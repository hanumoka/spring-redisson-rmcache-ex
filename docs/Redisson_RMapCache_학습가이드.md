# Spring + Redisson + RMapCache 완벽 가이드

> **학습 목표**: Redisson의 RMapCache를 이해하고 Spring Boot에서 활용하는 방법 학습
>
> **난이도**: 중급
>
> **예상 학습 시간**: 2-3시간
>
> **사전 지식**: Spring Boot 기본, Redis 기본 개념

---

## 목차

1. [Redis와 Redisson 개요](#1-redis와-redisson-개요)
2. [RMapCache란?](#2-rmapcache란)
3. [환경 구성](#3-환경-구성)
4. [RMapCache 기본 사용법](#4-rmapcache-기본-사용법)
5. [TTL과 MaxIdleTime](#5-ttl과-maxidletime)
6. [Near Cache (로컬 캐시)](#6-near-cache-로컬-캐시)
7. [실전 활용 패턴](#7-실전-활용-패턴)
8. [주의사항 및 Best Practices](#8-주의사항-및-best-practices)
9. [트러블슈팅](#9-트러블슈팅)
10. [참고 자료](#10-참고-자료)

---

## 1. Redis와 Redisson 개요

### 1.1 Redis란?

Redis(Remote Dictionary Server)는 인메모리 데이터 저장소입니다.

**핵심 특징**:
- Key-Value 저장소
- 다양한 자료구조 지원 (String, List, Set, Hash, Sorted Set 등)
- 영속성 옵션 (RDB, AOF)
- Pub/Sub 메시징
- 클러스터 모드 지원

### 1.2 Redisson이란?

Redisson은 Redis를 위한 Java 클라이언트 라이브러리입니다.

**일반 Redis 클라이언트 (Jedis, Lettuce) vs Redisson**:

| 항목 | Jedis/Lettuce | Redisson |
|------|---------------|----------|
| **추상화 수준** | 낮음 (명령어 직접 사용) | 높음 (Java 컬렉션 인터페이스) |
| **분산 자료구조** | 직접 구현 필요 | 기본 제공 (Map, Set, Queue 등) |
| **분산 락** | 직접 구현 필요 | RLock 제공 |
| **TTL 관리** | 수동 관리 | 자동 관리 (RMapCache) |
| **학습 곡선** | 낮음 | 중간 |

### 1.3 Redisson 주요 자료구조

```
Redisson 분산 컬렉션
├── RMap          - 분산 Map (TTL 없음)
├── RMapCache     - 분산 Map + TTL 지원 ⭐
├── RSet          - 분산 Set
├── RList         - 분산 List
├── RQueue        - 분산 Queue
├── RDeque        - 분산 Deque
├── RSortedSet    - 분산 Sorted Set
├── RLock         - 분산 락
└── RSemaphore    - 분산 세마포어
```

---

## 2. RMapCache란?

### 2.1 정의

`RMapCache`는 Redisson에서 제공하는 **TTL(Time-To-Live) 지원 분산 Map**입니다.

### 2.2 RMap vs RMapCache

```java
// RMap - 기본 분산 Map
RMap<String, User> map = redisson.getMap("users");
map.put("user:1", user);  // 만료되지 않음, 영구 저장

// RMapCache - TTL 지원 분산 Map
RMapCache<String, User> cache = redisson.getMapCache("userCache");
cache.put("user:1", user, 10, TimeUnit.MINUTES);  // 10분 후 자동 삭제
```

| 기능 | RMap | RMapCache |
|------|------|-----------|
| Key-Value 저장 | O | O |
| TTL (만료시간) | X | O |
| MaxIdleTime (유휴시간) | X | O |
| 자동 만료 삭제 | X | O |
| Near Cache | X | O |
| 메모리 사용량 | 적음 | 약간 더 많음 |

### 2.3 RMapCache 사용 사례

1. **세션 캐시**: 로그인 세션 정보 저장 (30분 TTL)
2. **API 응답 캐시**: 외부 API 호출 결과 캐싱 (5분 TTL)
3. **조회 데이터 캐시**: DB 조회 결과 캐싱 (1시간 TTL)
4. **토큰 저장**: JWT 리프레시 토큰 (7일 TTL)
5. **Rate Limiting**: API 호출 횟수 제한 (1분 TTL)

### 2.4 내부 동작 원리

RMapCache는 Redis의 Hash와 Sorted Set을 조합하여 구현됩니다.

```
Redis 내부 구조:
├── {name}           - Hash (실제 데이터)
├── {name}:timeout   - Sorted Set (TTL 관리)
└── {name}:idle      - Sorted Set (MaxIdleTime 관리)
```

**만료 처리 방식**:
1. **Lazy Eviction**: 조회 시점에 만료 여부 확인 후 삭제
2. **Active Eviction**: 백그라운드에서 주기적으로 만료 항목 삭제

---

## 3. 환경 구성

### 3.1 Docker로 Redis 실행

```yaml
# docker-compose.yml
version: '3.8'

services:
  redis:
    image: redis:7-alpine
    container_name: sado-redis
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    command: redis-server --appendonly yes
    networks:
      - sado-network

volumes:
  redis-data:

networks:
  sado-network:
    driver: bridge
```

```bash
# 실행
docker-compose up -d redis

# 확인
docker exec -it sado-redis redis-cli ping
# 결과: PONG
```

### 3.2 Gradle 의존성

```groovy
// build.gradle
dependencies {
    // Redisson Spring Boot Starter
    implementation 'org.redisson:redisson-spring-boot-starter:3.27.0'

    // 또는 Redisson만 사용 (Spring Boot 자동 설정 없이)
    // implementation 'org.redisson:redisson:3.27.0'
}
```

### 3.3 application.yml 설정

```yaml
# application.yml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      # password: your-password  # 필요시

# Redisson 상세 설정 (선택)
redisson:
  config: |
    singleServerConfig:
      address: "redis://localhost:6379"
      connectionPoolSize: 10
      connectionMinimumIdleSize: 5
      idleConnectionTimeout: 10000
      connectTimeout: 10000
      timeout: 3000
      retryAttempts: 3
      retryInterval: 1500
```

### 3.4 Redisson 설정 클래스

**방법 1: application.yml 기반 (간단)**

```java
// RedissonConfig.java
@Configuration
public class RedissonConfig {

    @Value("${spring.data.redis.host}")
    private String host;

    @Value("${spring.data.redis.port}")
    private int port;

    @Bean
    public RedissonClient redissonClient() {
        Config config = new Config();
        config.useSingleServer()
              .setAddress("redis://" + host + ":" + port)
              .setConnectionPoolSize(10)
              .setConnectionMinimumIdleSize(5);

        return Redisson.create(config);
    }
}
```

**방법 2: YAML 파일 기반 (상세 설정)**

```java
@Configuration
public class RedissonConfig {

    @Bean
    public RedissonClient redissonClient() throws IOException {
        Config config = Config.fromYAML(
            new ClassPathResource("redisson.yml").getInputStream()
        );
        return Redisson.create(config);
    }
}
```

```yaml
# src/main/resources/redisson.yml
singleServerConfig:
  address: "redis://localhost:6379"
  connectionPoolSize: 10
  connectionMinimumIdleSize: 5
  idleConnectionTimeout: 10000
  connectTimeout: 10000
  timeout: 3000
  retryAttempts: 3
  retryInterval: 1500
codec: !<org.redisson.codec.JsonJacksonCodec> {}
```

---

## 4. RMapCache 기본 사용법

### 4.1 RMapCache 생성

```java
@Service
@RequiredArgsConstructor
public class CacheService {

    private final RedissonClient redissonClient;

    public void example() {
        // RMapCache 생성 (이름으로 구분)
        RMapCache<String, User> userCache = redissonClient.getMapCache("userCache");
        RMapCache<Long, Study> studyCache = redissonClient.getMapCache("studyCache");

        // 같은 이름 = 같은 캐시 (분산 환경에서 공유)
        RMapCache<String, User> sameCache = redissonClient.getMapCache("userCache");
    }
}
```

### 4.2 기본 CRUD 연산

```java
@Service
@RequiredArgsConstructor
public class UserCacheService {

    private final RedissonClient redissonClient;

    private RMapCache<String, User> getUserCache() {
        return redissonClient.getMapCache("userCache");
    }

    // CREATE / UPDATE
    public void put(String userId, User user) {
        RMapCache<String, User> cache = getUserCache();

        // 1. 단순 저장 (만료 없음)
        cache.put(userId, user);

        // 2. TTL 설정하여 저장 (10분 후 만료)
        cache.put(userId, user, 10, TimeUnit.MINUTES);

        // 3. TTL + MaxIdleTime 설정
        // TTL: 1시간, MaxIdleTime: 30분 (둘 중 먼저 도달하면 만료)
        cache.put(userId, user, 1, TimeUnit.HOURS, 30, TimeUnit.MINUTES);
    }

    // READ
    public User get(String userId) {
        return getUserCache().get(userId);
    }

    // DELETE
    public void remove(String userId) {
        getUserCache().remove(userId);
    }

    // EXISTS
    public boolean exists(String userId) {
        return getUserCache().containsKey(userId);
    }

    // SIZE
    public int size() {
        return getUserCache().size();
    }

    // CLEAR ALL
    public void clearAll() {
        getUserCache().clear();
    }
}
```

### 4.3 조건부 연산

```java
public class ConditionalOperations {

    private final RMapCache<String, User> cache;

    // putIfAbsent - 키가 없을 때만 저장
    public User putIfAbsent(String userId, User user) {
        // 이미 존재하면 기존 값 반환, 없으면 저장 후 null 반환
        return cache.putIfAbsent(userId, user, 10, TimeUnit.MINUTES);
    }

    // fastPut - 이전 값 반환 없이 저장 (더 빠름)
    public void fastPut(String userId, User user) {
        // put()은 이전 값을 반환하지만, fastPut()은 반환 안 함
        cache.fastPut(userId, user, 10, TimeUnit.MINUTES);
    }

    // fastPutIfAbsent - 조건부 저장 (빠름)
    public boolean fastPutIfAbsent(String userId, User user) {
        // 저장 성공하면 true, 이미 존재하면 false
        return cache.fastPutIfAbsent(userId, user, 10, TimeUnit.MINUTES);
    }

    // replace - 키가 있을 때만 교체
    public User replace(String userId, User newUser) {
        return cache.replace(userId, newUser, 10, TimeUnit.MINUTES);
    }

    // computeIfAbsent - 없으면 계산 후 저장
    public User computeIfAbsent(String userId) {
        return cache.computeIfAbsent(userId, key -> {
            // DB에서 조회하여 캐시에 저장
            return userRepository.findById(key).orElse(null);
        });
    }
}
```

### 4.4 벌크 연산

```java
public class BulkOperations {

    private final RMapCache<String, User> cache;

    // 여러 개 한번에 저장
    public void putAll(Map<String, User> users) {
        cache.putAll(users);

        // TTL 포함 (각 항목에 동일한 TTL 적용)
        cache.putAll(users, 10, TimeUnit.MINUTES);
    }

    // 여러 개 한번에 조회
    public Map<String, User> getAll(Set<String> userIds) {
        return cache.getAll(userIds);
    }

    // 모든 키 조회
    public Set<String> getAllKeys() {
        return cache.keySet();
    }

    // 모든 값 조회
    public Collection<User> getAllValues() {
        return cache.values();
    }

    // 모든 Entry 조회
    public Set<Map.Entry<String, User>> getAllEntries() {
        return cache.entrySet();
    }
}
```

---

## 5. TTL과 MaxIdleTime

### 5.1 TTL (Time-To-Live)

**정의**: 데이터가 저장된 시점부터 만료까지의 시간

```java
// 10분 TTL
cache.put("key", value, 10, TimeUnit.MINUTES);

// 시간 흐름
// 0분: 저장
// 5분: 조회 가능
// 10분: 조회 가능
// 10분 1초: 만료 (자동 삭제)
```

```
저장 ────────────────────────────────> 만료
 │                                      │
 0분                                   10분
      (조회해도 TTL은 변하지 않음)
```

### 5.2 MaxIdleTime

**정의**: 마지막 접근(조회/수정) 이후 유휴 시간

```java
// 10분 MaxIdleTime
cache.put("key", value, 0, TimeUnit.MINUTES, 10, TimeUnit.MINUTES);
// 첫 번째 인자 0 = TTL 없음

// 시간 흐름
// 0분: 저장
// 5분: 조회 → idle 타이머 리셋
// 15분: 조회 → idle 타이머 리셋
// 25분 1초: 만료 (마지막 조회 후 10분 경과)
```

```
저장      조회       조회                        만료
 │         │          │                          │
 0분      5분        15분                       25분
          └─ idle 리셋 ─┘─────── 10분 경과 ───────┘
```

### 5.3 TTL + MaxIdleTime 조합

```java
// TTL: 1시간, MaxIdleTime: 30분
cache.put("key", value, 1, TimeUnit.HOURS, 30, TimeUnit.MINUTES);
```

**만료 조건**: TTL 또는 MaxIdleTime 중 **먼저 도달하는 조건**

**시나리오 1**: 활발하게 사용하는 경우
```
저장   조회   조회   조회   조회   만료(TTL)
 │      │      │      │      │      │
0분   20분   40분   50분   55분   60분
              (MaxIdleTime 계속 리셋, TTL로 만료)
```

**시나리오 2**: 사용 안 하는 경우
```
저장                  만료(Idle)
 │                       │
0분                     30분
    (30분 동안 조회 없음, MaxIdleTime으로 만료)
```

### 5.4 TTL/MaxIdleTime 조회 및 갱신

```java
// 남은 TTL 조회 (밀리초)
long remainingTTL = cache.remainTimeToLive("key");

// TTL 갱신 (기존 값 유지, TTL만 변경)
cache.updateEntryExpiration("key", 20, TimeUnit.MINUTES, 10, TimeUnit.MINUTES);

// 또는 다시 put (값도 함께 갱신)
User user = cache.get("key");
cache.put("key", user, 20, TimeUnit.MINUTES);
```

### 5.5 사용 사례별 권장 설정

| 사용 사례 | TTL | MaxIdleTime | 이유 |
|----------|-----|-------------|------|
| **로그인 세션** | 24시간 | 30분 | 비활동 시 빠른 만료, 최대 24시간 |
| **API 응답 캐시** | 5분 | - | 데이터 신선도 보장 |
| **사용자 프로필** | 1시간 | 15분 | 자주 접근하면 유지, 아니면 삭제 |
| **리프레시 토큰** | 7일 | - | 고정 만료, 활동과 무관 |
| **조회수 카운터** | 1분 | - | Rate Limiting용 |

---

## 6. Near Cache (로컬 캐시)

### 6.1 Near Cache란?

Near Cache는 **Redis 앞에 로컬 메모리 캐시**를 두는 방식입니다.

```
[애플리케이션]
     │
     ▼
[Near Cache (로컬)]  ← 1차 캐시 (빠름, JVM 메모리)
     │
     ▼
[Redis (원격)]       ← 2차 캐시 (네트워크 통신)
```

**장점**:
- 네트워크 왕복 없이 로컬에서 즉시 응답
- Redis 부하 감소
- 지연 시간 대폭 감소 (< 1ms)

**단점**:
- JVM 메모리 사용 증가
- 분산 환경에서 일관성 관리 필요

### 6.2 Near Cache 설정

```java
@Configuration
public class NearCacheConfig {

    @Bean
    public RedissonClient redissonClient() {
        Config config = new Config();
        config.useSingleServer()
              .setAddress("redis://localhost:6379");

        return Redisson.create(config);
    }

    // Near Cache가 활성화된 RMapCache 생성
    public <K, V> RMapCache<K, V> createNearCachedMap(
            RedissonClient redisson,
            String name,
            Class<K> keyClass,
            Class<V> valueClass) {

        // Near Cache 설정
        NearCacheConfiguration<K, V> nearCacheConfig =
            NearCacheConfiguration.<K, V>builder()
                .maxSize(10000)                           // 로컬 캐시 최대 크기
                .ttl(5, TimeUnit.MINUTES)                 // 로컬 TTL
                .maxIdle(1, TimeUnit.MINUTES)             // 로컬 MaxIdle
                .evictionPolicy(NearCacheEvictionPolicy.LRU)  // LRU 정책
                .build();

        // MapCache 옵션
        MapCacheOptions<K, V> options = MapCacheOptions.<K, V>defaults()
                .nearCache(nearCacheConfig);

        return redisson.getMapCache(name, options);
    }
}
```

### 6.3 Near Cache 사용 예시

```java
@Service
public class StudyCacheService {

    private final RMapCache<Long, StudyDTO> studyCache;

    public StudyCacheService(RedissonClient redissonClient) {
        // Near Cache 설정
        NearCacheConfiguration<Long, StudyDTO> nearCacheConfig =
            NearCacheConfiguration.<Long, StudyDTO>builder()
                .maxSize(1000)
                .ttl(5, TimeUnit.MINUTES)
                .evictionPolicy(NearCacheEvictionPolicy.LRU)
                .cacheUpdateMode(NearCacheUpdateMode.READ_INVALIDATE)
                .build();

        MapCacheOptions<Long, StudyDTO> options = MapCacheOptions.<Long, StudyDTO>defaults()
                .nearCache(nearCacheConfig);

        this.studyCache = redissonClient.getMapCache("studyCache", options);
    }

    public StudyDTO getStudy(Long studyId) {
        // 1차: Near Cache 조회 (로컬)
        // 2차: Redis 조회 (없으면)
        return studyCache.get(studyId);
    }
}
```

### 6.4 Near Cache 업데이트 모드

```java
NearCacheConfiguration.<K, V>builder()
    .cacheUpdateMode(NearCacheUpdateMode.READ_INVALIDATE)  // 기본값
    .build();
```

| 모드 | 동작 | 적합한 상황 |
|------|------|-------------|
| `READ_INVALIDATE` | 다른 노드에서 변경 시 로컬 캐시 무효화 | 쓰기 적은 경우 |
| `READ_UPDATE` | 다른 노드에서 변경 시 로컬 캐시도 업데이트 | 쓰기 많은 경우 |

### 6.5 Near Cache 주의사항

```java
// 주의: Near Cache 사용 시 메모리 관리
NearCacheConfiguration.<K, V>builder()
    .maxSize(10000)              // 반드시 최대 크기 설정
    .ttl(5, TimeUnit.MINUTES)    // 로컬 TTL 설정 (Redis TTL과 별도)
    .build();

// Near Cache의 TTL은 Redis TTL보다 짧게 설정 권장
// Redis TTL: 1시간, Near Cache TTL: 5분
// → 5분마다 Redis에서 최신 데이터 가져옴
```

---

## 7. 실전 활용 패턴

### 7.1 Cache-Aside 패턴 (가장 일반적)

```java
@Service
@RequiredArgsConstructor
public class StudyService {

    private final StudyRepository studyRepository;
    private final RedissonClient redissonClient;

    private RMapCache<Long, StudyDTO> getCache() {
        return redissonClient.getMapCache("studyCache");
    }

    public StudyDTO getStudy(Long studyId) {
        RMapCache<Long, StudyDTO> cache = getCache();

        // 1. 캐시 조회
        StudyDTO cached = cache.get(studyId);
        if (cached != null) {
            return cached;  // Cache Hit
        }

        // 2. Cache Miss → DB 조회
        Study study = studyRepository.findById(studyId)
                .orElseThrow(() -> new StudyNotFoundException(studyId));

        StudyDTO dto = StudyDTO.from(study);

        // 3. 캐시에 저장
        cache.put(studyId, dto, 1, TimeUnit.HOURS);

        return dto;
    }

    @Transactional
    public StudyDTO updateStudy(Long studyId, StudyUpdateRequest request) {
        // 1. DB 업데이트
        Study study = studyRepository.findById(studyId)
                .orElseThrow(() -> new StudyNotFoundException(studyId));
        study.update(request);
        studyRepository.save(study);

        // 2. 캐시 무효화 (또는 갱신)
        getCache().remove(studyId);
        // 또는: getCache().put(studyId, StudyDTO.from(study), 1, TimeUnit.HOURS);

        return StudyDTO.from(study);
    }

    @Transactional
    public void deleteStudy(Long studyId) {
        // 1. DB 삭제
        studyRepository.deleteById(studyId);

        // 2. 캐시 삭제
        getCache().remove(studyId);
    }
}
```

### 7.2 Write-Through 패턴

```java
@Service
@RequiredArgsConstructor
public class UserService {

    private final UserRepository userRepository;
    private final RedissonClient redissonClient;

    private RMapCache<Long, UserDTO> getCache() {
        return redissonClient.getMapCache("userCache");
    }

    @Transactional
    public UserDTO createUser(UserCreateRequest request) {
        // 1. DB 저장
        User user = new User(request.getName(), request.getEmail());
        User saved = userRepository.save(user);

        UserDTO dto = UserDTO.from(saved);

        // 2. 캐시에도 동시 저장 (Write-Through)
        getCache().put(saved.getId(), dto, 1, TimeUnit.HOURS);

        return dto;
    }
}
```

### 7.3 세션 캐시

```java
@Service
@RequiredArgsConstructor
public class SessionCacheService {

    private final RedissonClient redissonClient;

    private RMapCache<String, SessionData> getSessionCache() {
        return redissonClient.getMapCache("sessions");
    }

    // 세션 생성
    public void createSession(String sessionId, SessionData data) {
        // TTL: 24시간, MaxIdleTime: 30분
        getSessionCache().put(sessionId, data, 24, TimeUnit.HOURS, 30, TimeUnit.MINUTES);
    }

    // 세션 조회 (조회 시 MaxIdleTime 자동 리셋)
    public SessionData getSession(String sessionId) {
        return getSessionCache().get(sessionId);
    }

    // 세션 갱신 (로그인 연장)
    public void refreshSession(String sessionId) {
        SessionData data = getSessionCache().get(sessionId);
        if (data != null) {
            // TTL 갱신
            getSessionCache().put(sessionId, data, 24, TimeUnit.HOURS, 30, TimeUnit.MINUTES);
        }
    }

    // 로그아웃
    public void invalidateSession(String sessionId) {
        getSessionCache().remove(sessionId);
    }
}
```

### 7.4 API Rate Limiting

```java
@Service
@RequiredArgsConstructor
public class RateLimitService {

    private final RedissonClient redissonClient;

    private RMapCache<String, Integer> getRateLimitCache() {
        return redissonClient.getMapCache("rateLimit");
    }

    /**
     * API 호출 가능 여부 확인 및 카운트 증가
     * @param clientId 클라이언트 식별자
     * @param maxRequests 분당 최대 요청 수
     * @return 호출 가능하면 true
     */
    public boolean allowRequest(String clientId, int maxRequests) {
        String key = "rate:" + clientId;
        RMapCache<String, Integer> cache = getRateLimitCache();

        Integer currentCount = cache.get(key);

        if (currentCount == null) {
            // 첫 요청: 카운트 1로 시작, 1분 TTL
            cache.put(key, 1, 1, TimeUnit.MINUTES);
            return true;
        }

        if (currentCount >= maxRequests) {
            // 제한 초과
            return false;
        }

        // 카운트 증가 (TTL 유지)
        cache.put(key, currentCount + 1, 1, TimeUnit.MINUTES);
        return true;
    }

    // 남은 요청 횟수 조회
    public int getRemainingRequests(String clientId, int maxRequests) {
        String key = "rate:" + clientId;
        Integer currentCount = getRateLimitCache().get(key);
        return currentCount == null ? maxRequests : Math.max(0, maxRequests - currentCount);
    }
}
```

### 7.5 분산 환경에서 캐시 동기화

```java
@Service
@RequiredArgsConstructor
public class DistributedCacheService {

    private final RedissonClient redissonClient;

    private RMapCache<Long, ProductDTO> getProductCache() {
        return redissonClient.getMapCache("productCache");
    }

    // 캐시 이벤트 리스너 등록
    @PostConstruct
    public void setupListeners() {
        RMapCache<Long, ProductDTO> cache = getProductCache();

        // 항목 추가 이벤트
        cache.addListener(new EntryCreatedListener<Long, ProductDTO>() {
            @Override
            public void onCreated(EntryEvent<Long, ProductDTO> event) {
                log.info("Cache entry created: key={}", event.getKey());
            }
        });

        // 항목 만료 이벤트
        cache.addListener(new EntryExpiredListener<Long, ProductDTO>() {
            @Override
            public void onExpired(EntryEvent<Long, ProductDTO> event) {
                log.info("Cache entry expired: key={}", event.getKey());
            }
        });

        // 항목 삭제 이벤트
        cache.addListener(new EntryRemovedListener<Long, ProductDTO>() {
            @Override
            public void onRemoved(EntryEvent<Long, ProductDTO> event) {
                log.info("Cache entry removed: key={}", event.getKey());
            }
        });
    }
}
```

---

## 8. 주의사항 및 Best Practices

### 8.1 직렬화

```java
// 기본: Jackson JSON 직렬화
// 캐시에 저장할 객체는 직렬화 가능해야 함

@Data
@NoArgsConstructor
@AllArgsConstructor
public class UserDTO implements Serializable {
    private Long id;
    private String name;
    private String email;
    private LocalDateTime createdAt;
}

// 커스텀 코덱 설정 (선택)
Config config = new Config();
config.setCodec(new JsonJacksonCodec());  // 기본값
// 또는
config.setCodec(new Kryo5Codec());        // 더 빠름, 더 작은 크기
// 또는
config.setCodec(new MarshallingCodec()); // Java 직렬화
```

### 8.2 Key 설계

```java
// Bad: 너무 단순한 키
cache.put("1", user);  // 어떤 엔티티인지 불명확

// Good: 네임스페이스 포함
cache.put("user:1", user);
cache.put("study:123", study);
cache.put("session:abc-def", session);

// 복합 키가 필요한 경우
String key = String.format("study:%d:series:%d", studyId, seriesId);
cache.put(key, series);
```

### 8.3 TTL 설정 전략

```java
// Bad: 너무 긴 TTL (메모리 낭비)
cache.put("key", value, 30, TimeUnit.DAYS);

// Bad: 너무 짧은 TTL (캐시 효과 없음)
cache.put("key", value, 1, TimeUnit.SECONDS);

// Good: 데이터 특성에 맞는 TTL
// - 자주 변경되는 데이터: 5-15분
// - 거의 안 변하는 데이터: 1-24시간
// - 세션 데이터: TTL 24시간 + MaxIdleTime 30분
```

### 8.4 Null 처리

```java
// 주의: null 값 저장 시
cache.put("key", null);  // NullPointerException 발생 가능

// 해결책 1: null 체크
if (value != null) {
    cache.put("key", value, 10, TimeUnit.MINUTES);
}

// 해결책 2: Optional 사용
public Optional<User> getUser(Long id) {
    User user = cache.get("user:" + id);
    return Optional.ofNullable(user);
}

// 해결책 3: Null Object 패턴
public static final User NULL_USER = new User(-1L, "NULL", "NULL");

User cached = cache.get("user:" + id);
if (cached == null) {
    User fromDb = userRepository.findById(id).orElse(null);
    cache.put("user:" + id, fromDb != null ? fromDb : NULL_USER, 10, TimeUnit.MINUTES);
    return fromDb;
}
return cached.getId() == -1L ? null : cached;
```

### 8.5 대용량 데이터 주의

```java
// Bad: 큰 객체 저장
List<Study> allStudies = studyRepository.findAll();  // 10만 건
cache.put("allStudies", allStudies);  // 메모리 폭발!

// Good: 개별 항목으로 저장
for (Study study : studies) {
    cache.put("study:" + study.getId(), StudyDTO.from(study), 1, TimeUnit.HOURS);
}

// Good: ID 목록만 저장
List<Long> studyIds = studies.stream().map(Study::getId).toList();
cache.put("studyIds:page:0", studyIds, 5, TimeUnit.MINUTES);
```

### 8.6 트랜잭션과 캐시

```java
@Service
@RequiredArgsConstructor
public class StudyService {

    @Transactional
    public void updateStudy(Long id, StudyUpdateRequest request) {
        Study study = studyRepository.findById(id).orElseThrow();
        study.update(request);
        studyRepository.save(study);

        // 주의: 트랜잭션 커밋 전에 캐시 갱신하면
        // 트랜잭션 롤백 시 캐시와 DB 불일치 발생!

        // Bad: 트랜잭션 내부에서 캐시 갱신
        cache.put("study:" + id, StudyDTO.from(study));
    }

    // Good: 트랜잭션 완료 후 캐시 갱신
    @Transactional
    public StudyDTO updateStudy(Long id, StudyUpdateRequest request) {
        Study study = studyRepository.findById(id).orElseThrow();
        study.update(request);
        return StudyDTO.from(studyRepository.save(study));
    }

    // 서비스 호출 후 캐시 갱신
    public void updateStudyWithCache(Long id, StudyUpdateRequest request) {
        StudyDTO updated = updateStudy(id, request);
        cache.put("study:" + id, updated, 1, TimeUnit.HOURS);
    }
}

// 또는 이벤트 기반 캐시 갱신
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void handleStudyUpdated(StudyUpdatedEvent event) {
    cache.put("study:" + event.getStudyId(), event.getStudyDTO(), 1, TimeUnit.HOURS);
}
```

---

## 9. 트러블슈팅

### 9.1 연결 실패

**에러**:
```
org.redisson.client.RedisConnectionException: Unable to connect to Redis server
```

**원인 및 해결**:
```bash
# 1. Redis 실행 확인
docker ps | grep redis

# 2. Redis 접속 테스트
docker exec -it sado-redis redis-cli ping

# 3. 방화벽/포트 확인
telnet localhost 6379

# 4. application.yml 설정 확인
spring:
  data:
    redis:
      host: localhost  # Docker 내부에서는 redis 또는 컨테이너 IP
      port: 6379
```

### 9.2 직렬화 오류

**에러**:
```
com.fasterxml.jackson.databind.exc.InvalidDefinitionException:
Cannot construct instance of `java.time.LocalDateTime`
```

**해결**:
```java
// Jackson에서 Java 8 날짜 타입 지원 설정
@Configuration
public class RedissonConfig {

    @Bean
    public RedissonClient redissonClient() {
        Config config = new Config();
        config.useSingleServer().setAddress("redis://localhost:6379");

        // Jackson ObjectMapper 커스텀 설정
        ObjectMapper mapper = new ObjectMapper();
        mapper.registerModule(new JavaTimeModule());
        mapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);

        config.setCodec(new JsonJacksonCodec(mapper));

        return Redisson.create(config);
    }
}
```

### 9.3 메모리 부족

**에러**:
```
MISCONF Redis is configured to save RDB snapshots, but is currently not able to persist on disk.
```

**해결**:
```bash
# Redis 메모리 설정 확인
docker exec -it sado-redis redis-cli INFO memory

# maxmemory 설정 (redis.conf 또는 명령어)
docker exec -it sado-redis redis-cli CONFIG SET maxmemory 256mb
docker exec -it sado-redis redis-cli CONFIG SET maxmemory-policy allkeys-lru
```

### 9.4 TTL이 적용되지 않음

**문제**: `put()` 후에도 데이터가 만료되지 않음

**원인 및 해결**:
```java
// 원인 1: TTL 단위 착각
cache.put("key", value, 10, TimeUnit.SECONDS);  // 10초 (아닌 10분)

// 원인 2: RMap 사용 (RMapCache 아님)
RMap<String, User> map = redissonClient.getMap("users");  // TTL 미지원!
RMapCache<String, User> cache = redissonClient.getMapCache("users");  // 올바름

// 확인: Redis에서 TTL 확인
docker exec -it sado-redis redis-cli TTL "users:key"
```

### 9.5 동시성 문제

**문제**: 여러 인스턴스에서 동시에 캐시 갱신 시 경쟁 상태

**해결**: 분산 락 사용
```java
public StudyDTO getStudyWithLock(Long studyId) {
    String lockKey = "lock:study:" + studyId;
    RLock lock = redissonClient.getLock(lockKey);

    try {
        // 락 획득 (최대 10초 대기, 30초 후 자동 해제)
        if (lock.tryLock(10, 30, TimeUnit.SECONDS)) {
            try {
                // 캐시 확인
                StudyDTO cached = cache.get(studyId);
                if (cached != null) {
                    return cached;
                }

                // DB 조회 및 캐시 저장
                Study study = studyRepository.findById(studyId).orElseThrow();
                StudyDTO dto = StudyDTO.from(study);
                cache.put(studyId, dto, 1, TimeUnit.HOURS);
                return dto;
            } finally {
                lock.unlock();
            }
        }
        throw new RuntimeException("Failed to acquire lock");
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
        throw new RuntimeException(e);
    }
}
```

---

## 10. 참고 자료

### 공식 문서
- [Redisson 공식 문서](https://redisson.org/documentation.html)
- [Redisson GitHub](https://github.com/redisson/redisson)
- [Redisson RMapCache API](https://www.javadoc.io/doc/org.redisson/redisson/latest/org/redisson/api/RMapCache.html)
- [Redis 공식 문서](https://redis.io/documentation)

### 추천 학습 자료
- Redisson Wiki - RMapCache: https://github.com/redisson/redisson/wiki/7.-Distributed-collections#712-map
- Redis University (무료 강의): https://university.redis.com/

### 관련 SADO 프로젝트 문서
- `docs/08_Redis_활용_요구사항.md` - Redis 분산락 및 캐싱 시나리오
- `docs/07_최종_구현_계획.md` - Week 14 Redis 학습 계획

---

## 핵심 정리

### 3줄 요약
1. **RMapCache = RMap + TTL 지원**: Entry별 만료 시간 설정 가능
2. **TTL vs MaxIdleTime**: TTL은 고정 만료, MaxIdleTime은 접근 시 리셋
3. **Near Cache**: 로컬 캐시로 성능 향상, 분산 환경 일관성 주의

### 체크리스트
- [ ] Docker로 Redis 실행
- [ ] Spring Boot + Redisson 의존성 추가
- [ ] RedissonClient Bean 설정
- [ ] RMapCache 기본 CRUD 테스트
- [ ] TTL/MaxIdleTime 동작 확인
- [ ] Cache-Aside 패턴 구현
- [ ] Near Cache 설정 (선택)

---

**작성일**: 2025-12-21
**다음 단계**: 직접 실습 후 질문하기
