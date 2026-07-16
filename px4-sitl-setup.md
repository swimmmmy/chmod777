# PX4 SITL + QGroundControl 실증 환경 구축 (VMware Ubuntu)

[실증 방법](../README.md#실증-방법)의 1단계(정상 비행 확인) 환경 구축 과정과, VMware Ubuntu에서 재현 시 필요한 절차를 정리한 문서이다.

## 목차

- [환경](#환경)
- [사전 설치](#사전-설치)
- [실행 절차](#실행-절차)
- [문제 원인과 해결](#문제-원인과-해결)
- [참고: 트러블슈팅](#참고-트러블슈팅)

---

## 환경

| 항목 | 값 |
| --- | --- |
| Host OS | Windows (VMware Workstation) |
| Guest OS | Ubuntu 24.04.4 LTS |
| VM 리소스 | Memory 8GB, Processors 4 이상 권장 |
| Docker | 29.1.3 |
| PX4 이미지 | `jonasvautherin/px4-gazebo-headless` |
| QGroundControl | Daily build (AppImage, x86_64) |

> 초기 2코어/4GB로는 Gazebo 물리 시뮬레이션이 CPU를 따라가지 못해 `Accel #0 fail: TIMEOUT` 오류가 반복됨. 4코어/8GB 이상으로 증설 후 안정화됨.

---

## 사전 설치

### Docker

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER && newgrp docker
```

### QGroundControl

```bash
sudo usermod -aG dialout "$(id -un)"
sudo apt install -y libfuse2 libxcb-xinerama0 libxkbcommon-x11-0 libxcb-cursor-dev \
  gstreamer1.0-plugins-bad gstreamer1.0-libav gstreamer1.0-gl python3-gi python3-gst-1.0

wget https://d176tv9ibo4jno.cloudfront.net/builds/master/QGroundControl-x86_64.AppImage
chmod +x QGroundControl-x86_64.AppImage
```

> 예전에 흔히 쓰이던 `latest/QGroundControl.AppImage` 링크는 현재 403 Forbidden으로 막혀 있다. 위 `builds/master/` 경로를 사용해야 한다.

---

## 실행 절차

**1. PX4 SITL 실행** (터미널 1)

```bash
docker run --rm -it --network host \
  -e PX4_HOME_LAT=36.4800 -e PX4_HOME_LON=127.2890 -e PX4_HOME_ALT=0.0 \
  jonasvautherin/px4-gazebo-headless
```

`pxh>` 프롬프트가 뜰 때까지 대기한다. (좌표는 충청권 기준으로 설정)

**2. PX4 콘솔에서 통신 설정** (`pxh>`에 순서대로 입력)

```
param set MAV_0_BROADCAST 1
mavlink stop-all
mavlink start -x -u 14555 -r 4000000 -t 127.0.0.1 -o 14550
```

**3. QGroundControl 실행** (터미널 2)

```bash
./QGroundControl-x86_64.AppImage
```

**4. 연결 확인**

자동연결이 되지 않으면 수동으로 링크를 생성한다.
QGC 좌상단 `Disconnected` 클릭 → Comm Links → Add → Type: UDP, Listening Port: `14550` → Save → Connect

**5. 이륙 테스트** (`pxh>`)

```
commander takeoff
```

QGC 화면 상태가 `Ready` → `Takeoff` → `비행중`으로 전환되고, 위성 개수·배터리·좌표가 표시되면 정상 비행 확인 단계가 완료된 것이다.

---

## 문제 원인과 해결

기본 설정(`docker run -p 14550:14550/udp`)으로는 QGC와 PX4가 연결되지 않았다. 원인을 역추적한 과정은 다음과 같다.

| 증상 | 원인 | 해결 |
| --- | --- | --- |
| QGC에서 `UDP Link error: Address already in use` | Docker의 포트 포워딩 프로세스(`docker-pr`)가 포트를 독점(bind)하여, QGC가 같은 포트를 열려는 시도와 충돌 | `--network host`로 컨테이너와 호스트 네트워크를 공유시켜 포트 포워딩 자체를 제거 |
| `--network host`로도 데이터 미수신 (`tcpdump`로 확인) | PX4 기본 설정상 MAVLink가 localhost 내부로만 제한됨 | `param set MAV_0_BROADCAST 1`로 외부 브로드캐스트 허용 |
| 여전히 UDP bind 충돌 반복 | PX4와 QGC가 동일 포트(14550)를 동시에 리스닝 시도 (UDP는 한 프로세스만 포트를 열 수 있음) | PX4는 `14555`에서 송신, `-o 14550`으로 QGC가 리스닝하는 `14550`을 목적지로 명시하여 포트 역할 분리 |
| `Accel #0 fail: TIMEOUT` 반복, Failsafe 발동 | VM(2코어/4GB)에서 Gazebo 물리 연산이 CPU를 따라가지 못함 | VM을 4코어/8GB로 증설. 완전히 해소되지는 않으나 이륙 직후 수 분간은 안정적으로 동작 |

핵심 진단 방법: `sudo tcpdump -i lo -n udp port <포트번호>`로 실제 UDP 트래픽 발생 여부를 직접 확인하는 것이 연결 문제를 좁혀나가는 데 가장 확실했다.

---

## 참고: 트러블슈팅

- Docker 컨테이너가 중복 실행되어 있으면(과거 `-p` 방식 컨테이너와 `--network host` 컨테이너가 동시에 남아있는 경우 등) Gazebo 인스턴스끼리 충돌하며 TIMEOUT을 유발한다. `docker ps -a`로 항상 확인 후 `docker rm -f $(docker ps -aq)`로 정리한다.
- QGC 프로세스가 창을 닫아도 완전히 종료되지 않고 포트를 계속 점유하는 경우가 있다. `pkill -9 -f QGroundControl`로 강제 종료한다.
- QGC 설정이 꼬였다고 판단되면 `rm -rf ~/.config/QGroundControl.org`로 초기화 후 재실행한다.
- VM 하드웨어 설정(Memory/Processors)은 VM이 완전히 꺼진 상태(Powered off)에서만 변경할 수 있다.

---

작성: 김수영 · CCSC 2026 팀 TNT
