# 주행 연동 API 명세

낙상 의심 감지 서버(대시보드)와 주행 노드를 연결하기 위한 문서다. **주행 노드가 서버로 보낸다.** 서버가 주행 노드에 먼저 보내는 것은 없고, "무시 구역인가?"에 대한 답만 돌려준다.

```
주행 노드 ──(위치·상태, 감지 이벤트, 사진, 판단 결과)──▶ 서버 ──▶ 대시보드
         ◀──────────(무시 구역인지 여부)───────────────
```

## 1. 공통 규칙

| 항목 | 규칙 |
|---|---|
| 서버 주소 | `http://127.0.0.1:8000` (서버와 주행 노드가 같은 PC) |
| 좌표 | **map 좌표계, 미터**. AMCL이 쓰는 지도와 대시보드 지도가 같은 지도여야 한다 |
| 방향(yaw) | 라디안, map 좌표계 기준 (x축 0, 반시계 +) |
| 시간 | 보내지 않는다. 서버가 받은 시각을 기록한다 |
| 사진 | JPEG 또는 PNG. 다른 형식은 400 에러 |
| 실패 | 서버가 꺼져 있어도 순찰은 멈추지 않게 예외 처리한다 |

## 2. 연동 방법 두 가지

**A. ROS 패키지 사용 (추천)** — `ros/fall_patrol_bridge`

- 위치·상태: `status_bridge` 노드를 켜고 토픽만 발행
- 감지 이벤트: `ServerClient` 클래스 import해서 함수 호출
- 빌드와 예시 코드: [ros/README.md](../ros/README.md)

**B. HTTP 직접 호출** — 다른 언어나 다른 구조를 쓸 때. 아래 3, 4장의 명세대로 보낸다.

브라우저에서 `http://127.0.0.1:8000/docs`를 열면 모든 API를 직접 눌러서 시험할 수 있다.

## 3. 로봇 위치·상태

대시보드 지도의 로봇 위치, 시야각, 로봇 상태창, 실시간 로그에 쓰인다.

### A 방식 (ROS)

```bash
ros2 run fall_patrol_bridge status_bridge
```

- 위치·방향은 TF `map → base_footprint`에서 자동으로 읽어서 0.2초마다 보낸다 (AMCL이 켜져 있어야 함)
- 주행 노드는 **상태가 바뀔 때** 아래 토픽만 발행하면 된다

| 토픽 | 타입 | 값 |
|---|---|---|
| `/fall_patrol/state` | `std_msgs/String` | 아래 상태 중 하나 |
| `/fall_patrol/waypoint` | `std_msgs/Int32` | 지금 가는 순찰 지점 번호 (선택) |

### B 방식 (HTTP)

`PUT /status` — 0.2초(5Hz) 정도로 계속 보낸다. 5초 넘게 안 오면 대시보드에 "로봇 응답 없음"이 뜬다.

```json
{ "state": "patrol", "x": 1.25, "y": -0.40, "yaw": 1.57, "current_waypoint": 3 }
```

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `state` | string | O | 아래 상태 중 하나 |
| `x`, `y` | float | | map 좌표 (m) |
| `yaw` | float | | 방향 (rad) |
| `current_waypoint` | int | | 순찰 지점 번호 |

### 로봇 상태 값

| 값 | 대시보드 표시 | 언제 |
|---|---|---|
| `idle` | 대기 | 순찰 시작 전, 정지 |
| `patrol` | 순찰 중 | 순찰 경로 이동 중 |
| `suspect` | 의심 감지 | 누운 사람을 처음 발견한 순간 |
| `approach` | 접근 중 | 사람에게 다가가는 중 |
| `observe` | 관찰 중 | 멈춰서 움직임을 지켜보는 중 |
| `alert` | 알림 | 낙상 의심으로 판단한 직후 (잠깐) |

## 4. 사람 감지 흐름

**순서가 중요하다.** 사진은 3장: ① 처음 발견했을 때(멀리서), ② 접근해서 멈췄을 때, ③ 판단을 내릴 때.

```
1. 누운 사람 발견          상태 suspect, 사진 ① 찍어서 보관
2. 무시 구역 확인           GET  /ignore-zones/check
     ignored: true  →      사진 ① 버리고 상태 patrol, 순찰 계속 (서버에 아무것도 저장 안 됨)
     ignored: false →      상태 approach, 접근
3. 도착해서 멈춤            상태 observe, 사진 ②
                           POST /events (사진 ①②)        ← 이 순간부터 이 위치는 무시 구역
4. 관찰 (15~30초)
5. 판단                    사진 ③
                           POST /events/{id}/snapshots (사진 ③)
                           PATCH /events/{id} (결과)
                           결과가 confirmed면 상태 alert
6. 순찰 재개                상태 patrol
```

### 4-1. 무시 구역 확인 — `GET /ignore-zones/check?x=&y=`

```bash
curl "http://127.0.0.1:8000/ignore-zones/check?x=1.20&y=-0.50"
```
```json
{ "ignored": false, "event_id": null }
```

- `ignored: true`면 이미 처리 중인 위치다. 다가가지 않고 지나간다
- 판정 기준 (서버가 계산): 요청 좌표에서 **1m 이내**에, 아래 이벤트가 하나라도 있으면 무시
  - 낙상 의심·판단 불가·관찰 중·확인됨 상태이고 **10분 이내**
  - 오탐이고 **3분 이내**
  - 관리자가 "처리 완료"를 누른 이벤트는 제외 (다시 감지해야 하므로)
- **서버가 응답하지 않으면 `ignored: false`로 보고 접근한다.** 놓치는 것보다 한 번 더 보는 게 안전하다

### 4-2. 이벤트 생성 — `POST /events` (multipart/form-data)

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `x`, `y` | float | 권장 | 사건 위치 (map, m). 없으면 무시 구역 판정에 안 쓰인다 |
| `confidence` | float | | 인식 신뢰도 0~1 |
| `detected_snapshot` | 파일 | 권장 | 사진 ① |
| `observing_snapshot` | 파일 | 권장 | 사진 ② |

```bash
curl -X POST http://127.0.0.1:8000/events \
  -F x=1.20 -F y=-0.50 -F confidence=0.82 \
  -F "detected_snapshot=@detected.jpg;type=image/jpeg" \
  -F "observing_snapshot=@observing.jpg;type=image/jpeg"
```

응답(201)의 **`id`를 저장해 둔다.** 이후 호출에 쓴다.

```json
{ "id": 12, "status": "observing", "x": 1.2, "y": -0.5, "confidence": 0.82, "snapshot_url": "/snapshots/....jpg", ... }
```

### 4-3. 사진 ③ 추가 — `POST /events/{id}/snapshots` (multipart/form-data)

| 필드 | 값 |
|---|---|
| `kind` | `decision` (① `detected`, ② `observing`도 나중에 따로 보낼 수 있음) |
| `snapshot` | 사진 파일 |

```bash
curl -X POST http://127.0.0.1:8000/events/12/snapshots \
  -F kind=decision -F "snapshot=@decision.jpg;type=image/jpeg"
```

### 4-4. 판단 결과 — `PATCH /events/{id}` (JSON)

```bash
curl -X PATCH http://127.0.0.1:8000/events/12 \
  -H "Content-Type: application/json" -d '{"status": "confirmed"}'
```

| 결과 값 | 뜻 | 대시보드 |
|---|---|---|
| `confirmed` | 낙상 의심 (관찰 시간 동안 움직임 없음) | 빨간 알림 배너 |
| `uncertain` | 판단 불가 (가려짐, 일부만 보임, 애매함) | 노란 알림 배너, 관리자가 판단 |
| `false_alarm` | 오탐 (일어남, 걸어감, 사람이 아님) | 알림 없음 |

`acknowledged`(확인), `resolved`(처리 완료)는 관리자가 대시보드에서 누르는 값이다. 주행 노드는 보내지 않는다.

### 4-5. 같은 흐름을 ServerClient로 (A 방식)

```python
import cv2
from fall_patrol_bridge.server_client import ServerClient, ServerError

client = ServerClient()

def jpg(frame):
    return cv2.imencode(".jpg", frame)[1].tobytes(), "image/jpeg"

detected = jpg(frame)                          # 1. 발견 (사진 ①)
if client.is_ignored(x, y):                    # 2. 무시 구역 (서버 꺼짐 → False)
    return
try:
    event_id = client.create_event(x, y, confidence, detected, jpg(arrived_frame))  # 3.
    ...                                        # 4. 관찰
    client.add_snapshot(event_id, "decision", jpg(best_frame))                      # 5.
    client.set_result(event_id, "confirmed")
except ServerError as error:
    node.get_logger().error(f"서버 기록 실패: {error}")  # 순찰은 계속
```

## 5. 같이 정해야 할 것

| 항목 | 선택지 | 제안 |
|---|---|---|
| 이벤트 좌표 `x, y` | 사람 위치를 계산 / 로봇이 멈춘 위치로 대신 | 처음엔 **로봇 위치**로 시작 (무시 구역 반경이 1m라 충분) |
| 관찰 시간 | 몇 초 동안 안 움직이면 낙상 의심인지 | 15~30초 |
| 지도 | 새로 매핑하면 `.yaml` + `.pgm`을 넘겨준다 | 대시보드 지도도 같은 지도로 교체 (`convert_map --clean`) |
| 사진 형식 | JPEG / PNG | JPEG (용량 작음) |

## 6. 혼자 테스트하는 방법

1. 서버 실행: [../README.md](../README.md)
2. `http://127.0.0.1:8000/docs`에서 API를 하나씩 눌러보거나 위 curl 예시를 실행
3. `http://127.0.0.1:8000` 대시보드에서 배너, 지도, 로그가 바뀌는지 확인
4. 전체 흐름 예시는 `backend/scripts/fake_robot.py` (실제 로봇 대신 이 순서대로 서버를 부르는 스크립트)
