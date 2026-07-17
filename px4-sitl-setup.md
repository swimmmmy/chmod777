# PX4 SITL + QGroundControl 실증 환경 구축 (VMware Ubuntu)

[실증 방법](../README.md#실증-방법) 1단계(정상 비행 확인)를 준비하면서 겪은 과정을 정리했다. 원래는 WSL로 진행했는데 Docker–QGroundControl 네트워크 연결이 계속 불안정해서, VMware Ubuntu Desktop 환경으로 옮겨서 다시 진행했다. 여러 번 시행착오를 겪었지만 결국 연결에 성공했고, 그 과정에서 알게 된 원인들을 같이 정리해둔다. 다음에 같은 걸 하는 팀원이 있으면 이 순서를 그대로 따라가면 된다.

## 목차

- [환경](#환경)
- [사전 설치](#사전-설치)
- [실행 절차](#실행-절차)
- [문제 원인과 해결](#문제-원인과-해결)
- [Gazebo 3D GUI 붙이기](#gazebo-3d-gui-붙이기)
- [참고: 트러블슈팅](#참고-트러블슈팅)

---

## 환경

| 항목 | 값 |
| --- | --- |
| Host OS | Windows (VMware Workstation) |
| Guest OS | Ubuntu 24.04.4 LTS |
| VM 리소스 | Memory 8GB, Processors 4 이상 권장 (Gazebo GUI까지 쓰려면 아래 참고) |
| Docker | 29.1.3 |
| PX4 이미지 | `jonasvautherin/px4-gazebo-headless` |
| QGroundControl | Daily build (AppImage, x86_64) |

처음엔 VM을 2코어/4GB로 시작했는데, PX4랑 QGC를 같이 켜니까 CPU가 못 버텨서 `Accel #0 fail: TIMEOUT` 오류가 계속 났다. 4코어/8GB로 늘리고 나서야 QGC 연결까지는 안정적으로 돌아갔다. (Gazebo 3D GUI를 추가로 켤 때는 이보다 더 필요하다 — 아래 섹션 참고.)

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

예전에 많이 쓰던 `latest/QGroundControl.AppImage` 링크는 지금 403 Forbidden으로 막혀 있다. 위의 `builds/master/` 경로로 받아야 한다.

---

## 실행 절차

**1. PX4 SITL 실행** (터미널 1)

```bash
docker run --rm -it --network host \
  -e PX4_HOME_LAT=36.4800 -e PX4_HOME_LON=127.2890 -e PX4_HOME_ALT=0.0 \
  jonasvautherin/px4-gazebo-headless
```

- `--rm` : 컨테이너 종료 시 자동 삭제 (테스트용이라 여러 번 켰다 껐다 하니까 안 지우면 계속 쌓임)
- `--network host` : 컨테이너가 VM의 네트워크를 그대로 공유하게 함. 원래는 `-p 14550:14550/udp`처럼 포트만 매핑하는 방식으로 했는데, 그러면 Docker의 포트 포워딩 프로세스가 포트를 독점해버려서 QGC가 못 들어옴. 이 옵션으로 그 문제를 우회함
- `-e PX4_HOME_LAT/LON/ALT` : 드론이 출발할 위치(위도/경도/고도). 충청권 좌표로 설정
- `jonasvautherin/px4-gazebo-headless` : PX4 + Gazebo가 미리 세팅된 Docker 이미지 이름

`pxh>` 프롬프트가 뜰 때까지 기다린다.

**2. PX4 콘솔에서 통신 설정** (`pxh>`에 순서대로 입력)

```
param set MAV_0_BROADCAST 1
mavlink stop-all
mavlink start -x -u 14555 -r 4000000 -t 127.0.0.1 -o 14550
```

- `param set MAV_0_BROADCAST 1` : PX4는 기본적으로 MAVLink 신호를 컨테이너 안(localhost)에서만 유지하도록 되어 있다. 이 값을 1로 켜야 밖으로도 신호가 나간다.
- `mavlink stop-all` : 기존에 돌던 MAVLink 채널을 전부 종료. 새 설정을 적용하려면 한 번 꺼야 한다.
- `mavlink start -x -u 14555 -r 4000000 -t 127.0.0.1 -o 14550` : MAVLink를 새로 시작하는 명령.
  - `-u 14555` : PX4가 자기 쪽에서 쓰는 로컬 포트
  - `-r 4000000` : 데이터 전송 속도(byte/s)
  - `-t 127.0.0.1` : 신호를 보낼 대상 IP (여기선 같은 VM 안이니 자기 자신)
  - `-o 14550` : 목적지 포트. QGC가 리스닝할 포트를 여기로 지정

  PX4와 QGC가 같은 14550 포트를 동시에 열려고(bind) 하면 충돌이 나서, PX4는 14555에서 보내고 QGC는 14550에서 받도록 포트를 나눈 것이 핵심이다.

**3. QGroundControl 실행** (터미널 2, 새 창)

```bash
./QGroundControl-x86_64.AppImage
```

**4. 연결 확인**

자동으로 연결되면 좋고, 안 되면 수동으로 링크를 만든다.
QGC 좌상단 `Disconnected` 클릭 → Comm Links → Add → Type: `UDP`, Listening Port: `14550` → Save → Connect

**5. 이륙 테스트** (`pxh>`)

```
commander takeoff
```

QGC 화면 상태가 `Ready` → `Takeoff` → `비행중`으로 바뀌고, 위성 개수·배터리·좌표가 표시되면 성공이다.

![이륙 성공 화면](takeoff_success.png)
> PX4 콘솔에서 `commander takeoff` 실행 후, QGC에서 "비행중 / Takeoff" 상태로 전환되고 위성 10개, 배터리 94%, 실제 좌표(충청권)가 지도 위에 표시되는 것까지 확인한 화면.

---

## 문제 원인과 해결

처음에 `docker run -p 14550:14550/udp` 방식으로 시도했을 때는 QGC가 계속 "Disconnected" 상태였다. 몇 시간에 걸쳐 원인을 하나씩 좁혀나간 과정은 다음과 같다.

| 증상 | 원인 | 해결 |
| --- | --- | --- |
| QGC에서 `UDP Link error: Address already in use` | Docker의 포트 포워딩 프로세스(`docker-pr`)가 포트를 독점(bind)해서, QGC가 같은 포트를 열려는 시도와 충돌 | `--network host`로 컨테이너와 호스트 네트워크를 공유시켜 포트 포워딩 자체를 없앰 |
| `--network host`로 바꿔도 데이터가 안 옴 (`tcpdump`로 확인) | PX4 기본 설정상 MAVLink가 localhost 내부로만 제한됨 | `param set MAV_0_BROADCAST 1`로 외부 브로드캐스트 허용 |
| 여전히 UDP bind 충돌 반복 | PX4와 QGC가 같은 포트(14550)를 동시에 리스닝하려고 시도 (UDP는 한 프로세스만 포트를 열 수 있음) | PX4는 `14555`에서 송신, `-o 14550`으로 QGC가 리스닝하는 `14550`을 목적지로 명시해서 포트 역할을 분리 |
| `Accel #0 fail: TIMEOUT` 반복, Failsafe 발동 | VM(2코어/4GB)에서 Gazebo 물리 연산이 CPU를 못 따라감 | VM을 4코어/8GB로 증설. 완전히 없어지진 않지만 이륙 직후 몇 분 동안은 안정적으로 동작함 |

가장 도움이 됐던 진단 방법은 `sudo tcpdump -i lo -n udp port <포트번호>`로 실제 UDP 트래픽이 오가는지 직접 확인하는 것이었다. 포트가 "열려있다"는 것과 "실제로 데이터가 흐른다"는 건 다른 문제였다.

---

## Gazebo 3D GUI 붙이기

여기까지는 QGC의 2D 지도 화면만으로 연결을 확인했다. 발표 시연을 위해 실제 Gazebo 3D 시뮬레이션 화면도 띄워봤다.

### GUI가 안 뜨는 이유와 해결

`jonasvautherin/px4-gazebo-headless` 이미지는 이름 그대로 headless(화면 없음)로 실행되도록 만들어져 있다. 하지만 이미지 안에 Gazebo GUI 실행 파일 자체는 포함되어 있어서, X11 화면을 컨테이너에 공유해주면 GUI를 띄울 수 있다.

**1. VM에서 Docker가 화면에 그릴 수 있도록 권한 허용**

```bash
xhost +local:docker
```

**2. PX4를 화면 공유 옵션과 함께 실행**

```bash
docker run --rm -it --network host \
  -e DISPLAY=$DISPLAY \
  -v /tmp/.X11-unix:/tmp/.X11-unix \
  -e PX4_HOME_LAT=36.4800 -e PX4_HOME_LON=127.2890 -e PX4_HOME_ALT=0.0 \
  jonasvautherin/px4-gazebo-headless
```

- `-e DISPLAY=$DISPLAY` : VM의 화면 주소를 컨테이너에 전달
- `-v /tmp/.X11-unix:/tmp/.X11-unix` : 화면 그리기 통로(소켓)를 컨테이너와 공유

**3. 컨테이너 안에서 GUI 클라이언트 실행**

새 터미널에서 실행 중인 컨테이너에 접속한다.

```bash
docker ps                          # 컨테이너 ID 확인
docker exec -it <컨테이너ID> bash
```

컨테이너 안에서:

```bash
gz sim -g
```

`-g`는 이미 돌고 있는 시뮬레이션 서버에 GUI만 따로 붙이는 옵션이다. Entity Tree에 `ground_plane`, `sunUTC`, `x500_0`(드론 모델)이 나타나고, 3D 뷰에 드론 모델이 렌더링되면 성공이다.

![Gazebo 3D GUI 실행 화면](gazebo_3d.png)
> `gz sim -g`로 GUI를 띄운 화면. Entity Tree에 `ground_plane`, `sunUTC`, `x500_0`이 로드되어 있고, 3D 뷰에 x500 드론 모델이 렌더링된 상태. 하단에 실시간 속도 80.92%가 표시된다. 배경이 회색인 건 렌더링 문제가 아니라 default world의 기본 지면 재질이 원래 무채색이라서 그렇다.

### 리소스 요구량

Gazebo 3D 렌더링은 물리 연산과 별개로 그래픽 렌더링 부하가 추가되기 때문에, QGC 연결만 할 때보다 훨씬 많은 자원이 필요했다.

| VM 스펙 | Gazebo GUI 실시간 속도 | 비고 |
| --- | --- | --- |
| 4코어 / 8GB | 약 74% | 화면이 눈에 띄게 끊기고 지직거림 |
| 8코어 / 16GB | 약 80~84% | 여전히 완벽히 매끈하진 않지만 캡처/시연 가능한 수준 |

VMware Settings → Display에서 **Accelerate 3D graphics**가 켜져 있는지도 확인해야 한다. 꺼져 있으면 GPU 가속 없이 소프트웨어 렌더링만으로 처리되어 훨씬 느려진다. Graphics memory를 늘릴 경우, VM 전체 Memory도 그에 맞춰 늘려야 한다는 경고가 뜰 수 있다(예: Graphics memory 8GB 설정 시 VM Memory가 최소 16GB는 되어야 함).

### 드론 두 대 동시 실행은 무리였음

편대 비행 시나리오를 염두에 두고 두 번째 PX4 인스턴스를 추가로 띄워봤다.

```bash
docker run --rm -it --network host \
  -e DISPLAY=$DISPLAY \
  -v /tmp/.X11-unix:/tmp/.X11-unix \
  -e PX4_HOME_LAT=36.4810 -e PX4_HOME_LON=127.2900 -e PX4_HOME_ALT=0.0 \
  -e PX4_INSTANCE=1 \
  -e PX4_SIM_MODEL=x500 \
  jonasvautherin/px4-gazebo-headless
```

두 번째 드론까지는 실행됐지만, 8코어/16GB로도 감당이 안 될 만큼 시스템 전체가 심하게 느려졌다. Gazebo GUI를 띄운 상태에서 드론을 여러 대 동시에 돌리는 건 이 VM 스펙으로는 현실적이지 않다고 판단해서, 편대 시나리오는 실제 시뮬레이션 대신 다이어그램으로 설명하는 쪽으로 방향을 잡았다.

---

## 참고: 트러블슈팅

- Docker 컨테이너가 여러 개 겹쳐서 실행되어 있으면(예전 `-p` 방식 컨테이너와 `--network host` 컨테이너가 동시에 남아있는 경우 등) Gazebo 인스턴스끼리 충돌하며 TIMEOUT을 유발한다. `docker ps -a`로 항상 확인하고 `docker rm -f $(docker ps -aq)`로 정리한다.
- QGC 프로세스가 창을 닫아도 완전히 안 죽고 포트를 계속 물고 있는 경우가 있었다. `pkill -9 -f QGroundControl`로 강제 종료하면 된다.
- QGC 설정이 꼬였다 싶으면 `rm -rf ~/.config/QGroundControl.org`로 초기화하고 다시 실행한다.
- VM 하드웨어 설정(Memory/Processors/Display)은 VM이 완전히 꺼진 상태(Powered off)에서만 바꿀 수 있다.
- 드론이 여러 대 필요한 시나리오라면, 이 VM 스펙 기준으로는 Gazebo GUI 없이 headless로만 돌리는 걸 권장한다.

---

작성: 김수영 · CCSC 2026 팀 TNT
