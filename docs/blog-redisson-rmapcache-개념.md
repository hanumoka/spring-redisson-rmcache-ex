# Redis TTL 만료 이벤트를 Spring에서 감지하기 - Redisson RMapCache 완벽 가이드

---

## TL;DR

> 1. **RMapCache**는 Redis HASH의 각 필드(엔트리)에 개별 TTL을 설정할 수 있는 Redisson 자료구조
> 2. **EntryExpiredListener**로 만료 이벤트를 안정적으로 수신 (Redis 기본 TTL로는 불가능)
> 3. **주의**: 만료 처리는 클라이언트(Spring)에서 하므로, 서버가 죽으면 만료 처리도 중단됨

---

## 들어가며

### 문제 상황

Redis를 캐시로 사용하다 보면 이런 상황을 마주하게 됩니다.

> "Redis에 저장한 데이터가 TTL로 만료될 때, 그 시점에 특정 로직을 실행하고 싶은데..."

예를 들어:
- 세션이 만료될 때 사용자 로그아웃 처리
- 임시 데이터가 삭제될 때 DB 상태 업데이트
- 캐시가 만료될 때 알림 발송

일반적인 Redis TTL은 데이터가 **조용히** 사라집니다. 애플리케이션은 데이터가 언제 삭제되었는지 알 수 없습니다.

```
[일반 Redis TTL의 한계]
데이터 저장 → 10분 후 → 데이터 사라짐 (아무도 모름)
```

### 해결책: Redisson RMapCache

Redisson의 `RMapCache`와 `EntryExpiredListener`를 사용하면 이 문제를 해결할 수 있습니다.

```
[RMapCache + EntryExpiredListener]
데이터 저장 → 10분 후 → 만료 이벤트 발생 → 콜백 실행!
```

### 이 글에서 다루는 내용

| Part | 내용 |
|------|------|
| **Part 1** | 개념 이해 - Redisson, RMapCache, TTL/MaxIdleTime |
| **Part 2** | 내부 동작 원리 - ZSET 구조, EvictionScheduler |
| **Part 3** | 이벤트 처리 - EntryExpiredListener, 분산 환경 |
| **Part 4** | 운영 가이드 - 서버 다운, 스레드 모델, 체크리스트 |

---

# Part 1: 개념 이해

---

## Redisson이란?

> **핵심**: Redis를 Java 컬렉션처럼 사용하게 해주는 고수준 클라이언트

### 다른 Redis 클라이언트와의 비교

| 클라이언트 | 특징 | 추상화 수준 |
|-----------|------|------------|
| **Jedis** | 가장 오래된 Java Redis 클라이언트. 동기 방식. | 낮음 (Redis 명령어 래퍼) |
| **Lettuce** | Spring Boot 기본. 비동기/리액티브 지원. Netty 기반. | 중간 |
| **Redisson** | 분산 자료구조 제공. 이벤트 리스너, 분산 락 등 고급 기능. | 높음 |

### Redisson의 핵심 철학

```java
// 일반 Java 코드
Map<String, User> users = new HashMap<>();
users.put("user1", new User("홍길동"));

// Redisson 코드 (거의 동일!)
RMap<String, User> users = redisson.getMap("users");
users.put("user1", new User("홍길동"));  // Redis에 저장됨!
```

### Redisson이 제공하는 추가 가치

| 기능 | 설명 |
|-----|------|
| **자동 직렬화** | Java 객체를 자동으로 직렬화/역직렬화 |
| **원자성 보장** | 복잡한 연산을 Lua 스크립트로 원자적 실행 |
| **이벤트 리스너** | 데이터 변경/만료 시 콜백 (Redis 기본 기능 아님) |
| **분산 락** | RLock, RSemaphore 등 동기화 도구 |

### 주요 분산 자료구조

| Redisson 타입 | Java 대응 | 설명 |
|--------------|----------|------|
| `RMap` | `Map` | 분산 해시맵 (TTL 없음) |
| `RMapCache` | `Map` + TTL | **TTL/MaxIdleTime 지원** |
| `RSet` | `Set` | 분산 집합 |
| `RList` | `List` | 분산 리스트 |
| `RLock` | - | 분산 락 |

---

## RMap vs RMapCache

> **핵심 차이**: TTL 지원 여부

### RMap (기본 분산 맵)

```java
RMap<String, String> map = redisson.getMap("myMap");
map.put("key", "value");  // 영구 저장 (삭제 전까지 유지)
```

- 데이터가 명시적으로 삭제하기 전까지 **영구 보관**
- TTL 기능 없음

### RMapCache (캐시용 분산 맵)

```java
RMapCache<String, String> cache = redisson.getMapCache("myCache");
cache.put("key", "value", 10, TimeUnit.MINUTES);  // 10분 후 자동 삭제
```

- **TTL**: 지정 시간 후 자동 삭제
- **MaxIdleTime**: 마지막 접근 후 미사용 시 삭제
- **만료 이벤트 리스너**: 데이터 만료 시 콜백 가능

---

## TTL과 MaxIdleTime

### TTL (Time-To-Live)

> 데이터 저장 시점부터 **고정 시간** 후 삭제

```java
cache.put("session123", userData, 30, TimeUnit.MINUTES);
```

```
시간 흐름 →
[저장] -------- 30분 --------→ [자동 삭제]
   ↑                              ↑
   0분                           30분

* 중간에 get() 호출해도 시간 연장 안 됨
```

**사용 사례**: 세션 만료, 인증 토큰, 캐시 갱신 주기

### MaxIdleTime (최대 유휴 시간)

> 마지막 **접근 시점**부터 미사용 시 삭제

```java
// TTL=무제한, MaxIdleTime=10분
cache.put("key", value, 0, TimeUnit.MINUTES, 10, TimeUnit.MINUTES);
```

```
시간 흐름 →
[저장] --- [읽기] --- [읽기] ---------- 10분 미접근 --------→ [삭제]
   ↑         ↑          ↑                                      ↑
   0분       5분        8분                                    18분
             (리셋)     (리셋)                            (마지막 접근+10분)
```

**사용 사례**: 활동 기반 세션, 자주 사용되는 데이터만 유지

### TTL + MaxIdleTime 조합

```java
// TTL=60분, MaxIdleTime=10분
cache.put("key", value, 60, TimeUnit.MINUTES, 10, TimeUnit.MINUTES);
```

| 시나리오 | 결과 |
|---------|------|
| 10분간 접근 없음 | MaxIdleTime에 의해 삭제 |
| 60분 경과 | TTL에 의해 삭제 (자주 접근해도) |

**동작**: 둘 중 **먼저 만족되는 조건**에 의해 삭제

---

# Part 2: 내부 동작 원리

---

## RMapCache의 내부 구조

> **핵심**: RMapCache는 Redis 기본 TTL(EXPIRE)을 사용하지 않고, ZSET으로 만료 시간을 관리

### 왜 Redis 기본 TTL을 쓰지 않는가?

```
# Redis 기본 TTL의 한계
HSET myHash field1 "value1"
HSET myHash field2 "value2"
EXPIRE myHash 600   # myHash "전체"가 삭제됨
                    # field1만 따로 TTL 설정 불가!
```

**문제**: Redis TTL은 **키 단위**로만 설정 가능. HASH의 **개별 필드에는 TTL 불가**.

### RMapCache의 해결책: ZSET 활용

```
┌─────────────────────────────────────────────────────────────┐
│ Redis 내부 저장 구조 (myCache 생성 시)                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 1. myCache (HASH) - 실제 데이터                              │
│    ├─ key1 → serialized_value1                             │
│    └─ key2 → serialized_value2                             │
│                                                             │
│ 2. redisson__timeout__set:{myCache} (ZSET) - TTL 관리        │
│    ├─ score: 1703124300000 → "key1"   # 만료 timestamp      │
│    └─ score: 1703124600000 → "key2"                        │
│                                                             │
│ 3. redisson__idle__set:{myCache} (ZSET) - MaxIdleTime 관리   │
│    └─ score: 1703124300000 → "key1"                        │
│                                                             │
│ 4. redisson_map_cache_expired:{myCache} (Topic) - Pub/Sub   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**ZSET을 쓰는 이유**: score로 정렬되어 있어 만료된 키 조회가 O(log N + M)으로 빠름

```
ZRANGEBYSCORE redisson__timeout__set:{myCache} 0 {현재시간}
→ "현재 시간 기준 만료된 모든 키" 한 번에 조회
```

---

## EvictionScheduler: 만료 처리의 핵심

> **핵심**: Redisson 클라이언트(Spring)에서 동작하는 백그라운드 스케줄러

### 동작 방식: 동적 스케줄링

```
┌────────────────────────────────────────────────────────────┐
│  EvictionScheduler 동작 방식                                │
│                                                            │
│  1. 다음 만료 예정 시간 조회                                 │
│     ZRANGEBYSCORE timeout_zset 0 +inf LIMIT 0 1            │
│     → 가장 빨리 만료될 항목의 timestamp 확인                 │
│                                                            │
│  2. 해당 시간까지 대기 (sleep)                              │
│                                                            │
│  3. 만료된 항목들 일괄 처리                                  │
│     - HDEL myCache "key"        # 데이터 삭제               │
│     - ZREM timeout_zset "key"   # ZSET에서 제거             │
│     - PUBLISH expired_topic     # 이벤트 발행               │
│                                                            │
│  4. 1번으로 돌아감                                          │
└────────────────────────────────────────────────────────────┘
```

**핵심**: 1초마다 폴링하지 않고, **다음 만료 시간에 맞춰 깨어남** → CPU 효율적

### 예시

```
현재 시간: 10:00:00

ZSET 상태:
  score: 10:05:00 → key1  (5분 후 만료)
  score: 10:10:00 → key2  (10분 후 만료)

스케줄러 동작:
  1. 가장 빠른 만료 = 10:05:00 (key1)
  2. 5분 동안 sleep
  3. 10:05:00에 깨어남 → key1 삭제 + 이벤트 발행
  4. 다음 만료 = 10:10:00 (key2)
  5. 5분 동안 sleep...
```

### EvictionScheduler는 어디서 실행되는가?

```
┌─────────────────────────────────────────────────────────────┐
│                      JVM (Spring Application)               │
│                                                             │
│  ┌─────────────────────┐    ┌─────────────────────────────┐ │
│  │   Spring 스레드 풀    │    │    Redisson 스레드 풀        │ │
│  │                     │    │                             │ │
│  │  - @Async 스레드     │    │  - Netty EventLoop 스레드    │ │
│  │  - @Scheduled 스레드 │    │  - EvictionScheduler ◀ 여기  │ │
│  │  - HTTP 요청 스레드   │    │  - Pub/Sub 리스너 스레드     │ │
│  └─────────────────────┘    └─────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

**중요**: Spring의 스케줄러가 아닌 **Redisson 자체 스레드**에서 실행

---

## Redis TTL vs RMapCache TTL 비교

| 항목 | Redis 기본 TTL | RMapCache TTL |
|-----|---------------|---------------|
| **TTL 단위** | 키 전체 | HASH 필드(엔트리) 단위 |
| **만료 주체** | Redis 서버 | Redisson 클라이언트 |
| **만료 감지** | 수동적 (접근 시) + 샘플링 | 능동적 (스케줄러) |
| **서버 다운 시** | Redis가 알아서 삭제 | **삭제 안 됨** ⚠️ |
| **이벤트 안정성** | Fire-and-forget (유실 가능) | Pub/Sub + 재시작 시 일괄 처리 |
| **이벤트 내용** | 키 이름만 | 키 + 값 모두 포함 |
| **MaxIdleTime** | 미지원 | 지원 |

---

# Part 3: 이벤트 처리

---

## EntryExpiredListener

> **핵심**: TTL/MaxIdleTime 만료 시 콜백을 받을 수 있는 리스너

### 기본 사용법

```java
RMapCache<String, UserData> cache = redisson.getMapCache("sessions");

cache.addListener(new EntryExpiredListener<String, UserData>() {
    @Override
    public void onExpired(EntryEvent<String, UserData> event) {
        String key = event.getKey();        // "session:user1"
        UserData value = event.getValue();  // 만료된 데이터 객체

        // 만료 시 실행할 로직
        userService.handleSessionExpired(key, value);
    }
});
```

### 이벤트 리스너 종류

| 리스너 | 발생 시점 |
|-------|----------|
| `EntryCreatedListener` | 새 항목 추가 시 |
| `EntryUpdatedListener` | 기존 항목 수정 시 |
| `EntryRemovedListener` | 명시적 삭제 시 (`remove()` 호출) |
| `EntryExpiredListener` | **TTL/MaxIdleTime 만료 시** |

### 동작 흐름

```
[데이터 저장]                              [만료 처리]
     │                                        │
     │  cache.put("key", value, 10, MINUTES)  │
     │                                        │
     ▼                                        ▼
 ┌───────┐                              ┌───────────┐
 │ Redis │  ──── 10분 경과 ────────────→ │ Scheduler │
 │ HASH  │                              │ ZSET 스캔  │
 │ ZSET  │                              └─────┬─────┘
 └───────┘                                    │
                                              ▼
                                      ┌───────────────┐
                                      │ 만료 처리      │
                                      │ - HDEL        │
                                      │ - ZREM        │
                                      │ - PUBLISH     │
                                      └───────┬───────┘
                                              │
                                              ▼
                                      ┌───────────────┐
                                      │ onExpired()   │
                                      │ 콜백 실행      │
                                      └───────────────┘
```

---

## 분산 환경에서의 이벤트 처리

### 문제: 모든 인스턴스가 같은 이벤트를 받음

```
                         [Redis]
                            │
                      PUBLISH 이벤트
                            │
          ┌─────────────────┼─────────────────┐
          ▼                 ▼                 ▼
     [Spring #1]       [Spring #2]       [Spring #3]
          │                 │                 │
     onExpired()       onExpired()       onExpired()
     콜백 실행          콜백 실행          콜백 실행

     → 3개 인스턴스 모두 동일한 이벤트 수신!
```

### 두 가지 전략

#### 전략 1: 모든 인스턴스가 처리

**적합한 작업**: 로컬 상태 동기화 (Near Cache 무효화, 로컬 메모리 정리)

```
만료 이벤트 발생
    ├── Spring #1: 로컬 캐시에서 해당 키 제거
    ├── Spring #2: 로컬 캐시에서 해당 키 제거
    └── Spring #3: 로컬 캐시에서 해당 키 제거
```

#### 전략 2: 한 인스턴스만 처리 (분산 락)

**적합한 작업**: DB 업데이트, 외부 API 호출, 알림 발송 (한 번만 실행되어야 하는 작업)

```java
cache.addListener(new EntryExpiredListener<String, UserData>() {
    @Override
    public void onExpired(EntryEvent<String, UserData> event) {
        String lockKey = "lock:expired:" + event.getKey();
        RLock lock = redisson.getLock(lockKey);

        if (lock.tryLock()) {  // 락 획득 시도
            try {
                // 한 인스턴스만 이 코드 실행
                processExpiredEvent(event);
            } finally {
                lock.unlock();
            }
        }
        // 락 획득 실패 = 다른 인스턴스가 처리 중 → 스킵
    }
});
```

```
만료 이벤트 발생
    ├── Spring #1: 락 획득 시도 → 성공! → DB 업데이트 실행
    ├── Spring #2: 락 획득 시도 → 실패 → 스킵
    └── Spring #3: 락 획득 시도 → 실패 → 스킵
```

---

# Part 4: 운영 가이드

---

## 서버 다운 시 만료 데이터 처리

> **핵심**: Spring 서버가 죽으면 EvictionScheduler도 죽어서 만료 처리가 중단됨

### 서버 다운 시나리오

```
[정상 상태]
┌─────────────────────────────────────────────────────────┐
│  Spring 서버 (실행 중)                                   │
│  EvictionScheduler (동작 중) → ZSET 스캔 → 만료 처리     │
└─────────────────────────────────────────────────────────┘
                         │
                         ▼
                      [Redis] - 정상적으로 만료 처리됨


[서버 다운 상태]
┌─────────────────────────────────────────────────────────┐
│  Spring 서버 ❌ (죽음)                                   │
│  EvictionScheduler ❌ (함께 죽음)                        │
└─────────────────────────────────────────────────────────┘

                      [Redis]
                         │
           ┌─────────────┴─────────────┐
           │                           │
     HASH: key1, key2           ZSET: 만료 정보
     (삭제 안 됨!)              (그대로 남아있음)
```

**결과**: 만료 예정 데이터가 **삭제되지 않고 Redis에 그대로 남음**

### 서버 재시작 시

```
[서버 재시작]
EvictionScheduler 시작
    │
    │  1. ZSET 스캔: ZRANGEBYSCORE timeout_zset 0 {현재시간}
    │
    │  2. 만료된 키들 발견!
    │     - key1 (10분 전 만료 예정이었음)
    │     - key2 (5분 전 만료 예정이었음)
    │
    │  3. 일괄 삭제 + 이벤트 발행
    ▼
EntryExpiredListener 콜백도 일괄 실행됨!
```

### 분산 환경에서의 안전장치

```
                         [Redis]
                            │
          ┌─────────────────┼─────────────────┐
          ▼                 ▼                 ▼
     [Spring #1]       [Spring #2]       [Spring #3]

Spring #1 다운 → #2, #3이 만료 처리 계속
Spring #1, #2 다운 → #3이 만료 처리 계속
전부 다운 → 만료 처리 중단 ⚠️
```

**권장**: 분산 환경에서 최소 1개 인스턴스는 항상 살아있어야 함

### 실무적 영향

| 상황 | 결과 | 영향 |
|-----|------|-----|
| 서버 잠깐 다운 (수 분) | 재시작 시 밀린 만료 일괄 처리 | 큰 문제 없음 |
| 서버 오래 다운 (수 시간~일) | 만료 안 된 데이터가 Redis 메모리 점유 | 메모리 부족 위험 |
| 분산 환경 (여러 인스턴스) | 하나라도 살아있으면 만료 처리됨 | 안정적 |
| 모든 인스턴스 다운 | 모든 만료 처리 중단 | 위험 |

---

## 스레드 모델과 운영 주의사항

### Redisson의 스레드 구조

```
┌─────────────────────────────────────────────────────────────────┐
│                    Redisson 내부 스레드 구조                      │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  1. Netty EventLoopGroup                                │   │
│  │     - Redis I/O 처리 (읽기/쓰기)                         │   │
│  │     - 기본 스레드 수: CPU 코어 * 2                        │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  2. ScheduledExecutorService                            │   │
│  │     - EvictionScheduler (만료 처리)                      │   │
│  │     - 커넥션 풀 유지                                     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  3. Pub/Sub 전용 스레드                                  │   │
│  │     - 이벤트 리스너 콜백 실행 ◀ EntryExpiredListener     │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### ⚠️ 주의: 리스너 콜백에서 블로킹 금지

**문제**: EntryExpiredListener 콜백은 **Netty EventLoop 스레드**에서 실행됨

```java
// ❌ 잘못된 예시 - Netty 스레드 블로킹
cache.addListener(new EntryExpiredListener<String, UserData>() {
    @Override
    public void onExpired(EntryEvent<String, UserData> event) {
        // ❌ 동기 DB 호출 - 블로킹!
        userRepository.save(user);

        // ❌ 동기 HTTP 호출 - 블로킹!
        restTemplate.postForObject(url, request, Response.class);

        // ❌ Thread.sleep - 절대 금지!
        Thread.sleep(1000);
    }
});
```

**결과**: Netty 스레드가 블로킹되면 **모든 Redis 통신이 멈춤**

**해결책**: 별도 스레드에서 비동기 처리

```java
// ✅ 올바른 예시 - 별도 스레드풀에서 처리
@Service
public class CacheEventHandler {

    private final ExecutorService executor = Executors.newFixedThreadPool(10);

    public void setupListener(RMapCache<String, UserData> cache) {
        cache.addListener(new EntryExpiredListener<String, UserData>() {
            @Override
            public void onExpired(EntryEvent<String, UserData> event) {
                // Netty 스레드에서는 작업을 위임만 함
                executor.submit(() -> {
                    // 별도 스레드에서 블로킹 작업 수행
                    userRepository.save(event.getValue());
                    notificationService.send(event.getKey());
                });
            }
        });
    }
}
```

### 설정 예시

```java
@Configuration
public class RedissonConfig {

    @Bean(destroyMethod = "shutdown")  // 중요! 종료 시 리소스 정리
    public RedissonClient redissonClient() {
        Config config = new Config();
        config.useSingleServer()
              .setAddress("redis://localhost:6379")
              .setConnectionPoolSize(64)
              .setConnectionMinimumIdleSize(24)
              .setSubscriptionConnectionPoolSize(50);  // Pub/Sub용

        config.setNettyThreads(32);  // 기본값: CPU * 2
        config.setThreads(16);

        return Redisson.create(config);
    }
}
```

---

## 운영 체크리스트

### 필수 확인 항목

| 항목 | 확인 사항 | 중요도 |
|-----|----------|-------|
| **리스너 콜백** | 블로킹 작업 없는지? 별도 스레드 사용? | ⭐⭐⭐ |
| **종료 처리** | `destroyMethod = "shutdown"` 설정? | ⭐⭐⭐ |
| **분산 락** | 한 번만 실행할 작업에 RLock 사용? | ⭐⭐⭐ |
| **스레드 풀** | 트래픽에 맞는 크기 설정? | ⭐⭐ |
| **커넥션 풀** | Pub/Sub용 커넥션 충분한지? | ⭐⭐ |
| **메모리** | JVM 힙 여유 있는지? | ⭐⭐ |
| **모니터링** | 커넥션, 스레드, 메모리 모니터링? | ⭐⭐ |

### Redis 메모리 보호 설정

```
# redis.conf
maxmemory 2gb
maxmemory-policy allkeys-lru  # 메모리 부족 시 LRU로 삭제
```

---

## RMapCache 선택 시 트레이드오프

### 얻는 것

- ✅ HASH 필드(엔트리)별 개별 TTL 설정
- ✅ MaxIdleTime 지원
- ✅ 안정적인 만료 이벤트 리스너
- ✅ 만료된 데이터의 값(value)도 이벤트에 포함
- ✅ 재시작 시 밀린 만료 일괄 처리

### 감수해야 하는 것

- ⚠️ 클라이언트가 죽으면 만료 처리 중단
- ⚠️ 분산 환경에서 최소 하나의 인스턴스는 항상 살아있어야 함
- ⚠️ 모든 인스턴스 다운 시 "좀비 데이터" 누적 가능
- ⚠️ 리스너 콜백에서 블로킹 작업 금지

---

# FAQ

---

### Q1: RMapCache와 Spring Cache 추상화를 함께 쓸 수 있나요?

**A**: 네, Redisson은 Spring Cache 추상화를 지원합니다. 하지만 `@Cacheable` 등의 어노테이션으로는 EntryExpiredListener를 직접 등록하기 어렵습니다. 만료 이벤트가 필요하다면 직접 RMapCache를 사용하는 것이 좋습니다.

### Q2: TTL 만료 이벤트가 정확히 만료 시점에 발생하나요?

**A**: 거의 정확하지만, 약간의 지연이 있을 수 있습니다. EvictionScheduler가 다음 만료 시간에 맞춰 깨어나지만, 처리 시간과 네트워크 지연으로 밀리초~초 단위 오차가 발생할 수 있습니다.

### Q3: 만료 이벤트에서 value가 null로 올 수 있나요?

**A**: RMapCache의 EntryExpiredListener는 만료 시점의 value를 포함합니다. 단, 직렬화 문제가 있거나 데이터가 손상된 경우 null이 될 수 있으니 null 체크를 권장합니다.

### Q4: Near Cache와 RMapCache를 함께 쓰면 이벤트가 어떻게 동작하나요?

**A**: Near Cache가 활성화되면 로컬 캐시에도 데이터가 저장됩니다. 만료 이벤트 발생 시 모든 인스턴스가 이벤트를 받아 로컬 Near Cache도 무효화됩니다.

### Q5: 분산 환경에서 여러 인스턴스가 동시에 만료 처리를 시도하면?

**A**: Redisson 내부적으로 ZSET에서 키를 삭제할 때 원자적으로 처리합니다. 먼저 삭제에 성공한 인스턴스만 이벤트를 발행하므로, 중복 삭제는 발생하지 않습니다. 다만 **이벤트는 모든 인스턴스가 받으므로**, 비즈니스 로직 중복 실행 방지를 위해 분산 락을 사용해야 합니다.

### Q6: 서버가 오래 다운되어 있다가 재시작하면 어떻게 되나요?

**A**: 재시작 시 EvictionScheduler가 ZSET을 스캔하여 밀린 만료를 일괄 처리합니다. 이때:
- 많은 양의 만료 데이터가 한꺼번에 처리되어 순간적인 부하 발생 가능
- EntryExpiredListener 콜백도 일괄 실행됨
- 오래된 데이터의 value는 그대로 유지되어 있음 (삭제 안 됐으므로)

---

# 핵심 정리

| 개념 | 설명 |
|-----|------|
| **Redisson** | Redis를 Java 컬렉션처럼 사용하게 해주는 고수준 클라이언트 |
| **RMap** | 분산 HashMap (TTL 없음, 영구 저장) |
| **RMapCache** | TTL/MaxIdleTime을 지원하는 분산 Map |
| **TTL** | 저장 시점부터 고정 시간 후 만료 |
| **MaxIdleTime** | 마지막 접근부터 미사용 시 만료 |
| **ZSET** | RMapCache가 만료 시간을 관리하는 내부 자료구조 |
| **EvictionScheduler** | 클라이언트에서 동작하는 만료 처리 스케줄러 |
| **EntryExpiredListener** | 만료 이벤트 콜백 리스너 |
| **RLock** | 분산 환경에서 중복 처리 방지용 분산 락 |

---

## 다음 단계

다음 글에서는 실제로 Spring Boot 프로젝트에서:

1. Redisson 의존성 추가
2. RedissonClient 설정
3. RMapCache 기본 CRUD 테스트
4. TTL 만료 이벤트 리스너 구현
5. 분산 락을 활용한 중복 처리 방지

를 직접 구현해보겠습니다.

---

## 참고 자료

- [Redisson 공식 문서](https://redisson.org/)
- [Redisson GitHub](https://github.com/redisson/redisson)
- [RMapCache API 문서](https://www.javadoc.io/doc/org.redisson/redisson/latest/org/redisson/api/RMapCache.html)
- [Redisson Spring Boot Starter](https://github.com/redisson/redisson/tree/master/redisson-spring-boot-starter)
