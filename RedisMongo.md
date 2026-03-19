# Redis와 MongoDB 비교, 그리고 이 프로젝트에서의 사용 방식

이 문서는 현재 Mini Redis 프로젝트를 기준으로, Redis와 MongoDB가 무엇이 다른지, 이 프로젝트 안에서는 각각 어떤 역할을 하는지, 그리고 실제 운영에서는 보통 어떻게 사용하는지를 이해하기 쉽게 정리한 문서입니다.

## 1. 가장 먼저 한 줄로 이해하기

- Redis는 보통 **빠른 메모리 기반 저장소**로 사용합니다.
- MongoDB는 보통 **문서를 오래 저장하는 데이터베이스**로 사용합니다.

즉 아주 단순하게 보면:

- Redis: 빠름
- MongoDB: 오래 보관하고 구조화해서 저장하기 좋음

물론 실제로는 더 복잡하지만, 처음에는 이렇게 이해해도 충분합니다.

## 2. Redis와 MongoDB의 성격 차이

### Redis

Redis는 기본적으로 메모리 중심 저장소입니다.

주요 특징:

- 매우 빠른 읽기/쓰기
- 캐시로 많이 사용됨
- TTL이 매우 강력함
- 단순 key-value 접근이 쉬움
- 세션, 토큰, 카운터, 랭킹 같은 용도에 잘 맞음
- 데이터 타입이 다양함

쉽게 말하면:

"빨리 넣고, 빨리 꺼내고, 빨리 지워야 하는 데이터"에 강합니다.

### MongoDB

MongoDB는 문서(document) 기반 데이터베이스입니다.

주요 특징:

- 디스크 기반 장기 저장에 적합
- JSON 비슷한 구조로 데이터 저장 가능
- 구조가 비교적 유연함
- 복잡한 필드를 가진 데이터를 저장하기 좋음
- 애플리케이션의 본 데이터 저장소로 자주 사용됨

쉽게 말하면:

"조금 더 구조적인 데이터를 오래 보관하는 저장소"에 잘 맞습니다.

## 3. 비유로 이해하기

비유하면 아래와 같습니다.

- Redis: 책상 위 메모장
- MongoDB: 문서 보관함

책상 위 메모장은 꺼내 쓰기 빠르지만, 영구 보관용은 아닙니다.  
문서 보관함은 꺼내 쓰는 속도는 상대적으로 덜 중요하지만, 데이터를 오래 안정적으로 관리하기 좋습니다.

## 4. 실제 운영에서 각각 언제 쓰는가

### Redis를 많이 쓰는 경우

- 로그인 세션 저장
- 이메일 인증번호 저장
- 비밀번호 재설정 토큰 저장
- API rate limit 카운터
- 캐시
- 짧은 시간 유지되는 작업 상태
- 실시간 랭킹

### MongoDB를 많이 쓰는 경우

- 사용자 프로필 저장
- 게시글/댓글 저장
- 주문/문서/설정 데이터 저장
- 로그성 문서 저장
- 서비스의 메인 데이터 저장소

## 5. 실제 운영에서는 둘 중 하나만 쓰는가

보통은 아닙니다.  
실제 서비스에서는 Redis와 MongoDB를 **같이 쓰는 경우가 많습니다**.

대표적인 패턴은 이렇습니다.

### 패턴 1. MongoDB는 원본, Redis는 캐시

가장 흔한 패턴입니다.

- 실제 원본 데이터는 MongoDB에 저장
- 자주 읽는 값은 Redis에 캐시
- Redis가 비어 있으면 MongoDB에서 읽어서 다시 Redis에 올림

예시:

1. 사용자 프로필 원본은 MongoDB에 있음
2. API 요청이 오면 먼저 Redis를 확인
3. Redis에 없으면 MongoDB에서 읽음
4. 읽은 값을 Redis에 몇 초 또는 몇 분 캐시

이 방식의 장점:

- 응답 속도가 빨라짐
- MongoDB 부하가 줄어듦

### 패턴 2. Redis는 임시 상태, MongoDB는 영구 저장

- Redis에는 지금 당장 필요한 임시 상태 저장
- MongoDB에는 최종 결과 저장

예시:

- Redis: 현재 로그인 상태, OTP, 임시 큐 상태
- MongoDB: 사용자 계정 정보, 주문 정보, 기록 데이터

### 패턴 3. Redis로 먼저 빠르게 처리하고, MongoDB에 나중에 반영

이건 조금 더 복잡한 패턴입니다.

- 먼저 Redis에 빠르게 쓰기
- 이후 백그라운드에서 MongoDB에 반영

이 방식은 빠르지만 데이터 정합성 설계가 더 어려워집니다.

## 6. 이 프로젝트에서는 Redis와 MongoDB가 어떻게 들어가 있는가

현재 프로젝트에서는 **Redis가 주 저장 경로**이고, **MongoDB는 선택적 외부 저장 연동**입니다.

즉 핵심 구조는 이렇습니다.

```text
CLI
-> TCP / RESP
-> CommandManager
-> command handlers
-> Redis engine
-> StorageManager / TTL / Persistence / Invalidation
-> optional MongoManager
```

여기서 중요한 점은:

- 이 프로젝트의 중심은 `Redis engine`
- 메인 저장은 `StorageManager`
- TTL, persistence, invalidation도 Redis 쪽 기능
- MongoDB는 바깥쪽에 붙는 선택적 연동

즉 현재 구조는:

"MongoDB를 중심으로 하는 DB 서버"가 아니라  
"Mini Redis 서버에 MongoDB sync 기능이 선택적으로 붙은 구조"입니다.

## 7. 코드 기준으로 보면 어디서 연결되는가

### 7-1. 환경변수로 Mongo 사용 여부 결정

Mongo 설정은 [config.py](D:/jungleCamp/Projects/miniRedis/src/mini_redis/config.py)에서 읽습니다.

사용되는 환경변수:

- `MINI_REDIS_MONGO_ENABLED`
- `MINI_REDIS_MONGO_URI`
- `MINI_REDIS_MONGO_DB`
- `MINI_REDIS_MONGO_COLLECTION`
- `MINI_REDIS_MONGO_SERVER_SELECTION_TIMEOUT_MS`

즉 MongoDB는 항상 강제되는 것이 아니라, 환경변수로 켜는 선택 기능입니다.

### 7-2. bootstrap에서 MongoManager를 주입

[bootstrap.py](D:/jungleCamp/Projects/miniRedis/src/mini_redis/bootstrap.py)에서:

- `MongoAdapter`
- `MongoManager`

를 만들고, 이를 `Redis` 엔진에 주입합니다.

즉 Mongo 기능은 별도 매니저로 분리되어 있고, Redis 엔진이 필요할 때만 호출합니다.

### 7-3. Redis 엔진에서 실제 Mongo 호출

[redis.py](D:/jungleCamp/Projects/miniRedis/src/mini_redis/engine/redis.py)에서 Mongo 관련 동작이 일어납니다.

현재 코드 기준 주요 연결은 아래와 같습니다.

- `SET` -> `self._mongo.write_value(...)`
- `INCR` -> `self._mongo.write_value(...)`
- `DELETE` -> `self._mongo.delete_key(...)`
- `FLUSHDB` -> `self._mongo.clear()`
- `INFO MONGO` -> `self._mongo.info()`
- `BENCHMARK MONGO`, `BENCHMARK HYBRID` 지원

즉 현재 프로젝트는 MongoDB를 읽기 중심 원본 저장소처럼 쓰는 구조라기보다,  
**Redis 동작 후 Mongo에도 반영하는 write-through 성격의 연동**에 가깝습니다.

## 8. 이 프로젝트에서 Redis와 MongoDB의 역할을 나눠서 보면

### Redis 쪽 역할

- 실제 명령 실행의 중심
- in-memory key-value 저장
- TTL 관리
- tag invalidation
- AOF / snapshot persistence
- CLI/TCP/RESP를 통한 서버 흐름 전체

관련 핵심 파일:

- [redis.py](D:/jungleCamp/Projects/miniRedis/src/mini_redis/engine/redis.py)
- [manager.py](D:/jungleCamp/Projects/miniRedis/src/mini_redis/storage/manager.py)
- [ttl.py](D:/jungleCamp/Projects/miniRedis/src/mini_redis/storage/ttl.py)
- [manager.py](D:/jungleCamp/Projects/miniRedis/src/mini_redis/persistence/manager.py)
- [manager.py](D:/jungleCamp/Projects/miniRedis/src/mini_redis/invalidation/manager.py)

### MongoDB 쪽 역할

- 외부 저장소 연동
- write/delete/clear 수행
- 연결 상태와 마지막 작업 정보 제공
- 별도 benchmark 대상

관련 핵심 파일:

- [mongo_adapter.py](D:/jungleCamp/Projects/miniRedis/src/mini_redis/storage/mongo_adapter.py)
- [mongo_manager.py](D:/jungleCamp/Projects/miniRedis/src/mini_redis/storage/mongo_manager.py)
- [mongoredis.md](D:/jungleCamp/Projects/miniRedis/src/mini_redis/storage/mongoredis.md)

## 9. 이 프로젝트에서 MongoDB가 Redis를 대체하는가

아니요.  
현재 구조에서는 MongoDB가 Redis를 대체하지 않습니다.

정확히 말하면:

- Redis가 메인 서버 구조
- MongoDB는 선택적 보조 저장 경로

즉 이 프로젝트는:

- "Redis처럼 동작하는 서버"가 본체이고
- MongoDB는 "연동 가능한 외부 저장소"입니다

이 차이를 꼭 기억하면 이해가 쉬워집니다.

## 10. 실제 운영에서 Redis와 MongoDB를 같이 쓴다면 보통 어떻게 설계하는가

현재 프로젝트 구조를 기준으로 이해하기 쉽게 실제 운영 패턴을 정리하면 아래와 같습니다.

### 운영 패턴 A. MongoDB 원본 + Redis 캐시

가장 일반적입니다.

- MongoDB에 실제 데이터 저장
- Redis에 자주 읽는 데이터 캐시

예시:

- MongoDB: 사용자 정보
- Redis: `user:123:profile` 캐시

흐름:

1. API 요청이 들어옴
2. Redis에서 먼저 조회
3. 없으면 MongoDB에서 읽음
4. 읽은 값을 Redis에 TTL과 함께 저장

장점:

- 빠름
- 데이터 원본이 명확함
- 운영 패턴이 잘 알려져 있음

### 운영 패턴 B. Redis 임시 상태 + MongoDB 영구 저장

예시:

- Redis: 인증코드, 세션, 작업 진행 상태
- MongoDB: 회원정보, 주문 정보, 기록

장점:

- 각 저장소의 장점을 잘 살림

### 운영 패턴 C. Redis write-through + MongoDB 동기 반영

이 프로젝트의 현재 구조는 이 패턴과 가장 비슷합니다.

예시 흐름:

1. `SET key value`
2. Redis 메모리에 먼저 저장
3. 같은 요청 흐름에서 MongoDB에도 upsert
4. 응답 반환

장점:

- Redis와 Mongo 상태를 어느 정도 같이 맞출 수 있음

단점:

- Mongo가 느리면 응답도 느려질 수 있음
- 외부 DB 장애가 전체 요청에 영향을 줄 수 있음
- 운영 설계가 단순하지 않음

즉 실운영에서는 이 패턴을 쓸 수는 있지만, 신중하게 선택해야 합니다.

## 11. 운영 관점에서 둘을 같이 쓸 때 주의할 점

### 11-1. 누가 원본(source of truth)인지 정해야 함

가장 중요합니다.

질문은 이것입니다.

- 진짜 데이터의 기준은 Redis인가?
- 아니면 MongoDB인가?

이걸 정하지 않으면 장애 상황에서 매우 헷갈립니다.

보통은:

- Redis = 캐시
- MongoDB = 원본

으로 두는 경우가 많습니다.

### 11-2. 동기/비동기 반영 전략을 정해야 함

MongoDB 반영을:

- 요청 중에 바로 할지
- 나중에 비동기로 할지

결정해야 합니다.

동기 반영은 단순하지만 느려질 수 있고,  
비동기 반영은 빠르지만 정합성 관리가 더 어렵습니다.

### 11-3. 장애 시 동작을 정해야 함

예를 들어:

- Redis는 살아 있는데 MongoDB가 죽으면?
- MongoDB는 살아 있는데 Redis가 재시작하면?
- 둘의 데이터가 달라지면?

이 상황에 대한 정책이 필요합니다.

### 11-4. TTL 데이터와 영구 데이터의 차이를 구분해야 함

Redis는 TTL에 아주 잘 맞지만, MongoDB는 보통 영구 데이터 저장에 더 많이 사용됩니다.

그래서 아래를 구분해야 합니다.

- 잠깐 있다 사라져야 하는 데이터
- 오래 보관해야 하는 데이터

둘을 같은 방식으로 다루면 설계가 꼬일 수 있습니다.

## 12. 이 프로젝트를 보면서 이해하면 좋은 포인트

이 프로젝트를 기준으로 보면:

- Redis는 "서버 그 자체"
- MongoDB는 "연동되는 외부 저장소"

입니다.

즉 질문을 이렇게 바꾸면 이해가 쉬워집니다.

- "이 서버는 Redis처럼 어떻게 동작하는가?"
- "그 과정에서 MongoDB는 어디에 붙는가?"

현재 답은:

- Redis 쪽은 직접 구현되어 있음
- MongoDB는 그 위에 선택적으로 연결되어 있음

## 13. 초보자 관점에서 가장 쉬운 정리

처음에는 아래처럼 이해하면 됩니다.

### Redis

- 빠른 임시 저장소
- 캐시, TTL, 세션, 카운터에 강함

### MongoDB

- 오래 보관하는 문서 DB
- 서비스 본 데이터 저장에 강함

### 이 프로젝트

- Redis 스타일 서버를 직접 구현한 것
- 필요하면 MongoDB에도 같이 기록할 수 있게 만든 구조

## 14. 추천 이해 순서

이 문서를 읽은 뒤에는 아래 파일 순서로 보면 이해가 잘 됩니다.

1. [config.py](D:/jungleCamp/Projects/miniRedis/src/mini_redis/config.py)
2. [bootstrap.py](D:/jungleCamp/Projects/miniRedis/src/mini_redis/bootstrap.py)
3. [redis.py](D:/jungleCamp/Projects/miniRedis/src/mini_redis/engine/redis.py)
4. [mongo_manager.py](D:/jungleCamp/Projects/miniRedis/src/mini_redis/storage/mongo_manager.py)
5. [mongo_adapter.py](D:/jungleCamp/Projects/miniRedis/src/mini_redis/storage/mongo_adapter.py)
6. [mongoredis.md](D:/jungleCamp/Projects/miniRedis/src/mini_redis/storage/mongoredis.md)

## 15. 한 줄 요약

Redis는 빠른 메모리 저장소, MongoDB는 오래 보관하는 문서 DB로 이해하면 되고,  
현재 프로젝트에서는 Redis가 본체이고 MongoDB는 선택적으로 붙는 외부 저장 경로입니다.

## 16. 만약 이 프로젝트를 "Mongo 원본 + Redis 캐시" 구조로 바꾸면?

이제 한 단계 더 나아가서 생각해보면, 현재 프로젝트는:

- Redis 스타일 서버가 본체
- MongoDB는 선택적 write-through 연동

구조에 가깝습니다.

그런데 실제 운영에서 더 흔한 구조는 아래입니다.

- MongoDB = 원본 데이터 저장소
- Redis = 빠른 캐시

즉 질문을 바꾸면:

"지금 프로젝트를 MongoDB가 진짜 원본이고, Redis는 캐시처럼 동작하는 구조로 바꾸면 무엇이 달라질까?"

가 됩니다.

## 17. 가장 큰 차이: 데이터의 기준이 바뀜

현재 구조에서는 Redis 쪽이 먼저 동작하고, MongoDB는 그 결과를 따라가는 쪽입니다.

```text
command
-> Redis memory write
-> optional Mongo sync
```

하지만 Mongo 원본 + Redis 캐시 구조로 바꾸면 기준이 이렇게 바뀝니다.

```text
command
-> MongoDB write or read
-> Redis cache update or cache lookup
```

즉 가장 큰 변화는:

- 지금: Redis가 중심
- 변경 후: MongoDB가 중심

입니다.

이 차이는 단순히 저장 위치가 바뀌는 것이 아니라, 전체 설계 기준이 바뀌는 것입니다.

## 18. 읽기 흐름은 어떻게 달라지는가

현재 프로젝트에서는 `GET`이 기본적으로 메모리 저장소를 봅니다.

지금 흐름:

```text
GET key
-> Redis engine
-> StorageManager.get(key)
-> 값 반환
```

Mongo 원본 + Redis 캐시 구조로 바꾸면 보통 아래 흐름이 됩니다.

```text
GET key
-> 먼저 Redis 캐시 조회
-> 있으면 바로 반환
-> 없으면 MongoDB 조회
-> 조회한 값을 Redis에 TTL과 함께 캐시
-> 반환
```

이걸 보통 cache-aside 패턴이라고 많이 부릅니다.

장점:

- 자주 읽는 데이터는 매우 빠름
- Redis가 비어 있어도 MongoDB에서 복구 가능

단점:

- 첫 조회는 느릴 수 있음
- 캐시 만료/무효화 설계를 잘 해야 함

## 19. 쓰기 흐름은 어떻게 달라지는가

현재 프로젝트에서는 `SET`이 Redis 메모리 저장을 먼저 수행합니다.

현재 흐름:

```text
SET key value
-> StorageManager.set()
-> TTL / invalidation 처리
-> persistence append
-> optional Mongo write
```

Mongo 원본 구조로 바꾸면 보통 아래 둘 중 하나를 선택합니다.

### 방식 A. Mongo 먼저 쓰고 Redis 캐시 갱신

```text
SET key value
-> MongoDB upsert
-> 성공하면 Redis 캐시 갱신
-> 응답 반환
```

장점:

- 원본 데이터가 항상 먼저 기록됨
- source of truth가 명확함

단점:

- Mongo가 느리면 쓰기도 느려짐

### 방식 B. Mongo 먼저 쓰고 Redis 캐시는 삭제

```text
SET key value
-> MongoDB upsert
-> 관련 Redis key 삭제
-> 다음 GET 때 다시 캐시 채움
```

장점:

- 캐시 일관성 관리가 상대적으로 단순함

단점:

- 쓰기 직후 첫 조회는 캐시 miss가 날 수 있음

실무에서는 "쓰기 후 캐시 삭제"도 꽤 자주 씁니다.

## 20. DELETE는 어떻게 바뀌는가

현재는:

```text
DELETE key
-> Redis에서 삭제
-> optional Mongo delete
```

Mongo 원본 구조라면:

```text
DELETE key
-> MongoDB에서 삭제
-> Redis 캐시에서도 삭제
```

즉 삭제의 기준도 MongoDB가 됩니다.

## 21. TTL은 더 조심해서 설계해야 함

현재 프로젝트는 TTL이 Redis 쪽 핵심 기능입니다.

하지만 Mongo 원본 구조로 가면 TTL은 두 종류로 나눠 생각해야 합니다.

### 1. 캐시 TTL

이건 Redis에만 적용되는 TTL입니다.

예:

- 사용자 프로필을 60초 동안만 캐시

원본인 MongoDB 데이터는 지워지지 않고, 캐시만 사라집니다.

### 2. 실제 데이터 TTL

이건 원본 데이터 자체가 일정 시간이 지나면 의미가 없어지는 경우입니다.

예:

- 인증코드
- 임시 토큰
- 짧은 세션 정보

이 경우는 Mongo 쪽에서도 TTL 정책이 필요할 수 있습니다.

즉 Mongo 원본 구조에서는:

- Redis TTL = 캐시 수명
- Mongo TTL = 실제 데이터 수명

을 구분해야 합니다.

## 22. 지금 프로젝트 기준으로 구조는 어떻게 바뀌는가

현재 흐름:

```text
CLI
-> TCP / RESP
-> CommandManager
-> handlers
-> Redis engine
-> StorageManager 중심
-> optional MongoManager
```

Mongo 원본 + Redis 캐시 흐름으로 바꾸면 개념적으로는 이렇게 바뀝니다.

```text
CLI
-> TCP / RESP
-> CommandManager
-> handlers
-> service layer or Redis engine
-> Redis cache lookup/update
-> MongoDB source read/write
```

즉 `StorageManager`의 위치가 바뀝니다.

지금은:

- StorageManager = 메인 저장소

바뀐 후에는:

- StorageManager = 캐시 계층

이 됩니다.

## 23. 코드 레벨에서는 어떤 점을 바꿔야 하는가

이 프로젝트를 정말 그렇게 바꾸려면 아래 변경이 필요합니다.

### 23-1. `get()`의 책임 변경

[redis.py](D:/jungleCamp/Projects/miniRedis/src/mini_redis/engine/redis.py)의 `get()`은 지금 메모리 저장소 중심입니다.

바뀐 구조에서는:

1. Redis 캐시 조회
2. miss면 Mongo 조회
3. Mongo 결과를 Redis 캐시에 저장
4. 반환

흐름으로 바뀌어야 합니다.

### 23-2. `set()`의 기준 변경

지금은 `StorageManager.set()`이 먼저입니다.

바뀐 구조에서는:

- Mongo 먼저 쓰기
- 성공 후 캐시 갱신 또는 캐시 무효화

방식으로 바뀌어야 합니다.

### 23-3. `delete()`의 기준 변경

지금은 메모리 삭제가 중심입니다.

바뀐 구조에서는:

- Mongo 삭제
- 캐시 삭제

순서가 자연스럽습니다.

### 23-4. `StorageManager`의 의미 재정의

지금 `StorageManager`는 Redis 자체 데이터 저장소입니다.

Mongo 원본 구조에서는:

- 영구 저장소가 아니라 캐시 저장소
- 빠른 조회용 레이어

로 의미가 바뀝니다.

### 23-5. persistence 역할 축소 또는 변경

현재 프로젝트는 Redis 메모리 상태를 AOF/snapshot으로 저장합니다.

그런데 Mongo가 원본이 되면 질문이 생깁니다.

- Redis 캐시 상태를 굳이 AOF/snapshot으로 길게 보존해야 하는가?

많은 경우 답은 "꼭 그럴 필요는 없다"입니다.

왜냐하면 캐시는 날아가도 Mongo에서 다시 채울 수 있기 때문입니다.

즉 이 구조로 가면 persistence는:

- 아예 단순화되거나
- 캐시 warm-up 보조 용도로 축소되거나
- 운영 옵션으로만 남을 수 있습니다

## 24. 장점은 무엇인가

Mongo 원본 + Redis 캐시 구조로 바꾸면 아래 장점이 있습니다.

- 원본 데이터 기준이 명확함
- Redis 재시작 시 복구가 쉬움
- 캐시는 날려도 Mongo에서 다시 채울 수 있음
- 실무에서 널리 쓰는 익숙한 패턴과 가까워짐
- 장애 분석이 더 쉬워질 수 있음

## 25. 단점은 무엇인가

반대로 아래 단점도 있습니다.

- 현재 프로젝트의 "Redis 자체 서버" 정체성이 약해짐
- GET 경로가 더 복잡해짐
- 캐시 일관성 설계가 필요함
- 캐시 무효화 정책이 더 중요해짐
- Mongo 장애가 읽기/쓰기 경로에 더 직접적으로 영향을 줌

즉 Mini Redis를 "직접 Redis를 구현하는 프로젝트"로 볼지,  
"Mongo 앞단 캐시 서버"로 볼지 성격이 달라집니다.

## 26. 어떤 경우에 이 구조가 더 잘 맞는가

아래 같은 경우에는 Mongo 원본 + Redis 캐시 구조가 더 현실적입니다.

- 사용자/게시글/문서 같은 본 데이터가 이미 Mongo에 있는 서비스
- 읽기 속도 개선이 주목적인 경우
- Redis는 캐시로만 두고 싶은 경우
- Redis가 날아가도 서비스 복구가 쉬워야 하는 경우

예시:

- 사용자 프로필 조회 캐시
- 게시글 상세 조회 캐시
- 자주 읽는 설정값 캐시
- 대시보드 집계 결과 캐시

## 27. 반대로 현재 구조가 더 잘 맞는 경우

지금 프로젝트 같은 Redis 중심 구조가 더 잘 맞는 경우도 있습니다.

- TTL 중심 임시 데이터 서버를 만들고 싶을 때
- 캐시 무효화 엔진 자체를 만들고 싶을 때
- Redis 스타일 동작을 직접 구현하고 싶을 때
- 교육용 / 실험용 / 데모용 서버를 만들고 싶을 때

즉 목적에 따라 선택이 달라집니다.

## 28. 현실적인 추천

현재 프로젝트를 당장 Mongo 원본 + Redis 캐시 구조로 완전히 바꾸기보다는, 아래처럼 가는 것이 현실적입니다.

### 추천 1. 현재 구조 유지

- Mini Redis 자체 서버로 계속 발전
- TTL / invalidation / inspect 기능 강화

### 추천 2. 별도 모드 추가

- "Redis-native mode"
- "Mongo-source cache mode"

처럼 모드를 나누는 방법도 있습니다.

이렇게 하면:

- 학습용 구조도 유지할 수 있고
- 실무형 캐시 패턴 실험도 가능해집니다

## 29. 최종 요약

현재 프로젝트는 "Redis가 본체, Mongo는 선택 연동" 구조입니다.

만약 이를 "Mongo 원본 + Redis 캐시" 구조로 바꾸면:

- 데이터 기준이 Mongo로 이동하고
- Redis는 빠른 캐시 계층이 되며
- `get/set/delete/persistence`의 의미가 전반적으로 바뀝니다

이 구조는 실제 서비스에서는 더 흔하지만,  
현재 프로젝트의 정체성은 지금보다 "Redis 구현체"보다는 "캐시 계층 서버"에 가까워지게 됩니다.
