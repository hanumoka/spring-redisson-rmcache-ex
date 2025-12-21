# spring-redisson-rmapcache 프로젝트

## 프로젝트 개요

Spring Boot + Redisson + RMapCache 학습을 위한 실습 프로젝트

- **언어**: Java 21
- **프레임워크**: Spring Boot 4.0.1
- **빌드 도구**: Gradle
- **패키지명**: com.hanumoka.springredissonrmapcache

## 학습 목적

- Redisson RMapCache의 동작 원리 이해
- TTL, MaxIdleTime 설정 및 활용
- Near Cache 설정
- 실전 캐싱 패턴 (Cache-Aside, Write-Through 등)

## 프로젝트 구조

```
spring-redisson-rmapcache/
├── src/
│   ├── main/java/com/hanumoka/springredissonrmapcache/
│   └── main/resources/
├── docs/
│   └── Redisson_RMapCache_학습가이드.md   # 학습 가이드
├── build.gradle
└── CLAUDE.md                               # 이 파일
```

## 학습 가이드

- **경로**: `docs/Redisson_RMapCache_학습가이드.md`
- **내용**: Redis, Redisson 개요, RMapCache 사용법, TTL/MaxIdleTime, Near Cache, 실전 패턴

## 📝 블로그 작성 지원

### 사용자 블로그 작성 계획

**사용자는 학습 내용을 Notion에 블로그 형식으로 직접 작성합니다.**

#### Claude의 블로그 지원 역할
1. **블로그 초안 작성 지원**
   - 사용자가 학습한 내용을 블로그 형식으로 정리 요청 시 초안 제공
   - Notion에 붙여넣기 편한 마크다운 형식으로 작성

2. **블로그 구조 제안**
   - 들어가며 (문제 상황, 학습 동기)
   - 본론 (해결 과정, 핵심 개념)
   - 트러블슈팅 (겪은 문제와 해결 방법)
   - 마무리 (배운 점, 다음 단계)

3. **작성 시점**
   - 특정 기능 구현 완료 후
   - 트러블슈팅 해결 후
   - 사용자가 요청할 때

#### 블로그 작성 요청 예시
- "RMapCache TTL 테스트한 내용을 블로그 글로 정리해줘"
- "Near Cache 설정 과정을 블로그 글로 작성해줘"
- "방금 해결한 직렬화 에러를 트러블슈팅 블로그로 정리해줘"

## 실습 체크리스트

- [ ] Docker로 Redis 실행
- [ ] Redisson 의존성 추가
- [ ] RedissonClient Bean 설정
- [ ] RMapCache 기본 CRUD 테스트
- [ ] TTL/MaxIdleTime 동작 확인
- [ ] Cache-Aside 패턴 구현
- [ ] Near Cache 설정 (선택)

## 관련 프로젝트

- **sado_be**: 메인 학습 프로젝트 (16주 계획)
- 이 프로젝트는 RMapCache 집중 학습을 위한 별도 실습 환경
