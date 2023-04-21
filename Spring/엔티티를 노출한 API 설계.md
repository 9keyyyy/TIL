# 엔티티를 노출한 API 생성하기

## 📍 @JsonIgnore 사용하기
- 양방향 연관관계의 엔티티에서 한 쪽에는 해당 어노테이션을 붙여줘야 무한 루프를 막을 수 있음

예시를 살펴보며 이해해보면, 

### Controller
```java
@GetMapping("/api/v1/simple-orders")
public List<Order> ordersV1() {
    List<Order> all = orderRepository.findAllByString(new OrderSearch());
    return all;
}
```


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

    @Enumerated(EnumType.STRING)
    private OrderStatus status; // 주문상태 [ORDER, CANCEL]
}


@Entity
@Getter
@Setter
public class Member {
    @Id @GeneratedValue
    @Column(name = "member_id")
    private Long id;

    @NotEmpty
    private String name;

    @Embedded
    private Address address;

    @JsonIgnore
    @OneToMany(mappedBy = "member") // mappedBy를 통해 연관관계의 거울임을 명시. Order 테이블의 member 필드가 주인
    private List<Order> orders = new ArrayList<>();
}

```
- 위의 Order 클래스는 Member를 필드로 갖고 있고, Member 클래스 또한 필드에 order를 가지고 있음
- 한 쪽에 `@JsonIgnore` 어노테이션을 붙이지 않으면, 위의 컨트롤러에서처럼 api에서 엔티티를 바로 사용하는 경우 무한 루프를 돌면서 계속 객체를 뽑아냄
- 즉, **양방향 연관관계에서는 한 쪽에 `@JsonIgnore` 어노테이션을 붙여 무한 루프가 돌지 않도록 해주어야 함**

</br></br>

## 📍 Hibernate5Module 사용하기
### 위의 API를 포스트맨으로 호출 시, 아래와 같은 오류가 뜸 

<img width="945" alt="스크린샷 2023-04-21 오후 6 31 04" src="https://user-images.githubusercontent.com/62213813/233601417-bdae1b04-e631-45ac-89ef-4bb88cf83b27.png">

- 해당 오류는 LAZY fetch 때문에 뜨는 것
- Order class를 보면 member는 fetch가 LAZY로 되어있음 > 지연 로딩
- 지연 로딩의 경우, DB에서 데이터를 가지고 올 때 order에서만 데이터를 가지고 옴 (member X) 
- 때문에 hibernate에서 아래와 같은 형식으로 프록시 객체를 가져옴 (bytebuddy) + member 값이 필요하게 되었을 때 DB에서 값 가져옴(프록시 초기화)
    ```java
    private Member member = new ByteBuddyInterceptor();
    ```
- 즉, member가 순수한 자바 객체가 아니라 발생하는 것!

</br>

### 해결 방법
- hibernate5module 사용 : 초기화 된 것은 값 내보내고, 그렇지 않은 경우 null값 반환하게 해줌
- `implementation 'com.fasterxml.jackson.datatype:jackson-datatype-hibernate5-jakarta'` 를 build.gradle에 추가
- main application 코드에 스프링 빈으로 등록해준 후 로딩 어떻게 할 지 설정
    ```java
	@Bean
	Hibernate5JakartaModule hibernate5Module(){
		Hibernate5JakartaModule hibernate5Module = new Hibernate5JakartaModule();
		hibernate5Module.configure(Hibernate5Module.Feature.FORCE_LAZY_LOADING, true);
		return hibernate5Module;
	}
    ```
- 위의 코드 상으로는 FORCE_LAZY_LOADING으로 설정했기 때문에, jpa 실행 시점에 강제로 로딩을 해옴
- FORCE_LAZY_LOADING으로 설정하지 않는 경우에는 강제로 컨트롤러에서 로딩하는 방법이 있음

</br></br>

## 📍 엔티티를 직접 노출해 API를 불러오는 방법의 문제점
1. API 스펙 상 필요 없는 정보를 받아오게 될 수 있음
2. 성능 상 이슈 : 필요 없는 정보를 받아오므로 
   - 위의 예시를 보면 hibernate5module를 통해 지연 로딩인 필드를 다 강제로 가져오게끔 함 > 그만큼 쿼리나감
   - hibernate5module을 사용하는 것보다는 dto로 변환해서 반환하는 것이 더 좋은 방법!
   - **강제 초기화를 막기 위해 필드를 EAGER fetch로 바꾸는 건 절대 해서는 안됨** > 성능 이슈



</br></br></br></br></br></br>
** '실전! 스프링 부트와 JPA 활용2' 강의를 듣고 작성했습니다.