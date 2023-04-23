# API 개발 컬렉션 조회 최적화

## 📍 컬렉션 조회 - 엔티티를 DTO로 변환 : 기본 (V2)
### Entity
```java

@Entity
@Table(name = "orders")
@Getter
@Setter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Order {
    @Id @GeneratedValue
    @Column(name="order_id")
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "member_id") // foreign key가 무엇인지 명시
    private Member member;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
    private List<OrderItem> orderItems = new ArrayList<>();

    @OneToOne(fetch = FetchType.LAZY, cascade = CascadeType.ALL)
    @JoinColumn(name ="delivery_id")    //연관관계 주인. FK가짐
    private Delivery delivery;

    private LocalDateTime orderDate; // 주문시간
}
```

### Controller
```java
@GetMapping("/api/v2/orders")
public List<OrderDto> ordersV2() {
    List<Order> orders = orderRepository.findAllByString(new OrderSearch());
    List<OrderDto> collect = orders.stream()
            .map(o -> new OrderDto(o))
            .collect(Collectors.toList());
    return collect;
}

@Getter
static class OrderDto{
    private Long orderId;
    private String name;
    private LocalDateTime orderDate;
    private OrderStatus orderStatus;
    private Address address;
//        private List<OrderItem> orderItems; // 엔티티 외부 노출하면 안됨!
    private List<OrderItemDto> orderItems;

    public OrderDto(Order order) {
        orderId = order.getId();
        name = order.getMember().getName();
        orderDate = order.getOrderDate();
        orderStatus = order.getStatus();
        address = order.getDelivery().getAddress();
        orderItems = order.getOrderItems().stream()
                .map(orderItem -> new OrderItemDto(orderItem))
                .collect(Collectors.toList());
    }


    @Getter
    static class OrderItemDto {

        private String itemName;
        private int orderPrice;
        private int count;
        public OrderItemDto(OrderItem orderItem) {
            itemName = orderItem.getItem().getName();
            orderPrice = orderItem.getOrderPrice();
            count = orderItem.getCount();
        }
    }
}
```
- 지연 로딩으로 인해 너무 많은 수의 SQL을 실행하게 됨
- 지연 로딩은 영속성 컨텍스트에 있으면 영속성 컨텍스트에 있는 엔티티를 사용하고, 없으면 SQL 쿼리 실행
- SQL 실행 수 (최악의 경우)
  - order : 1
  - member, address : N (order 조회 수 만큼)
  - orderItem : N (order 조회 수 만큼)
  - item : N (orderItem 조회 수 만큼)

</br></br>

## 📍 컬렉션 조회 - 엔티티를 DTO로 변환 : fetch join 최적화 (V3)

### Controller
```java
@GetMapping("/api/v3/orders")
public List<OrderDto> ordersV3() {
    List<Order> orders = orderRepository.findAllWithItem();

    List<OrderDto> collect = orders.stream()
            .map(o -> new OrderDto(o))
            .collect(Collectors.toList());
    return collect;
}
```

### Repository
```java
public List<Order> findAllWithItem() {
    return em.createQuery(
            "select distinct o from Order o"+   // distinct가 있으면 jpa가 자체적으로 order id가 같을 경우 중복 제거 (sql에서는 모든 필드 다 똑같아야 중복 제거)
            " join fetch o.member m" +
            " join fetch o.delivery d" +
            " join fetch o.orderItems oi" +
            " join fetch oi.item i", Order.class)
            .getResultList();
}
```
- fetch join으로 SQL이 1번만 실행됨
- `distinct`를 사용 : 1대다 조인이 있어 DB row가 증가하는데, 이에 따라 같은 order 엔티티 조회 수도 증가하게됨 > jpa의 distinct는 sql에 distinct를 추가 + 같은 엔티티 조회 시 어플리케이션에서 중복을 걸러줌
- 단점? **페이징 불가능**
  - 컬렉션 페치 조인을 사용할 경우 페이징이 불가능함
  - 페이징을 할 경우 하이버네이트는 경고 로그를 남기면서 모든 데이터를 DB에서 읽어오고, **메모리에서 페이징**을 해버림 > 이는 매우 위험
- 컬렉션 페치 조인은 1개만 사용할 수 있음 (컬렉션 둘 이상일 경우 데이터가 부정합하게 조회될 수 있으므로)

</br></br>

## 📍 컬렉션 조회 - 엔티티를 DTO로 변환 : 페이징과 한계 돌파 (V3.1)
- 컬렉션을 페치 조인하면 페이징이 불가능
  - 컬렉션을 페치 조인할 경우 일대다 조인이 발생하므로 데이터가 예측할 수 없이 증가
  - 일대다에서 "일"을 기준으로 페이징을 하는 것이 목적인데, 데이터는 "다"를 기준으로 row 생성
- 이 경우 하이버네이트는 경고 로그를 남기고 모든 db데이터를 읽어서 메모리에서 페이징 시도 > 최악의 경우 out of memory 나고 장애 발생

### 한계 돌파 : 페이징 + 컬렉션 엔티티를 함께 하려면?
- ToOne 관계를 모두 페치 조인 : ToOne 관계는 row 수를 증가시키지 않기 때문에 페이징 쿼리에 영향 x
- 컬렉션은 지연 로딩으로 조회
- 지연 로딩 최적화를 위해 `hibernate.default_batch_fetch_size`, `@BatchSize`를 적용

### Controller
```java
@GetMapping("/api/v3.1/orders")
public List<OrderDto> ordersV3_page(
        @RequestParam(value = "offset", defaultValue = "0") int offset,
        @RequestParam(value = "limit", defaultValue = "100") int limit){
    List<Order> orders = orderRepository.findAllWithMemberDelivery(offset, limit);

    List<OrderDto> collect = orders.stream()
            .map(o -> new OrderDto(o))
            .collect(Collectors.toList());
    return collect;
}
```

### Repository
```java
public List<Order> findAllWithMemberDelivery(int offset, int limit) {
    return em.createQuery(
            "select o from Order o" +
                    " join fetch o.member m" +
                    " join fetch o.delivery d", Order.class
    ).setFirstResult(offset)
            .setMaxResults(limit)
            .getResultList();
}
```

### application.yml
```yaml
jpa:
hibernate:
    ddl-auto: create-drop 
properties:
    hibernate:
    format_sql: true
    default_batch_fetch_size: 100
```
- 글로벌하게 적용하고 싶을 때는 `default_batch_fetch_size` 추가 : 설정한 사이즈 만큼 IN 쿼리로 조회
- 각 필드마다 사이즈를 다르게 적용하고 싶을 때는 해당 클래스로 가서 `@BatchSize` 어노테이션 적용

### 쿼리가 날라간 결과 확인
```bash
 select
        o1_0.order_id,
        d1_0.delivery_id,
        d1_0.city,
        d1_0.street,
        d1_0.zipcode,
        d1_0.status,
        m1_0.member_id,
        m1_0.city,
        m1_0.street,
        m1_0.zipcode,
        m1_0.name,
        o1_0.order_date,
        o1_0.status 
    from
        orders o1_0 
    join
        member m1_0 
            on m1_0.member_id=o1_0.member_id 
    join
        delivery d1_0 
            on d1_0.delivery_id=o1_0.delivery_id offset ? rows fetch first ? rows only
```
```bash
select
        o1_0.order_id,
        o1_0.order_item_id,
        o1_0.count,
        o1_0.item_id,
        o1_0.order_price 
    from
        order_item o1_0 
    where
        o1_0.order_id in(?,?)
```
```bash
select
        i1_0.item_id,
        i1_0.dtype,
        i1_0.name,
        i1_0.price,
        i1_0.stock_quantity,
        i1_0.artist,
        i1_0.etc,
        i1_0.author,
        i1_0.isbn,
        i1_0.actor,
        i1_0.director 
    from
        item i1_0 
    where
        i1_0.item_id in(?,?,?,?)
```
**장점**
- 원래 1, N, N 으로 나가던 쿼리가 1, 1, 1이 됨
- 조인보다 DB데이터 전송량이 최적화됨
- 페치 조인 방식과 비교해서 쿼리 호출 수가 약간 증가하지만, DB 데이터 전송량이 감소
- **컬렉션 페치 조인은 페이징이 불가능하지만 이 방법은 페이징이 가능**
  

### V3, V3.1 비교
- V3에서는 쿼리는 1개가 나가지만 일대다 조인으로 쿼리를 날린 데이터가 늘어나고, 이를 어플리케이션으로 모두 불러오기 때문에 중복된 데이터가 전송되므로 전송량 자체가 많아짐
- V3.1은 쿼리는 3개가 나가지만 쿼리 자체가 최적화되어 있음. 즉, 중복 없이 정확한 데이터만 전송

### 결론
- ToOne 관계는 페치 조인해도 페이징에 영향을 주지 않음. 따라서 ToOne 관계는 페치 조인으로 쿼리 수를 줄이고, 나머지는 `hibernate.default_batch_fetch_size`로 최적화하자!
- `hibernate.default_batch_fetch_size` 사이즈는 100 ~ 1000 사이를 선택하는 것을 권장. 이 전략은 SQL IN 절을 사용하는데, 데이터베이스에 따라 IN 절 파라미터를 1000으로 제한하기도 하므로.
- 1000으로 잡을 경우 1000개를 DB에서 불러오므로 DB에 순간 부하가 증가할 수 있음 (순간 부하를 어디까지 견딜 수 있는지로 결정하는 것이 좋음)

</br></br>

## 📍 컬렉션 조회 - JPA에서 DTO 직접 조회 : 기본 (V4)
### Controller
```java
@GetMapping("/api/v4/orders")
public List<OrderQueryDto> ordersV4(){
    return orderQueryRepository.findOrderQueryDtos();
}
```
### Repository
```java
@Data
public class OrderQueryDto {

     @JsonIgnore
     private Long orderId;
     private String name;
     private LocalDateTime orderDate;
     private OrderStatus orderStatus;
     private Address address;
     private List<OrderItemQueryDto> orderItems;

     public OrderQueryDto(Long orderId, String name, LocalDateTime orderDate, OrderStatus orderStatus, Address address) {
          this.orderId = orderId;
          this.name = name;
          this.orderDate = orderDate;
          this.orderStatus = orderStatus;
          this.address = address;
     }
}

@Data
public class OrderItemQueryDto {

    private Long orderId;
    private String itemName;
    private int orderPrice;
    private int count;

    public OrderItemQueryDto(Long orderId, String itemName, int orderPrice, int count) {
        this.orderId = orderId;
        this.itemName = itemName;
        this.orderPrice = orderPrice;
        this.count = count;
    }
}

public List<OrderQueryDto> findOrderQueryDtos(){
    List<OrderQueryDto> result = findOrders();

    result.forEach(o -> {
        List<OrderItemQueryDto> orderItems = findOrderItems(o.getOrderId());
        o.setOrderItems(orderItems);
    });

    return result;
}

private List<OrderItemQueryDto> findOrderItems(Long orderId) {
    return em.createQuery(
            "select new jpabook.jpashop.repository.order.query.OrderItemQueryDto(oi.order.id, i.name, oi.orderPrice, oi.count)" +
                    " from OrderItem oi" +
                    " join oi.item i" +
                    " where oi.order.id = :orderId", OrderItemQueryDto.class)
            .setParameter("orderId", orderId)
            .getResultList();
}

private List<OrderQueryDto> findOrders() {
    return em.createQuery(
                    "select new jpabook.jpashop.repository.order.query.OrderQueryDto(o.id, m.name, o.orderDate, o.status, d.address)" +
                            " from Order o" +
                            " join o.member m" +
                            " join o.delivery d", OrderQueryDto.class)
            .getResultList();
}

```
- query는 루트에서 1번, 컬렉션 N번 실행
- ToOne 관계 먼저 조회 > ToMany 관계 별도로 처리
  - `findOrders()` > `findOrderItems()` 루프
  - ToOne 관계는 조인해도 row 수 증가 X
  - ToMany 관계는 조인하면 row 수 증가
- 즉, row 수가 증가하지 않는 ToOne 관계는 조인으로 최적화하기 쉬우므로 한번에 조회하고, ToMany 관계는 최적화하기 어려우므로 별도의 메서드로 조회



</br></br>

## 📍 컬렉션 조회 - JPA에서 DTO 직접 조회 : 컬렉션 조회 최적화 (V5)

### Controller
```java
@GetMapping("/api/v5/orders")
public List<OrderQueryDto> ordersV5(){
    return orderQueryRepository.findAllByDto_optimization();
}
```

### Repository
```java
public List<OrderQueryDto> findAllByDto_optimization() {
    List<OrderQueryDto> result = findOrders();

    // order id 추출
    List<Long> orderIds = result.stream()
            .map(o -> o.getOrderId())
            .collect(Collectors.toList());

    // query 1번
    List<OrderItemQueryDto> orderItems = em.createQuery(
                    "select new jpabook.jpashop.repository.order.query.OrderItemQueryDto(oi.order.id, i.name, oi.orderPrice, oi.count)" +
                            " from OrderItem oi" +
                            " join oi.item i" +
                            " where oi.order.id in :orderIds", OrderItemQueryDto.class)
            .setParameter("orderIds", orderIds)
            .getResultList();

    // 메모리에서 값 세팅
    Map<Long, List<OrderItemQueryDto>> orderItemMap = orderItems.stream()
            .collect(Collectors.groupingBy(orderItemQueryDto -> orderItemQueryDto.getOrderId()));
    result.forEach(o -> o.setOrderItems(orderItemMap.get(o.getOrderId())));

    return result;
}
```
- ToOne 관계들을 먼저 조회하고, 여기서 얻은 식별자 orderId로 ToMany 관계인 `OrderItem`을 한꺼번에 조회
- map을 사용해서 매칭 성능 향성 (O(1))
- 쿼리 총 2번 나가게 되는데, 나간 쿼리를 보면
  - 루트 1번 : member , delivery 조인해서 order 가져오는 쿼리
  - 컬렉션 1번 : orderItem과 item 조인하되, IN 쿼리로 orderItem 한번에 가져옴
    ```
    select
            o1_0.order_id,
            m1_0.name,
            o1_0.order_date,
            o1_0.status,
            d1_0.city,
            d1_0.street,
            d1_0.zipcode 
        from
            orders o1_0 
        join
            member m1_0 
                on m1_0.member_id=o1_0.member_id 
        join
            delivery d1_0 
                on d1_0.delivery_id=o1_0.delivery_id

    select
            o1_0.order_id,
            i1_0.name,
            o1_0.order_price,
            o1_0.count 
        from
            order_item o1_0 
        join
            item i1_0 
                on i1_0.item_id=o1_0.item_id 
        where
            o1_0.order_id in(?,?)
    ```



</br></br>

## 📍 컬렉션 조회 - JPA에서 DTO 직접 조회 : 플랫 데이터 최적화 (V6)
### Controller
```java
@GetMapping("/api/v6/orders")
public List<OrderQueryDto> ordersV6(){
    List<OrderFlatDto> flats = orderQueryRepository.findAllByDto_flat();

    return flats.stream()
            .collect(groupingBy(o -> new OrderQueryDto(o.getOrderId(),
                            o.getName(), o.getOrderDate(), o.getOrderStatus(), o.getAddress()),
                    mapping(o -> new OrderItemQueryDto(o.getOrderId(),
                            o.getItemName(), o.getOrderPrice(), o.getCount()), toList())
            )).entrySet().stream()
            .map(e -> new OrderQueryDto(e.getKey().getOrderId(),
                    e.getKey().getName(), e.getKey().getOrderDate(), e.getKey().getOrderStatus(),
                    e.getKey().getAddress(), e.getValue()))
            .collect(toList());
}
```
- 위의 코드에서 `OrderQueryDto`가 groupingby 할 기준을 정해주기 위해 `OrderQueryDto` 클래스에 `@EqualsAndHashCode(of = "orderId")` 어노테이션을 추가해주어야 함


### Repository
```java
@Data
public class OrderFlatDto {
    private Long orderId;
    private String name;
    private LocalDateTime orderDate;
    private OrderStatus orderStatus;
    private Address address;

    public OrderFlatDto(Long orderId, String name, LocalDateTime orderDate, OrderStatus orderStatus, Address address, String itemName, int orderPrice, int count) {
        this.orderId = orderId;
        this.name = name;
        this.orderDate = orderDate;
        this.orderStatus = orderStatus;
        this.address = address;
        this.itemName = itemName;
        this.orderPrice = orderPrice;
        this.count = count;
    }

    private String itemName;
    private int orderPrice;
    private int count;

}

public List<OrderFlatDto> findAllByDto_flat() {
    return em.createQuery(
            "select new " +
                    " jpabook.jpashop.repository.order.query.OrderFlatDto(o.id, m.name, o.orderDate, o.status, d.address, i.name, oi.orderPrice, oi.count)" +
                    " from Order o" +
                    " join o.member m" +
                    " join o.delivery d" +
                    " join o.orderItems oi " +
                    " join oi.item i", OrderFlatDto.class)
            .getResultList();
}
```
- query 1번 호출
- 조인으로 인해 db에서 애플리케이션에 전달하는 데이터에 중복 데이터가 추가됨 (상황에 따라 V5 보다 느릴 수 있음)
- 어플리케이션에서 추가 작업이 큼 (FlatDto > QueryDto로 분해하는 작업이 추가되므로)
- 페이징 불가능 

</br></br></br></br></br></br>
** '실전! 스프링 부트와 JPA 활용2' 강의를 듣고 작성했습니다.