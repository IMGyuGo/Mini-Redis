# CLI Client Structure Guide

이 문서는 `src/mini_redis/cli/client.py`가 프로젝트 전체 구조 안에서 어디에 놓여 있고, 어떤 역할만 맡고 있는지를 프로젝트 구조와 비교해서 정리한 문서입니다.

## 1. 프로젝트 전체에서 CLI가 놓인 위치

```text
miniRedis/
|-- README.md
|-- img/
|-- data/
|-- docs/
|-- tests/
`-- src/
    `-- mini_redis/
        |-- cli_main.py
        |-- server_main.py
        |-- bootstrap.py
        |-- config.py
        |-- types.py
        |-- cli/
        |   |-- client.py
        |   `-- parser.py
        |-- protocol/
        |   `-- resp.py
        |-- network/
        |   |-- tcp_client.py
        |   |-- tcp_server.py
        |   `-- timing.py
        |-- commands/
        |   |-- manager.py
        |   |-- queue.py
        |   |-- catalog.py
        |   `-- handlers/
        |-- engine/
        |   `-- redis.py
        |-- storage/
        |   |-- manager.py
        |   |-- ttl.py
        |   |-- mongo_adapter.py
        |   |-- mongo_manager.py
        |   `-- benchmark.py
        |-- invalidation/
        |   `-- manager.py
        `-- persistence/
            |-- manager.py
            |-- aof.py
            |-- meta.py
            `-- rdb.py
```

핵심은 `cli/`가 프로젝트의 맨 앞단, 즉 사용자 입력을 처음 받는 계층이라는 점입니다.

## 2. CLI 폴더만 따로 보면

```text
src/mini_redis/cli/
|-- client.py   # 사용자와 직접 만나는 터미널 인터페이스
`-- parser.py   # 입력 문자열을 Command 형태로 변환
```

즉, `cli` 폴더는 아주 작지만 역할이 분명합니다.

- `client.py`: 입력 받고, 출력하고, 로컬 명령을 처리하고, 서버 요청을 보냄
- `parser.py`: 사용자가 친 문자열을 공통 명령 구조로 정리함

## 3. client.py가 프로젝트 안에서 연결되는 위치

`client.py`는 혼자 동작하지 않고, 아래 순서로 다른 모듈과 연결됩니다.

```text
user input
-> cli/client.py
-> cli/parser.py
-> protocol/resp.py
-> network/tcp_client.py
-> network/tcp_server.py
-> commands/manager.py
-> commands/handlers/*.py
-> engine/redis.py
-> storage / ttl / persistence / invalidation / mongo
```

여기서 중요한 점은 `client.py`가 직접 `Redis`를 호출하지 않는다는 것입니다.

## 4. client.py가 맡는 일

`src/mini_redis/cli/client.py`는 아래 일만 담당합니다.

- 프롬프트 출력
- 배너 출력
- `.help`, `.demo`, `.clear`, `.exit` 같은 로컬 메타 명령 처리
- 사용자가 입력한 일반 명령을 파싱기로 넘김
- TCP 클라이언트를 통해 서버에 명령 전송
- 서버 응답을 보기 좋게 출력
- `WATCH`, `LIVESET` 같은 CLI 보조 기능 제공

## 5. client.py가 맡지 않는 일

아래 책임은 `client.py`의 역할이 아닙니다.

- RESP 규칙 자체를 정의하는 일
- 소켓 서버를 여는 일
- 명령 라우팅을 결정하는 일
- 실제 key-value 저장을 수행하는 일
- TTL 만료를 계산하는 일
- AOF / snapshot 저장을 수행하는 일
- invalidation tag 인덱스를 관리하는 일

이런 책임은 각각 다른 폴더로 분리되어 있습니다.

## 6. 프로젝트 구조와 비교했을 때의 정확한 경계

### CLI 계층

- 파일: `cli/client.py`, `cli/parser.py`
- 관심사: 사용자 경험, 입력, 출력

### Protocol 계층

- 파일: `protocol/resp.py`
- 관심사: RESP 인코딩/디코딩

### Network 계층

- 파일: `network/tcp_client.py`, `network/tcp_server.py`, `network/timing.py`
- 관심사: TCP 전송, 요청/응답 전달

### Command 계층

- 파일: `commands/manager.py`, `commands/queue.py`, `commands/handlers/`
- 관심사: 명령 진입점 통일, FIFO 실행, 명령별 handler 분기

### Engine 계층

- 파일: `engine/redis.py`
- 관심사: 내부 매니저 오케스트레이션

### Data/Support 계층

- 파일: `storage/*`, `persistence/*`, `invalidation/*`
- 관심사: 실제 저장, TTL, 복구, invalidation, 외부 연동

## 7. client.py를 읽을 때 보면 좋은 포인트

`client.py`를 읽을 때는 아래 순서로 보면 이해가 쉽습니다.

1. `run()`
2. `_handle_local_input()`
3. `_run_server_command()`
4. `_run_watch_mode()`
5. `_run_liveset_mode()`
6. `_render_response()`

이 순서대로 보면 "입력 -> 분기 -> 서버 전송 -> 응답 출력" 구조가 자연스럽게 보입니다.

## 8. 한 줄 요약

`client.py`는 Mini Redis의 시작점이지만, 실제 Redis 동작을 처리하는 곳은 아닙니다.  
사용자 입력을 받아서 서버 구조로 안전하게 넘기고, 결과를 다시 사용자에게 보여주는 "가장 바깥쪽 인터페이스 계층"입니다.
