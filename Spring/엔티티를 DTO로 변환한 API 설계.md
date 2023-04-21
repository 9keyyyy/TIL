# 엔티티를 DTO로 변환한 API 설계

## 📍 엔티티를 DTO로 변환하기 - 기본
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

- order를 보면 member와 delivery 엔티티와 연관되어 Lazy 로딩되는 것을 알 수 있음

### Controller
```java
@GetMapping("/api/v2/simple-orders")
public List<SimpleOrderDto> ordersV2(){
    List<Order> orders = orderRepository.findAllByString(new OrderSearch());
    List<SimpleOrderDto> result = orders.stream()
            .map(o -> new SimpleOrderDto(o))
            .collect(Collectors.toList());
    return result;
}

@Data
static class SimpleOrderDto {
    public SimpleOrderDto(Order order) {
        orderId = order.getId();
        name = order.getMember().getName(); //LAZY 초기화 : 영속성 컨텍스트가 없는 경우 DB 쿼리 날림
        orderDate = order.getOrderDate();
        orderStatus = order.getStatus();
        address = order.getDelivery().getAddress(); //LAZY 초기화
    }

    private Long orderId;
    private String name;
    private LocalDateTime orderDate;
    private OrderStatus orderStatus;
    private Address address; //배송지 정보
}

```
- api를 호출할 경우 dto에서 name, address가 초기화 될 때 LAZY 초기화가 되는 것을 알 수 있음 (지연 로딩은 영속성 컨택스트가 없는 경우 db에 쿼리를 날려 데이터를 받아오기 때문)
- 주문이 2개 있다고 할 때, 쿼리가 나가는 순서를 정리해보면
  1. Order > SQL 1번 > 결과 주문 수 2개 > 2번 루프 돔
  2. 1차 루프 : member, delivery SQL 각각 1번 
  3. 2차 루프 : member, delivery SQL 각각 1번 
- 즉, 총 쿼리 5번
- N + 1 문제 발생 : 1 + 회원 N + 배송 N (최악의 경우)
</br></br>


## 📍 엔티티를 DTO로 변환하기 - fetch join 사용하기
위의 코드를 최적화하기 위해 fetch join을 사용해보면,

### Controller
```java
@GetMapping("/api/v3/simple-orders")
public List<SimpleOrderDto> orderV3(){
    List<Order> orders = orderRepository.findAllWithMemberDelivery();
    List<SimpleOrderDto> result = orders.stream()
            .map(o -> new SimpleOrderDto(o))
            .collect(Collectors.toList());
    return result;
}
```
### Repository
```java
public List<Order> findAllWithMemberDelivery() {
    return em.createQuery(
            "select o from Order o" +
                    " join fetch o.member m" +
                    " join fetch o.delivery d", Order.class
    ).getResultList();
}
```
- 엔티티를 페치 조인(fetch join)을 사용해서 **쿼리 1번**에 조회
- 페치 조인으로 order > member, delivery 는 이미 조회된 상태이므로 지연 로딩이 일어나지 않음
- 실무에서 자주 사용되는 기법 (재사용성도 있음)
- api를 호출할 경우 아래와 같이 쿼리를 날리게 됨
  ```
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
            on d1_0.delivery_id=o1_0.delivery_id
  ```

</br></br>

## 📍 jpa에서 dto 바로 조회
fetch join 사용에서 더 쿼리를 최적화하기 위해 엔티티를 가져와서 dto에 대입하는 방법이 아니라 바로 dto를 조회하는 방법을 사용해보면,

### Controller
```java
@GetMapping("/api/v4/simple-orders")
public List<OrderSimpleQueryDto> orderV4(){
    return orderRepository.findOrderDtos();
}
```
### Repository
```java
public List<OrderSimpleQueryDto> findOrderDtos() {
    return em.createQuery(
            "select new jpabook.jpashop.repository.OrderSimpleQueryDto(o.id, m.name, o.orderDate, o.status, d.address) " +
                    " from Order o" +
            " join o.member m" +
            " join o.delivery d", OrderSimpleQueryDto.class)
            .getResultList();
}
```
```java
public class OrderSimpleQueryDto {
    public OrderSimpleQueryDto(Long orderId, String name, LocalDateTime orderDate, OrderStatus orderStatus, Address address) {
        this.name = name;
        this.orderDate = orderDate;
        this.orderId = orderId;
        this.orderStatus = orderStatus;
        this.address = address;
    }

    private Long orderId;
    private String name;
    private LocalDateTime orderDate;
    private OrderStatus orderStatus;
    private Address address; //배송지 정보
}

```
- 일반적인 SQL을 사용할 때처럼 원하는 값을 선택해서 조회
- `new` 명령어를 사용해 JPQL의 결과를 DTO로 즉시 변환
- fetch join을 사용한 것에 비해 조금 더 성능이 최적화됨 (select 절에서 필요한 정보만 선택해 불러오므로 네트워크 용량 최적화)
- 화면에는 최적화되어 있지만 재사용성은 없음
- api를 호출할 경우 아래와 같이 쿼리가 생성됨
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
    ```

</br></br>

## 📍 쿼리 선택 방식 권장 순서 (엔티티를 DTO로 변환 / DTO 바로 조회 방식 비교)
- 우선, 엔티티를 DTO로 변환하는 방법 선택 
- 필요 시 fetch join으로 성능 최적화 > 대부분의 성능 이슈 해결됨
- 그래도 안될 경우 DTO로 직접 조회하는 방법 선택
- 최후의 방법은 JPA가 제공하는 네이티브 SQL이나 스프링 JDBC Template을 사용해서 SQL을 직접 사용



</br></br></br></br></br></br>
** '실전! 스프링 부트와 JPA 활용2' 강의를 듣고 작성했습니다.