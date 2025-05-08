# User-service

### 🔖 서비스 개요
-   **주제**: 사용자 관리 및 계좌 서비스
-   **주요 역할**:
    -   사용자 인증 및 계정 관리
    -   가상 계좌 생성 및 관리

</br>

### 🌏 주요 로직

<img width="943" alt="주문 처리 흐름도" src="https://github.com/user-attachments/assets/0edb90be-ebb0-4b72-a83b-9baf6ed56ba8" />


</br>

### 🔗 주요 기능

#### 1️⃣ 사용자 인증 및 계정 관리
-   구글 소셜 로그인 지원 및 JWT 기반 인증 시스템 구현
-   사용자 프로필 정보 관리 및 업데이트 기능 제공
-   보안 강화를 위한 토큰 관리 및 갱신 메커니즘 구현

#### 2️⃣ 가상 계좌 관리
-   회원가입 시 모의 투자용 가상 자금 자동 제공
-   실시간 계좌 잔고 관리 및 업데이트 처리
-   주문 가능 금액 검증 및 계좌 잔액 변동 이벤트 처리

#### 3️⃣ 이벤트 처리 및 연동
-   Order 서비스와 연동 통한 계좌 잔액 검증
-   Matching 서비스로부터 체결 이벤트 수신 및 처리
-   RabbitMQ 활용한 비동기 이벤트 처리 구현

### 🌈 개선 사항

#### 1️⃣ 동시성 문제 해결 - <ins>락 및 AtomicReference 적용으로 데이터 정합성 확보</ins>

**📌 문제점**
-   주문 시 `account` 테이블 동시 접근으로 잔액 및 보유 주식 수 동기화 오류 발생.
-   데이터 부정합 가능성 존재.

<img alt="동시성 문제 발생 상황" src="https://github.com/user-attachments/assets/7b4c784c-a42d-478f-be4c-fae1dfce99d3" />


**✅ 해결 과정**
-   **1단계: `synchronized` 키워드 한계 분석**
    -   초기 JPA 트랜잭션 서비스 메소드에 `synchronized` 적용 시도.
    -   JVM 레벨 락으로 인한 Spring 환경에서의 심각한 성능 저하 확인.
-   **2단계: 락 범위 최소화 및 `AtomicReference` 도입**
    -   JPA Repository 메소드 수준으로 락 범위 축소.
    -   `AtomicReference` 활용, 여러 스레드에서 안전한 객체 참조 및 업데이트로 동시성 제어 강화.
-   **3단계: 비관적 락 최종 적용**
    -   금융 로직의 데이터 정합성 확보 위해 비관적/낙관적 락 비교.
    -   동시 접근 시 충돌 가능성 높아 낙관적 락만으로 불충분 판단.
    -   특정 엔티티 트랜잭션 시작 시점 선점하는 비관적 락 적용으로 데이터 정합성 확보.

---

#### 2️⃣ DB 커넥션 풀 최적화 - <ins>HikariCP 및 MySQL 설정 조정을 통한 성능 향상</ins>

**📌 문제점**
-   락 적용 후에도 HikariCP 타임아웃 지속 (주요 원인: 커넥션 고갈).
-   JPA 사용 시 EntityManager 필요 DB 커넥션 부족.
    -   **트랜잭션 처리 흐름**: `Thread → EntityManager → HikariCP → MySQL DB`
    -   Spring 환경, 스레드 기본 작업 단위는 JPA 트랜잭션.
    -   1 JPA 트랜잭션 = 1 영속성 컨텍스트 = 1 HikariCP 커넥션 할당.
    -   다수 동시 트랜잭션 발생 시 할당 가능 DB 커넥션 부족이 근본 원인.
      
<img alt="DB 커넥션 부족 문제" src="https://github.com/user-attachments/assets/babf1115-8f55-4b5b-975d-cb9c9d6d512d" />


**✅ 해결 과정**
-   **1단계: HikariCP 풀 크기 산정 및 조정**
    -   `@Transactional` 내부에서 1 영속성 컨텍스트 = 1 DB 커넥션 사용 인지.
    -   TPS 기반 적정 풀 크기 계산 (예: 40 TPS, 쿼리당 1초 소요 시 약 40개 + 버퍼 10 ~ 20%  =  44 ~ 48개).
    -   HikariCP 고정 크기 풀 권장 따라 `maximum-pool-size`와 `minimum-idle` 동일 설정.
    -   기본값(10)에서 부하 테스트 및 계산 기반으로 `maximum-pool-size=50` (또는 권장 공식 기반 30)으로 상향 조정 (실제값은 환경/테스트 따라 최종 결정).
-   **2단계: MySQL `max_connections` 설정 동시 조정**
    -   애플리케이션 단 커넥션 풀 증가에도 DB 서버 수용 `max_connections` 낮으면 효과 없음.
    -   HikariCP `maximum-pool-size` 증가에 맞춰 MySQL `max_connections` 값도 상향 조정, 양단 충분한 커넥션 확보.
    
<img alt="커넥션 풀 설정 최적화" src="https://github.com/user-attachments/assets/b8827da4-334a-4f90-813f-e952ebc8bbaa" />

