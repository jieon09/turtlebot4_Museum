# TurtleBot4 Museum Security System

웹캠과 TurtleBot4 AMR을 활용한 **박물관 도난 방지 스마트 보안 시스템**입니다.
고정 웹캠과 TurtleBot4의 OAK-D 카메라로 전시품과 도둑 객체를 탐지하고, 이상 상황이 발생하면 관리자 웹 대시보드에 알림을 표시하며 AMR이 도둑 위치로 출동해 추적합니다.

---

## 프로젝트 개요

기존 박물관 보안 시스템은 인력 중심 감시에 의존하기 때문에 사각지대나 야간 감시 공백이 발생할 수 있습니다.
본 프로젝트는 **YOLO 기반 객체 탐지**, **Flask 모니터링 서버**, **SQLite 이벤트 로그**, **ROS2 기반 TurtleBot4 자율주행**을 연동하여 도난 상황을 자동으로 감지하고 대응하는 시스템을 목표로 합니다.

### 주요 기능

* 웹캠 기반 전시품 실시간 탐지
* TurtleBot4 OAK-D 카메라 기반 전시품 및 도둑 탐지
* 전시품 미탐지 시 도난 이벤트 발생
* 관리자 웹 대시보드에서 영상, 로그, 로봇 상태 확인
* Alert Popup을 통한 위험 상황 알림
* ROS2 / Nav2 기반 TurtleBot4 순찰
* 도둑 위치 계산 후 AMR 추적 모드 전환
* robot2, robot8 간 중복 추적 방지를 위한 상태 공유

---

## 시스템 구성

```text
turtlebot4_Museum/
├── README.md
└── C3_museum_src/
    ├── AMR/
    ├── Detection/
    └── System Monitor/
```

### 1. System Monitor

관리자 웹 대시보드와 서버를 담당합니다.

* Flask 기반 웹 서버
* 관리자 로그인 및 세션 관리
* 전시품 등록, 수정, 삭제
* 실시간 영상 스트리밍 표시
* 도난 이벤트 로그 저장
* Alert Popup 출력
* 보안 모드 ON/OFF 관리

### 2. Detection

YOLO 기반 객체 탐지를 담당합니다.

* 웹캠 영상 기반 전시품 탐지
* TurtleBot4 OAK-D RGB / Depth 영상 기반 객체 탐지
* DB에 등록된 전시품 목록과 현재 탐지 결과 비교
* 전시품 미검출 시 missing list 생성
* 도둑 탐지 시 Depth와 TF를 이용한 map 좌표 계산
* 처리된 영상을 서버로 업로드

### 3. AMR

TurtleBot4 자율주행과 추적을 담당합니다.

* ROS2 기반 TurtleBot4 제어
* Nav2 기반 waypoint 순찰
* `/arrive_position` 토픽을 통한 도착 위치 알림
* `/thief_position` 토픽 수신 후 추적 모드 전환
* 도둑 위치 기준 일정 거리 유지 목표점 계산
* 상황 종료 신호 수신 시 추적 종료

---

## 기술 스택

| 구분               | 사용 기술                 |
| ---------------- | --------------------- |
| OS               | Ubuntu 22.04          |
| Robot Middleware | ROS2 Humble           |
| Robot            | TurtleBot4            |
| Navigation       | Nav2, AMCL, SLAM      |
| Camera           | USB Webcam, OAK-D Pro |
| Detection        | YOLOv8, OpenCV        |
| Web Server       | Flask                 |
| Database         | SQLite3               |
| Language         | Python3               |
| Visualization    | Web Dashboard, RViz2  |

---

## 시스템 동작 흐름

1. 관리자가 Flask 웹 서버에 로그인합니다.
2. 보안 모드가 활성화되면 웹캠 YOLO와 TurtleBot4 YOLO가 객체 탐지를 시작합니다.
3. YOLO는 현재 영상에서 탐지된 전시품 목록과 DB에 등록된 전시품 목록을 비교합니다.
4. 등록된 전시품이 일정 시간 이상 탐지되지 않으면 도난 상황으로 판단합니다.
5. 서버는 이벤트 로그를 저장하고 관리자 화면에 Alert Popup을 표시합니다.
6. TurtleBot4는 순찰 중 특정 전시품 위치에 도착하면 `/arrive_position` 토픽을 발행합니다.
7. 도둑이 탐지되면 OAK-D Depth와 TF 변환을 이용해 도둑의 map 좌표를 계산합니다.
8. AMR은 `/thief_position`을 수신하고 추적 모드로 전환합니다.
9. 도둑 위치가 갱신되면 Nav2 Goal을 반복 갱신하며 추적합니다.
10. 관리자가 상황 종료를 확인하면 `/situation_end` 토픽을 통해 추적을 종료합니다.

---

## 주요 ROS2 Topic

| Topic                         | Message Type                              | 설명                 |
| ----------------------------- | ----------------------------------------- | ------------------ |
| `/api/security_status`        | `std_msgs/Bool`                           | 보안 모드 ON/OFF 상태    |
| `/robot2/start_patrol_signal` | `std_msgs/Bool`                           | robot2 순찰 시작 신호    |
| `/robot8/start_patrol_signal` | `std_msgs/Bool`                           | robot8 순찰 시작 신호    |
| `/robot2/arrive_position`     | `std_msgs/Int32`                          | robot2 특정 위치 도착 신호 |
| `/robot8/arrive_position`     | `std_msgs/Int32`                          | robot8 특정 위치 도착 신호 |
| `/robot2/thief_position`      | `geometry_msgs/PointStamped`              | robot2 기준 도둑 위치 좌표 |
| `/robot8/thief_position`      | `geometry_msgs/PointStamped`              | robot8 기준 도둑 위치 좌표 |
| `/robot2/amcl_pose`           | `geometry_msgs/PoseWithCovarianceStamped` | robot2 현재 위치       |
| `/robot8/amcl_pose`           | `geometry_msgs/PoseWithCovarianceStamped` | robot8 현재 위치       |
| `/situation_end`              | `std_msgs/Bool`                           | 상황 종료 신호           |
| `/robot2/catch_thief_2`       | `std_msgs/Bool`                           | robot2 도둑 탐지 상태    |
| `/robot8/catch_thief_8`       | `std_msgs/Bool`                           | robot8 도둑 탐지 상태    |

---

## 실행 전 설정

### 1. ROS2 환경 설정

```bash
source /opt/ros/humble/setup.bash
```

워크스페이스를 사용하는 경우:

```bash
source ~/rokey_ws/install/setup.bash
```

### 2. Python 패키지 설치

```bash
pip install -r requirements.txt
```

### 3. 서버 주소 수정

Detection 코드 내부의 서버 주소를 현재 Host PC IP에 맞게 수정합니다.

```python
SERVER = "http://<HOST_PC_IP>:5000"
```

### 4. YOLO 모델 경로 수정

YOLO 모델 경로를 현재 PC 환경에 맞게 수정합니다.

```python
YOLO("/path/to/best.pt")
```

---

## 실행 방법

### 1. System Monitor 실행

```bash
cd C3_museum_src/System\ Monitor
python3 app.py
```

### 2. WebCam Detection 실행

```bash
cd C3_museum_src/Detection
python3 yolo_tt_result2_web.py
```

### 3. TurtleBot4 Detection 실행

robot2:

```bash
cd C3_museum_src/Detection
python3 yolo_tt_result2.py
```

robot8:

```bash
cd C3_museum_src/Detection
python3 yolo_tt_result8.py
```

### 4. AMR 실행

```bash
cd C3_museum_src/AMR
python3 real_final_2.py
```

> 파일명과 경로는 실제 PC별 배치 구조에 맞게 수정하여 실행합니다.

---

## 결과 화면

YOLO는 전시품과 도둑 객체를 실시간으로 탐지하고, 탐지 결과를 바운딩 박스로 표시합니다.

```text
hand
frog
bubble
thief
```

관리자 웹 대시보드에서는 다음 정보를 확인할 수 있습니다.

* 웹캠 영상
* TurtleBot4 카메라 영상
* AMR 상태
* 보안 모드 상태
* 도난 이벤트 로그
* 위험 상황 Alert Popup

---

## 주요 구현 내용

### 전시품 미검출 판단

YOLO 탐지 결과와 DB에 등록된 전시품 목록을 비교하여, 등록된 전시품이 현재 영상에서 탐지되지 않으면 missing list에 추가합니다.
missing list가 존재하면 서버에 이벤트를 전송하고, 관리자 화면에 알림을 표시합니다.

### 도둑 위치 계산

도둑 객체가 탐지되면 Bounding Box 중심점과 Depth 값을 이용해 카메라 기준 3D 좌표를 계산합니다.
이후 TF 변환을 통해 map 좌표계 기준 도둑 위치를 계산하고 `/thief_position` 토픽으로 발행합니다.

### AMR 추적

AMR은 도둑 위치와 자신의 현재 위치를 기준으로 방향 벡터를 계산합니다.
도둑에게 직접 충돌하지 않도록 일정 거리 떨어진 지점을 Nav2 Goal로 설정하고, 도둑 위치가 변경되면 Goal을 주기적으로 갱신합니다.

### 로봇 협동

robot2와 robot8은 `catch_thief` 상태 토픽을 공유하여 동일한 도둑 좌표를 중복 계산하지 않도록 제어합니다.

---

## 테스트 시나리오

1. 관리자가 웹 대시보드에 로그인합니다.
2. 보안 모드를 활성화합니다.
3. 웹캠과 TurtleBot4 카메라에서 전시품을 탐지합니다.
4. 전시품이 사라지면 도난 이벤트가 발생합니다.
5. 서버는 Alert Popup과 이벤트 로그를 출력합니다.
6. 도둑이 탐지되면 AMR이 추적 모드로 전환됩니다.
7. 관리자가 상황 종료를 누르면 AMR은 추적을 종료합니다.

---

## 프로젝트 의의

본 프로젝트는 단순 CCTV 감시를 넘어, 객체 탐지와 자율 이동 로봇을 결합하여 이상 상황을 자동 인지하고 실제 공간에서 대응하는 스마트 보안 시스템을 구현했다는 점에 의미가 있습니다.
웹 대시보드, YOLO 탐지, ROS2 Navigation, 다중 AMR 협동을 하나의 흐름으로 통합하여 실제 박물관 보안 환경에 적용 가능한 구조를 설계했습니다.
