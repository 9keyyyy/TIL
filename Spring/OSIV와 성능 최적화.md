# OSIV와 성능 최적화

- Open Session In View: 하이버네이트
- Open EntityManager In View: JPA

## 📍 OSIV ON
<img width="534" alt="스크린샷 2023-04-23 오후 9 32 37" src="https://user-images.githubusercontent.com/62213813/233840023-cbf5f155-64c5-4af2-91cb-283a8c7fa827.png"> </br>

- `spring.jpa.open-in.view` : true 기본값
- 위의 기본값을 뿌리면서, 애플리케이션 시작 시점에 warn 로그를 남김
- 기본적으로 jpa는 트랜잭션을 시작할 때 db 커넥션을 가져옴 (영속성 컨텍스트) > api가 유저에게 반환될 때까지 살아있음
- 즉, 고객의 요청에 Response가 갈 때까지 영속성 컨텍스트가 살아있기 때문에 View Template이나 API 컨트롤러에서 지연 로딩이 가능했던 것
- 지연 로딩은 영속성 컨텍스트가 살아있어야 가능 > 영속성 컨텍스트는 데이터베이스 커넥션을 유지해야 함
- 하지만 이 전략은 너무 오랜시간동안 db connection 리소스를 사용 > 실시간 트래픽이 중요한 애플리케이션에서는 커넥션이 모자랄 수 있음 > 장애

## 📍 OSIV OFF
<img width="534" alt="스크린샷 2023-04-23 오후 9 51 47" src="https://user-images.githubusercontent.com/62213813/233840868-1234c00c-1eff-4b0a-9229-8e7b7bb9a2a3.png"> </br>

- `spring.jpa.open-in.view` : false
- 트랜잭션이 종료될 때 영속성 컨텍스트를 닫음 + 데이터베이스 커넥션 반환
- OSIV를 끄면 지연 로딩을 트랜잭션 안에서 처리해야 함 > 트랜잭션이 끝나기 전 지연 로딩을 강제로 호출해 두어야 함 (or fetch join 사용)
- 아래와 같이 컨트롤러에서 지연 로딩을 한 경우, 프록시를 초기화 할 수 없다는 에러가 뜸 (`LazyInitialzizationException`)
  ```java
    @GetMapping("/api/v1/simple-orders")
    public List<Order> ordersV1() {
    List<Order> all = orderRepository.findAllByString(new OrderSearch());
    for (Order order : all) {
        order.getMember().getName();        // Lazy 로딩 강제 초기화 : 실제 필드(name)를 불러오는 경우
        order.getDelivery().getAddress();   // Lazy 로딩 강제 초기화
    }
    return all;
    }
  ```

## 📍 커멘드와 쿼리 분리
- 실무에서 **OVIS를 끈 상태**로 복잡성을 관리하는 좋은 방법이 Command와 Query를 분리하는 것
- 보통 비지니스 로직은 특정 엔티티 몇 개를 등록하거나 수정하는 것이므로 성능이 크게 문제되지 않음
- 성능 이슈는 조회에서 발생 (복잡한 화면을 출력하기 위한 쿼리 - 화면에 맞추어 성능 최적화 하는 것이 중요)
- 때문에 크고 복잡한 애플리케이션 개발 시, 이 둘의 관심사를 명확하게 분리하는 것이 유지보수 관점에서 좋음

예를 들어,
### OrderService
- OrderService : 핵심 비지니스 로직
- OrderQueryService : 화면이나 API에 맞춘 서비스 (주로 읽기 전용 트랜잭션 사용)

보통 서비스 계층에서 트랜잭션을 유지하기 때문에, 두 서비스 모두 트랜잭션을 유지하면서 지연 로딩을 사용할 수 있음


</br></br></br></br></br></br>
** '실전! 스프링 부트와 JPA 활용2' 강의를 듣고 작성했습니다.ㅑ