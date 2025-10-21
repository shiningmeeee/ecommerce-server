# E-Commerce 플랫폼

포인트 충전식 결제 + 재고 관리 + 선착순 쿠폰 발급 기반의 이커머스 서비스

## 🗂️ 문서

- [API 명세서](./requirements/api_docs.yml)
- [ERD](./requirements/ERD.md)
- [인프라 구성도](./requirements/infra.md)

## 🎯 목표 시나리오

### 상품 구매 플로우
1. **상품 조회**: 사용자는 상품 목록을 조회하고 상세 정보를 확인한다.
2. **포인트 충전**: 결제를 위해 필요한 포인트를 미리 충전한다.
3. **쿠폰 발급**: 선착순 쿠폰 이벤트 참여 시 동시성 제어를 통해 중복 방지.
4. **주문/결제**: 재고 차감과 포인트 차감이 단일 트랜잭션으로 처리.
5. **외부 연동**: 주문 완료 시 데이터 플랫폼으로 비동기 전송.

## ⚙️ 기술 스택

- **Backend**: Java 21, Spring Boot 3, JPA (MySQL)
- **Cache & Lock**: Redis (Redisson) → 인기 상품 캐싱, 분산 락 관리
- **Message Queue**: Kafka → 주문 데이터 비동기 전송
- **인증**: JWT → 사용자 인증 & 권한 관리
- **테스트**: JUnit5 + Mockito + Testcontainers

## 📂 프로젝트 구조

```
src/main/java/com/ecommerce
├── clean
│   ├── order          # ✅ 주문/결제 (클린 아키텍처)
│   ├── product        # ✅ 상품 관리 (클린 아키텍처)
│   ├── point          # ✅ 포인트 충전/사용 (클린 아키텍처)
│   └── coupon         # ✅ 쿠폰 발급/사용 (클린 아키텍처)
└── layered
    └── statistics     # ✅ 통계 조회 (레이어드 아키텍처)
```

## 🛠 Infrastructure Layer 구조

```
clean/{domain}/adapter
├── in/
│   └── web/                              # API Controller
├── out/
│   ├── persistence/                      # JPA 구현체
│   │   ├── *RepositoryAdapter.java
│   │   └── *Mapper.java
│   ├── lock/
│   │   ├── RedisStockLockAdapter.java   # 재고 락 처리
│   │   └── RedisCouponLockAdapter.java  # 쿠폰 락 처리
│   ├── cache/
│   │   └── RedisProductCacheAdapter.java # 상품 캐싱
│   └── external/
│       └── DataPlatformAdapter.java      # 외부 데이터 플랫폼 연동
```

## 🚀 API 요약

### 1️⃣ 상품 (Product)
- `GET /api/v1/products/{productId}` → 단일 상품 상세 조회
- `GET /api/v1/products` → 상품 목록 조회
- `GET /api/v1/products/top-selling` → 인기 판매 상품 TOP 5 (최근 3일)

### 2️⃣ 포인트 (Point)
- `GET /api/v1/point/{userId}` → 사용자 잔여 포인트 조회
- `POST /api/v1/point/{userId}/charge` → 포인트 충전

### 3️⃣ 쿠폰 (Coupon)
- `POST /api/v1/coupon/issue/first-come-first-served` → 선착순 쿠폰 발급
- `GET /api/v1/coupon/{userId}` → 보유 쿠폰 목록 조회

### 4️⃣ 주문 (Order)
- `POST /api/v1/order` → 주문 생성 및 결제 처리

## ✅ 핵심 설계 포인트

| 항목 | 설명 |
|------|------|
| **동시성 제어** | 재고/포인트/쿠폰 처리 시 Redisson 분산 락으로 동시성 문제 해결 |
| **트랜잭션 관리** | 재고 차감 + 포인트 차감 + 주문 생성을 ACID 보장 트랜잭션으로 처리 |
| **선착순 처리** | Redis Sorted Set/List를 활용한 쿠폰 발급 순서 관리 |
| **캐싱 전략** | 인기 상품 정보는 Redis 캐싱으로 DB 부하 감소 |
| **비동기 처리** | Kafka를 통한 주문 데이터 외부 플랫폼 전송 |
| **클린 아키텍처** | 주문/포인트/쿠폰은 port in/out 기반 책임 분리 |
| **레이어드 아키텍처** | 통계 조회는 단순 Service-Repository 구조 |

## 📊 데이터베이스 구조

### 주요 테이블
- **USER**: 사용자 정보 및 포인트 잔액 관리
- **POINT_HISTORY**: 포인트 충전/사용 이력
- **ITEM**: 상품 정보 (가격, 재고)
- **ORDER**: 주문 정보
- **ORDER_DETAIL**: 주문 상세 (상품별 수량)
- **COUPON**: 쿠폰 정보 (할인율/할인금액)
- **COUPON_HISTORY**: 쿠폰 발급/사용 이력

## 🏗️ 인프라 아키텍처

```
Client → Microservices (상품/회원/쿠폰/주문 서버)
           ↓
      DataStores (MySQL + Redis Cache)
           ↓
      Message Queues (주문생성 → 주문완료 → 주문 후처리)
           ↓
      External Data Platform
```

## 🧪 테스트 구조

```
src/test/java/com/ecommerce
├── clean/order
│   ├── application/service/
│   │   ├── OrderServiceTest.java
│   │   └── PaymentServiceTest.java
│   └── integration/
│       ├── OrderFlowIntegrationTest.java
│       ├── OrderConcurrencyTest.java
│       └── PaymentIdempotencyTest.java
├── clean/product
│   ├── application/service/ProductServiceTest.java
│   └── integration/StockConcurrencyTest.java
├── clean/point
│   ├── application/service/PointServiceTest.java
│   └── integration/PointChargeConcurrencyTest.java
├── clean/coupon
│   ├── application/service/CouponServiceTest.java
│   └── integration/CouponIssueConcurrencyTest.java
└── layered/statistics
    └── StatisticsServiceTest.java
```

## 🧪 주요 통합 테스트

- **OrderFlowIntegrationTest** → 전체 주문 플로우 (상품 조회 → 쿠폰 적용 → 주문 → 결제)
- **StockConcurrencyTest** → 동시 재고 차감 시 정확성 보장
- **PointChargeConcurrencyTest** → 포인트 충전 동시성 테스트
- **CouponIssueConcurrencyTest** → 선착순 쿠폰 발급 중복 방지
- **PaymentIdempotencyTest** → 중복 결제 방지 (멱등성)

## 🔧 환경 설정

### 필수 요구사항
- JDK 21+
- MySQL 8.0+
- Redis 7.0+
- Kafka 3.0+ (Optional)

### 실행 방법

```bash
# 로컬 환경 실행
./gradlew bootRun

# 테스트 실행
./gradlew test

# 통합 테스트만 실행
./gradlew integrationTest
```