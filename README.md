# User-service

### 🔖 서비스 개요
  - **주제**: 사용자 관리 및 계좌 서비스
  - **주요 역할**: 
    - 사용자 인증 및 계정 관리
    - 가상 계좌 생성 및 관리
    

</br>

### 🌏 주요 로직
<img width="943" alt="주문 처리 흐름도" src="https://github.com/user-attachments/assets/0edb90be-ebb0-4b72-a83b-9baf6ed56ba8" />

</br>

### 🔗 주요 기능  

#### 1️⃣ 사용자 인증 및 계정 관리
- 구글 소셜 로그인 지원 및 JWT 기반 인증 시스템 구현
- 사용자 프로필 정보 관리 및 업데이트 기능 제공
- 보안 강화를 위한 토큰 관리 및 갱신 메커니즘 구현

#### 2️⃣ 가상 계좌 관리
- 회원가입 시 모의 투자용 가상 자금 자동 제공
- 실시간 계좌 잔고 관리 및 업데이트 처리
- 주문 가능 금액 검증 및 계좌 잔액 변동 이벤트 처리

#### 3️⃣ 이벤트 처리 및 연동
- 주문 서비스와의 연동을 통한 계좌 잔액 검증
- 매칭 서비스로부터 체결 이벤트 수신 및 처리
- RabbitMQ를 활용한 비동기 이벤트 처리 구현


### 🌈 개선 사항

#### 1️⃣ 동시성 문제 해결 - <ins>락 및 AtomicReference 적용으로 데이터 정합성 확보</ins>

**📌 문제 상황**
- 주문 기능 실행 시, `account` 테이블에 다수의 트랜잭션이 동시에 접근하면서 잔액 및 보유 주식 수에 대한 동기화가 제대로 이루어지지 않는 문제 발생.
- 이로 인해 데이터 부정합이 발생할 가능성이 존재.

- ![image](https://github.com/user-attachments/assets/7b4c784c-a42d-478f-be4c-fae1dfce99d3)


**✅ 개선 방향**
- **1단계: `synchronized` 키워드 사용의 한계점 분석**
    - 초기에는 각 JPA를 통해 DB에 접근하는 트랜잭션 서비스 메소드에 `synchronized` 키워드를 사용하여 스레드 동기화를 시도.
    - 그러나 `synchronized`는 JVM 레벨에서 작동하며, 특히 멀티스레드 기반의 Spring 환경에서는 해당 메소드 전체에 락을 걸어 심각한 성능 저하를 유발.

- **2단계: 락 적용 범위 최소화 및 `AtomicReference` 도입**
    - 성능 저하를 개선하기 위해, JPA Repository 인터페이스의 메소드 수준으로 락 적용 범위를 최소화.
    - 동시에 여러 스레드에서 안전하게 객체를 참조하고 업데이트할 수 있도록 `AtomicReference`를 활용하여 동시성 제어를 강화.

- **3단계: 비관적 락 도입**
    - 데이터 정합성이 매우 중요한 금융 관련 로직의 특성을 고려하여 비관적 락과 낙관적 락(Optimistic Lock)을 비교 분석.
    - 현재 로직에서는 동시 접근 시 데이터 충돌 가능성이 높아 낙관적 락만으로는 데이터 일관성을 완벽히 보장하기 어렵다고 판단.
    - 따라서 특정 엔티티에 대해 트랜잭션 시작 시점에 락을 선점하여 다른 트랜잭션의 수정을 막는 비관적 락을 적용하여 데이터 정합성을 확보.

---

#### 2️⃣ DB 커넥션 풀 최적화 - <ins>HikariCP 및 MySQL 설정 조정을 통한 성능 향상</ins>

**📌 문제 상황**
- 동시성 문제 해결을 위해 락을 적용했음에도 불구하고, HikariCP(DB 커넥션 풀) 자체의 타임아웃이 발생하는 현상 지속.
    - 주요 원인: HikariCP 커넥션 고갈 (Connection Pool Exhaustion).
    
- JPA 사용 시 EntityManager가 필요로 하는 DB 커넥션 부족 문제 발생.
    - **트랜잭션 처리 흐름:** `Thread → EntityManager(영속성 컨텍스트) → HikariCP → MySQL DB`
    - Spring 환경에서 각 스레드의 기본 작업 단위는 JPA 구현체(EntityManager)를 통한 트랜잭션 단위로 취급됨.
    - 하나의 JPA 트랜잭션은 하나의 영속성 컨텍스트를 가지며, 이는 HikariCP로부터 하나의 DB 커넥션을 할당받아 사용.
    - 결과적으로, 동시에 많은 트랜잭션이 발생할 경우 할당 가능한 DB 커넥션 수가 부족해지는 것이 근본적인 문제로 파악됨.
 
     ![image](https://github.com/user-attachments/assets/babf1115-8f55-4b5b-975d-cb9c9d6d512d)


**✅ 개선 방향**
- **1단계: HikariCP 풀 크기 산정 및 조정**
    - `@Transactional` 어노테이션 내부에서는 하나의 영속성 컨텍스트가 생성되며, 이 컨텍스트는 하나의 DB 커넥션을 사용함을 인지.
    - TPS(초당 트랜잭션 수)를 기반으로 적절한 커넥션 풀 크기를 계산.
        - 예시: 평균적으로 초당 40개의 트랜잭션이 발생하고, 각 쿼리 처리에 평균 1초가 소요된다면 이론적으로 약 40개의 커넥션이 필요.
        - 예상치 못한 트래픽 증가(스파이크)에 대비하기 위해 계산된 값에 10-20%의 버퍼를 추가하여 최종 커넥션 수를 결정 (예: 44~48개).
    - **고정 크기 풀(Fixed-size Pool) 사용:** HikariCP는 최적의 성능을 위해 가변 크기 풀보다는 고정 크기 풀 사용을 권장.
        - 이를 위해 `spring.datasource.hikari.maximum-pool-size`와 `spring.datasource.hikari.minimum-idle` 값을 동일하게 설정.
        - 기존 기본값(통상 10)에서 서비스 부하 테스트 및 계산된 값을 바탕으로 `spring.datasource.hikari.maximum-pool-size=50` (또는 권장 공식 `connections = ((core_count * 2) + effective_spindle_count)`에 따른 값, 예: 30)으로 상향 조정.
        (*실제 적용값은 시스템 환경과 부하테스트 결과에 따라 최종 결정*)

- **2단계: MySQL `max_connections` 설정 동시 조정**
    - 애플리케이션 단(HikariCP)에서 커넥션 풀 크기를 늘리더라도, 데이터베이스 서버(MySQL) 자체에서 수용할 수 있는 최대 커넥션 수(`max_connections` 파라미터)가 낮게 설정되어 있으면 실질적인 효과가 없음.
    - 따라서 HikariCP의 `maximum-pool-size` 증가에 맞춰 MySQL의 `max_connections` 값도 적절히 상향 조정하여 애플리케이션과 데이터베이스 양단에서 충분한 커넥션을 확보하도록 설정.
 
  ![image](https://github.com/user-attachments/assets/b8827da4-334a-4f90-813f-e952ebc8bbaa)

