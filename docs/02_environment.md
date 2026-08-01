# 02. 재현 스택 구축 및 실험 설계

## 목적

NKS의 블랙박스 구간을 관측 가능한 형태로 재현하고, 각 계층이 성능에 미치는 영향을 개별적으로 분리 측정할 수 있는 환경을 구성합니다.

---

## 1. 물리 인프라

### 호스트 사양

| 항목 | 사양 |
|------|------|
| CPU | Intel Core i5-12400 (6C/12T, Alder Lake) |
| Memory | 64GB DDR4 Dual Channel |
| Storage | WD_BLACK SN850X 1TB NVMe (PCIe 4.0) |
| Host OS | Xubuntu 22.04 LTS |
| Hypervisor | KVM / libvirt |

### 디스크 파티션 구성

```
nvme-WD_BLACK_SN850X_1000GB (931.5GB)
├─ part1    487MB    /boot/efi
├─ part2    488.3GB  /  (호스트 OS 루트)
├─ part3    100GB    → ubuntu01 Ceph OSD
├─ part4    100GB    → ubuntu02 Ceph OSD
├─ part5    100GB    → ubuntu03 Ceph OSD
└─ part6    49.4GB   → L0/L1 벤치마크 전용
```

**part6는 이 실험을 위해 별도로 생성한 파티션입니다.** L0에서는 호스트가 직접 사용하고, L1에서는 동일 파티션을 KVM 게스트에 패스스루하여 측정합니다. 같은 물리 영역을 사용하므로 L0↔L1 비교에서 변인이 KVM 가상화 하나로 통제됩니다.

> **주의:** 장치명 `nvme0n1`/`nvme1n1`은 재부팅 시 뒤바뀝니다. 실험 전 반드시 `/dev/disk/by-id/` 경로로 실제 디스크를 확인해야 합니다.
> → [07_troubleshooting.md #2](07_troubleshooting.md)

---

## 2. 가상 머신 구성

| VM | OpenStack 역할 | Ceph 역할 | vCPU | RAM | 측정 계층 |
|----|---------------|-----------|:----:|:---:|:---------:|
| controller | Controller | MON, MGR | 4 | 12GB | — |
| ubuntu01 | Compute | MON, OSD | 4 | 12GB | **L1, L2, L3** |
| ubuntu02 | Compute | MON, OSD | 4 | 12GB | — |
| ubuntu03 | Compute | MGR(standby), OSD | 4 | 12GB | — |

### MON 3개 구성 근거

Ceph MON은 Paxos 기반 합의를 수행하므로 **홀수 구성이 필요합니다.** 짝수일 경우 네트워크 분단 시 어느 쪽도 과반을 확보하지 못해 쿼럼이 붕괴될 수 있습니다. 초기 4-MON 구성에서 ubuntu03의 MON을 제거하여 3개로 조정했습니다.

### 측정 VM 스펙 통일

L1~L3는 ubuntu01에서, L4~L5는 OpenStack 인스턴스(k8s-worker1)에서 측정합니다. 두 환경의 vCPU/RAM이 다르면 계층 비교에 CPU 성능 차이가 섞이므로, k8s-worker1을 **4 vCPU / 12GB로 리사이즈**하여 ubuntu01과 동일하게 맞췄습니다.

---

## 3. 소프트웨어 스택

| 구성요소 | 버전 | 배포 도구 |
|----------|------|-----------|
| OpenStack | 2024.1 Caracal | Kolla-Ansible (컨테이너 기반) |
| Ceph | Quincy 17 | cephadm |
| Kubernetes | v1.31.14 | kubeadm |
| CSI Driver | cloud-provider-openstack | Helm |


### 스토리지 풀 구성

| Pool | Replica | PG | 용도 |
|------|:-------:|:--:|------|
| volumes | 3 | 32 | Cinder 블록 스토리지 (측정 대상) |
| vms | 3 | 32 | Nova 인스턴스 디스크 |
| images | 3 | 32 | Glance 이미지 |
| .mgr | 3 | 1 | cephadm 내부 관리 |

---

## 4. 계층 설계

### 설계 원칙

각 계층은 직전 계층 대비 **정확히 하나의 변인만** 추가되도록 구성했습니다.

| 계층 | 측정 위치 | 대상 경로 | 추가 변인 |
|:----:|-----------|-----------|-----------|
| L0 | 호스트 | `/dev/nvme*p6` (raw) | — |
| L1 | ubuntu01 | `/dev/vdd` (raw, 동일 파티션) | KVM 가상화 |
| L2 | ubuntu01 | `/dev/rbd0` (raw) | Ceph + Replica 3 |
| L3 | ubuntu01 | `/mnt/rbd-l3/testfile` | ext4 파일시스템 |
| L4 | k8s-worker1 | `/mnt/rbd-l4/testfile` | 중첩 가상화 |
| L5 | K8s Pod | `/data/testfile` | CSI + 컨테이너 |
| L6 | NKS Pod | `/data/testfile` | managed 플랫폼 |

### 변인 통제 방식

**L1 ↔ L2:** 동일한 VM(ubuntu01)에서 대상만 물리 파티션 → RBD로 변경. CPU, 커널, 메모리 조건이 완전히 동일하므로 차이는 Ceph 계층뿐입니다.

**L4 ↔ L5:** 동일한 VM(k8s-worker1)에서 측정. K8s Pod는 `nodeSelector`로 해당 노드에 고정 배치했습니다.

```yaml
spec:
  nodeSelector:
    kubernetes.io/hostname: k8s-worker1
```

**L2 ↔ L3:** raw 디스크와 파일시스템 비용을 정량화한 후 상위 계층으로 진행했습니다. NKS 측정이 파일시스템 기준이므로 어딘가에서 전환이 필요했으며, 이를 측정 대상으로 삼았습니다.

---

## 5. 측정 조건

### fio 프로파일

```
공통 옵션
  --direct=1              OS 페이지 캐시 우회
  --ioengine=libaio       비동기 I/O (iodepth 실효화)
  --size=2G               잡당 워킹셋
  --runtime=60            측정 시간
  --time_based            시간 기준 종료
  --group_reporting       잡 통합 리포트
  --offset_increment=2G   잡별 영역 분리 (4K 테스트만)
```

| 종목 | rw | bs | iodepth | numjobs | 총 Queue Depth | 측정 대상 |
|------|----|----|:-------:|:-------:|:--------------:|-----------|
| randwrite-4k | randwrite | 4k | 32 | 4 | 128 | 랜덤 쓰기 IOPS |
| randread-4k | randread | 4k | 32 | 4 | 128 | 랜덤 읽기 IOPS |
| seqwrite-1m | write | 1M | 16 | 1 | 16 | 순차 쓰기 대역폭 |
| seqread-1m | read | 1M | 16 | 1 | 16 | 순차 읽기 대역폭 |

### 4종을 측정한 이유

성능은 두 개의 축으로 결정되며, 4종은 그 조합입니다.

```
              작은 블록(4K)          큰 블록(1M)
              요청당 고정비용 지배    데이터 전송량 지배
쓰기 →    randwrite-4k          seqwrite-1m     (복제 있음)
읽기 →    randread-4k           seqread-1m      (복제 없음)
```
### 왜 4K와 1M인가

두 축의 양 극단을 잡기 위해서입니다.

|	|4K 랜덤|	1M 순차 |
|------|----|----|
| 초당 요청 수	| 수십만	| 수천 |
| 부담 요인	| 요청 처리 횟수	| 데이터 전송량 |
| 실제 사례	| DB 트랜잭션, 로그	| 백업, 파일 복사 |
| 측정 지표	| IOPS	| 대역폭(MB/s) |

단일 종목만 측정했다면 NKS의 **읽기·쓰기 동일 현상**을 발견할 수 없었고, 따라서 QoS 가설에 도달하지 못했을 것입니다.

---

## 6. 측정 신뢰성 확보

### 6-1. SSD 프리컨디셔닝

신규 생성 파티션은 모든 블록이 미기록(trimmed) 상태이므로, SSD 내부 가비지 컬렉션 부담 없이 일시적으로 높은 성능을 보입니다. 이를 정상 성능으로 오독하면 계층 비교가 무의미해집니다.

모든 raw 계층 측정 전 전체 영역을 순차 쓰기로 채워 steady state를 만들었습니다.

```bash
fio --name=precondition --filename=$DEV --direct=1 \
    --ioengine=libaio --rw=write --bs=1M --iodepth=32 --numjobs=1 \
    --loops=2 --group_reporting
```

**미실시 시 52% 편차가 발생했습니다.** → [07_troubleshooting.md #1](07_troubleshooting.md)

### 6-2. 워킹셋 분리

raw 디바이스에 `numjobs=4`를 지정하면 4개 잡이 모두 앞 2GB 영역을 경합합니다. 좁은 영역을 반복 덮어쓰면 GC 부담이 비정상적으로 증가합니다.

```
--offset_increment=2G
```

잡별로 2GB씩 다른 구간을 담당하도록 하여 총 8GB 워킹셋을 확보했습니다. 파일시스템 계층에서는 프리컨디셔닝으로 8GB 파일을 미리 생성해 조건을 맞췄습니다.


### 6-3. 계층별 동시 관측

fio 실행 중 하위 계층을 동시에 관측하여, 클라이언트가 보고한 수치와 실제 물리 I/O를 교차 검증했습니다.

| 관측 도구 | 위치 | 측정 대상 |
|-----------|------|-----------|
| `fio` | 각 계층 | Application 관점 IOPS/latency |
| `ceph osd perf` | controller | OSD별 apply/commit latency |
| `ceph -s` | controller | 클러스터 상태, client io |
| `iostat -x 1` | 호스트 | 물리 NVMe r/s, w/s, await, %util |

L0/L1에서 fio와 iostat이 1:1로 일치함을 확인한 것이 기준선이 되며, L2부터 이 비율이 깨지는 정도가 곧 Ceph의 쓰기 증폭입니다.

---

## 7. 측정 절차 요약

```
1. 대상 디바이스 확인 (/dev/disk/by-id 기준)
2. 프리컨디셔닝 (raw: 전체 영역 / fs: 8GB 파일 생성)
3. 관측 터미널 기동 (ceph osd perf, ceph -s, iostat)
4. fio 4종 순차 실행 (각 60초)
5. 결과 및 iostat 로그 수집
```

---

> 다음 문서: [03_layer_L0_L1.md](03_layer_L0_L1.md) — 물리 NVMe 기준선과 KVM 가상화 비용

← [README로 돌아가기](../README.md)
