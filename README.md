# Portal Universe 설정 저장소

**Spring Cloud Config Server**를 위한 중앙화된 설정 저장소입니다. Portal Universe 마이크로서비스들의 모든 환경별 설정을 Git으로 관리합니다.

## 📋 개요

이 저장소는 `portal-universe` 프로젝트의 설정 서버(Config Server)에서 사용됩니다. **환경별 설정(local, docker, kubernetes)** 과 **서비스별 설정**을 분리하여 관리함으로써, 코드와 설정을 완전히 분리하고 배포 파이프라인을 효율화합니다.

### 핵심 특징

✅ **중앙화된 설정 관리** - 모든 서비스의 설정을 한 곳에서 관리  
✅ **환경별 설정 분리** - local, docker, kubernetes 환경별 독립적인 설정  
✅ **Git 기반 버전 관리** - 설정 변경 이력 추적 및 롤백 가능  
✅ **동적 설정 갱신** - 애플리케이션 재시작 없이 설정 적용 (선택적)  
✅ **프로필 기반 구성** - Spring Profiles을 통한 유연한 설정 로드  

---

## 📁 저장소 구조

```
portal-universe-config-repo/
├── application.yml                    # 모든 환경의 기본값 (공통)
├── application-local.yml              # 로컬 개발 환경 공통 설정
├── application-docker.yml             # Docker 환경 공통 설정
├── application-kubernetes.yml         # Kubernetes 환경 공통 설정
│
├── api-gateway.yml                    # API 게이트웨이 기본 설정
├── api-gateway-local.yml              # API 게이트웨이 로컬 설정
├── api-gateway-docker.yml             # API 게이트웨이 Docker 설정
├── api-gateway-kubernetes.yml         # API 게이트웨이 K8s 설정
│
├── auth-service.yml                   # 인증 서비스 기본 설정
├── auth-service-local.yml             # 인증 서비스 로컬 설정
├── auth-service-docker.yml            # 인증 서비스 Docker 설정
├── auth-service-kubernetes.yml        # 인증 서비스 K8s 설정
│
├── blog-service.yml                   # 블로그 서비스 기본 설정
├── blog-service-local.yml             # 블로그 서비스 로컬 설정
├── blog-service-docker.yml            # 블로그 서비스 Docker 설정
├── blog-service-kubernetes.yml        # 블로그 서비스 K8s 설정
│
├── shopping-service.yml               # 쇼핑 서비스 기본 설정
├── shopping-service-local.yml         # 쇼핑 서비스 로컬 설정
├── shopping-service-docker.yml        # 쇼핑 서비스 Docker 설정
├── shopping-service-kubernetes.yml    # 쇼핑 서비스 K8s 설정
│
├── notification-service.yml           # 알림 서비스 기본 설정
├── notification-service-local.yml     # 알림 서비스 로컬 설정
├── notification-service-docker.yml    # 알림 서비스 Docker 설정
└── notification-service-kubernetes.yml # 알림 서비스 K8s 설정
```

---

## 🔄 설정 로드 순서

Spring Cloud Config Server는 다음 순서로 설정을 로드합니다 (마지막이 우선순위 최상):

```
1. application.yml (기본값, 모든 환경)
   ↓
2. application-{profile}.yml (환경별 공통)
   예: application-docker.yml, application-kubernetes.yml
   ↓
3. {service-name}.yml (서비스 기본값)
   예: auth-service.yml
   ↓
4. {service-name}-{profile}.yml (서비스 환경별)
   예: auth-service-docker.yml, auth-service-kubernetes.yml
```

**결과**: 뒤의 설정이 앞의 설정을 덮어씁니다.

### 예시: Auth Service Docker 환경

```yaml
# 1단계: 기본값 로드
spring.jpa.open-in-view: false  # application.yml

# 2단계: Docker 공통 설정으로 덮어쓰기
eureka.client.service-url.defaultZone: http://discovery-service:8761/eureka  
spring.kafka.bootstrap-servers: kafka:29092

# 3단계: Auth Service 기본값으로 덮어쓰기
spring.datasource.url: jdbc:mysql://...

# 4단계: Auth Service Docker 설정으로 최종 덮어쓰기
spring.datasource.url: jdbc:mysql://mysql-db:3306/auth_db
```

---

## 📝 설정 파일 상세 설명

### 1. 공통 설정 파일

#### `application.yml` (모든 환경의 기본값)
```yaml
spring:
  jpa:
    open-in-view: false  # 지연 로딩 문제 방지
  kafka:
    producer:
      acks: all  # 모든 레플리카에 기록되어야 확인
      retries: 3  # 재시도 횟수

logging:
  pattern:
    level: "[${spring.application.name},%X{traceId},%X{spanId}]"  # 분산 추적용
```

---

### 2. 환경별 공통 설정

#### `application-local.yml` (로컬 개발)
- **데이터베이스**: localhost 접속
- **서비스 디스커버리**: Eureka 비활성화 또는 로컬 설정
- **로깅**: DEBUG 레벨로 상세 로깅
- **보안**: SSL 비활성화

#### `application-docker.yml` (Docker Compose)
```yaml
eureka:
  client:
    service-url:
      defaultZone: http://discovery-service:8761/eureka

spring:
  kafka:
    bootstrap-servers: kafka:29092  # Docker 네트워크 호스트명

management:
  zipkin:
    tracing:
      endpoint: http://zipkin:9411/api/v2/spans  # Zipkin 추적
```

**특징**:
- 컨테이너 간 통신 (Docker 네트워크)
- Eureka로 자동 서비스 디스커버리
- 분산 추적 활성화
- Prometheus 메트릭 수집 활성화

#### `application-kubernetes.yml` (Kubernetes)
- **서비스 디스커버리**: Kubernetes DNS 활용
- **데이터베이스**: Kubernetes Service로 접속
- **설정**: ConfigMap/Secret 통합
- **모니터링**: 프로덕션급 설정

---

### 3. 서비스별 설정

#### `api-gateway.yml` (API 게이트웨이)
**주요 기능**:
- **라우팅 규칙**: 서비스별 경로 매핑
- **필터**: 요청/응답 필터링
- **Circuit Breaker**: 장애 격리
- **OIDC/OAuth2**: 인증 서비스 연동

**라우팅 예시**:
```yaml
spring.cloud.gateway.routes:
  - id: auth-service-api
    uri: lb://auth-service
    predicates:
      - Path=/api/auth/**
    filters:
      - StripPrefix=2
      - CircuitBreaker:
          fallbackUri: forward:/fallback/auth
```

#### `auth-service.yml` (인증 서비스)
```yaml
spring:
  security:
    oauth2:
      authorizationserver:
        issuer: http://localhost:8080/auth-service
```

**환경별 설정**:
- **local**: localhost 접속, 자세한 로깅
- **docker**: mysql-db 호스트명, 표준 포트
- **kubernetes**: MySQL Service DNS, 보안 강화

---

## 🚀 설정 적용 방법

### 1. Config Server가 저장소를 로드하는 방식

서비스가 시작될 때, **Config Server**에 다음 URL로 요청합니다:

```bash
GET http://config-server:8888/{service-name}/{profile}
```

**예시**:
```bash
# Auth Service Docker 환경
http://localhost:8888/auth-service/docker

# Blog Service Local 환경
http://localhost:8888/blog-service/local

# API Gateway Production 환경
http://localhost:8888/api-gateway/production
```

### 2. Config Server 응답 형식

서버는 모든 적용 가능한 설정을 병합하여 반환합니다:

```json
{
  "name": "auth-service",
  "profiles": ["docker"],
  "label": "main",
  "version": "abc123...",
  "propertySources": [
    {
      "name": "application.yml",
      "source": { "spring.jpa.open-in-view": false, ... }
    },
    {
      "name": "application-docker.yml",
      "source": { "eureka.client.service-url.defaultZone": "...", ... }
    },
    {
      "name": "auth-service.yml",
      "source": { "spring.datasource.url": "...", ... }
    },
    {
      "name": "auth-service-docker.yml",
      "source": { "spring.datasource.url": "...", ... }
    }
  ]
}
```

### 3. 애플리케이션에서 설정 로드

#### Spring Boot 시작 시

서비스의 `bootstrap.yml`에 Config Server 설정:

```yaml
spring:
  cloud:
    config:
      uri: http://config-service:8888
      name: auth-service
      profile: docker  # SPRING_PROFILES_ACTIVE 환경 변수로도 설정 가능
```

#### 동적 설정 갱신

Config Server 설정 변경 후 애플리케이션 재시작 없이 적용:

```bash
# 설정 새로고침
curl -X POST http://localhost:8080/actuator/refresh

# 또는 @RefreshScope 애노테이션으로 관리되는 Bean만 갱신
```

---

## 🔧 환경별 설정 선택

### Local 환경 (로컬 개발)

```bash
# 방법 1: 환경 변수 설정
export SPRING_PROFILES_ACTIVE=local

# 방법 2: application.yml에 설정
spring:
  profiles:
    active: local

# 방법 3: 명령어로 설정
./gradlew bootRun --args='--spring.profiles.active=local'
```

**적용되는 설정**:
```
application.yml
+ application-local.yml
+ {service-name}.yml
+ {service-name}-local.yml
```

### Docker Compose 환경

```bash
# docker-compose.yml에서 환경 변수 설정
environment:
  - SPRING_PROFILES_ACTIVE=docker
  - SPRING_CLOUD_CONFIG_URI=http://config-service:8888
```

**적용되는 설정**:
```
application.yml
+ application-docker.yml
+ {service-name}.yml
+ {service-name}-docker.yml
```

### Kubernetes 환경

```yaml
# k8s/services/auth-service-deployment.yml
containers:
  - name: auth-service
    env:
      - name: SPRING_PROFILES_ACTIVE
        value: kubernetes
      - name: SPRING_CLOUD_CONFIG_URI
        value: http://config-service:8888
```

**적용되는 설정**:
```
application.yml
+ application-kubernetes.yml
+ {service-name}.yml
+ {service-name}-kubernetes.yml
```

---

## 📊 주요 설정값 정리

### 데이터베이스 연결

| 환경 | 호스트 | 포트 | URL |
|------|--------|------|-----|
| **Local** | localhost | 3307 | `jdbc:mysql://localhost:3307/...` |
| **Docker** | mysql-db | 3306 | `jdbc:mysql://mysql-db:3306/...` |
| **K8s** | mysql.default.svc.cluster.local | 3306 | `jdbc:mysql://mysql.default.svc.cluster.local:3306/...` |

### Eureka Service Discovery

| 환경 | 설정 |
|------|------|
| **Local** | Disabled 또는 http://localhost:8761/eureka |
| **Docker** | `http://discovery-service:8761/eureka` |
| **K8s** | `http://discovery-service.default.svc.cluster.local:8761/eureka` |

### Kafka Bootstrap Servers

| 환경 | 설정 |
|------|------|
| **Local** | `localhost:9092` |
| **Docker** | `kafka:29092` (내부) / `localhost:9092` (외부) |
| **K8s** | `kafka.default.svc.cluster.local:9092` |

### Zipkin 분산 추적

| 환경 | 엔드포인트 |
|------|----------|
| **Local** | `http://localhost:9411/api/v2/spans` |
| **Docker** | `http://zipkin:9411/api/v2/spans` |
| **K8s** | `http://zipkin.default.svc.cluster.local:9411/api/v2/spans` |

---

## 🔐 보안 정보 관리

### 민감한 정보 처리

비밀번호, 토큰, API 키 등의 민감한 정보는 **환경 변수**로 주입합니다:

```yaml
spring:
  datasource:
    username: ${DB_USER:laze}  # 기본값은 'laze'
    password: ${DB_PASSWORD:password}  # 환경 변수로 오버라이드
```

### 환경 변수 설정 방법

#### Local 환경
```bash
export DB_USER=myuser
export DB_PASSWORD=mypassword
./gradlew bootRun
```

#### Docker 환경
```yaml
# docker-compose.yml
environment:
  - DB_USER=laze
  - DB_PASSWORD=password
```

#### Kubernetes 환경
```yaml
# k8s/secret.yml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  DB_USER: bGF6ZQ==  # base64 encoded
  DB_PASSWORD: cGFzc3dvcmQ=
```

---

## 📝 설정 추가 및 수정

### 새로운 서비스 추가

1. **설정 파일 생성**:
```bash
touch new-service.yml
touch new-service-local.yml
touch new-service-docker.yml
touch new-service-kubernetes.yml
```

2. **기본 설정 작성** (`new-service.yml`):
```yaml
spring:
  application:
    name: new-service
  
server:
  port: 8085
```

3. **환경별 설정 작성** (`new-service-docker.yml`):
```yaml
spring:
  datasource:
    url: jdbc:mysql://mysql-db:3306/new_db
    username: ${DB_USER:laze}
    password: ${DB_PASSWORD:password}
```

4. **Git에 커밋**:
```bash
git add new-service*.yml
git commit -m "Add configuration for new-service"
git push origin main
```

### 기존 설정 수정

1. **파일 수정**:
```bash
vim auth-service-docker.yml
# 설정 변경
```

2. **변경사항 커밋**:
```bash
git add auth-service-docker.yml
git commit -m "Update auth-service Docker configuration"
git push origin main
```

3. **애플리케이션에 적용**:
```bash
# Config Server가 자동으로 변경사항 감지
# 또는 수동 갱신
curl -X POST http://api-gateway:8080/actuator/refresh
```

---

## 🔍 설정 검증 및 테스트

### Config Server 설정 확인

```bash
# 특정 서비스의 설정 조회
curl http://localhost:8888/auth-service/docker

# 결과 예시
{
  "name": "auth-service",
  "profiles": ["docker"],
  "propertySources": [...]
}
```

### 서비스가 올바른 설정을 로드했는지 확인

```bash
# Actuator endpoints 확인
curl http://localhost:8081/actuator/env | grep -A5 spring.datasource

# 또는 로그 확인
docker-compose logs auth-service | grep "datasource"
```

### 프로필별 설정 테스트

```bash
# Local 프로필로 테스트
SPRING_PROFILES_ACTIVE=local ./gradlew :services:auth-service:test

# Docker 프로필로 테스트
SPRING_PROFILES_ACTIVE=docker ./gradlew :services:auth-service:test
```

---

## 🚨 트러블슈팅

### 문제: 설정이 적용되지 않음

**원인**: Config Server가 최신 설정을 가져오지 못함

**해결책**:
```bash
# 1. 저장소 최신화
git pull origin main

# 2. Config Server 재시작
docker-compose restart config-service

# 3. 서비스에서 설정 갱신
curl -X POST http://localhost:8080/actuator/refresh
```

### 문제: `java.net.UnknownHostException`

**원인**: 잘못된 호스트명 (환경 불일치)

**확인**:
```bash
# 현재 활성 프로필 확인
curl http://localhost:8081/actuator/env | jq '.propertySources[] | select(.name | contains("application")) | .name'

# 올바른 프로필 설정 확인
echo $SPRING_PROFILES_ACTIVE
```

### 문제: 데이터베이스 연결 실패

**원인**: 환경별 DB 호스트명 오류

**디버깅**:
```bash
# 현재 설정된 데이터베이스 URL 확인
curl http://localhost:8888/auth-service/docker | jq '.propertySources[] | select(.name | contains("auth-service")) | .source.spring.datasource.url'

# 실제 호스트에서 연결 테스트 (Docker)
docker exec mysql-db ping
docker exec auth-service curl http://mysql-db:3306
```

---

## 📚 참고 자료

- [Spring Cloud Config 공식 문서](https://spring.io/projects/spring-cloud-config)
- [Spring Cloud Config Server 가이드](https://cloud.spring.io/spring-cloud-config/reference/html/)

---

## 🤝 기여하기

### 설정 변경 워크플로우

1. **브랜치 생성**:
```bash
git checkout -b feat/update-config
```

2. **설정 수정 및 테스트**:
```bash
# 파일 수정
vim auth-service-docker.yml

# 변경사항 검증
curl http://localhost:8888/auth-service/docker
```

3. **커밋 및 푸시**:
```bash
git commit -m "Update auth-service database configuration for optimization"
git push origin feat/update-config
```

4. **Pull Request 생성**:
- 변경사항 설명
- 영향받는 서비스 명시
- 테스트 결과 포함

---

## 📝 주의사항

⚠️ **민감한 정보는 저장하지 마세요**
- 비밀번호는 환경 변수로 설정
- API 키는 Secret Management 사용
- 토큰은 런타임에 주입

⚠️ **프로덕션 설정**
- 로깅 레벨을 INFO 이상으로 설정
- 보안 헤더 활성화
- SSL/TLS 필수

⚠️ **설정 변경 시 주의**
- 모든 서비스에 미치는 영향 검토
- 변경 사항을 문서화
- 롤백 계획 준비

---

**마지막 업데이트**: 2025년 12월  
**Config Server 버전**: Spring Cloud Config 2025.0.0  
**관련 프로젝트**: [Portal Universe](https://github.com/L-a-z-e/portal-universe)
