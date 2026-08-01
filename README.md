# NKS Managed Kubernetes 스토리지 성능의 블랙박스 규명

> NKS의 I/O 성능 수치를 해석하기 위해 동등한 스택을 직접 구축하고,
> 계층별 성능 손실을 정량 분리한 프로젝트입니다.

---

## 한 줄 요약

NKS(NHN Kubernetes Service)에서 측정한 스토리지 성능이 **읽기와 쓰기 모두 동일한 2,000 IOPS**로 고정되는 현상을 발견했으나, managed 플랫폼 특성상 원인을 규명할 수 없었습니다. 동등한 스택(OpenStack + Ceph + K8s)을 직접 구축하여 물리 디스크부터 Pod까지 6개 계층의 성능을 분리 측정한 결과, **NKS의 수치는 하드웨어 성능이 아니라 QoS 정책의 결과**임을 Little's Law로 규명했습니다.

---

## 목차

1. [문제 인식 — NKS에서 관측된 이상 패턴](#1-문제-인식)
2. [접근 방법 — 계층 분리 실험 설계](#2-접근-방법)
3. [측정 환경](#3-측정-환경)
4. [계층별 측정 결과](#4-계층별-측정-결과)
5. [핵심 발견](#5-핵심-발견)
6. [트러블슈팅](#6-트러블슈팅)
7. [문서 목차](#7-문서-목차)

---

## 1. 문제 인식

NKS 클러스터에 PVC(10Gi, `cinder-default`)를 마운트하고 fio 4종을 측정한 결과입니다.

| 종목 | IOPS / Bandwidth | 평균 Latency |
|------|:----------------:|:------------:|
| randwrite 4K | 2,002 IOPS | 63.90 ms |
| randread 4K | 2,007 IOPS | 63.75 ms |
| seqwrite 1M | 53.2 MiB/s | 299.89 ms |
| seqread 1M | 64.1 MiB/s | 249.41 ms |

### 무엇이 이상한가

**읽기와 쓰기가 동일합니다.** 분산 스토리지에서 쓰기는 복제를 거치고 읽기는 거치지 않으므로, 통상 읽기가 2~3배 빠릅니다. 그런데 2,002와 2,007로 사실상 같습니다.

**변동이 없습니다.** 60초 측정 내내 순간 IOPS가 458~634 범위에 머물렀습니다. 실제 하드웨어라면 GC, 캐시 히트/미스에 따라 출렁입니다.

### 왜 원인을 알 수 없는가

managed Kubernetes는 CSI 아래 계층을 노출하지 않습니다.

| 관찰 대상 | NKS |
|-----------|:---:|
| Pod 내부 fio | 가능 |
| PVC ↔ 백엔드 볼륨 매핑 | 부분 |
| 스토리지 백엔드 종류·구성 | **불가** |
| OSD/디스크 지연 통계 | **불가** |
| 물리 디스크 I/O | **불가** |

63.9ms가 디스크 성능인지, 네트워크 지연인지, 정책적 제한인지 **측정만으로는 판단할 수 없습니다.**

> 상세 내용은 [docs/01_problem.md](docs/01_problem.md)를 참고해 주십시오.

---

## 2. 접근 방법

블랙박스 내부를 알 수 없다면, **동등한 스택을 직접 만들어 각 계층의 기여도를 분리 측정**하는 방식을 택했습니다.

### 계층 설계

각 단계가 직전 대비 **변인 하나만** 추가하도록 구성했습니다.

| 계층 | 측정 위치 | 대상 | 추가 변인 |
|:----:|-----------|------|-----------|
| **L0** | 호스트 (bare-metal) | NVMe 파티션 raw | — (기준선) |
| **L1** | KVM 게스트 | 동일 파티션 raw (패스스루) | KVM 가상화 |
| **L2** | KVM 게스트 | Ceph RBD raw | Ceph 분산 + Replica 3 |
| **L3** | KVM 게스트 | Ceph RBD + ext4 | 파일시스템 |
| **L4** | OpenStack 인스턴스 | Cinder 볼륨 + ext4 | 중첩 가상화 |
| **L5** | K8s Pod | PVC + ext4 | CSI + 컨테이너 |
| **L6** | NKS Pod | PVC + ext4 | **managed 플랫폼** |

L1과 L2는 **동일한 VM**에서 측정하여 Ceph 비용만 분리했습니다.
L4와 L5는 **동일한 VM**에서 측정하여 K8s 계층 비용만 분리했습니다.

### 측정 조건 통일

```
공통: direct=1, ioengine=libaio, size=2G, runtime=60s, time_based
      offset_increment=2G (잡별 영역 분리)

randwrite-4k : rw=randwrite, bs=4k,  iodepth=32, numjobs=4
randread-4k  : rw=randread,  bs=4k,  iodepth=32, numjobs=4
seqwrite-1m  : rw=write,     bs=1M,  iodepth=16, numjobs=1
seqread-1m   : rw=read,      bs=1M,  iodepth=16, numjobs=1
```

**4종을 측정한 이유는** 블록 크기(요청당 고정비용 vs 데이터 전송량)와 방향(복제 유무)이라는 두 축의 조합을 확보하기 위함입니다. 단일 종목만 측정했다면 NKS의 읽기·쓰기 동일 현상을 발견할 수 없었습니다.

### 측정 신뢰성 확보

| 항목 | 조치 |
|------|------|
| SSD 프리컨디셔닝 | 모든 raw 측정 전 전체 영역 순차 쓰기 (미실시 시 52% 오차 발생) |
| 워킹셋 분리 | `offset_increment=2G`로 잡별 영역 분리 |
| 측정 도구 통일 | 전 계층 `ljishen/fio` 컨테이너 사용, native fio와 0.5% 이내 일치 검증 |
| 계층별 동시 관측 | fio + `ceph osd perf` + `iostat -x` 동시 수집 |

> 상세 내용은 [docs/02_environment.md](docs/02_environment.md)를 참고해 주십시오.

---

## 3. 측정 환경

### 물리 호스트

| 항목 | 사양 |
|------|------|
| CPU | Intel Core i5-12400 (6C/12T) |
| Memory | 64GB DDR4 |
| Storage | WD_BLACK SN850X 1TB NVMe |
| OS | Xubuntu 22.04 LTS |

### 스택

```
Xubuntu 22.04 (bare-metal)
 └ KVM/libvirt
    ├ controller  : OpenStack Controller, Ceph MON/MGR
    ├ ubuntu01    : Compute, Ceph OSD  ← L1/L2/L3 측정 지점
    ├ ubuntu02    : Compute, Ceph OSD
    └ ubuntu03    : Compute, Ceph OSD
         └ OpenStack 인스턴스 (k8s-worker1)  ← L4/L5 측정 지점
              └ Kubernetes Pod
```

| 구성요소 | 버전 / 도구 |
|----------|------------|
| OpenStack | 2024.1 Caracal (Kolla-Ansible) |
| Ceph | Quincy (cephadm), OSD 3개, Replica 3 |
| Kubernetes | v1.31 (kubeadm) |
| CSI | cloud-provider-openstack (Cinder CSI) |

**측정 VM 스펙을 통일했습니다.** ubuntu01(4 vCPU / 12GB)과 k8s-worker1(4 vCPU / 12GB)을 동일 스펙으로 맞춰 계층 간 비교의 변인을 통제했습니다.

---

## 4. 계층별 측정 결과

### 전체 수치

| 종목 | L0 물리 | L1 KVM | L2 Ceph | L3 ext4 | L4 OpenStack | L5 K8s | L6 NKS |
|------|:-------:|:------:|:-------:|:-------:|:-----------:|:------:|:------:|
| randwrite 4K (IOPS) | 550,000 | 255,000 | 8,974 | 8,913 | 8,548 | 8,447 | **2,002** |
| randread 4K (IOPS) | 559,000 | 310,000 | 24,300 | 26,000 | 20,700 | 21,300 | **2,007** |
| seqwrite 1M (MiB/s) | 4,967 | 5,214 | 349 | 407 | 404 | 400 | **53.2** |
| seqread 1M (MiB/s) | 6,609 | 6,592 | 2,172 | 2,094 | 1,378 | 1,391 | **64.1** |

### 계층별 손실률 (randwrite 4K 기준)

| 구간 | 추가 계층 | 손실률 |
|------|-----------|:------:|
| L0 → L1 | KVM 가상화 | 53.6% |
| L1 → L2 | **Ceph 분산 + Replica 3** | **96.5%** |
| L2 → L3 | ext4 파일시스템 | 0.7% |
| L3 → L4 | OpenStack 중첩 가상화 | 4.1% |
| L4 → L5 | K8s CSI + 컨테이너 | 1.2% |
| L5 → L6 | **managed 플랫폼 QoS** | **76.3%** |

### 쓰기 증폭 (fio ↔ iostat 교차 측정)

| 계층 | randwrite 증폭률 | seqwrite 증폭률 |
|------|:---------------:|:--------------:|
| L0 / L1 | 1.0x | 1.0x |
| L2 이상 | **7.9x** | **2.7x** |

L0/L1에서 fio 보고량과 물리 디스크 기록량이 1:1로 일치했으나, Ceph 계층부터 어긋납니다. 읽기는 전 계층에서 1:1을 유지하는데, 복제가 쓰기 경로에만 존재하기 때문입니다.

---

## 5. 핵심 발견

### 발견 1 — 병목은 Ceph 단일 계층에 집중됩니다

물리 성능의 **98.4%가 L1→L2 구간에서 소실**됩니다. ext4, OpenStack 중첩 가상화, K8s CSI를 모두 합쳐도 6% 미만입니다.

> "K8s나 컨테이너 때문에 스토리지가 느리다"는 통념은 데이터로 반박됩니다. K8s CSI와 컨테이너 계층의 오버헤드는 1% 이내, 측정 오차 수준이었습니다.

### 발견 2 — 가상화 오버헤드는 하위 계층 성능에 따라 결정됩니다

동일한 virtio 계층이 L0→L1에서는 53.6% 손실을 유발했으나, L3→L4에서는 4.1%에 그쳤습니다.

```
L0 요청당 처리시간   1.82 µs  →  virtio 약 2 µs 추가  →  두 배 증가
L3 요청당 처리시간 112.2 µs  →  virtio 약 5 µs 추가  →  4% 증가
```

virtio가 붙이는 절대 비용은 유사하나, 기준값이 60배 차이나므로 상대적 영향이 달라집니다. **병목이 이미 다른 계층에 있으면 추가 계층의 비용은 흡수됩니다.**

### 발견 3 — 워크로드 유형에 따라 병목 계층이 다릅니다

| 종목 | 지배적 병목 | 근거 |
|------|-------------|------|
| randwrite 4K | Ceph 복제 + BlueStore | 증폭률 7.9x, 중첩 가상화 영향 4% |
| randread 4K | 요청당 고정비용 | 복제 없음, KVM에서 45% 손실 |
| seqwrite 1M | Ceph 복제 | 증폭률 2.7x |
| seqread 1M | **데이터 전송량** | 중첩 가상화에서 34% 손실 |

seqread만 중첩 가상화에서 큰 손실이 발생했습니다. 복제가 없어 Ceph 부담이 작은 대신, 초당 2GB의 메모리 복사가 추가 가상화 계층에서 그대로 비용이 되기 때문입니다.

### 발견 4 — NKS의 수치는 성능이 아니라 정책입니다

동등한 스택(L5)이 8,447 IOPS를 내는 반면 NKS는 2,002 IOPS에 고정됩니다. 세 가지 근거로 QoS 제한임을 규명했습니다.

**근거 1 — 읽기·쓰기 동일**
L0~L5 전 계층에서 읽기가 쓰기보다 2~3배 빨랐습니다. NKS만 2,002 vs 2,007로 동일합니다.

**근거 2 — 변동성 부재**
```
L5 randwrite : 순간 IOPS 36 ~ 4,517  (표준편차 52,126 µs)
L6 randwrite : 순간 IOPS 458 ~ 634   (표준편차  6,315 µs)
```

**근거 3 — Little's Law 검증**
```
Latency = Queue Depth / Throughput
        = 128 / 2,002 = 63.9 ms      (실측 63.90 ms)
```
63.9ms는 디스크 처리 시간이 아니라, 초당 2,000개를 통과시키는 게이트 뒤에서 128개 요청이 대기한 시간입니다.

### 발견 5 — Little's Law는 전 계층에서 성립합니다

| 계층 | 예측값 | 실측값 | 오차 |
|------|:------:|:------:|:----:|
| L0 randwrite | 232.7 µs | 232.47 µs | 0.1% |
| L1 randwrite | 502 µs | 501.17 µs | 0.2% |
| L2 randwrite | 14.26 ms | 14.259 ms | 0.01% |
| L4 randwrite | 14.97 ms | 14.968 ms | 0.01% |
| L6 randwrite | 63.9 ms | 63.90 ms | 0.0% |

전 계층·전 종목에서 오차 0.2% 이내로 일치합니다. 측정 환경에 외부 노이즈가 없었음을 검증하는 동시에, **동일한 분석 프레임을 self-managed와 managed 양쪽에 적용할 수 있음**을 보여줍니다.

---

## 6. 트러블슈팅

측정 과정에서 발견하고 해결한 방법론적 오류들입니다.

| # | 문제 | 영향 | 해결 |
|---|------|------|------|
| 1 | SSD 프리컨디셔닝 누락 | 동일 조건 측정에서 **52% 편차** | 전체 영역 순차 쓰기 후 재측정 → 0.5%로 수렴 |
| 2 | NVMe 장치명 비영속성 | iostat이 무관한 디스크 관측 | `/dev/disk/by-id` 영속 경로 사용 |
| 3 | cephadm 자동 디바이스 흡수 | 벤치마크 대상이 OSD로 편입 | `--unmanaged=true` 설정, OSD 제거 후 재측정 |
| 4 | shelve로 인한 스토리지 고갈 | 클러스터 full, 쓰기 차단 | 인스턴스 스냅샷 삭제, full-ratio 임시 조정 |
| 5 | 워킹셋 중첩 | 4개 잡이 동일 2GB 영역 경합 | `offset_increment=2G` 적용 |

> 상세 내용은 [docs/07_troubleshooting.md](docs/07_troubleshooting.md)를 참고해 주십시오.

---

## 7. 문서 목차

| 문서 | 내용 |
|------|------|
| [01_problem.md](docs/01_problem.md) | NKS 측정 결과와 관찰 가능성의 한계 |
| [02_environment.md](docs/02_environment.md) | 재현 스택 구축 및 실험 설계 |
| [03_layer_L0_L1.md](docs/03_layer_L0_L1.md) | 물리 NVMe 기준선과 KVM 가상화 비용 |
| [04_layer_L2_L3.md](docs/04_layer_L2_L3.md) | Ceph 분산 스토리지와 파일시스템 계층 |
| [05_layer_L4_L5.md](docs/05_layer_L4_L5.md) | 중첩 가상화와 K8s CSI 계층 |
| [06_comparison.md](docs/06_comparison.md) | NKS 대조 분석 및 QoS 규명 |
| [07_troubleshooting.md](docs/07_troubleshooting.md) | 측정 방법론 오류와 해결 과정 |

### 실험 재현

```bash
scripts/
├── validate_tools.sh      # fio 도구 검증 (native vs container)
├── precondition.sh        # SSD 프리컨디셔닝
├── bench_layer.sh         # 계층별 4종 측정
└── observe.sh             # 계층별 동시 관측
```

---

## References

- [Ceph BlueStore Configuration](https://docs.ceph.com/en/latest/rados/configuration/bluestore-config-ref/)
- [OpenStack Cinder RBD Driver](https://docs.openstack.org/cinder/latest/configuration/block-storage/drivers/ceph-rbd-volume-driver.html)
- [cloud-provider-openstack (Cinder CSI)](https://github.com/kubernetes/cloud-provider-openstack)
- [fio Documentation](https://fio.readthedocs.io/en/latest/)
