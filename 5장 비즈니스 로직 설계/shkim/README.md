# 5장 비즈니스 로직 설계
- 도메인 이벤트는 애그리거트가 발행합니다.
```java
public class Ticket {

  public List<TicketDomainEvent> accept(LocalDateTime readyBy) {
    this.acceptTime = LocalDateTime.now();
    this.readyBy = readyBy;
    return singletonList(new TicketAcceptedEvent(readyBy)); // 이벤트 반환
  }
}
```

```java
public class KitchenService {

  @Autowired
  private TicketRepository ticketRepository;

  @Autowired
  private TicketDomainEventPublisher domainEventPublisher;

  public void accept(long ticketId, LocalDateTime readyBy) {
    Ticket ticket = ticketRepository.findById(ticketld).orElseThrow(() ->
            new TicketNotFoundException(ticketld));

    List<TicketDomainEvent> events = ticket.accept(readyBy);
    domainEventPublisher.publish(Ticket.class, orderld, events); // 도메인이벤트발행
  }
}
```

```java
public interface DomainEventPublisher {

  void publish(String aggregateType, Object aggregateId, List<DomainEvent>domainEvents);

}
```

## 주방 서비스 비즈니스 로직
- Restaurant 애그리거트와 Ticket 애그리거트는 이 서비스의 메인 애그리거트입니다.
  - Restaurant 애그리거트는 음식점 메뉴 및 운영 시간을 알고 있는 상태에서 주문을 검증할 수 있습니다.
  - 티켓은 배달원이 픽업할 수 있게 음식점이 미리 준비해야 할 주문을 나타냅니다.
- 주방 서비스에는 인바운드 어댑터가 3개 있습니다
  - RESTAPI: 음식점 점원이 사용하는 UI가 호출하는 RESTAPI. KitchenService를 호출하여 Ticket을 생성/수정합니다.
  - KitchenServiceCommandHandler: 사가가 호출하는 비동기 요청/응답 API. KitchenService를 호출하여 Ticket을 생성/수정합니다.
  - KitchenServiceEventConsumer: Restaurantservice 가 발행한 이벤트를 구독합니다. KitchenService를 호출하여 Restaurant을 생성/수정합니다.
- 아웃바운드 어댑터는 2개입니다.
  - DB 어댑터: TicketRepository. RestaurantRepository 인터페이스를 구현하여 DB에 접근합니다.
  - DomainEventPublishingAdapter: DomainEventPublisher 인터페이스를 구현하여 Ticket 도 메인 이벤트를 발행합니다.

```java
public class Ticket {

  public static ResultWithAggregateEvents<Ticket, TicketDomainEvent> create(long restaurantId, Long id, TicketDetails details) {
    return new ResultWithAggregateEvents<>(new Ticket(restaurantId, id, details));
  }

  public List<TicketPreparationStartedEvent> preparing() {
    switch (state) {
      case ACCEPTED:
        this.state = Ticketstate.PREPARING;
        this.preparingTime = LocalDateTime.now();
        return singletonList(new TicketPreparationStartedEvent());
      default:
        throw new UnsupportedStateTransitionException(state);
    }
  }

  public List<TicketDomainEvent> cancel() {
    switch (state) {
      case AWAITING_ACCEPTANCE:
      case ACCEPTED:
        this.previousState = state;
        this.state = Ticketstate.CANCEL_PENDING; return emptyList();
      default:
        throw new UnsupportedStateTransitionException(state);
    }
  }
}
```
- create()는 Ticket을 생성하고 preparing()은 음식점에서 주문을 준비하기 시작할 때 호출됩니다.
- preparing()은 주문 상태를 PREPARING으로 변경하고, 그 시간을 기록한 후 이벤트를 발행합니 다.
- cancel()는 사용자가 주문을 취소할 때 호출됩니다. 이 메서드는 취소가 가능한 상태면 주문 상태 변경 후 이벤트를 반환하지만, 취소가 불가능할 경우 예외를 던집니다.
- 이 세 메서드는 이벤트, 커맨드 메시지, REST API 요청에 반응하여 호출됩니다.

### KitchenService 도메인 서비스
- KitchenService는 주방 서비스의 인바운드 어댑터가 호출합니다.
- 주문 상태를 변경하는 accept(), reject(). preparing() 등의 메서드는 각각 애그리거트를 가져와 애그리거트 루트에 있는 해당 메서드를 호출한 후 도메인 이벤트를 발행합니다.

```java
public class KitchenService {

  @Autowired
  private TicketRepository ticketRepository;

  @Autowired
  private TicketDomainEventPublisher domainEventPublisher;

  public void accept(long ticketId, LocalDateTime readyBy) {
    Ticket ticket = ticketRepository.findById(ticketId).orElseThrow(() ->
            new TicketNotFoundException(ticketld));
    List<TicketDomainEvent> events = ticket.accept(readyBy);
    domainEventPublisher.publish(ticket, events); // 도메인 이벤트 발행
  }
}
```
- accept()는 음식점에서 새 주문을 접수할 때 다음 두 매개변수를 전달받아 호출됩니다.

### KitchenServiceCommandHandler 클래스
- 주문 서비스에 구현된 사가가 전송한 커맨드 메시지를 처리하는 어댑터입니다.
- KitchenService를 호출하여 Ticket을 생성/수정하는 핸들러 메서드가 커맨드별로 정의되어 있습니다

```java
public class KitchenServiceCommandHandler {

  @Autowired
  private KitchenService kitchenService;

  private Message createTicket(CommandMessage<CreateTicket> cm) {
    CreateTicket command = cm.getCommand();
    long restaurantId = command.getRestaurantId();
    Long ticketId = command.getOrderId();
    TicketDetails ticketDetails = command.getTicketDetails();

    try {
      Ticket ticket = kitchenService.createTicket(restaurantId,
              ticketld, ticketDetails);
      CreateTicketReply reply = new CreateTicketReply(ticket.getId());
      return withSuccess(reply);
    } catch (RestaurantDetailsVerificationException e) {
      return withFailure();
    }
  }

  private Message confirmCreateTicket (CommandMessage<ConfirmCreateTicket> cm) {
    Long ticketId = cm.getCommand().getTicketId();
    kitchenService.confirmCreateTicket(ticketId);
    return withSuccess();
  }
}
```

## 주문 서비스 비즈니스 로직
- 주문 서비스는 주문을 생성. 수정, 취소하는 API를 제공하는 서비스입니다. (컨슈머가 주로 호출)
- Order 애그리거트가 중 심을 차지하고 있지만, 음식점 서비스 데이터의 부분 레플리카인 Restaurant 애그리거트도 있습니다.
  - 덕분에 주문 서비스가 주문 품목을 검증하고 단가를 책정하는 일도 할 수 있습니다.
- 비즈니스 로직은 Order/Restaurant 애그리거트 외에도 OrderService, OrderRepository, RestaurantRepository, CreateOrderSaga 같은 여러 사가로 구성됩니다.

- 인바운드 어댑터
  - REST API: 컨슈머가 사용하는 UI가 호출하는 RESTAPI, OrderService를 호출하여 Order를 생성/수정합니다.
  - OrderEventConsumer: 음식점 서비스가 발행한 이벤트를 구독합니다. OrderService를 호출하여 Restaurant 레플리카를 생성/수정합니다.
  - OrderCommandHandler: 사가가 호출하는 비동기 요청/응답 기반의 API. OrderService를 호출하여 Order를 수정합니다.
  - SagaReplyAdapter: 사가 응답 채널을 구독하고 사가를 호출합니다.
- 아웃바운드 어댑터
  - DB 어댑터: OrderRepository 인터페이스를 구현하여 주문 서비스 DB에 접근합니다.
  - DomainEventPublishingAdapter: DomainEventPublisher 인터페이스를 구현하여 Order 도메인 이벤트를 발행합니 다.
  - OutboundCommandMessageAdapter: CommandPublisher 인터페이스를 구현한 클래스입니다. 커맨드 메시지를 사가 참여자에게 보냅니다.

```java
public class Order {

  public static ResultWithDomainEvents<Order, OrderDomainEvent> createOrder(long consumerId, Restaurant restaurant, List<OrderLineItem> orderLinelterns) {
    Order order = new Order(consumerId, restaurant.getld(), orderLineltems);
    List<OrderDomainEvent> events = singletonList(new OrderCreatedEvent(
                    new OrderDetails(consumerld, restaurant.getld(), orderLineltems, order. getOrderTotaK)),
            restaurant.getName());
    return new ResultWithDomainEvents<>(order, events);
  }

  public Order(long consumerld, long restaurantld, List<OrderLineItem> orderLineltems) {
    this.consumerld = consumerld;
    this.restaurantld = restaurantld;
    this.orderLineltems = new OrderLineltems(orderLineltems);
    this.state = APPROVAL_PENDING;
  }

  public List<OrderDomainEvent> noteApproved() {
    switch (state) {
      case APPROVAL_PENDING:
        this.state = APPROVED;
        return singletonList(new OrderAuthorized());
      default:
        throw new UnsupportedStateTransitionException(state);
    }
  }

  public List<OrderDomainEvent> noteRejected() {
    switch (state) {
      case APPROVAL_PENDING:
        this.state = REJECTED;
        return singletonList(new OrderRejected());
      default:
        throw new UnsupportedStateTransitionException(state);
    }
  }
}
```

- createOrder()는 주문을 생성하고 OrderCreatedEvent를 발행하는 정적 팩토리 메서드입니다.
  - OrderCreatedEvent는 주문 품목. 총액. 음식점 II). 음식점명 등 주문 내역이 포함된. 강화된 이벤 트입니다.
- Order는 처음에 APPROVAL_PENDING 상태로 출발합니다.
  - CreateOrderSaga 완료 시 소비자의 신용카드 승인까지 성공하면 noteApproved(), 서비스 중 하나라도 주문을 거부하거나 신용카드 승인이 실패하면 noteRejected()가 호출됩니다.
  - Order 애그리거트에 있는 메서드는 대부분 애그 리거트 상태에 따라 동작이 결정됩니다. Ticket 애그리거트처럼 이벤트 역시 발생시킵니다.

### OrderService 클래스

OrderService 클래스는 비즈니스 로직의 진입점입니다. 주문을 생성/수정하는 메서드가 모두 이 클래스에 있습니다. <br>
이 클래스의 메서드는 대부분 사가를 만들어 Order 애그리거트 생성/수정을 오케스트레이션합니다.

```java
@Transactional
public class OrderService {

  @Autowired
  private OrderRepository orderRepository;

  @Autowired
  private SagaManager<CreateOrderSagaState> createOrderSagaManager;

  @Autowired
  private SagaManager<ReviseOrderSagaState> reviseOrderSagaManagement;

  @Autowired
  private OrderDomainEventPublisher orderAggregateEventPublisher;

  public Order createOrder(long consumerId, long restaurantId, List<MenuItemIdAndQuantity> lineItems) {
    Restaurant restaurant = restaurantRepository.findByld(restaurantld)
            .orElseThrow(() -> new RestaurantNotFoundException(restaurantld));

    List<OrderLineItem> orderLineItems = makeOrderLinelterns(lineitems, restaurant); // Order 애그리거트 생성

    ResultWithDomainEvents<Order, OrderDomainEvent> orderAndEvents = Order.createOrder(consumerId, restaurant, orderLineltems);

    Order order = orderAndEvents.result;

    orderRepository.save(order); // Order를 DB에 저장

    orderAggregateEventPublisher.publish(order, orderAndEvents.events); // 도메인 이벤트 발행

    OrderDetails orderDetails = new OrderDetails(consumerld, restaurantld, orderLineltems, order.getOrderTotal());
    CreateOrderSagaState data = new CreateOrderSagaState(order.getId(), orderDetails);

    createOrderSagaManager.create(data, Order.class, order.getId()); // CreateOrderSaga 생성

    return order;
  }

  public Order reviseOrder(long orderId, OrderRevision orderRevision) {
    Order order = orderRepository.findByld(orderId) // Order 조회
            .orElseThrow(() -> new OrderNotFo니ndException(orderld));

    ReviseOrderSagaData sagaData = new ReviseOrderSagaData(order.getConsumerId(), orderId, null, orderRevision);
    reviseOrderSagaManager.create(sagaData);
    return order;
  }

}
```

주문 서비스는 주문을 생성하고 수정 할때 사가에 전적으로 의존합니다. <br>
다른 서비스에 있는 데이터가 트랜잭션 관점에서 일관성이 보장되어야 하기 때문입니다. <br>
그래서 OrderService 메서드는 대부분 직접 Order를 업데이트하지 않고 사가를 만듭니다.




