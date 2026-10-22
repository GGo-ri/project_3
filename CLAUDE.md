# CLAUDE.md

## 사용자와 대화

- **답변은 반드시 한국어로 한다.** 코드, 명령어, 경로, API 이름만 원문 그대로 둔다. 영어로 답하면 매우 불쾌해한다
- 커밋에 `Co-Authored-By: Claude` 같은 Claude 표시를 넣지 않는다
- 사용자는 국비 교육 팀프로젝트(5명, 3주)에서 **서버·DB·웹 대시보드 담당**이다. 로봇 주행, 인식(YOLO) 모델은 다른 팀원 담당이고 HTTP API로만 연결한다
- 사용자는 ROS 개념(AMCL, TF 등)을 깊이 알지는 않는다. 명령어를 줄 때는 무엇을 하는 건지 한 줄씩 쉽게 설명한다
- 서버·가짜 로봇 같은 프로세스는 Claude가 백그라운드로 띄우고 끄는 걸 선호한다. 사용자는 브라우저만 본다
- 노트북(Windows)과 이 리눅스 PC 두 곳에서 작업한다. **작업이 끝나면 커밋하고 `git push`까지 해야** 노트북 쪽 Claude가 이어받을 수 있다

## 프로젝트

TurtleBot3가 실내를 순찰하다가 낙상이 의심되는 사람을 발견하면 접근해서 관찰하고, 결과를 서버에 기록한다. 관리자는 웹 대시보드에서 확인·처리한다. 화면과 문서에서는 "낙상"이 아니라 "낙상 의심"이라고 쓴다.

```
터틀봇 --영상--> PC: ROS 노드(주행, YOLO) --HTTP--> FastAPI 서버 --> SQLite
                 PC: web_video_server(8080) --영상--> 대시보드 (서버가 / 에서 서빙)
```

- 서버와 ROS/YOLO는 **같은 리눅스 PC**에서 돈다. 로봇 노드는 `http://127.0.0.1:8000`으로 서버를 부른다
- 서버에 ROS 의존성을 넣지 않는다
- 상세 설계(감지 흐름, 상태, 무시 구역, 데이터 모델, API)는 [GUIDE.md](GUIDE.md), 실행 방법은 [README.md](README.md)

## 구조

```
backend/
  app/            FastAPI 서버 (main, config, database, models, schemas, activity, routers/)
  static/         대시보드 (index.html, app.js, style.css) — 프레임워크 없음
  maps/           대시보드용 지도 (map.png, map.yaml) + 가짜 로봇 경로 (patrol.json)
  scripts/        seed(더미 데이터), fake_robot(가짜 로봇), convert_map(SLAM 지도 변환), map_cleaner 등
  data/           app.db + snapshots/ (git 제외, 서버가 자동 생성)
ros/
  fall_patrol_bridge/   ROS 2 패키지: status_bridge 노드(TF map→base_footprint를 PUT /status로 전송),
                        ServerClient(주행·인식 노드가 import해서 서버 호출). 사용법은 ros/README.md
```

## 실행 (리눅스)

```bash
cd backend
python3 -m venv .venv && .venv/bin/pip install -r requirements.txt   # 처음 한 번
.venv/bin/python -m uvicorn app.main:app --host 0.0.0.0 --port 8000  # 서버
.venv/bin/python -m scripts.fake_robot --observe 8                   # 가짜 로봇 (다른 터미널)
.venv/bin/python -m scripts.seed                                     # 더미 데이터 (DB를 지우고 새로 만든다)
```

- 서버와 가짜 로봇은 백그라운드로 띄우고, 사용자는 브라우저(`http://localhost:8000`)만 본다
- 데이터를 비울 때는 서버를 끄고 `backend/data`를 지운다

## 정해진 설계 (바꾸기 전에 사용자와 상의)

- **감지 흐름**: 누운 사람 발견(사진 ① 보관) → `GET /ignore-zones/check` → 무시면 사진 버리고 순찰, 아니면 접근 → 도착해서 사진 ② → `POST /events`(사진 ①②) → 관찰 → 사진 ③ `POST /events/{id}/snapshots` → `PATCH /events/{id}` 결과
- **이벤트 상태**: observing → confirmed / uncertain / false_alarm → acknowledged → resolved
- **무시 구역**: 별도 테이블 없이 이벤트로 계산. 1m 이내, 10분(오탐 3분) 이내, resolved가 아니면 무시
- **사진**: 파일은 `data/snapshots/`, DB에는 `event_snapshots`(kind: detected, observing, decision)로 기록
- **로그**: 서버가 API 호출을 받을 때 `activity_logs`에 자동 기록. 다른 파트가 따로 보내지 않는다
- **대시보드 배치**: 위쪽은 카메라 | 지도(90° 회전) + 로봇 상태, 아래쪽은 이벤트 | 실시간 로그. 위아래 모두 정중앙 기준 반반. 다크 모던 테마. 사용자가 여러 번 조정해서 확정한 배치라 함부로 바꾸지 않는다
- 시간은 UTC로 저장하고 `Z`를 붙여 응답, 화면에서 KST로 표시
- Python 3.10 기준 (ROS2 Humble). 3.11 이상 문법을 쓰지 않는다

## 코드 규칙

- 타입힌트를 쓰고 주석은 최소화한다
- 오류는 `HTTPException`
- 상태값 enum은 `app/models.py` 한 곳에서만 정의한다
- `create_all`만 쓴다. 모델을 바꾸면 `backend/data`를 지우고 다시 만든다

## 실제 로봇 카메라 연동 (완료, 2026-10-22)

- 로봇: `v4l2_camera_node` + `turtlebot3_bringup`, 토픽은 **`/image_raw`**
- PC: `web_video_server` (8080). 대시보드 `CONFIG.videoTopic = "/image_raw"`, `CONFIG.videoType = "ros_compressed"`
- `localhost:8080`에서 원본(`/image_raw`) 스트림을 직접 열지 않는다. 와이파이가 꽉 차서 영상이 멈춘다
- 실제 실행 명령, 로봇 주소, ROS_DOMAIN_ID(33)는 [실행방법.txt](실행방법.txt)

## 리눅스 PC에서 할 일 (2026-10-22 기준, 위에서부터 순서대로)

### 1. ZIP 폴더 → git 폴더로 옮기기
이전에는 GitHub에서 ZIP으로 받은 `~/project_3-main`을 썼다. git과 연결이 안 돼서 push가 안 됐고, 작업을 `GGo-ri/123` 저장소에 웹 업로드로 올렸다(내용은 project_3에 이미 합침, 사용자가 삭제 예정).

```bash
pkill -f "uvicorn app.main:app"                          # 예전 폴더에서 켜진 서버 끄기
cd ~ && git clone https://github.com/GGo-ri/project_3.git
cd ~/project_3/backend && python3 -m venv .venv && .venv/bin/pip install -r requirements.txt
cp -r ~/project_3-main/backend/data ~/project_3/backend/ # 기존 DB·사진 유지 (선택)
```
- `실행방법.txt`의 경로 `~/project_3-main`을 `~/project_3`로 바꿔서 실행한다
- 카메라·대시보드가 예전처럼 나오면 옮기기 완료. 확인되면 `~/project_3-main`은 지워도 된다 (사용자에게 묻고)

### 2. status_bridge 빌드
```bash
mkdir -p ~/fall_patrol_ws/src && cp -r ~/project_3/ros/fall_patrol_bridge ~/fall_patrol_ws/src/
cd ~/fall_patrol_ws && colcon build --packages-select fall_patrol_bridge && source install/setup.bash
```

### 3. 실제 로봇 위치가 대시보드에 뜨는지 확인
위치 추정(AMCL)은 **주행 담당 영역**이다. 우리 `status_bridge`는 AMCL 결과(TF `map → base_footprint`)를 읽어서 서버로 넘기기만 한다.

1. 로봇: bringup + 카메라 (`실행방법.txt` [1])
2. PC: 서버 + web_video_server (`실행방법.txt` [2])
3. PC: AMCL만 켜기 (주행 없이 위치만 확인할 때)
   ```bash
   export TURTLEBOT3_MODEL=waffle_pi ROS_DOMAIN_ID=33
   ros2 launch nav2_bringup localization_launch.py map:=<지도>/map.yaml params_file:=$(ros2 pkg prefix turtlebot3_navigation2)/share/turtlebot3_navigation2/param/waffle_pi.yaml
   ros2 run rviz2 rviz2 -d $(ros2 pkg prefix nav2_bringup)/share/nav2_bringup/rviz/nav2_default_view.rviz
   ```
   주행까지 할 땐 `ros2 launch turtlebot3_navigation2 navigation2.launch.py map:=...` (AMCL 포함)
4. RViz **2D Pose Estimate**로 로봇 위치를 찍는다. 라이다 점이 지도 벽과 겹치면 성공
5. `ros2 run fall_patrol_bridge status_bridge` → 대시보드 지도에 RViz와 같은 위치로 로봇이 뜨면 성공

- AMCL 지도와 **대시보드 지도(`backend/maps/`)가 같은 지도**여야 위치가 맞는다. 지금 대시보드 지도는 `my-car-astar`의 실제 미로 지도(`maze_map`)를 `--clean`으로 정리한 것
- `status_bridge`가 "map 좌표의 로봇 위치를 아직 모름"만 띄우면 AMCL이 안 켜졌거나 2D Pose Estimate를 안 찍은 것
- 노트북 WSL에서는 `turtlebot3_ws`의 Nav2 파라미터 플러그인 이름 형식(`::` vs `/`) 문제로 planner_server가 안 켜졌다. 같은 에러가 나면 3번(AMCL만)으로 먼저 확인
- 시뮬레이션(Gazebo turtlebot3_world)에서는 status_bridge, 상태 토픽, ServerClient 이벤트 흐름 모두 2026-10-22에 확인함

### 4. 새 지도가 오면 대시보드 지도 교체
주행 담당이 새로 매핑하면 원본은 로봇(AMCL)용으로 그대로 두고, 대시보드용만 변환한다. pgm을 그냥 덮어쓰면 안 된다 (png 변환과 원점 보정이 필요).
```bash
cd ~/project_3/backend && .venv/bin/python -m scripts.convert_map <새 지도>/map.yaml --clean
```
- 서버 재시작 없이 대시보드 새로고침(F5)
- 세로로 긴 지도면 `static/app.js`의 `CONFIG.mapRotation: 90`, 가로로 길면 `0`
- 위치가 맞는 걸 확인하고 `backend/maps`를 커밋·푸시

## 주행 담당과의 연동 (진행 중)

- 연동 문서 [docs/API.md](docs/API.md)를 전달함 (ROS 방식만: `status_bridge` + `/fall_patrol/state` 토픽 + `ServerClient` 함수 4개)
- 구조: **주행 노드가 감지 흐름 전체를 지휘**한다. 인식은 주행에게만 결과를 주고, 서버는 주행 노드만 부른다
- 상태 토픽 값: `idle`, `patrol`, `suspect`, `approach`, `observe`, `alert` (소문자 정확히, 바뀔 때 한 번씩)
- 사진은 카메라 토픽의 최신 화면 한 장을 꺼내 JPEG로 보낸다. 인식이 그린 화면 토픽이 생기면 그걸 쓰는 게 좋다
- 사건 좌표는 **사람이 있는 위치**로 보내 달라고 했다. 로봇 위치를 그대로 쓰면 보는 방향마다 좌표가 달라져서 무시 구역(1m)이 안 맞고 같은 사람을 여러 번 기록한다. 간단히는 "로봇이 바라보는 방향으로 사람까지 거리만큼 앞" (docs/API.md "사람 좌표"). 서버는 받은 좌표를 그대로 저장·표시·판정한다
- 아직 정할 것: 사람 좌표 계산 방식, 관찰 시간(15~30초), 접근 거리(1~1.5m), 새 지도 공유
- 주행 담당 GitHub 계정을 project_3 Collaborators에 추가할 예정. 주행 코드도 같은 저장소에 올리기로

## 남은 일

- 서버: 2분 넘게 "관찰 중"인 이벤트를 자동으로 "판단 불가"로 (주행 노드가 결과 전에 꺼질 때 대비)
- 서버: 사진 저장 후 에러가 나면 파일만 남는 문제 → 에러 시 저장한 파일 삭제
- 대시보드: 낙상 의심 알림 소리
- 스피커(로봇사이언스몰 Adafruit 미니 USB 스피커 3369)와 마이크(디바이스마트 Coms BT042 USB 마이크) 구매 예정, 둘 다 라즈베리파이 빈 USB 포트 2개에 꽂음. 음성 안내 + Whisper(PC에서 실행) 키워드 판단은 로봇·인식 쪽, 서버는 음성 응답 텍스트와 녹음 파일 저장·대시보드 표시 담당
- 발표용 통계 (감지→확인 평균 시간, 오탐률)
- 나중에: MCU(LED·부저) 연동, 텔레그램 알림
