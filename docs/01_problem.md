# 01. 문제 인식 — NKS 스토리지 성능의 해석 어려움

## 배경

MSP 엔지니어가 고객사의 managed Kubernetes 환경을 운영할 때, "스토리지가 느리다"는 문의는 흔한 요청 유형이라고 생각합니다. 이때 필요한 것은 수치 자체가 아니라 **그 수치가 왜 나오는지에 대한 해석**입니다.

이 프로젝트는 NKS(NHN Kubernetes Service)에서 스토리지 성능을 측정하던 중, 수치는 얻었으나 해석하기 어려운 상황에 직면하면서 시작되었습니다.

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

## 3. 관측된 특이 패턴

### 패턴 1 — 읽기와 쓰기가 거의 동일합니다

```
randwrite 4K : 2,002 IOPS
randread  4K : 2,007 IOPS   ← 차이 0.25%
```

분산 스토리지에서 쓰기는 복제본 전파와 저널링을 거치고, 읽기는 Primary 노드가 응답하는 구조가 일반적입니다. 따라서 읽기가 쓰기보다 빠를 것으로 예상됩니다.

두 값이 0.25% 차이로 수렴한다는 것은, **측정된 수치가 스토리지의 물리적 특성을 온전히 반영하지 않을 가능성**을 시사한다고 생각했습니다.

### 패턴 2 — 변동성이 매우 낮습니다

60초 측정 구간의 순간 IOPS 범위입니다.

```
randwrite : min 458 ~ max 634   (표준편차 16.00)
randread  : min 462 ~ max 780   (표준편차 23.08)
```

일반적으로 플래시 기반 스토리지는 가비지 컬렉션, 캐시 히트율 변화, 다른 테넌트의 부하 등에 따라 순간 성능이 변동하는 것으로 알려져 있습니다. 60초 내내 좁은 범위를 유지하는 것은 다소 이례적으로 보였습니다.

### 패턴 3 — Latency가 상대적으로 높습니다

평균 63.9ms는 NVMe SSD 기준으로 높은 값입니다. 일반적인 NVMe 랜덤 쓰기 지연은 수십~수백 마이크로초 수준으로 알려져 있습니다.

그러나 이 값이 **디스크 처리 시간에서 비롯된 것인지, 대기열에서 비롯된 것인지 구분하기 어렵습니다.**

---

## 4. 해석이 어려운 이유

세 가지 가설이 유사한 측정 결과를 만들 수 있다고 판단했습니다.

| 가설 | 설명 | 검증에 필요한 것 |
|------|------|-----------------|
| A. 하드웨어 한계 | 백엔드 디스크가 실제로 느림 | 물리 디스크 I/O 통계 |
| B. 네트워크 지연 | 스토리지 노드까지의 왕복 지연 | 네트워크 홉·대역폭 정보 |
| C. QoS 정책 | 볼륨당 IOPS 상한 설정 | 백엔드 정책 설정값 |

그런데 managed Kubernetes에서는 이 중 어느 것도 직접 확인하기 어렵습니다.

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

CSI 인터페이스 아래는 플랫폼 사업자의 관리 영역이며, 사용자에게 노출되지 않습니다.

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

수치는 있으나 해석의 근거가 부족한 상태를 해소하기 위해, 다음과 같은 접근을 택했습니다.

> **블랙박스 내부를 직접 볼 수 없다면, 동등한 형태의 스택을 재현하여 각 계층이 성능에 미치는 영향을 개별적으로 측정해 본다.**

재현 대상은 NKS와 유사한 프로비저닝 경로를 갖는 구성으로 설정했습니다.

```
NKS  : Pod → PVC → cinder-default → Cinder → ??? → ???
재현 : Pod → PVC → cinder-rbd     → Cinder → Ceph RBD → NVMe
                                              ↑ 여기부터 직접 관측 가능
```

`cinder-default`라는 StorageClass 이름은 NKS 역시 OpenStack Cinder를 프로비저너로 사용함을 시사합니다. 그 아래 백엔드가 Ceph인지 다른 스토리지인지는 공개 정보로 확인할 수 없지만, 통상 분산 스토리지를 구성하기에 ceph를 기술스택으로 선정했습니다.
이 프로젝트에서는 **확인 불가 영역으로 명시**하고 진행합니다.

따라서 이후 비교는 "동일한 구성 간의 비교"가 아니라, **"Pod에서 PVC에 접근했을 때 나타나는 성능"이라는 관점에서의 대조**로 이해해 주시기 바랍니다.

---

## 6. 검토 목표

재현 스택에서 다음을 확인하고자 했습니다.

| # | 질문 | 확인 방법 |
|---|------|-----------|
| 1 | 각 계층은 성능에 얼마나 기여하는가 | 계층별 단계적 측정 |
| 2 | 병목은 주로 어느 계층에 있는가 | 계층 간 손실률 비교 |
| 3 | 동등한 형태의 스택 성능은 NKS와 얼마나 다른가 | L5 ↔ L6 대조 |
| 4 | NKS의 63.9ms는 무엇으로 설명될 수 있는가 | Little's Law 검토 |

이 중 4번은 **직접 검증이 아닌 간접 추론**이 될 수밖에 없다는 점을 전제로 진행했습니다.

---

> 다음 문서: [02_environment.md](02_environment.md) — 재현 스택 구축 및 실험 설계

← [README로 돌아가기](../README.md)
