# NKS 스토리지 성능 계층 분석

> NKS의 I/O 성능 수치를 해석하기 위해 동등한 형태의 스택을 직접 구축하고,
> 계층별 성능 손실을 정량적으로 분리해 본 프로젝트입니다.

---

## 한 줄 요약

NKS(NHN Kubernetes Service)에서 측정한 스토리지 성능이 **읽기와 쓰기 모두 약 2,000 IOPS**로 수렴하는 현상을 발견했으나, managed 플랫폼 특성상 원인을 확인할 수 없었습니다. 
동등한 형태의 스택(OpenStack + Ceph + K8s)을 직접 구축하여 물리 디스크부터 Pod까지 6개 계층의 성능을 분리 측정한 결과, **NKS의 수치는 백엔드 하드웨어의 성능 한계라기보다 QoS 정책의 결과일 가능성이 높다**고 판단했습니다.

<img width="1420" height="559" alt="image" src="https://github.com/user-attachments/assets/da544028-71a1-4bf4-90cc-f55dfde5f044" />

> 본 프로젝트의 결론은 외부에서 관측 가능한 지표만을 근거로 한 추론입니다. NKS의 내부 구성이나 정책 설정값을 직접 확인한 것은 아닙니다.

---

## 목차

1. [문제 인식 — NKS에서 관측된 패턴](#1-문제-인식)
2. [접근 방법 — 계층 분리 실험 설계](#2-접근-방법)
3. [측정 환경](#3-측정-환경)
4. [계층별 측정 결과](#4-계층별-측정-결과)
5. [주요 발견](#5-주요-발견)
6. [트러블슈팅](#6-트러블슈팅)
7. [한계](#7-한계)
8. [문서 목차](#8-문서-목차)

---

## 1. 문제 인식

NKS 클러스터에 PVC(10Gi, `cinder-default`)를 마운트하고 fio 4종을 측정한 결과입니다.

| 종목 | IOPS / Bandwidth | 평균 Latency |
|------|:----------------:|:------------:|
| randwrite 4K | 2,002 IOPS | 63.90 ms |
| randread 4K | 2,007 IOPS | 63.75 ms |
| seqwrite 1M | 53.2 MiB/s | 299.89 ms |
| seqread 1M | 64.1 MiB/s | 249.41 ms |

### 무엇이 특이한가

**읽기와 쓰기가 거의 동일합니다.** 분산 스토리지에서 쓰기는 복제를 거치고 읽기는 거치지 않으므로, 통상 읽기가 더 빠를 것으로 예상됩니다. 그런데 2,002와 2,007로 차이가 0.25%에 불과합니다.

**변동성이 매우 낮습니다.** 60초 측정 내내 순간 IOPS가 458~634 범위에 머물렀습니다. 실제 하드웨어라면 GC, 캐시 히트율 변화 등에 따라 더 큰 변동이 나타날 것으로 예상됩니다.

### 왜 원인을 확인할 수 없는가

managed Kubernetes는 CSI 아래 계층을 노출하지 않습니다.

| 관찰 대상 | Self-managed | NKS |
|-----------|:------------:|:---:|
| Pod 내부 fio | 가능 | 가능 |
| PVC ↔ 백엔드 볼륨 매핑 | 가능 | 부분 |
| 스토리지 백엔드 종류·구성 | 가능 | **불가** |
| OSD/디스크 지연 통계 | 가능 | **불가** |
| 물리 디스크 I/O | 가능 | **불가** |

63.9ms가 디스크 처리 시간인지, 네트워크 지연인지, 정책적 제한인지 **측정값만으로는 판단하기 어렵습니다.**

> 상세 내용은 [docs/01_problem.md](docs/01_problem.md)를 참고해 주십시오.

---

## 2. 접근 방법

블랙박스 내부를 직접 볼 수 없다면, **동등한 형태의 스택을 만들어 각 계층의 기여도를 분리 측정**하는 방식을 택했습니다.

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

**4종을 측정한 이유는** 블록 크기(요청당 고정비용 vs 데이터 전송량)와 방향(복제 유무)이라는 두 축의 조합을 확보하기 위함입니다. 단일 종목만 측정했다면 NKS의 읽기·쓰기 동일 현상을 발견하기 어려웠을 것으로 생각됩니다.

### 측정 신뢰성 확보

| 항목 | 조치 |
|------|------|
| SSD 프리컨디셔닝 | 모든 raw 측정 전 전체 영역 순차 쓰기 (미실시 시 52% 편차 발생) |
| 워킹셋 분리 | `offset_increment=2G`로 잡별 영역 분리 |
| 측정 도구 통일 | 전 계층 `ljishen/fio` 컨테이너 사용, native fio와 0.5% 이내 일치 확인 |
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
| L5 → L6 | **managed 플랫폼** | **76.3%** |

### 쓰기 증폭 (fio ↔ iostat 교차 측정)

| 계층 | randwrite 증폭률 | seqwrite 증폭률 |
|------|:---------------:|:--------------:|
| L0 / L1 | 1.0x | 1.0x |
| L2 이상 | **7.9x** | **2.7x** |

L0/L1에서 fio 보고량과 물리 디스크 기록량이 1:1로 일치했으나, Ceph 계층부터 어긋납니다. 읽기는 전 계층에서 1:1을 유지하는데, 복제가 쓰기 경로에만 존재하기 때문으로 보입니다.

---

## 5. 주요 발견

### 발견 1 — 병목이 Ceph 계층에 집중되어 있습니다

물리 성능의 **98.4%가 L1→L2 구간에서 소실**되었습니다. ext4, OpenStack 중첩 가상화, K8s CSI를 모두 합쳐도 6% 미만입니다.

> 본 환경에서 K8s CSI와 컨테이너 계층의 오버헤드는 1% 이내로, 측정 오차와 구분되지 않았습니다. "K8s나 컨테이너 때문에 스토리지가 느리다"는 통념은 이번 측정 결과와 부합하지 않는 것으로 보입니다.

### 발견 2 — 가상화 오버헤드는 하위 계층 성능에 따라 다르게 나타납니다

동일한 virtio 계층이 L0→L1에서는 53.6% 손실을 유발했으나, L3→L4에서는 4.1%에 그쳤습니다.

```
L0 요청당 처리시간   1.82 µs  →  약 2 µs 추가  →  두 배 증가
L3 요청당 처리시간 112.2 µs  →  약 5 µs 추가  →  4% 증가
```

추가되는 시간의 절대값은 유사하나 기준값이 약 60배 차이나므로, 상대적 영향이 달라지는 것으로 판단됩니다. **병목이 이미 다른 계층에 있으면 추가 계층의 비용은 상대적으로 흡수되는 것으로 보입니다.**

### 발견 3 — 워크로드 유형에 따라 지배적 병목이 다릅니다

| 종목 | 추정 지배 요인 | 근거 |
|------|---------------|------|
| randwrite 4K | Ceph 복제 + BlueStore | 증폭률 7.9x, 중첩 가상화 영향 4% |
| randread 4K | 요청당 고정비용 | 복제 없음, KVM에서 45% 손실 |
| seqwrite 1M | Ceph 복제 | 증폭률 2.7x |
| seqread 1M | **데이터 전송량** | 중첩 가상화에서 34% 손실 |

seqread만 중첩 가상화에서 큰 손실이 관측되었습니다. 복제가 없어 Ceph 부담이 작은 대신, 초당 2GB의 메모리 복사가 추가 가상화 계층에서 비용으로 나타난 것으로 보입니다.

### 발견 4 — NKS의 수치는 QoS 정책의 결과일 가능성이 높습니다

동등한 형태의 스택(L5)이 8,447 IOPS를 보이는 반면 NKS는 2,002 IOPS에 수렴합니다. 세 가지 관측이 QoS 제한을 시사합니다.

**관측 1 — 읽기·쓰기 동일**
L2~L5 전 계층에서 읽기가 쓰기보다 2.4~2.9배 빨랐습니다. NKS만 2,002 vs 2,007로 동일합니다.

**관측 2 — 변동성 부재**
```
L5 randwrite : 순간 IOPS 36 ~ 4,517  (clat 표준편차 52,126 µs)
L6 randwrite : 순간 IOPS 458 ~ 634   (clat 표준편차  6,315 µs)
```

**관측 3 — Little's Law 부합**
```
Latency = Queue Depth / Throughput
        = 128 / 2,002 = 63.93 ms      (실측 63.90 ms, 오차 0.05%)
```
계산값과 실측값이 거의 일치한다는 것은, 63.9ms의 대부분이 디스크 처리 시간이 아니라 큐 대기 시간일 가능성을 시사합니다.

> 다만 이는 외부 관측 지표에서 도출한 추론이며, 실제 정책 설정값을 확인한 것은 아닙니다.

### 발견 5 — Little's Law가 전 계층에서 성립합니다

| 계층 | 예측값 | 실측값 | 오차 |
|------|:------:|:------:|:----:|
| L0 randwrite | 232.7 µs | 232.47 µs | 0.1% |
| L1 randwrite | 502 µs | 501.17 µs | 0.2% |
| L2 randwrite | 14.26 ms | 14.259 ms | 0.01% |
| L4 randwrite | 14.97 ms | 14.968 ms | 0.01% |
| L6 randwrite | 63.93 ms | 63.90 ms | 0.05% |

전 계층·전 종목에서 오차 0.3% 이내로 일치했습니다. 측정 구간에 외부 노이즈가 적었음을 시사하는 동시에, **동일한 분석 프레임을 self-managed와 managed 양쪽에 적용할 수 있음**을 보여줍니다.

---

## 6. 트러블슈팅

측정 과정에서 발견하고 해결한 방법론적 오류들입니다.

| # | 문제 | 영향 | 해결 |
|---|------|------|------|
| 1 | SSD 프리컨디셔닝 누락 | 동일 조건 측정에서 **52% 편차** | 전체 영역 순차 쓰기 후 재측정 → 0.5%로 수렴 |
| 2 | NVMe 장치명 비영속성 | iostat이 무관한 디스크 관측 | `/dev/disk/by-id` 영속 경로 사용 |
| 3 | cephadm 자동 디바이스 흡수 | 벤치마크 대상이 OSD로 편입 | `--unmanaged=true` 설정, OSD 제거 후 재측정 |
| 4 | OSD 제거 순서 오류 | 클러스터 full, 쓰기 차단 | 임계값 임시 조정 후 데이터 정리 |
| 5 | shelve로 인한 스토리지 고갈 | 288GB 소모, 클러스터 full | 인스턴스 스냅샷 삭제 |
| 6 | 워킹셋 중첩 | 4개 잡이 동일 2GB 영역 경합 | `offset_increment=2G` 적용 |

특히 1, 2, 6번은 **오류 메시지 없이 잘못된 데이터를 산출**하는 유형이었습니다. 측정값이 예상과 다를 때 도구나 환경 차이로 결론짓기 전에 측정 조건 자체를 의심하는 절차가 필요하다고 판단했습니다.

> 상세 내용은 [docs/07_troubleshooting.md](docs/07_troubleshooting.md)를 참고해 주십시오.

---

## 7. 한계

본 프로젝트의 결론은 다음 제약 하에서 도출되었습니다.

| 항목 | 내용 |
|------|------|
| 백엔드 미확인 | NKS의 스토리지 백엔드가 무엇인지 확인할 수 없음. StorageClass명으로 Cinder 사용만 추정 |
| QoS 수치 미검증 | 볼륨 크기·타입에 따라 상한이 달라지는지 미확인 |
| 단일 스핀들 제약 | OSD 3개가 동일 물리 디스크의 파티션을 사용. 실제 분산 환경의 네트워크 특성 미반영 |
| 덮어쓰기 워크로드 한정 | ext4 저널링 비용은 파일 생성·확장 워크로드에서 재측정 필요 |
| 단일 측정 | 각 계층·종목당 1회 측정. 통계적 신뢰구간 미산출 |

**NKS 백엔드를 확인할 수 없다는 점**은 이 프로젝트의 근본적 제약이자, 동시에 프로젝트를 시작하게 된 이유이기도 합니다. 관측이 제한된 환경에서 어디까지 추론할 수 있는지를 탐색한 시도로 봐주시면 좋겠습니다.

---

## 8. 문서 목차

| 문서 | 내용 |
|------|------|
| [01_problem.md](docs/01_problem.md) | NKS 측정 결과와 관찰 가능성의 한계 |
| [02_environment.md](docs/02_environment.md) | 재현 스택 구축 및 실험 설계 |
| [03_layer_L0_L1.md](docs/03_layer_L0_L1.md) | 물리 NVMe 기준선과 KVM 가상화 비용 |
| [04_layer_L2_L3.md](docs/04_layer_L2_L3.md) | Ceph 분산 스토리지와 파일시스템 계층 |
| [05_layer_L4_L5.md](docs/05_layer_L4_L5.md) | 중첩 가상화와 K8s CSI 계층 |
| [06_comparison.md](docs/06_comparison.md) | NKS 대조 분석 및 QoS 검토 |
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
