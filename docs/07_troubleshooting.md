# 07. 측정 방법론 오류와 해결 과정

측정 과정에서 발견한 오류와 해결 절차를 기록합니다. 단순 장애 대응이 아니라, **잘못된 측정이 잘못된 결론으로 이어질 뻔한 지점**을 중심으로 정리했습니다.

---

## 목차

1. [SSD 프리컨디셔닝 누락으로 인한 52% 측정 편차](#1-ssd-프리컨디셔닝-누락으로-인한-52-측정-편차)
2. [NVMe 장치명 비영속성으로 인한 관측 대상 오지정](#2-nvme-장치명-비영속성으로-인한-관측-대상-오지정)
3. [cephadm 자동 디바이스 흡수로 인한 벤치마크 대상 오염](#3-cephadm-자동-디바이스-흡수로-인한-벤치마크-대상-오염)
4. [OSD 제거 순서 오류와 클러스터 용량 고갈](#4-osd-제거-순서-오류와-클러스터-용량-고갈)
5. [shelve로 인한 스토리지 고갈](#5-shelve로-인한-스토리지-고갈)
6. [측정 도구 검증 및 워킹셋 중첩](#6-측정-도구-검증-및-워킹셋-중첩)

---

## 1. SSD 프리컨디셔닝 누락으로 인한 52% 측정 편차

### 증상

측정 도구(native fio 3.28 vs 컨테이너 fio 3.6)의 결과 일치 여부를 검증하기 위해 동일 조건으로 측정했으나, 52%의 차이가 발생했습니다.

| 도구 | 측정 시각 | IOPS |
|------|-----------|:----:|
| 컨테이너 fio 3.6 | 07-30 20:24 | 578,000 |
| native fio 3.28 | 07-31 05:25 | 380,000 |

### 원인 분석

처음에는 fio 버전 차이를 의심했으나, 측정 시각을 확인하면서 다른 설명이 가능하다고 판단했습니다.

컨테이너 측정은 **파티션 생성 직후** 수행되었습니다. 이 시점의 SSD는 대부분의 블록이 미기록(trimmed) 상태이므로, 가비지 컬렉션 부담 없이 빈 블록에 기록할 수 있어 일시적으로 높은 성능을 보이는 것으로 알려져 있습니다. SNIA에서는 이를 FOB(Fresh Out of Box) 상태로 정의하고 있습니다.

컨테이너 측정 과정에서 49.4GB 파티션에 132GB를 기록했습니다. 동일 영역을 약 2.7회 덮어쓴 결과 SSD 내부가 steady state에 가까워졌고, 그 상태에서 native 측정이 수행된 것으로 보입니다.

대역폭 변동폭이 이러한 해석을 뒷받침합니다.

```
컨테이너 : min   56 MB/s ~ max 1,172 MB/s   (표준편차 81,405 IOPS)
native   : min 1,039 MB/s ~ max 3,856 MB/s   (표준편차 34,610 IOPS)
```

컨테이너 측정은 초반에 매우 빠르다가 GC가 개입하며 급락하는 패턴을 보였고, 이로 인해 평균이 높게 산출된 것으로 판단됩니다.

부수적으로, raw 디바이스에 `numjobs=4`를 지정하면 **4개 잡이 모두 앞 2GB 영역을 경합**하는 문제도 확인되었습니다. 좁은 영역의 반복 덮어쓰기는 GC 부담을 증가시키는 요인으로 작용했을 것으로 보입니다.

### 해결

두 가지 조치를 적용했습니다.

```bash
# 1. 프리컨디셔닝 — 전체 영역을 순차 쓰기로 2회 채움
fio --name=precondition --filename=$DEV --direct=1 \
    --ioengine=libaio --rw=write --bs=1M --iodepth=32 --numjobs=1 \
    --loops=2 --group_reporting

# 2. 잡별 워킹셋 분리
--offset_increment=2G
```

### 결과

| 도구 | IOPS | Bandwidth | 평균 latency |
|------|:----:|:---------:|:------------:|
| native (3.28) | 426,000 | 1,664 MiB/s | 299.99 µs |
| 컨테이너 (3.6) | 428,000 | 1,673 MiB/s | 298.35 µs |
| **차이** | **+0.5%** | +0.5% | -0.5% |

52% 편차가 0.5%로 수렴했습니다. **원인이 fio 버전이 아니라 측정 조건이었을 가능성이 높다고 판단됩니다.**

### 교훈

플래시 스토리지 벤치마크에서 프리컨디셔닝은 표준 절차로 권고되고 있습니다. 이를 누락하면 도구나 환경 차이로 오인할 수 있는 규모의 편차가 발생할 수 있음을 확인했습니다. 이 검증을 통해 이후 전 계층에서 동일 컨테이너 이미지를 사용하되, 필요 시 native fio로 대체해도 무방하다는 근거를 확보했습니다.

---

## 2. NVMe 장치명 비영속성으로 인한 관측 대상 오지정

### 증상

L4 측정을 위해 `iostat -x 1 nvme0n1p3 nvme0n1p4 nvme0n1p5`를 실행했으나, 파티션 하나만 출력되고 수치 변화가 관측되지 않았습니다.

### 원인 분석

`parted`로 확인한 결과 `nvme0n1`이 512GB NTFS 파티션을 가진 Windows 디스크였습니다. OSD가 위치한 WD_BLACK 1TB 디스크는 `nvme1n1`으로 인식되어 있었습니다.

리눅스에서 NVMe 컨트롤러 초기화 순서는 보장되지 않으므로, **재부팅 시 `nvme0n1`과 `nvme1n1`이 교체될 수 있습니다.**

실제로 프로젝트 진행 중 최소 두 차례 이름이 뒤바뀐 것으로 확인됩니다.

```
초기 측정 시     : WD 1TB = nvme1n1
중간 측정 시     : WD 1TB = nvme0n1
L4 측정 시       : WD 1TB = nvme1n1
```

### 위험성

잘못된 디스크를 관측하면 **"부하를 가했는데 반응이 없다"는 결과**가 나오며, 이를 실제 성능 특성으로 오독할 위험이 있습니다. 예를 들어 "캐시가 모두 흡수했다"거나 "물리 디스크가 포화되지 않았다"는 잘못된 결론에 도달할 수 있습니다.

### 해결

측정 전 영속 경로로 실제 장치명을 확인하는 절차를 추가했습니다.

```bash
# 영속 경로 확인
ls -l /dev/disk/by-id/ | grep -i "wd_black"

# 현재 장치명 자동 추출
basename $(readlink -f /dev/disk/by-id/nvme-WD_BLACK*part3)
```

libvirt 도메인 정의에서도 영속 경로를 사용하도록 확인했습니다.

```
vdb → /dev/disk/by-id/nvme-WD_BLACK_SN850X_1000GB_25494J805585-part3
```

### 교훈

`/dev/sdX`, `/dev/nvmeXnY` 형태의 커널 할당 이름은 영속적이지 않습니다. 스크립트, 설정 파일, 모니터링 대상 지정에는 `/dev/disk/by-id` 또는 UUID를 사용하는 것이 안전합니다. 특히 관측 도구가 잘못된 대상을 가리키는 경우 오류가 아니라 **정상적으로 보이는 잘못된 데이터**가 산출되므로 발견이 어렵습니다.

---

## 3. cephadm 자동 디바이스 흡수로 인한 벤치마크 대상 오염

### 증상

L1 측정을 위해 물리 파티션을 KVM 게스트에 연결한 후, 클러스터 상태를 확인하니 OSD가 3개에서 4개로 증가해 있었습니다.

```
osd: 4 osds: 4 up (since 10h), 4 in
usage: 197 GiB used, 152 GiB / 349 GiB avail   ← 300GB → 349GB
health: HEALTH_WARN
        Failed to apply 1 service(s): osd.all-available-devices
        Too many repaired reads on 1 OSDs
```

### 원인 분석

cephadm의 `all-available-devices` 서비스 정책이 활성화되어 있으면, **비어 있는 블록 디바이스를 자동으로 OSD로 편입**하는 동작을 합니다.

관측된 정황을 종합하면 다음 순서로 진행된 것으로 보입니다.

```
1. virsh attach-disk로 벤치마크용 파티션을 ubuntu01에 연결
2. cephadm이 이를 "사용 가능한 빈 디바이스"로 인식
3. 자동으로 LVM 생성 후 osd.0으로 편입
4. L1 fio가 해당 디바이스를 raw로 덮어씀 → OSD 데이터 손상
5. "Too many repaired reads" 경고 발생
```

벤치마크 대상 디바이스가 스토리지 풀의 일부가 된 상태에서 raw 쓰기가 수행된 것으로 판단됩니다.

### 해결

```bash
# 1. 자동 흡수 중단 (최우선)
ceph orch apply osd --all-available-devices --unmanaged=true

# 2. 해당 OSD 제거
ceph osd out 0
ceph osd safe-to-destroy 0
ceph orch daemon rm osd.0 --force
ceph osd purge 0 --yes-i-really-mean-it

# 3. 디바이스 LVM 흔적 제거 후 재연결
virsh detach-disk vm-2 vdc --live
virsh attach-disk vm-2 /dev/nvme0n1p6 vdc \
  --driver qemu --subdriver raw --cache none --targetbus virtio --live
```

측정 신뢰성 확보를 위해 **L1 전체를 재측정**했습니다. 재측정 결과 기존 값과 3~8% 차이가 있었으며, 깨끗한 상태에서 얻은 데이터를 최종 채택했습니다.

### 교훈

자동화 도구는 명시적 지시 없이도 리소스를 편입할 수 있습니다. 벤치마크나 테스트 목적으로 디바이스를 추가할 때는 **자동 흡수 정책을 먼저 비활성화**하는 것이 안전합니다. 이 사례는 운영 환경에서 디스크 교체·증설 작업 시에도 유사하게 발생할 수 있을 것으로 생각됩니다.

---

## 4. OSD 제거 순서 오류와 클러스터 용량 고갈

### 증상

osd.0 제거 후 클러스터가 HEALTH_ERR 상태에 진입했습니다.

```
health: HEALTH_ERR
        2 full osd(s)
        Degraded data redundancy: 3009/46371 objects degraded (6.489%)
        Full OSDs blocking recovery: 10 pgs recovery_toofull
        4 pool(s) full

usage: 264 GiB used, 36 GiB / 300 GiB avail   (88%)
```

### 원인 분석

OSD 제거 시 리밸런싱 완료를 기다리지 않고 진행한 것이 주된 원인으로 보입니다.

```
제거 전 : 349GB 중 197GB 사용 (56%)
제거 후 : 300GB 중 264GB 사용 (88%)
```

Replica 3 환경에서 OSD 하나를 제거하면 해당 OSD가 보유하던 데이터를 나머지 OSD가 나눠 가져야 합니다. 총 용량은 감소하는데 저장할 데이터 양은 동일하므로 사용률이 급등합니다.

full 임계값(기본 95%)에 도달한 OSD가 발생하면 **쓰기가 차단되며, 복구 작업 자체도 쓰기를 필요로 하므로 교착 상태**에 빠지는 것으로 보입니다.

### 해결

```bash
# 1. 임계값 임시 상향으로 작업 공간 확보
ceph osd set-full-ratio 0.98
ceph osd set-nearfull-ratio 0.95
ceph osd set-backfillfull-ratio 0.96

# 2. 불필요 데이터 삭제 (5절 참조)

# 3. 임계값 원복
ceph osd set-full-ratio 0.95
ceph osd set-nearfull-ratio 0.85
ceph osd set-backfillfull-ratio 0.90
```

### 교훈

OSD 제거는 `ceph osd out` 이후 `ceph -s`가 `active+clean`이 될 때까지 대기한 후 진행하는 것이 안전합니다. 또한 제거 전에 **남은 OSD가 전체 데이터를 수용할 수 있는지** 용량을 사전 계산할 필요가 있습니다.

full 상태에서는 삭제 작업조차 실패합니다. `rbd rm`은 이미지를 trash로 이동시키는데, 이 메타데이터 기록 또한 쓰기 작업이기 때문인 것으로 보입니다.

```
librbd::trash::MoveRequest: failed to add image to trash: (28) No space left on device
```

---

## 5. shelve로 인한 스토리지 고갈

### 증상

L4 측정용 인스턴스 생성 시 `No valid host was found` 오류가 발생했습니다. RAM 확보를 위해 미사용 K8s 노드를 `shelve` 처리했으나, 이후 Ceph 클러스터가 full 상태에 도달했습니다.

### 원인 분석

`openstack server shelve`는 인스턴스를 정지시키는 것이 아니라, **인스턴스 디스크 전체를 Glance 이미지로 복사**한 후 인스턴스를 삭제하는 동작으로 알려져 있습니다.

```
k8s-control-shelved  : 32 GiB
k8s-worker1-shelved  : 32 GiB
k8s-worker2-shelved  : 32 GiB
                       ─────
                       96 GiB × Replica 3 = 288 GiB
```

RAM을 반납하려는 조치가 스토리지를 288GB 소모하는 결과로 이어졌습니다. 이미지 상태가 `saving`에서 멈춰 있었는데, 복사 도중 클러스터가 full에 도달해 쓰기가 차단된 것으로 보입니다.

또한 `SHUTOFF` 상태의 인스턴스는 Nova가 자원을 계속 예약하므로 **정지만으로는 RAM이 반납되지 않는다는 점**도 함께 확인했습니다.

### 해결

`unshelve` 복구를 시도했으나 full 상태로 인해 실패했습니다. Glance에서 이미지를 삭제한 후, RBD에 남은 고아 이미지를 직접 제거했습니다.

```bash
# Glance 삭제
openstack image delete <shelved-image-id>

# RBD 락 확인 및 해제
rbd status images/<image-id>
docker restart glance_api        # watcher 해제

# 임계값 상향 후 삭제
ceph osd set-full-ratio 0.98
rbd rm images/<image-id>
```

RAM 확보는 다른 방식으로 해결했습니다. 불필요한 인스턴스를 삭제하여 컴퓨트 노드 하나를 완전히 비운 후, 측정용 인스턴스를 해당 노드로 리사이즈했습니다.

### 교훈

`shelve`는 RAM과 스토리지를 교환하는 성격의 작업으로 보입니다. 스토리지 여유가 충분하지 않은 환경에서는 사용을 재고할 필요가 있습니다. 자원 확보가 목적이라면 인스턴스 삭제 또는 스펙 축소가 더 적절할 것으로 생각됩니다.

또한 hypervisor의 가용 RAM은 전체 합계가 아니라 **개별 노드 기준**으로 판단해야 합니다. `free_ram_mb 23999`가 표시되더라도 3개 노드에 분산되어 있으면 12GB 인스턴스를 배치할 수 없습니다.

---

## 6. 측정 도구 검증 및 워킹셋 중첩

### 배경

NKS 측정에 `ljishen/fio`(fio 3.6)를 사용했으므로 전 계층에서 동일 도구를 사용하고자 했으나, 컨테이너 실행이 측정에 영향을 주는지 확인이 필요했습니다.

### 검증 설계

컨테이너는 가상화가 아니라 프로세스 격리이므로 호스트 커널을 그대로 사용합니다. 다만 다음 두 조건이 충족되어야 측정이 유효할 것으로 판단했습니다.

| 조건 | 이유 |
|------|------|
| 측정 대상이 이미지 레이어 밖에 있을 것 | overlayfs 오버헤드 배제 |
| cgroup I/O 제한이 없을 것 | 인위적 제한 배제 |

raw 디바이스는 `-v /dev:/dev`로, 파일시스템은 bind mount로 전달하여 두 조건을 충족시켰습니다. bind mount는 K8s PVC 마운트와 동일한 메커니즘이므로 조건이 일치하는 것으로 보았습니다.

```bash
# raw
docker run --rm --privileged -v /dev:/dev ljishen/fio --filename=/dev/xxx

# 파일시스템
docker run --rm -v /mnt/target:/data ljishen/fio --filename=/data/testfile
```

### 검증 결과

프리컨디셔닝과 워킹셋 분리를 적용한 후 재검증한 결과, native fio와 0.5% 이내로 일치했습니다. (1절 참조)

### 컨테이너 런타임 차이 대응

L4 측정 노드는 Kubernetes 워커이므로 `containerd`가 설치되어 있어 `docker.io` 패키지와 충돌이 발생했습니다.

```
containerd.io : Conflicts: containerd
```

Docker 설치 대신 `ctr`로 동일 이미지를 실행하여 도구 일관성을 유지했습니다.

```bash
ctr run --rm \
  --mount type=bind,src=/mnt/rbd-l4,dst=/data,options=rbind:rw \
  docker.io/ljishen/fio:latest <name> \
  fio --filename=/data/testfile ...
```

### 부수 확인 — fio 버전 간 리포트 차이

절대값(IOPS, 대역폭, latency)은 일치하나, `bw` 통계의 `per=` 백분율 산출 기준이 다른 것으로 확인됩니다.

```
fio 3.6  : per=25.05%   (잡별 대역폭 기준으로 보임)
fio 3.28 : per=99.53%   (그룹 합산 기준으로 보임)
```

결과 해석 시 혼동하지 않도록 기록해 둡니다.

---

## 종합

이 프로젝트에서 발견한 오류는 두 유형으로 구분해 볼 수 있습니다.

| 유형 | 사례 | 특징 |
|------|------|------|
| **명시적 실패** | 3, 4, 5번 | 오류 메시지 발생, 발견이 비교적 용이 |
| **암묵적 오염** | 1, 2, 6번 | 정상 동작처럼 보이나 데이터가 잘못됨 |

후자가 더 위험하다고 생각합니다. 프리컨디셔닝을 누락해도 fio는 정상 종료되고, 잘못된 디스크를 관측해도 iostat은 오류 없이 출력합니다. **측정값이 예상과 다를 때 도구나 환경 차이로 결론짓기 전에, 측정 조건 자체를 먼저 의심하는 절차**가 필요하다고 판단됩니다.

1번 사례에서 52% 차이를 "fio 버전 차이"로 결론지었다면, 이후 모든 계층 비교의 신뢰성이 흔들렸을 것으로 보입니다.

---

← [README로 돌아가기](../README.md)
