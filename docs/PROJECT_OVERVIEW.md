# Project Overview

이 문서는 네트워크 장애 감시 및 우회 라우팅 시뮬레이션의 역할, 데이터 흐름, UI/API, 검증 기준을 정리한다.

## 런타임 구조

- Python 표준 라이브러리만 사용한다.
- 각 역할은 독립 프로세스로 실행할 수 있다.
- 역할 간 통신은 TCP + 줄 단위 JSON 메시지를 사용한다.
- 기본 viewer와 Web UI runtime은 필요한 node role process를 supervisor로 자동 실행할 수 있다.

## 구현된 역할

- Host Simulator
  - 주기적으로 host 관측값을 갱신한다.
  - `CPU_SPIKE`, `SERVICE_DOWN`, `LATENCY_HIGH` fault를 수동 on/off 또는 duration 방식으로 주입할 수 있다.
  - Host는 raw observation만 제공하고 fault 판단은 Local Agent가 담당한다.
- Local Agent
  - Host 상태를 polling한다.
  - 정상 상태 변화는 `HOST_STATE_UPDATE` event로 만든다.
  - fault 조건을 감지하면 fault event를 생성한다.
  - 반복된 동일 fault는 중복 event 생성을 억제한다.
  - primary 실패 시 같은 `event_id`로 backup 경로를 시도한다.
- Relay R1 / R2
  - primary 경로의 hop-by-hop forwarding을 담당한다.
  - ACK 대기, retry, duplicate suppression, route mismatch 거부를 수행한다.
  - downstream 실패 근거를 allowlist schema의 `downstream_error`로 보존한다.
- Relay R1B / R2B
  - constrained backup 경로를 담당한다.
  - primary와 backup 사이의 임의 교차 forwarding은 허용하지 않는다.
- Monitor
  - event를 수신하고 중복을 제거한다.
  - host 상태 table, 최근 event 요약, ACK 결과, sink 처리 결과를 유지한다.
  - route summary, route trace, 장애 위치 추정 정보를 표시할 수 있도록 status detail에 포함한다.
  - reset 이후 중복 제외 누적 event 수를 `total_logged`로 기록한다.
- Controller/UI
  - node status를 수집한다.
  - start, pause, reset, kill, fault, ackdrop, delay 명령을 처리한다.
  - 전체 overview와 focused node monitor를 제공한다.

## 데이터 경로

- primary: `local-agent -> r1 -> r2 -> monitor`
- backup: `local-agent -> r1b -> r2b -> monitor`
- 금지된 교차 경로: `r1 -> r2b`, `r1b -> r2`

## 메시지와 상태

- `EVENT`: Host 상태 또는 fault를 나타내는 data-plane event
- `ACK`: hop-by-hop 수신 확인
- `CONTROL`: controller가 node에 전달하는 제어 명령
- `STATUS`: node가 자신의 상태와 detail을 보고하는 메시지
- `STATUS_REPORT`: shared control token을 포함해 controller에 전달되는 status wrapper

## UI와 API

- Terminal viewer
  - `python main.py`로 실행한다.
  - 비대화형 환경에서는 scripted scenario를 자동 실행한다.
  - `focus <node>`, `overview`, `focus all`로 관찰 대상을 전환한다.
- External controller client
  - `python main.py --controller --control-token <token>`로 실행한다.
  - viewer/controller와 같은 shared token이 필요하다.
- Web UI
  - `python -m web_ui.server --web-port 8080`로 실행한다.
  - TUI 출력을 파싱하지 않고 controller runtime state JSON을 읽어 렌더링한다.
  - 주요 endpoint는 `GET /`, `GET /api/state`, `POST /api/control`, `POST /api/power`다.

## 검증 상태

- `python -m unittest` 기준 121개 테스트가 통과한다.
- Python LSP error diagnostics 기준 error가 없다.
- `GIT_MASTER=1 git diff --check`가 통과한다.
- `python main.py --duration 8 --scripted`로 scripted scenario smoke를 확인했다.
- `python -m web_ui.server --web-port 18080 --control-port 19110 --duration 8`로 Web UI runtime smoke를 확인했다.

## 실행과 제어 참고

저장소 루트에서 `python main.py`를 실행하면 터미널 viewer와 필요한 역할 프로세스가 함께 시작된다. 비대화형 환경에서는 scripted scenario를 자동 실행한다. 브라우저 화면은 `python -m web_ui.server --web-port 8080`으로 시작하며, 기본 주소는 `http://127.0.0.1:8080`이다.

역할별 독립 실행에는 `python main.py --role <role>`을 사용한다. `role`에는 `controller`, `host`, `agent`, `relay-r1`, `relay-r2`, `relay-r1b`, `relay-r2b`, `monitor`가 들어간다. 독립 역할, 독립 Controller/UI, 외부 controller client는 기본적으로 동일한 shared control token이 필요하다. 의도적인 무인증 제어는 `--allow-unauthenticated-control`로 명시한다.

외부 controller client 연결 예시는 다음과 같다.

```bash
python main.py --controller --host 127.0.0.1 --port 9110 --control-token <token>
```

이미 실행 중인 runtime에 Web UI만 연결할 때는 supervisor를 끈다.

```bash
python -m web_ui.server --web-port 8080 --no-supervisor --control-token <token>
```

터미널 viewer에서 `focus <node>`, `overview`, `focus all`로 관찰 대상을 바꿀 수 있다. `<node>`는 `host`, `agent`, `r1`, `r1b`, `r2`, `r2b`, `monitor` 중 하나다. `focus`는 화면 전환이며 node에 CONTROL 메시지를 보내지 않는다.

| 명령 | 관찰 목적 |
|---|---|
| `start`, `pause`, `reset` | 전체 역할의 실행 상태 변경 |
| `start r1`, `pause r1`, `reset r2`, `kill monitor` | 특정 역할의 상태 변경 |
| `fault cpu on/off`, `fault service on/off`, `fault latency on/off` | Host fault 주입과 해제 |
| `ackdrop` | ACK 손실 실험 |
| `delay r1 1.5` 등 | Relay 처리 지연 실험 |
| `help`, `quit`, `exit` | 명령 확인과 종료 |

standalone role과 `--fixed-node-ports` Web UI 실행의 기본 포트는 Host `9101`, Agent `9102`, R1 `9103`, R2 `9104`, Monitor `9105`, R1B `9106`, R2B `9107`, Controller/UI `9110`이다. 일반 Web UI supervisor 실행은 역할 프로세스에 빈 포트를 동적으로 할당한다.

Web UI의 주요 endpoint는 `GET /`(화면), `GET /api/state`(상태 snapshot), `POST /api/control`(명령), `POST /api/power`(supervisor 역할 전원)이다. Web UI는 터미널 출력을 파싱하지 않는다.

현재 구현의 검증 명령은 `python -m unittest`와 `git diff --check`다.
