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


# 체크포인트

# 1. Saga (Pub / Sub)
![image](https://user-images.githubusercontent.com/121836061/212212975-c85e8ee0-61ac-4777-b093-cdc30f440070.png)








# 2. CQRS

case1) MenuRead

![image](https://user-images.githubusercontent.com/121836061/212215253-d164e23f-7a43-43e9-b84c-ddaf9f7a9089.png)














case2) OrderRead

![image](https://user-images.githubusercontent.com/121836061/212004410-66b51d31-9c75-427c-b9d7-bc7f42dc365c.png)



















case3) OrderStatus

![image](https://user-images.githubusercontent.com/121836061/212004566-00da7698-3bac-4b30-9d4d-2ced5d0c700c.png)













# 3. Compensation / Correlation


Compensation(보상) : 실패/에러에 대한 롤백 프로그래밍 처리
->주문취소시 요리가 이미 시작시에는 취소할 수 없도록 처리한다.
----------------------------------------------------------------
    @PreOrderCancel
    public static void OrderCancelRelay(OrderCancelled OrderCancelled){
        Order Order = new Order();
        OrderStore orderStore = new OrderStore();
        Customer customer = new Customer();
        if(orderStore.getCookStart().equals("시작됨")){
            customer.sendMessage("요리가 시작되어 취소할 수 없습니다.");
             repository().save(order);
    }
----------------------------------------------------------------

Correlation(상호보완) : 정-> 역방향(취소)에 대한 프로그래밍 처리 
-> 주문취소시 결재취소 같이 진행
------------------------------------------------
    @PostOderCancel
    public void PayCancel(PayCanceled PayCanceled) {
        Order order = new Order();
        Payment payment = new Payment();
        if(order.getStatus().equals("주문취소됨")){
            payment.setStatus("결재취소");
             repository().save(payment);
        }
------------------------------------------------


![image](https://user-images.githubusercontent.com/121836061/212009752-961651b4-0338-4d51-92bc-67659a357840.png)











