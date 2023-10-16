# 4장 트랜잭션 관리: 사가

## 4.1 마이크로서비스 아키텍처에서의 트랜잭션 관리

<img src="./img/1.jpg" alt="" width="500" />

### 사가

- 비동기 메세징을 이용한 일련의 로컬 트랜잭션
- 여러 서비스의 데이터를 업데이트하는 시스템 커맨드마다 사가를 하나씩 정의

<img src="./img/2.jpg" alt="" width="500" />

- 메시지를 통해 사가 참여자를 느슨하게 결합 & 사가가 반드시 완료되도록 보장

#### 사가는 보상 트랜잭션으로 변경분을 롤백한다

<img src="./img/3.jpg" alt="" width="500" />

- 사가는 단계마다 로컬 DB에 변경분을 커밋하기에 자동 롤백 불가능
- 4번째 단계에서 실패하면 1~3번째 단계에서 적용된 commit undo &rarr; 보상 트랜잭션(= compensating transaction)<br>
&rArr; (n + 1) 사가 트랜잭션이 실패하면 이전 n개의 트랜잭션 undo

<img src="./img/4.jpg" alt="" width="500" />

- 1~3번째 단계 = `보상 트랜잭션` &rarr; 실패할 가능성이 있는 다음 단계에 있음
- 4번째 단계 = `피봇 트랜잭션` &rarr; 절대로 실패하지 않는 단계
- 5~6번째 단계 = `재시도 가능 트랜잭션` &rarr; 항상 성공 단계

## 4.2 사가 편성

- `choreography`: 의사 결정과 순서화를 사가 참여자에게 맡김 &rarr; 이벤트 교환 방식으로 통신
- `orchestration`: 사가 오케스트레이터에 중앙화 &rarr; 오케스트레이터가 사가 참여자에게 커맨드 메시지 전송

### 4.2.1 코레오그래피 사가

<img src="./img/5.jpg" alt="" width="500" />

- 사가 참여자가 서로 이벤트 소통
- 각 참여자가 자신 DB 업데이트 후 다음 참여자 트리거하는 이벤트 발행
- 발행/구독 방식으로 소통

#### 확실한 이벤트 기반 통신

1. 사가 참여자가 **자신의 DB를 업데이트하고, DB 트랜잭션 일부로 이벤트 발행**하도록 해야함
2. 사가 참여자는 **자신이 수신한 이벤트와 자신이 가진 데이터 연관** 지을 수 있어야 함
- ex) 신용카드 승인됨 이벤트를 받은 주문 서비스는 이에 해당하는 주문 찾을 수 있어야 함
- 해결책으로 데이터를 매핑할 수 있도록 상관관계 ID 포함된 이벤트 발행

### 4.2.2 오케스트레이션 사가

> 사가 오케스트레이터가 비동기 요청/응답을 주고 받으며 사가 참여자를 호출하고 처리 과정에 따라 커맨드 메시지 전송

<img src="./img/6.jpg" alt="" width="500" />

#### 사가 오케스트레이터를 상태 기계로 모델링

> 상태 기계 &rarr; 상태 + 이벤트에 의해 트리거되는 상태 전이

<img src="./img/7.jpg" alt="" width="450" />

- 티켓 생성 상태 &rarr; 신용카드 승인, 주문 거부 상태로 전이
- 다양한 응답에 따라 다양한 상태 전이를 거치면서 결국 주문 승인, 거부 상태 중 한쪽으로 귀결

## 4.3 비격리 문제 처리

> 사가는 격리성이 제외된 ACD 트랜잭션

- `원자성`: 사가는 트랜잭션을 모두 완료하거나 모든 변경분 undo
- `일관성`: 서비스 내부의 참조 무결성은 로컬 DB가, 여러 서비스에 걸친 참조 무결성은 서비스가 처리
- `지속성`: 로컬 DB로 처리

&rArr; 격리가 안되어 비정상(= anomaly) 발생 가능 &rarr; 사가를 동시에 실행한 결과와 순차 실행한 결과가 달라질 수 있음

### 4.3.1 비정상 개요

> 비정상이란, 트랜잭션이 차례대로 실행되지 않은 것처럼 데이터를 읽고 쓰게 되는 현상

- `lost update`: 한 사가의 변경분을 다른 사가가 못 읽고 덮어씀
- `dirty reads`: 사가 업데이트를 하지 않은 변경분을 다른 트랜잭션이나 사가가 조회
- `fuzzy/non-repeatable read`: 한 사가의 상이한 두 단계가 같은 데이터를 읽어도 결과가 달라지는 현상 &rarr; 다른 사가가 그 사이 업데이트를 했기에 발생

#### lost update

1. 주문 생성 사가 첫 번째 단계에서 주문 생성
2. 사가 실행 중 주문 취소 사가가 주문 취소
3. 주문 생성 사가 마지막 단계에서 주문 승인

&rArr; 주문 생성 사가는 주문 취소 사가가 업데이트한 데이터를 덮어 쓰게 되고, 고객은 자신이 주문 취소한 음식 배달받음

#### dirty read

1. 주문 취소 사가: 신용 잔고 &uarr;
2. 주문 생성 사가: 신용 잔고 &darr;
3. 주문 취소 사가: 신용 잔고 줄이는 보상 트랜잭션 실행

&rArr; 주문 생성 사가는 신용 잔고를 dirty read 하게 되고, 소비자는 신용 한도를 초과하는 주문 발생 가능

### 4.3.2 비격리 대책

- `semantic lock`: 애플리케이션 수준 락
- `communtative updates`: 업데이트 작업은 어떤 순서로 실행해도 되게끔 설계
- `pessimistic view`: 사가 단계 순서를 재조정하여 비즈니스 리스크 최소화
- `re-read value`: 데이터를 덮어 쓸 때 전에 변경된 내용은 없는지 다시 확인
- `version file`: 순서를 재조정할 수 있게 업데이트 기록
- `by value`: 동시성 메커니즘 동적 선택

#### 사가의 구조

- `보상 가능 트랜잭션`: 보상 트랜잭션으로 롤백 가능한 트랜잭션
- `피봇 트랜잭션`: 사가의 진행/중단 지점 &rarr; 피봇 트랜잭션이 커밋되면 사가는 완료될 때까지 실행
- `재시도 가능 트랜잭션`: 반드시 성공하는 트랜잭션

<img src="./img/8.jpg" alt="" width="500" />

## 4.4 주문 서비스 및 주문 생성 사가 설계

<img src="./img/9.jpg" alt="" width="500" />

### 4.4.1 OrderService 클래스

> 주문 생성/관리 담당 서비스 API 계층이 호출하는 도메인 서비스

- `SagaManager`: CreateOrderSaga 사가 생성 &rarr; 사가 오케스트레이터와 참여자 작성 클래스

<img src="./img/10.jpg" alt="" width="500" />

```java
@Transactional
public class OrderServce {
    public Order createOrder(OrderDetails orderDetails) {
        ...
        ResultWithEvents<Order> orderAndEvents = Order.createdOrder(...); // Order 생성
        Order order = orderAndEvents.result;
        orderRepository.save(order);                                      // DB Order 저장
        
        eventPublisher.publish(
            Order.class,
            Long.toString(order.getId()),
            orderAndEvents.events
        );                                                               // 도메인 이벤트 발행
        
        CreateOrderSagaState data = new CreateOrderSagaState(order.getId(), orderDetails); // 사가 생성
        createOrderSagaManager.create(data, Order.class, order.getId());
        
        return order;
    }
}
```

- Order, OrderDetails ID가 포함된 CreateOrderSagaState를 SagaManager.create()에 넘겨 CreateOrderSaga 생성
- SagaManager가 사가 오케스트레이터 인스턴스 생성 후 사가 참여자에게 커맨드 메세지 전달되고 사가 오케스트레이터를 DB에 저장

### 4.4.2 주문 생성 사가 구현

- `CreateOrderSaga`: 사가 상태 기계를 정의한 싱글턴 클래스 &rarr; CreateOrderSagaState로 커맨드 메시지 생성하고 사가 참여자 프록시가 지정한 메세지 채널을 통해 참여자에게 메세지 전달
- `CreateOrderSagaState`: 사가 저장 상태 &rarr; 커맨드 메시지 생성
- `사가 참여자 프록시 클래스`: 프록시 크래스마다 커맨드 채널, 커맨드 메시지 타입, 반환형으로 구성된 사가 참여자 메시징 API 정의

```java
public class CreateOrderSagaState {
    private Long orderId;
    private OrderDetails orderDetails;
    private long ticketId;
    
    // CreateTicket command message 생성
    CreateTicket makeCreateTicketCommand() {
        return new CreateTicket(getOrderDetails().getRestaurantId(), getOrderId(), makeTicketDetails(getOrderDetails()));
    }
    
    // 새로 만든 티켓 ID 저장
    void handleCreateTicketReply(CreateTicketReply reply) {
        setTicketId(reply.getTicketId());
    }
    
    ...
}
```

#### 이벤추에이트 트램 사가 프레임워크

<img src="./img/11.jpg" alt="" width="500">

1. OrderService &rarr; CreateOrderSagaState 생성
2. OrderService &rarr; SagaManager를 호출하여 사가 인스턴스 생성
3. SagaMangager &rarr; Saga definition 첫 번째 단계 실행
4. CreateOrderSagaState를 호출하여 커맨드 메시지 생성
5. SagaManager &rarr; 커맨드 메세지를 사가 참여자에게 보냄
6. SagaManager &rarr; 사가 인스턴스 DB 저장

## 의문점

1. 발행한 도메인 이벤트와 생성된 사가 내의 커멘드의 차이점

- 도메인 이벤트: 말 그대로 내부 DB 비즈니스 로직을 처리하는 이벤트 &rarr; 내부
- 사가 내 커맨드: 다른 사가 참여자에게 요청하는 커맨드 &rarr; 외부

2. DB에 저장하는 사가 인스턴스란? ex) 사가 인스턴스 그자체를 DB에 저장하는건지, 사가 인스턴스 내 사가 상태와 같은 필드를 저장하는건지

- 이벤추에이트 트램 사가 프레임워크가 처리해주는 기능

