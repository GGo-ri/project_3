# CLAUDE.md

## 사용자와 대화

- **답변은 반드시 한국어로 한다.** 코드, 명령어, 경로, API 이름만 원문 그대로 둔다. 영어로 답하면 매우 불쾌해한다
- 커밋에 `Co-Authored-By: Claude` 같은 Claude 표시를 넣지 않는다
- 사용자는 국비 교육 팀프로젝트(5명, 3주)에서 **서버·DB·웹 대시보드 담당**이다. 로봇 주행, 인식(YOLO) 모델은 다른 팀원 담당이고 HTTP API로만 연결한다

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

## 지금 할 일: 실제 로봇 카메라 연동

1. 로봇에서 `turtlebot3_bringup`과 카메라(`v4l2_camera`)를 띄운다. 이 경우 토픽은 `/camera/image_raw`가 아니라 **`/image_raw`**다
2. PC에서 `ros2 topic hz /image_raw`로 영상이 들어오는지 확인 (`ROS_DOMAIN_ID`가 로봇과 같아야 함)
3. `sudo apt install -y ros-humble-web-video-server` 후 `ros2 run web_video_server web_video_server`
4. `http://localhost:8080`에서 토픽 영상이 단독으로 나오는지 확인
5. `backend/static/app.js`의 `CONFIG.videoTopic`을 `/image_raw`로 바꾼다
6. 와이파이로 끊기면 압축 영상(`type=ros_compressed`) 사용을 검토한다

## 실제 로봇에 status_bridge 붙이기

`ros/README.md` 순서대로 빌드하고, 로봇 bringup + 위치 추정(AMCL)이 떠 있는 상태에서 `ros2 run fall_patrol_bridge status_bridge`. 대시보드 지도에서 로봇이 움직이면 성공. 시뮬레이션(Gazebo turtlebot3_world)에서는 2026-10-22에 확인함. 주행 노드는 `/fall_patrol/state`(String)로 상태를 발행하면 된다.

## 남은 일

- 이벤트 좌표를 어떻게 구할지 로봇 파트와 합의 (지금은 nullable)
- 스피커(Adafruit USB 미니 스피커)와 마이크(Coms BT042 USB) 구매 예정. 음성 안내 + Whisper(PC에서 실행) 키워드 판단은 로봇·인식 쪽, 서버는 음성 응답 텍스트와 녹음 파일 저장·표시 담당
- 사진 저장 후 에러가 나면 파일만 남는 문제: 에러 시 저장한 파일을 지우도록 처리
- 다른 파트에 넘길 API 명세 정리
- 나중에: MCU(LED·부저) 연동, 텔레그램 알림
