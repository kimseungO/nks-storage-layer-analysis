# 01. 문제 인식 — NKS 스토리지 성능의 해석 불가능성

## 배경

MSP 엔지니어가 고객사의 managed Kubernetes 환경을 운영할 때, 서비스 지연이 일어나는 원인 가운데 스토리지io 지연은 주요 지연 원인 중 하나입니다. 이때 필요한 것은 수치 자체가 아니라 **그 수치가 왜 나오는지에 대한 해석**입니다.

이 프로젝트는 NKS(NHN Kubernetes Service)에서 직접 스토리지 io를 측정해 보고, 직접 볼 수 없느 가려진 인프라에 대해 직접 구현하고 측정하고 해석하는 과정을 담고있습니다.

---

## 1. 측정 환경

### 클러스터 구성

| 항목 | 값 |
|------|-----|
| 플랫폼 | NHN Cloud NKS |
| StorageClass | `cinder-default` |
| PVC 크기 | 10 Gi |
| 마운트 경로 | `/data` (Pod 내부) |
| 측정 도구 | `ljishen/fio` 컨테이너 (fio 3.6) |

### 측정 매니페스트

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: fio-test-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: cinder-default
  resources:
    requests:
      storage: 10Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: fio-test
  namespace: default
spec:
  containers:
  - name: fio
    image: ljishen/fio
    command: ["sleep", "3600"]
    volumeMounts:
    - name: test-vol
      mountPath: /data
  volumes:
  - name: test-vol
    persistentVolumeClaim:
      claimName: fio-test-pvc
```

### 측정 조건

```
공통: direct=1, ioengine=libaio, size=2G, runtime=60s, time_based

randwrite-4k : rw=randwrite, bs=4k,  iodepth=32, numjobs=4
randread-4k  : rw=randread,  bs=4k,  iodepth=32, numjobs=4
seqwrite-1m  : rw=write,     bs=1M,  iodepth=16, numjobs=1
seqread-1m   : rw=read,      bs=1M,  iodepth=16, numjobs=1
```

---

## 2. 측정 결과

| 종목 | IOPS | Bandwidth | 평균 Latency | p99 |
|------|:----:|:---------:|:------------:|:---:|
| randwrite 4K | 2,002 | 8,012 KiB/s | 63.90 ms | 78 ms |
| randread 4K | 2,007 | 8,029 KiB/s | 63.75 ms | 65 ms |
| seqwrite 1M | 53 | 53.2 MiB/s | 299.89 ms | 684 ms |
| seqread 1M | 64 | 64.1 MiB/s | 249.41 ms | 257 ms |

---

## 3. 관측된 이상 패턴

### 패턴 1 — 읽기와 쓰기가 동일합니다

```
randwrite 4K : 2,002 IOPS
randread  4K : 2,007 IOPS   ← 차이 0.25%
```

분산 스토리지에서 쓰기는 복제본 전파와 저널링을 거치고, 읽기는 Primary 노드가 즉시 응답합니다. 따라서 읽기가 쓰기보다 빠른 것이 일반적입니다.

두 값이 0.25% 차이로 일치한다는 것은, **측정된 수치가 스토리지의 물리적 특성을 반영하지 않을 가능성**을 시사합니다.


### 패턴 2 — Latency가 비정상적으로 높습니다

평균 63.9ms는 NVMe SSD 기준으로 매우 느린 값입니다. 일반적인 NVMe 랜덤 쓰기 지연은 수십~수백 마이크로초 수준입니다.

그러나 이 값이 **디스크가 느려서인지, 대기열이 길어서인지 구분할 수 없습니다.**

---

## 4. 해석이 불가능한 이유

세 가지 가설이 동일한 측정 결과를 만들 수 있습니다.

| 가설 | 설명 | 검증에 필요한 것 |
|------|------|-----------------|
| A. 하드웨어 한계 | 백엔드 디스크가 실제로 느림 | 물리 디스크 I/O 통계 |
| B. 네트워크 지연 | 스토리지 노드까지의 왕복 지연 | 네트워크 홉·대역폭 정보 |
| C. QoS 정책 | 볼륨당 IOPS 상한 설정 | 백엔드 정책 설정값 |

그런데 managed Kubernetes에서는 이 중 어느 것도 확인할 수 없습니다.

### 관찰 가능성 매트릭스

| 관찰 대상 | Self-managed | NKS |
|-----------|:------------:|:---:|
| Pod 내부 fio 측정 | 가능 | 가능 |
| PVC → PV 매핑 | 가능 | 가능 |
| PV → 백엔드 볼륨 매핑 | 가능 | 부분 |
| 스토리지 백엔드 종류 | 가능 | **불가** |
| 복제 정책 (Replica 수) | 가능 | **불가** |
| OSD/노드별 지연 통계 | 가능 | **불가** |
| 물리 디스크 I/O (iostat) | 가능 | **불가** |
| 쓰기 증폭률 교차 검증 | 가능 | **불가** |

CSI 인터페이스 아래는 플랫폼 사업자의 영역이며, 사용자에게 노출되지 않습니다.

```
    [ Pod ]           ← 측정 가능
       │
    [ PVC / CSI ]     ← 측정 가능
       │
  ─────┼───────────── 추상화 경계
       │
    [ ??? ]           ← 관측 불가
       │
    [ ??? ]           ← 관측 불가
```

---

## 5. 이 프로젝트의 출발점

수치는 있으나 해석이 없는 상태를 해소하기 위해, 다음과 같은 접근을 택했습니다.

> **블랙박스 내부를 직접 볼 수 없다면, 동등한 스택을 재현하여 각 계층이 성능에 미치는 영향을 개별적으로 측정한다.**

재현 대상은 NKS와 동일한 프로비저닝 경로를 갖는 구성입니다.

```
NKS  : Pod → PVC → cinder-default → Cinder → ??? → ???
재현 : Pod → PVC → cinder-rbd     → Cinder → Ceph RBD → NVMe
                                              ↑ 여기부터 직접 관측 가능
```

`cinder-default`라는 StorageClass 이름은 NKS 역시 OpenStack Cinder를 프로비저너로 사용함을 시사합니다. 다만 그 아래 백엔드가 Ceph인지 다른 스토리지인지는 공개 정보로 확정할 수 없으며, 이 프로젝트에서는 **확인 불가 영역으로 명시**하고 진행합니다.

---

## 6. 검증 가설

재현 스택에서 다음을 확인하고자 했습니다.

| # | 가설 | 확인 방법 |
|---|------|-----------|
| 1 | 각 계층은 성능에 얼마나 기여하는가 | 계층별 단계적 측정 |
| 2 | 병목은 어느 계층에 있는가 | 계층 간 손실률 비교 |
| 3 | 동등 스택의 성능은 NKS와 얼마나 다른가 | L5 ↔ L6 대조 |
| 4 | NKS의 63.9ms는 무엇으로 설명되는가 | Little's Law 검증 |

---

> 다음 문서: [02_environment.md](02_environment.md) — 재현 스택 구축 및 실험 설계

← [README로 돌아가기](../README.md)
