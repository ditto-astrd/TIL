# 개요
1. 헥사고날 아키텍쳐
2. 패키지 구조
3. 계층별 설명
4. 아키텍처 검사도구

## 1. 헥사고날 아키텍쳐
- 핵심 : 포트 & 어댑터 패턴을 적용하여 외부 인프라적 요소들의 변경으로 인해 내부의 도메인 로직과 비즈니스 로직의 변경을 최소화
- 외부 계층과 통신은 포트를 통해 이루어지며, 포트는 in, out으로 분류되어 인터페이스 또는 추상클래스로 정의
- 외부 계층 예시
  - in : Web Controller, Rest Controller, Cli 인터페이스 등
  - out : 영속 어댑터(DB), API 어댑터, 메세지 큐 어댑터, 파일 시스템 어댑터 등
- in 포트는 외부에서 내부의 비즈니스 로직 또는 도메인 로직을 사용하기 위해 호출될 때 사용되며,in 포트 구현체는 어플리케이션 계층에 구현
- out 포트는 내부의 비즈니스 로직 구현을 위해 외부의 데이터를 조회하거나 저장하는 등의 처리를 위해 사용되며, out 포트 구현체는 adapter 계층에 구현

## 2. 패키지 구조
```
com.project.member/
├── domain/
│   └── member/
│        ├── Member.java
│        └── MemberRepository.java
├── application/
│   ├── port/
│   │   ├── in/
│   │   │   ├── (I) JoinMemberUseCase.java
│   │   │   └── (I) GetMemberQuery.java
│   │   └── out/
│   │       ├── (I) UseItemPort.java
│   │       └── (I) LoadItemPort.java
│   └── service/
│       ├── JoinMemberService.java 
│       └── UseItemService.java 
├── adapter/
│   ├── in/
│   │   ├── web/
│   │   │   ├── JoinMemberController 
│   │   │   └── UseItemController
│   │   └── cli/
│   └── out/
│       ├── api/ 
│       │   └── UseItemApiAdapter.java
│       └── persistence/
│            └── LoadItemPersistenceAdapter.java
└── configuration/ 
     ├── SecurityConfig
     ├── FeignConfig 
     ├── DataSourceConfig
     └── WebMvcConfig
     
```

## 3. 계층별 설명
### domain
---
- 도메인 로직을 포함하는 중심 계층
  - 도메인 로직이란?
    - 현실 세계의 문제를 해결하는 규칙
    - ex ) 유저의 구매 금액이 충분한지 확인
    - ex ) 유저의 멤버십 등급에 따라 무료 충전 혜택을 계산
- 이 계층은 외부 의존성 없이 순수 도메인 로직과 도메인과 관련된 영속성 처리를 포함
  - 영속성 처리는 외부 인프라적 요소로 adapter 계층에서 처리하는 것이 클린 아키텍쳐 철학에 부입하지만, JPA를 사용할 경우 도메인과 관련된 영속성 처리에 한하여 도메인에 포함 (`만들면서 배우는 클린 아키텍쳐`에서 말하는 **지름길**)

### application
---
- 유스케이스를 구현하고 도메인 객체들의 흐름을 제어
- 포트에 정의하는 인터페이스 이름의 prefix/postpix는 in과 out을 구분하기 쉽게 특정 용어를 정해서 사용

<br>

#### port.in
- 외부에서 어플리케이션으로 들어오는 요청을 처리하기 위한 인터페이스를 정의
- 용어
  - UseCase : 저장, 수정, 삭제 등의 유스케이스
  - Query : 조회

<br>

#### port.out
- 어플리케이션에서 외부로 나가는 요청을 처리하기 위한 인터페이스를 정의
- 용어
  - Load : 조회
  - Port : Out 포트에는 일괄 Port를 Postfix로 지정
 
<br>

#### service
- 실제 비즈니스 로직을 구현하는 서비스 클래스들이 위치
- 도메인 로직을 보조하는 로직
  - ex ) 유저 머니 지급을 시도하는데 유효하지 않을 경우 에러메세지를 띄움
  - ex ) 유저의 쿠폰을 사용하기 위해 외부 서비스에 요청
  - ex ) 유저의 게임머니 잔액을 DB에 저장
- `private-package` 활용
  - 포트를 구현하는 서비스의 경우, 클래스 접근 제한자를 **package-private**로 지정하여 외부 계층에서 직접적으로 접근하지 못하도록 설정
  - 외부 계층에서는 포트를 이용하여 해당 기능을 사용할 수 있도록 유도

<br>

### adapter
- 외부 시스템과의 통신 담당

<br>

#### in
- 외부에서 어플리케이션으로 들어오는 요청을 처리하는 어댑터들이 위치함
- `web` : 웹 요청을 처리하는 컨트롤러들이 위치함

<br>

#### out
- 어플리케이션에서 외부로 나가는 요청을 처리하는 어댑터들이 위치함
- `api` : 외부 API와의 통신을 담당하는 어댑터들이 위치함
- `persistence` : 데이터 영속성을 담당하는 어댑터들이 위치함

<br>

### configuration
- 어플리케이션의 설정을 담당하는 클래스들이 위치함

## 아키텍쳐 검사 도구
### ArchUnit
- ArchUnit은 프로젝트 내 아키텍처가 올바르게 구성되어 있는지 확인하기 위한 도구
- 역할
  - 패키지와 클래스의 의존 관계 확인
  - 상속 관계, 순환 참조 검사
  - 직접 규칙을 정의해 코딩 컨벤션 준수 여부를 검사
### 적용 규칙
- `PaymentArchitectureTest.java`
  - 도메인 계층은 다른 계층에 의존하지 않는다.
  - 어플리케이션 계층은 어댑터와 설청계층에서만 접근 가능하다.
  - 포트 계층은 어댑터와 설정계층에서만 접근 가능하다.
  - 어댑터 계층은 도메인, 어플리케이션, 포트 계층에서 접근해서는 안된다.
