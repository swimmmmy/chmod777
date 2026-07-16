# GNSS 미코닝(Meaconing) 및 Pseudolite 기반 PNT 기만 시나리오

## Topic

> "GNSS 신호 물리적 시간 지연(Meaconing)을 통한 Timing(T) 정보 교란이 PNT(Position, Navigation, Timing)에 미치는 영향 분석"

공격자는 시스템을 해킹하거나 내부 네트워크에 침투하지 않는다. 단지 정상적인 GNSS 신호를 매우 짧은 시간(나노초 단위) 지연시켜 재전송할 뿐이다. 그러나 수신기는 이를 정상 신호로 인식할 수 있으며, 위치(Position), 항법(Navigation), 시각(Timing)에 오류가 발생해 PNT 기반 무기체계의 운용에 영향을 줄 수 있다.

---

## Team

- 박대범
- 진민교
- 김수영

---

## Background

### PNT란?

**Positioning(위치) + Navigation(항법) + Timing(시각)**

현대 군사 시스템은 대부분 PNT 정보를 기반으로 운용된다.

- 드론 및 무인기 비행
- 군함·잠수함 항법
- 유도무기 표적 추적
- 군 통신망 시간 동기화
- 지휘통제(C2) 시스템

그중에서도 **Timing(T)** 은 Position과 Navigation을 계산하는 기준이 되기 때문에, 아주 작은 시간 오차도 전체 PNT 정확도에 영향을 줄 수 있다.

[![PNT 구조도](https://github.com/swimmmmy/chmod777/raw/main/pnt_structure.png)](/swimmmmy/chmod777/blob/main/pnt_structure.png)
> 그림 1. PNT 시스템 구조도 (GNSS, SBAS, INS 포함)

---

### GNSS란?

GNSS(Global Navigation Satellite System)는 위성을 이용하여 PNT 정보를 제공하는 위성항법 시스템의 총칭이다.

| 국가  | 시스템     | 위성 수(약) |
| --- | ------- | ------- |
| 미국  | GPS     | 31      |
| 러시아 | GLONASS | 24      |
| 유럽  | Galileo | 27      |
| 중국  | BeiDou  | 35      |

[![GNSS 신호 수신 구조](https://github.com/swimmmmy/chmod777/raw/main/gnss_signal1.png)](/swimmmmy/chmod777/blob/main/gnss_signal1.png)
> 그림 2. GNSS 위성 신호 수신 구조 (Space Segment → User Segment)

[![GNSS 신호 수신 구조 (SBAS 포함)](https://github.com/swimmmmy/chmod777/raw/main/gnss_signal2.png)](/swimmmmy/chmod777/blob/main/gnss_signal2.png)
> 그림 2-1. SBAS 지상기지국을 포함한 GNSS 신호·보정값 흐름 (Space Segment / User Segment / Ground Segment)

**GNSS 신호의 특징**

- GPS 위성은 약 20,000km 이상의 고도에서 신호를 송신한다.
- 위성 송신 출력: 약 25W
- 지상 수신 전력: 약 -160dBW
- 지상에 도달하는 신호가 매우 약하기 때문에 전파 간섭이나 GNSS 기만 공격의 영향을 받을 가능성이 있다.

---

### 핵심 공격 기법

#### 1. 미코닝(Meaconing)

정상적인 GNSS 신호를 수신한 뒤, 내용은 변경하지 않고 시간만 아주 짧게 지연시켜 다시 송신하는 공격 기법

| 구분     | 재밍    | 스푸핑      | 미코닝            |
| ------ | ----- | -------- | -------------- |
| 방식     | 신호 방해 | 가짜 신호 생성 | 정상 신호 지연 후 재송신 |
| 데이터    | 없음    | 가짜       | 정상 데이터 그대로     |
| 탐지 난이도 | 쉬움    | 보통       | **상대적으로 어려움**  |

**미코닝이 탐지하기 어려운 이유**

1. 신호 내용이 정상 데이터와 동일하다.
2. 자연적인 전파 지연(전리층 오차)과 구별하기 어렵다.
3. 수신기는 더 강한 신호를 우선 선택한다 (Capture Effect).
4. 기존 GNSS 수신기에서는 이상 여부를 즉시 판단하기 어렵다.

---

#### 2. Pseudolite

Pseudolite(Pseudo + Satellite)는 지상 또는 드론·항공기 등에 설치되어 GNSS 위성처럼 신호를 송신하는 장치이다.

- GPS 위성보다 훨씬 가까운 위치에서 신호를 송신하므로 수신기에는 상대적으로 강한 신호로 인식된다.
- 드론이나 이동 차량에 탑재하면 물리적 침투 없이 원거리에서 공격이 가능하다.

---

#### 3. 두 기법의 결합 시나리오

```
드론 또는 이동 플랫폼
        ↓
Pseudolite 장비 운용
        ↓
Meaconing 방식의 지연 신호 송신
        ↓
GNSS 수신기 정상 신호로 인식
        ↓
PNT 정보 왜곡
```

---

## Scenario

### 가정

본 시나리오에서는 적이 드론이나 차량 등에 Pseudolite를 탑재한 뒤, 정상 GNSS 신호를 일정 시간 지연시켜 다시 송신하는 상황을 가정하였다.

### 단계별 흐름

```
① GNSS 신호 지연(Meaconing)
          ↓
② Timing(T) 정보에 미세한 오차 발생
          ↓
③ ΔR = c × Δt (거리 계산 오류)
          ↓
④ Position(Positioning) 오차 발생
          ↓
⑤ Navigation(항법) 오류 발생
          ↓
⑥ PNT 기반 시스템의 연쇄적인 운용 오류
```

- ΔR : 거리 오차
- c : 빛의 속도 (약 3×10⁸ m/s)
- Δt : 시간 오차

| 시간 오차  | 거리 오차   |
| ------ | ------- |
| 1 ns   | 약 30 cm |
| 10 ns  | 약 3 m   |
| 100 ns | 약 30 m  |

결국 GNSS에서는 Timing이 정확해야 Position과 Navigation도 정확하게 계산된다. 이번 시나리오는 작은 Timing 오차만으로도 PNT 전체에 어떤 변화가 생기는지 확인하는 데 목적이 있다.

---

### 예상 영향

| 대상    | 예상 영향              |
| ----- | ------------------ |
| 군용 드론 | 위치 오차로 임무 수행 오류 가능 |
| 항공기   | 항법 오차 및 접근 절차 영향   |
| 군함    | 항로 이탈 가능성          |
| 유도무기  | 표적 좌표 오차 발생 가능     |
| 군 통신망 | 시간 동기화 오류 가능       |
| 암호체계  | 시간 기반 인증 실패 가능     |

> ※ 실제 영향은 운용 환경, 복합항법(INS), 보안 기능 적용 여부 등에 따라 달라질 수 있다.

---

## 실증 방법

### PX4 SITL 기반 GNSS 오차 주입 시뮬레이션

실제 RF 장비 대신 시뮬레이션 환경을 이용해 GNSS 기만 상황을 구현한다.

**사용 도구**

- PX4 SITL (Docker, `jonasvautherin/px4-gazebo-headless`)
- Gazebo
- QGroundControl (GCS)
- Python + pymavlink

**실행 환경**

| 항목 | 값 |
| --- | --- |
| Host OS | Windows (VMware Workstation) |
| Guest OS | Ubuntu 24.04.4 LTS |
| VM 리소스 | Memory 8GB, Processors 4 이상 |

> 초기에는 WSL 기반으로 진행했으나, WSL–Docker–QGroundControl 간 네트워크 연결이 근본적으로 불안정하여 **VMware Ubuntu Desktop 환경으로 전환**하였다. VM 안에서 PX4와 QGroundControl을 함께 실행하여 네트워크 경계를 없애는 방식으로 연결 문제를 해결했다. 상세 구축 절차는 [docs/px4-sitl-setup.md](docs/px4-sitl-setup.md) 참고.

**실험 절차**

**1단계 — 정상 비행 확인** ✅ 완료

- PX4 SITL + Gazebo 환경 실행
- QGroundControl에서 드론 연결 및 `commander takeoff`로 이륙 확인
- 정상 GNSS 신호 상태에서 비행 경로 확인 및 캡처

**2단계 — GNSS 오차 단계별 주입** (진행 예정)

- Python + pymavlink 스크립트로 GPS 오차값 주입
- +10m → +50m → +100m → +300m 단계적으로 적용
- 각 단계별 비행 경로 변화 관찰 및 기록

| 오차    | 미코닝 시간 지연 | 예상 결과             |
| ----- | --------- | ----------------- |
| +10m  | ≈ 33ns    | 비행경로 미세 변화        |
| +50m  | ≈ 167ns   | Waypoint 이탈 시작    |
| +100m | ≈ 333ns   | RTH 오작동           |
| +300m | ≈ 1000ns  | Position Drift 심각 |

**3단계 — GCS 화면 비교**

| 정상 환경         | 기만 환경              |
| ------------- | ------------------ |
| 정상 GNSS 위치 정보 | 가상의 GNSS 위치 정보 적용  |
| 드론 위치 정상 표시   | 변경된 위치로 인식         |
| 정상적인 항법 수행    | 위치 정보 왜곡에 따른 항법 변화 |

> GCS 화면은 정상처럼 보이지만, 드론은 실제 위치가 아닌 기만된 GNSS 정보를 기준으로 위치를 인식하고 비행하게 된다.

[![실증 흐름도](https://github.com/swimmmmy/chmod777/raw/main/images/poc_flow.png)](/swimmmmy/chmod777/blob/main/images/poc_flow.png)
> 그림 3. PX4 SITL 기반 GNSS 오차 주입 실증 흐름

---

## 대응 방안

### 기술적 대응

**1. 다중 GNSS 교차검증**

- GPS, Galileo, BeiDou 등 복수의 위성항법 시스템을 동시 수신하여 신호 일관성을 교차검증한다.
- 단일 시스템 의존 구조에서 다중 시스템 병행 운용 구조로 전환이 필요하다.

**2. TESLA 프로토콜 기반 신호 인증**

- TESLA(Timed Efficient Stream Loss-tolerant Authentication)는 브로드캐스트 신호에 해시체인 기반 인증을 적용하는 프로토콜이다.
- 각 메시지마다 해시값을 연결하여 신호 위조 여부를 검증할 수 있으며, 미코닝 공격으로 인한 시간 지연 신호를 탐지하는 데 활용될 수 있다.
- 현재 GPS L1 신호에는 미적용 상태이므로 도입이 시급하다.

**3. 신호 방향 탐지 (AOA)**

- 안테나 어레이를 활용한 신호 도달 방향(Angle of Arrival) 분석으로 비정상적인 방향에서 수신되는 신호를 탐지한다.

**4. INS 복합항법 보완**

- GNSS 신호 이상 감지 시 관성항법장치(INS)로 자동 전환하는 복합항법 체계를 구축한다.
- 단, INS는 단기간 사용에 한정되므로 장기적인 백업 수단으로는 한계가 있다.

### 제도적 대응

**1. 군용 GNSS 수신기 미코닝 대응 기준 마련**

- 현재 군용 GNSS 수신기에는 미코닝 탐지 기준이 명시되어 있지 않다.
- 신호 인증 메커니즘 적용을 의무화하는 규정 마련이 필요하다.

**2. KPS 설계 단계부터 미코닝 대응 내재화**

- 2035년 완성 예정인 한국형 위성항법시스템(KPS) 설계 단계부터 미코닝 및 Pseudolite 공격에 대한 보안 요구사항을 반영해야 한다.

**3. 정기 신호 무결성 점검 의무화**

- 군용 PNT 시스템에 대한 정기적인 신호 무결성 점검 체계를 구축한다.

---

## Cases (실제 사례)

| 사례                 | 연도        | 내용                                              |
| ------------------ | --------- | ----------------------------------------------- |
| RQ-170 미군 드론 이란 착륙 | 2011      | 미코닝 방식으로 GPS 신호를 조작하여 군용 드론을 이란 영토에 착륙시킨 것으로 추정 |
| GNSS 스푸핑 500% 급증   | 2023→2024 | OPSGROUP 보고서 기준 전 세계 GNSS 스푸핑 사건 500% 증가        |
| NATO 군함 위치 스푸핑     | 2021      | 오데사 항구 인근 NATO 군함 2척의 위치 정보가 스푸핑 공격으로 왜곡됨       |
| 북한 GPS 교란          | 2023      | 한미연합훈련 기간 중 북한의 GPS 재밍으로 군 훈련 방해                |
| 이라크 상공 민항기 GPS 오류  | 2023      | 이란 소행으로 추정되는 GPS 기만으로 민항기 항법 장애 발생              |

---

## 향후 계획

- [x] 주제 최종 확정
- [ ] PNT-GNSS 데이터 흐름도(DFD) 작성
- [x] PX4 SITL + QGroundControl 실증 환경 구축 (VMware Ubuntu)
- [ ] GPS 오차 단계별 주입 Python 스크립트 작성 (+10m/+50m/+100m/+300m)
- [ ] GCS 화면 캡처 및 시연 영상 제작
- [ ] 본문 및 발표 자료 작성
- [ ] 참고문헌 정리

---

## References

[1] OPSGROUP, "GNSS Spoofing: The Threat to Aviation," 2024.

[2] Eurocontrol, "Aviation Information Note: GNSS Radio Frequency Interference (RFI)," 2024.

[3] U.S. Department of Defense (DoD), "Department of Defense Positioning, Navigation, and Timing (PNT) Enterprise Strategy," 2021.

[4] M. L. Psiaki and T. E. Humphreys, "GNSS Spoofing and Detection," *Proceedings of the IEEE*, vol. 104, no. 6, pp. 1258-1270, 2016.

[5] J. Steiner, S. Pleninger, and J. Hospodka, "Assessing the Vulnerability of Aviation Systems to GNSS Meaconing Attacks," in *2024 New Trends in Civil Aviation (NTCA)*, IEEE, 2024.

[6] 김동균 등, "GNSS 스푸핑 공격에 따른 시간 동기화 시스템의 취약성 분석 및 탐지 기법," 한국정보보호학회 논문지, 2022.

[7] 이용원, 손창근, 이규호, "국내 위성 사이버보안 가이드라인 발전 방안 연구," 방산안보연구, 제1권, 제1호, pp. 71-86, 2025.

[8] 박상현 등, "무인기(UAV) 항법 시스템 및 군 통신망에 대한 스푸핑 위협 분석," 한국항공운항학회지, 2023.

[9] 과학기술정보통신부, "한국형 위성항법시스템(KPS) 개발사업 예비타당성조사 통과," 보도자료, 2021년 6월 25일.

[10] SBG Systems, "Meaconing: the most common type of GNSS spoofing interference attacks," 2023.
