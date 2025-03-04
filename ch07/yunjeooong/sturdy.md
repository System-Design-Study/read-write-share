# 분산 시스템을 위한 유일 ID 생성기 설계

분산 시스템에서 `auto_Increment`으로 유일 ID를 생성하는 접근법은

1. 데이터베이스 서버 한대로 요구를 담당할 수 없고,
2. 여러 데이터베이스 서버를 쓰는 경우에는 지연 시간(delay)을 낮추기가 무척 힘들 것이다.

**유일성이 보장되는 ID의 몇 가지 예를 함께 살펴보자**

## 1.  요구 사항

- ID는 유일해야 함.
- ID는 숫자로만 구성되어야 함.
- ID는 64비트로 표현할 수 있는 값이어야 함.
- ID는 발급 날짜에 따라 정렬 가능해야 함.
- 초당 10,000개의 ID를 만들 수 있어야 함.

## 2. 개략적 설계안 제시 및 동의 구하기

요구 사항에 따른 유일성이 보장되는 ID를 만드는 방법.

- 다중 마스터 복제 (multi-master replication)
- UUID (Universally Unique Identifier)
- 티켓 서버 (ticket server)
- 트위터 스노플레이크 접근법 (twitter snowflake)

### 다중 마스터 복제(mutli-master replication)

데이터베이스의 auto-increment를 데이터베이스 서버의 수 만큼 증가시키는 방법. 
해당 서버가 생성한 이전 ID 값에 전체 서버의 수를 더한 값이 됨.

⭐️장점
- 데이터 베이스 수를 늘리면 초당 생성 가능ID 수를 늘릴 수 있음.

⭐️단점
- 여러 데이터 센터에 걸쳐 규모를 늘리기 어렵다.
- ID의 유일성이 보장되겠지만, 시간 흐름에 맞추어 커지도록 보장할 수는 없다.
- 서버를 추가하거나 삭제할 때도 잘 동작하도록 만들기 어렵다.

### UUID (Universally Unique Identifier)

컴퓨터 시스템에 저장되는 정보를 유일하게 식별하기 위한 128비트 자리 수로 아이디를 생성하는 방법.

⭐️장점
- 충돌 가능성이 자극히 낮음.
- 서버 간 조율 없이 독립적으로 생성 가능.
- 동기화 이슈 없음.

⭐️단점
- ID가 128비트로 길다.
- ID를 시간순으로 정렬할 수 없다.
- ID에 숫자 아닌 값이 포함될 수 있다.

### 티켓 서버 (ticket server)

`auto_increment` 기능을 갖춘 데이터베이스 서버, 즉 티켓 서버를 중앙 집중형으로 하나만 사용하는 것.

⭐️장점
- 유일성이 보장되는 오직 숫자로만 구성된 ID를 쉽게 만들 수 있다.
- 구현하기 쉽고, 중소 규모 애플리케이션에 적합하다.

⭐️단점
- 티켓 서버가 SPOF(Single-Point-of-Failure)가 된다. 서버에서 장애가 생기면 모든 시스템이 영향을 받는다.
    - 티켓 서버를 여러 대 준비하게 된다면, 데이터 동기화 같은 새로운 문제가 발생한다.

### 트위터 스노플레이크 접근법 (Twitter snowflake)

ID를 바로 생성하는 대신 생성해야 하는 ID의 구조를 여러 절로 분할하는 방법

ID는 64비트로 표현할 수 있는 값이어야 한다. 요구 사항에 따라 아래와 같이 설계.

64비트 아이디 구조
- 1비트 (사인 비트) - 41비트 (타임스탬프) - 5비트 (데이터 센터 ID) - 5비트 (서버ID) - 12비트 (일련 번호)
- 사인(sign) 비트 : 나중을 위해 유보해주는 값
- 타임스탬프(timeStamp) : 기원 시작으로 몇 밀리초가 경과했는지를 나타내는 값
- 데이터 센터 ID : 데이터 센터 ID
- 서버 ID : 데이터 센터 당 서버 ID. 데이터 센터당 32개 서버 사용 가능
- 일련번호 : 각 서버에서 ID를 생성할 때마다 일련번호를 1만큼 증가. 1밀리초가 경과할 때마다 0으로 초기화

## 3. 상세 설계

**요구 사항에 모두 일치하는 트위터 스노플레이크 접근법을 선택.**

Section 별 ID 생성 시기
- 데이터 센터 ID나 서버 ID는 시스템이 시작할 떄 결정되는 값.
    -  운영 중에서 바뀌지 않으나, 두개의 ID를 잘못 변경하게 되면 ID 충돌이 일어날 수 있기 때문에 작업 시 신중하게 할 것.
- 타임스탬프나 일련번호는 ID 생성기가 돌고 있는 중에 만들어지는 값.

### 타임 스탬프

타임 스탬프가 41비트일 경우 69년이 지나면 기원 시각을 바꾸거나 ID 체계를 다른 것으로 이전해야함.

### 일련번호

일련번호는 12비트이므로, 4096개의 값을 가질 수 있다. 어떤 서버가 같은 밀리초 동안 하나 이상의ID를 만들어 낸 경우에만 0보다 큰 값을 갖게 된다.


## 4. 마무리

추가적으로 나올 수 있는 이슈
- 시계 동기화(clock synchronization) : 여러 서버가 물리적으로 독립된 여러 장비에서 실행되는 경우에는 시계를 동기화 할 수 있는 방법이 있어야함.
    - NTP(Network Time Protocol) 를 사용하여 문제를 해결하는 방법이 있음.
- 각 절(section)의 길이 최적화 : 동시성(concurrency)이 낮고 수평이 긴 애플리케이션이라면 일련번호 절의 길이를 줄이고 타임스탬프 절의 길이를 늘리는 것이 효과적
- 고가용성(high availability) : ID 생성기는 필수 불가별(mission critical) 컴포넌트이므로 아주 높은 가용성을 제공해야함.