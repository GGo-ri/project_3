# 낙상 의심 감지 순찰 로봇 — 서버 · 대시보드

TurtleBot3가 순찰하다가 낙상이 의심되는 사람을 발견하면 서버에 기록하고, 관리자가 웹 대시보드에서 확인·처리하는 시스템의 서버와 대시보드다.

- 서버: FastAPI + SQLite (`backend/app`)
- 대시보드: HTML/JS, 서버가 `/`에서 같이 서빙 (`backend/static`)
- 테스트 도구: 더미 데이터, 가짜 로봇 (`backend/scripts`)
- ROS 연동: 로봇 위치 전송 노드, 서버 호출 클라이언트 (`ros/fall_patrol_bridge`) → [ros/README.md](ros/README.md)
- **주행 연동 가이드: [docs/API.md](docs/API.md)** ← 주행 담당은 이것부터 읽는다
- 설계 문서: [GUIDE.md](GUIDE.md)

## 리눅스(Ubuntu)에서 실행

### 처음 한 번

```bash
sudo apt install -y python3-venv
cd backend
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
```

### 서버 실행

```bash
cd backend
.venv/bin/python -m uvicorn app.main:app --host 0.0.0.0 --port 8000
```

브라우저에서 접속한다.

- 같은 PC: `http://localhost:8000`
- 같은 와이파이의 다른 기기: `http://<이 PC의 IP>:8000` (IP 확인: `hostname -I`)
- 다른 기기에서 안 열리면 방화벽을 연다: `sudo ufw allow 8000`

### 가짜 로봇으로 테스트 (다른 터미널)

```bash
cd backend
.venv/bin/python -m scripts.fake_robot --observe 8
```

실제 로봇 없이 순찰, 사람 감지, 사진 3장 업로드, 판단 결과 전송까지 실제 순서대로 서버를 호출한다.

### 더미 데이터 넣기 (선택)

서버를 끈 상태에서 실행한다. **기존 DB를 지우고** 더미 이벤트 20건과 사진, 처리 이력을 새로 만든다.

```bash
cd backend
.venv/bin/python -m scripts.seed
```

### 데이터 비우기

서버를 끄고 `backend/data` 폴더를 지운다. 서버를 다시 켜면 빈 DB가 새로 만들어진다.

```bash
rm -rf backend/data
```

## 실제 로봇·카메라를 붙일 때 바꿀 곳

| 항목 | 위치 | 내용 |
|---|---|---|
| 지도 | `backend/maps/` | SLAM 지도로 교체: `.venv/bin/python -m scripts.convert_map <경로>/map.yaml --clean` |
| 지도 회전 | `backend/static/app.js` `CONFIG.mapRotation` | 지도를 눕혀서 보여줄지 (90 또는 0) |
| 영상 토픽 | `backend/static/app.js` `CONFIG.videoTopic` | 실제 카메라 토픽 또는 YOLO 결과 이미지 토픽 |
| 영상 포트 | `backend/static/app.js` `CONFIG.videoPort` | web_video_server 포트 (기본 8080) |
| 영상 타입 | `backend/static/app.js` `CONFIG.videoType` | `ros_compressed`(와이파이 권장) 또는 `mjpeg`(원본 토픽, 대역폭 큼) |
| 카메라 화각 | `backend/static/app.js` `CONFIG.cameraFovDeg` | 실제 카메라 화각 |
| 무시 구역 | `backend/app/config.py` | 반경, 유지 시간 |
| 로봇 노드 | ROS 쪽 | 서버 주소는 `http://127.0.0.1:8000` |

로봇 쪽은 `ros/fall_patrol_bridge`를 쓴다. 위치 전송은 `status_bridge` 노드, 감지 이벤트 기록은 `ServerClient`로 한다. 자세한 사용법은 [ros/README.md](ros/README.md), API 순서는 [GUIDE.md](GUIDE.md)의 "감지 흐름" 참고.
