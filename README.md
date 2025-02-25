![image](https://user-images.githubusercontent.com/487999/79708354-29074a80-82fa-11ea-80df-0db3962fb453.png)

# 예제 - 음식배달 

본 예제는 MSA/DDD/Event Storming/EDA 를 포괄하는 분석/설계/구현/운영 전단계를 커버하도록 구성한 예제입니다.
이는 클라우드 네이티브 애플리케이션의 개발에 요구되는 체크포인트들을 통과하기 위한 예시 답안을 포함합니다.
- 체크포인트 : https://workflowy.com/s/assessment-check-po/T5YrzcMewfo4J6LW


# Table of contents

- [예제 - 음식배달](#---)
  - [서비스 시나리오](#서비스-시나리오)
  - [체크포인트](#체크포인트)
  - [분석/설계](#분석설계)
  - [구현:](#구현-)
    - [DDD 의 적용](#ddd-의-적용)
    - [폴리글랏 퍼시스턴스](#폴리글랏-퍼시스턴스)
    - [폴리글랏 프로그래밍](#폴리글랏-프로그래밍)
    - [동기식 호출 과 Fallback 처리](#동기식-호출-과-Fallback-처리)
    - [비동기식 호출 과 Eventual Consistency](#비동기식-호출-과-Eventual-Consistency)
  - [운영](#운영)
    - [CI/CD 설정](#cicd설정)
    - [동기식 호출 / 서킷 브레이킹 / 장애격리](#동기식-호출-서킷-브레이킹-장애격리)
    - [오토스케일 아웃](#오토스케일-아웃)
    - [무정지 재배포](#무정지-재배포)
  - [신규 개발 조직의 추가](#신규-개발-조직의-추가)

# 서비스 시나리오

배달의 민족 커버하기 - https://1sung.tistory.com/106

기능적 요구사항
1. 고객이 메뉴를 선택하여 주문한다
1. 고객이 결제한다
1. 주문이 되면 주문 내역이 입점상점주인에게 전달된다
1. 상점주인이 확인하여 요리해서 배달 출발한다
1. 고객이 주문을 취소할 수 있다
1. 주문이 취소되면 배달이 취소된다
1. 고객이 주문상태를 중간중간 조회한다
1. 주문상태가 바뀔 때 마다 카톡으로 알림을 보낸다

비기능적 요구사항
1. 트랜잭션
    1. 결제가 되지 않은 주문건은 아예 거래가 성립되지 않아야 한다  Sync 호출 
1. 장애격리
    1. 상점관리 기능이 수행되지 않더라도 주문은 365일 24시간 받을 수 있어야 한다  Async (event-driven), Eventual Consistency
    1. 결제시스템이 과중되면 사용자를 잠시동안 받지 않고 결제를 잠시후에 하도록 유도한다  Circuit breaker, fallback
1. 성능
    1. 고객이 자주 상점관리에서 확인할 수 있는 배달상태를 주문시스템(프론트엔드)에서 확인할 수 있어야 한다  CQRS
    1. 배달상태가 바뀔때마다 카톡 등으로 알림을 줄 수 있어야 한다  Event driven

추가 요구사항
1. 5번 시나리오 변경 : 상점주는 주문이 접수 된 후 결제가 되었는지 조회할 수 있다.
1. 시나리오 추가 : 상점주가 주문을 거절하면 주문 내역이 취소된다.

# 체크포인트

- 분석 설계


  - 이벤트스토밍: 
    - 스티커 색상별 객체의 의미를 제대로 이해하여 헥사고날 아키텍처와의 연계 설계에 적절히 반영하고 있는가?
    - 각 도메인 이벤트가 의미있는 수준으로 정의되었는가?
    - 어그리게잇: Command와 Event 들을 ACID 트랜잭션 단위의 Aggregate 로 제대로 묶었는가?
    - 기능적 요구사항과 비기능적 요구사항을 누락 없이 반영하였는가?    

  - 서브 도메인, 바운디드 컨텍스트 분리
    - 팀별 KPI 와 관심사, 상이한 배포주기 등에 따른  Sub-domain 이나 Bounded Context 를 적절히 분리하였고 그 분리 기준의 합리성이 충분히 설명되는가?
      - 적어도 3개 이상 서비스 분리
    - 폴리글랏 설계: 각 마이크로 서비스들의 구현 목표와 기능 특성에 따른 각자의 기술 Stack 과 저장소 구조를 다양하게 채택하여 설계하였는가?
    - 서비스 시나리오 중 ACID 트랜잭션이 크리티컬한 Use 케이스에 대하여 무리하게 서비스가 과다하게 조밀히 분리되지 않았는가?
  - 컨텍스트 매핑 / 이벤트 드리븐 아키텍처 
    - 업무 중요성과  도메인간 서열을 구분할 수 있는가? (Core, Supporting, General Domain)
    - Request-Response 방식과 이벤트 드리븐 방식을 구분하여 설계할 수 있는가?
    - 장애격리: 서포팅 서비스를 제거 하여도 기존 서비스에 영향이 없도록 설계하였는가?
    - 신규 서비스를 추가 하였을때 기존 서비스의 데이터베이스에 영향이 없도록 설계(열려있는 아키택처)할 수 있는가?
    - 이벤트와 폴리시를 연결하기 위한 Correlation-key 연결을 제대로 설계하였는가?

  - 헥사고날 아키텍처
    - 설계 결과에 따른 헥사고날 아키텍처 다이어그램을 제대로 그렸는가?
![image](https://user-images.githubusercontent.com/117880601/206364771-4b676cbf-e230-45ad-b21f-79a2ef8c3edf.png)




- 구현

  - Saga (Pub / Sub) :
    - 고객이 order에서 주문을 하면 store는 주문이 접수됨을 확인한다.

    - 접수 소스코드
      - ![접수_소스](https://user-images.githubusercontent.com/117880601/205803641-2c715cc3-328a-479a-ba3c-056184c12a1a.PNG)


    - 주문하기
      - ![주문_order](https://user-images.githubusercontent.com/117880601/205802710-475e91a8-08fa-43b9-b102-41d3696e9814.PNG)


    - 상점 접수확인
      - ![주문_store](https://user-images.githubusercontent.com/117880601/205804026-b4ae592f-22ad-4647-a58f-0f8144a12477.PNG)

  - CQRS :
    - 주문정보가 변경될 때마다 고객은 MyPage에서 변경된 상태 정보를 조회한다.

    - ReadModel 속성
      - ![CQRS_이미지2](https://user-images.githubusercontent.com/117880601/205804669-292f7d62-f10d-4c0f-b552-b90fc3a9c0b3.PNG)

    - ReadModel 소스
      - ![CQRS_소스_이미지2](https://user-images.githubusercontent.com/117880601/205804725-8c5bd735-efa3-45ed-abf5-c34041df6787.PNG)

    - 주문하기
      - ![CQRS_실행_이미지1-1](https://user-images.githubusercontent.com/117880601/205804857-aa9b7110-fbd6-40c2-8ad3-2cc9e32b023e.PNG)

    - 마이페이지 확인
      - ![CQRS_실행_이미지2](https://user-images.githubusercontent.com/117880601/205804984-38d30030-e4fd-46b7-b161-d4499b1f2030.PNG)

  - Compensation / Correlation :
    - 주문을 한 후 취소를 한다.

    - 주문시 상점 접수 소스
      - ![3_소스_1](https://user-images.githubusercontent.com/117880601/205805302-aa477388-86af-4237-a7a6-c0162444f0c4.PNG)

    - 취소 상점 취소 소스
      - ![3_소스_2](https://user-images.githubusercontent.com/117880601/205805370-51158218-cf6b-497a-a00e-4330f33a6bdf.PNG)

    - 주문 실행
      - ![3_실행_1](https://user-images.githubusercontent.com/117880601/205805465-1cf978d0-c51e-48ac-8372-f682455909e5.PNG)

    - 상점 접수 확인
      - ![3_실행_2](https://user-images.githubusercontent.com/117880601/205805496-8e341f64-3b8c-4dbd-aef9-d9f7957a320e.PNG)

    - 취소 실행
      - ![3_실행_3](https://user-images.githubusercontent.com/117880601/205805584-9e3bb5e4-1736-41c5-84f2-3620d3695184.PNG)

    - 취소 후 상점 확인
      - ![3_실행_4](https://user-images.githubusercontent.com/117880601/205805652-4bf6b3de-db0e-438c-bdaa-d1ac3c81e087.PNG)

  - Request / Response :
    - 주문을 할때 결제 정보를 Request / Response 방식으로 변경하여 수행

    - Pay Domain 속성 수정
      - ![image](https://user-images.githubusercontent.com/117880601/205825192-b7627cbb-53a2-47f2-b398-ed8eab2e0af7.png)

    - Order 소스 수정
      - ![4_소스_주문](https://user-images.githubusercontent.com/117880601/205825279-ffbb0a72-1b11-4b1d-9e6e-9688bc433384.PNG)

    - 결제 Controller 소스 수정
      - ![4_소스_결제_Controller](https://user-images.githubusercontent.com/117880601/205825375-18c45a0a-a1d9-4907-880b-9b0cda0d4724.PNG)

    - 결제 Domain 소스 수정
      - ![4_소스_결제_Domain](https://user-images.githubusercontent.com/117880601/205825437-425c3c79-f77c-4caa-9db1-f61ab3750098.PNG)

    - 주문 실행
      - ![4_주문_1](https://user-images.githubusercontent.com/117880601/205825561-440f3cca-8fe4-4787-a651-3bf55c6980f5.PNG)

    - 결제 컨테이너가 정상일 경우
      - ![4_실행_정상](https://user-images.githubusercontent.com/117880601/205825685-23e291fb-c2be-466b-a66c-c7767a8e0e37.PNG)

    - 결제 컨테이너가 비정상일 경우 주문 시 에러
      - ![4_실행_주문_에러](https://user-images.githubusercontent.com/117880601/205825756-d9f46d82-c9e0-46c0-be06-ce2214ffb7d1.PNG)
    
    

  - Circuit Breaker
    - story 컨테이너가 중지 되어 있을 때 주문 취소 시 조리중으로 표현한다.

    - 서킷브레이커_설정
      - ![서킷브레이커_설정](https://user-images.githubusercontent.com/117880601/205855732-fa7b9ef0-f0f3-4a55-bed1-0cbb3959ff06.PNG)

    - 서킷브레이커_스토어_딜레이
      - ![서킷브레이커_스토어_딜레이](https://user-images.githubusercontent.com/117880601/205855802-f159c04e-7b86-4825-b8b8-bac143d9b526.PNG)

    - 서킷브레이커_주문취소_에러 생성
      - ![서킷브레이커_주문취소_에러발생](https://user-images.githubusercontent.com/117880601/205855885-388570c2-4b58-4503-aa62-cd801d4e4ac5.PNG)

    - 서킷브레이커_페일백설정
      - ![서킷브레이커_폐일백설정](https://user-images.githubusercontent.com/117880601/205856061-34799fe2-6a1a-4153-8436-b74c5262623b.PNG)

    - 서킷브레이커_페일백파일추가
      - ![서킷브레이커_페일백파일추가](https://user-images.githubusercontent.com/117880601/205855985-204e2ca2-fc8d-4c67-b308-8744a2131de9.PNG)

  - Gateway / Ingress :
    - gateway 의 application.yml 파일 설정으로 Gateway를 통하여 컨테이너를 호출한다.

    - Gateway 설정 확인
      - ![6_설정_!](https://user-images.githubusercontent.com/117880601/205806027-0775e622-6fa3-423a-b6d6-9816d07411fe.PNG)

    - Gateway를 통하여 order 컨터이너 호출
      - ![6_실행_1](https://user-images.githubusercontent.com/117880601/205806113-e7223211-b23e-4341-b603-eda59415ced0.PNG)

    - order 컨테이너를 직접 호출하여 주문 상태를 확인
      - ![6_실행_2](https://user-images.githubusercontent.com/117880601/205806161-a8f46cfa-e086-4632-ae65-ded891a68e18.PNG)
    






