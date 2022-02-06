# API Basic



## Member API



### SaveMember



#### V1

```java
@RestController
@RequiredArgsConstructor
public class MemberApiController {
    
private final MemberService memberService;

    /**
     * Post 로직
     */

    // 첫 버전의 회원 등록 api
    @PostMapping("/api/v1/members")
    private CreateMemberResponse saveMemberV1(@RequestBody @Valid Member member){
        Long id = memberService.join(member);
        return new CreateMemberResponse(id);
    }

	@Data
    static class CreateMemberRequest{
        @NotEmpty
        private String name;
    }


    @Data
    static class CreateMemberResponse{
        private Long id;

        public CreateMemberResponse(Long id) {
            this.id = id;
        }

    }
}
```

- #### SaveMemberV1은 엔티티를 직접 받는다.

- #### @RestController은 @Controller, @ResponseBody가 모두 포합되어 있으며 @ResponseBody는 클라이언트로 자바 객체를 응답할 수 있게 한다(응답에는 XML, JSON 형식 등이 있다).

- #### @RequestBody는 JSON, XML 요청을 받을 수 있게 한다.

- #### @Valid는 @RequestBody로 들어오는 데이터에 대한 검증을 수행하며 세부 사항은(@NotNull, @Email, @NotEmpty 등) 객체 내부에 정의해야 한다. 세부 사항에 대한 에러 로그는 스프링 부트를 설정해야 하며 에러 발생 시 에러 내용이 응답에 message로 전송된다. 이전 화면에서 들어오는 데이터로 컨트롤러에서 validation 하는 것을 presentation batching 이라고 한다.

- #### 엔티티는 범용적으로 사용될 수 있어야 하며 1 : 1 매핑 문제가 발생하면 안됀다.

- #### SaveMemberV1은 엔티티에 세부 사항을 작성 했으며 여러 API 용도로 사용되기 힘들게 설계되었다.



#### V2

```java
    @PostMapping("/api/v2/members")
    public  CreateMemberResponse saveMemberV2(@RequestBody @Valid CreateMemberRequest request){
        Member member = new Member();
        // JSON에서 읽는 방식
        member.setName(request.getName());
        Long id = memberService.join(member);

        return new CreateMemberResponse(id);
    }
```

- #### SaveMemberV2는 SaveMemberV1와 다르게 DTO를 사용하여 SaveMemberV1의 문제점을 보완한다.

- #### DTO를 사용하여 API 스펙이 변하지 않게 했으며 세부 사항을 DTO에 작성하여 클라이언트의 요청에 대한 스펙을 명확히 할 수 있다.



### UpdateMember



#### V2

```java
	/**
     * update 로직
     */

    @PutMapping("/api/v2/members/{id}")
    public UpdateMemberResponse updateMemberV2(
            @PathVariable("id") Long id,
            @RequestBody @Valid UpdateMemberRequest request){

        memberService.update(id, request.getName());
        Member findMember = memberService.findOne(id);
        return new UpdateMemberResponse(findMember.getId(), findMember.getName());
    }

    @Data
    static class UpdateMemberRequest{
        private String name;
    }

    // DTO는 로직이 크게 있는 것도 아니므로 @AllArgsConstructor을 사용한다. 다른 곳에서는 사용하지 말자
    // @AllArgsConstructor로 모든 파라미터에 값을 생성자로 넣어줘야 한다
    @Data
    @AllArgsConstructor
    static class UpdateMemberResponse{
        private Long id;
        private String name;
    }
```

```java
public class MemberService{

	private final MemberRepository memberRepository;
    
    @Transactional
    public void update(Long id, String name) {
        Member member = memberRepository.findOne(id);
        member.setName(name);
    }

}
```

- #### UpdateMemberV2의 findOne()은 서비스 계층을 호출하는 것이며 MemberService의 update는 리포지토리 계층을 호출하는 것이다.

- #### 각 클래스의 조회, 업데이트 로직을 분리하여 유지보수를 쉽게 하였다. Update()메소드의 반환형이 Member이라면 메소드 내에 command(업데이트), query(조회)가 모두 같이 있는 것이므로 반환형을 void로 한다.

- #### PUT 방식은 전체 업데이트를 할 때 사용한다. 부분 업데이트를 하려면 PATCH, POST룰 사용하는 것이 좋다.



### Members



#### V1

```java
	@GetMapping("/api/v1/members")
    public List<Member> memberV1(){
        return memberService.findMembers();
    }
```

- #### MembersV1은 엔티티를 직접 조회하는 방식이다. 즉, 엔티티의 모든 값이 노출되어 위험하다.

- #### 이 방법은 엔티티 스펙이 계속 변경될 수 있으며 각 API가 모두 만족할 스펙을 만드는 것은 어려우므로 엔티티가 범용적으로 사용되지 못하게 한다.

- #### 엔티티 전체를 반환하기 때문에 원하지 않는 정보를 전송하여 부하가 커지게 한다.

- #### 컬렉션을 직접 반환하면 향후 API 스펙을 변경하기 어렵다. 즉, 컬렉션 데이터와 추가로 다른 데이터를 추가하여 전송할 수 없다. 확정성과 유연성이 보장되지 않는 것이다.

- #### @JsonIgnore을 객체 위에 선언하면 그 데이터는 응답으로 전송되지 않는다.



#### V2

```java
    @GetMapping("/api/v2/members")
    public Result memberV2(){
        List<Member> findMembers = memberService.findMembers();
        // List Member를 List DTO로 바꾼 것이다.
        List<MemberDto> collect = findMembers.stream()
                .map(m -> new MemberDto(m.getName()))
                .collect((Collectors.toList()));

        //
        return new Result(collect.size(), collect);
    }

    @Data
    @AllArgsConstructor
    static class Result<T>{
        // 유연하게 변수 추가 가능하다
        private int count;
        private T data;
    }

    @Data
    @AllArgsConstructor
    static class MemberDto{
        private String name;
    }
```

- #### MembersV2는 DTO를 사용하여 MembersV1의 문제점을 보완한다.

- #### @AllArgsConstructor로 자동으로 변수에 대한 생성자가 생성된다.

- #### DTO를 사용하여 API 스펙에 정확히 맞는 데이터만 저장한다.

- #### 컬렉션을 제네릭으로 감싸서 JSON을 한단계 더 생성하여 컬렉션 뿐만 아니라 다른 데이터도 함께 응답으로 전송할 수 있게 한다.



# Lazy Loading & Inquiry Optimization



## OrderSimple API



### Orders



#### V1

```java
/**
 * xToOne(ManyToOne, OneToOne)
 * Order
 * Order -> Member
 * Order -> Delivery
 */
@RestController
@RequiredArgsConstructor
public class OrderSimpleApiController {

    private final OrderRepository orderRepository;
    private final OrderSimpleQueryRepository orderSimpleQueryRepository;

    @GetMapping("/api/v1/simple-orders")
    public List<Order> ordersV1(){
        List<Order> all = orderRepository.findAllByString(new OrderSearch());
        for (Order order : all) {
            order.getMember().getName();
            order.getDelivery().getAddress();
        }
        return all;
    }
}
```

- #### 주문, 배송정보, 회원을 조회할 것이다.

- #### OrdersV1은 엔티티를 직접 노출시키는 방법이다.

- #### FindAllByString() 메소드만 사용하면 원하는 데이터 외에 다른 데이터도 클라이언트에게 전송하게 되어 낭비가 발생한다.

- #### FindAllByString() 메소드를 사용할 때 엔티티 사이의 양방향 매핑 때문에 무한 loop에 빠질 수 있다. 사용 빈도수가 더 적은 엔티티의 객체에 @JsonIgnore하여 방지할 수 있다.

- #### Lazy 로딩으로 Order 객체 내부에 초기화되지 않고 ByteBuddyInterceptor이라는 프록시 객체로 남아있는 객체가 많다. Jackson은 프록시를 어떤 방식으로 json 파일을 생성해야할지 모르므로 Hibernate5Module을 사용하여 직접 빈에 등록해야 한다. 초기화 되지 않은 객체는 null로 반환될 것이다.

- #### 하지만 Hibernate5Module을 사용해도 아직 엔티티를 직접 노출하고 있다. 앞서 말한 엔티티를 직접 노출하여 발생하는 문제점이 아직 존재한다는 뜻이다.



#### V2

```java
    @GetMapping("/api/v2/simple-orders")
    public List<SimpleOrderDto> ordersV2(){
        // ORDER 2개 조회
        // N + 1 -> 1 + 회원 N + 배송 N
        List<Order> orders = orderRepository.findAllByString(new OrderSearch());

        // 2개
        List<SimpleOrderDto> result = orders.stream()
                .map(o -> new SimpleOrderDto(o))
                .collect(Collectors.toList());

        return result;
    }

    @Data
    static class SimpleOrderDto{
        private Long orderId;
        private String name;
        private LocalDateTime orderDate;
        private OrderStatus orderStatus;
        private Address address;

        public SimpleOrderDto(Order order) {
            orderId = order.getId();
            name = order.getMember().getName();
            orderDate = order.getOrderDate();
            orderStatus = order.getStatus();
            address = order.getDelivery().getAddress();
        }
    }
```

- #### OrdersV2는 DTO를 사용하여 OrderV1의 엔티티 직접 노출 문제를 보완한다.

- #### 원하는 데이터를 DTO를 통해 저장하고 Lazy 로딩을 통해 초기화한다.

- #### Order, member, delivery Lazy 로딩을 통해 초기화하여 1 + N + N 쿼리가 실행되어 너무 많은 쿼리를 요구하게 되는 문제점이 있다. OrdersV1과 OrdersV2의 쿼리 개수는 동일하다.



#### V3

```java
    // Entity를 조회 후 dto로 변환
    // 로직 재활용 가능 - 엔티티 조회하기 때문
    // V4와 장단점 tradeoff 있다
    @GetMapping("/api/v3/simple-orders")
    public List<SimpleOrderDto> ordersV3(){
        List<Order> orders = orderRepository.findAllWithMemberDelivery();
        List<SimpleOrderDto> result = orders.stream()
                .map(o -> new SimpleOrderDto(o))
                .collect(Collectors.toList());

        return result;
    }
```

```java
	/** 
	* OrderRepository
	*/
	public List<Order> findAllWithMemberDelivery() {
        return em.createQuery(
                "select o from Order o" +
                        " join fetch o.member m" +
                        " join fetch o.delivery d", Order.class
        ).getResultList();
    }
```

- #### OrdersV3는 페치 조인을 이용하여 하나의 쿼리로 원하는 데이터의 객체를 조인하여 OrderV2의 문제점을 보완한다.

- #### 엔티티를 조회 후 DTO로 변환하는 과정이다.

- #### 엔티티를 조회하기 때문에 로직을 재활용할 수 있다.



#### V4

```java
    // Entity 조회하지 않고 바로 dto로 저장
    // 불필요한 정보 조회 하지 않고 조금 더 성능 최적화 가능
    @GetMapping("/api/v4/simple-orders")
    public List<OrderSimpleQueryDto> ordersV4(){
        return orderSimpleQueryRepository.findOrderDtos();
    }
```

```java
/**
* OrderSimpleQueryRepository
*/
@Repository
@RequiredArgsConstructor
public class OrderSimpleQueryRepository {

    private final EntityManager em;

    public List<OrderSimpleQueryDto> findOrderDtos() {
        return em.createQuery(
                "select new jpabook.jpashop.repository.order.simplequery.OrderSimpleQueryDto(o.id, m.name, o.orderDate, o.status, d.address) from Order o" +
                        " join o.member m" +
                        " join o.delivery d", OrderSimpleQueryDto.class
        ). getResultList();

    }

}
```

```java
/**
* OrderSimpleQueryDto
*/
@Data
public class OrderSimpleQueryDto {
    private Long orderId;
    private String name;
    private LocalDateTime orderDate;
    private OrderStatus orderStatus;
    private Address address;

    public OrderSimpleQueryDto(Long orderId, String name, LocalDateTime orderDate, OrderStatus orderStatus, Address address) {
        this.orderId = orderId;
        this.name = name;
        this.orderDate = orderDate;
        this.orderStatus = orderStatus;
        this.address = address;
    }
}

```

- #### OrdersV4는 엔티티를 조회 후 DTO로 변환하는 것이 아닌 DTO로 바로 조회하는 방식이다.

- #### 원하는 데이터만 조회하기 때문에 select 절에 new를 사용한다.

- #### Select 절에서 원하는 데이터를 직접 선택하여 어플리케이션 네트워크 용량을 최적화한다. 데이터 크기의 차이가 클 때 OrdersV3보다 성능이 좋지만 일반적인 경우에서는 성능차이가 거의 없다.

- #### 이 방식은 엔티티를 직접 조회하지 않으며 화면에 의존적이다. 또한, 클래스의 범용성이 감소하며 리포지토리에 API 스펙이 모두 포함되어 있어서 논리적 계층의 의미가 모호해진다. 따라서 OrderRepository에 로직을 생성하지 않고 따로 클래스를 생성한다(OrderSimpleQueryRepository).



### Arrangement



1. #### 엔티티를 DTO로 변환하는 방법을 선택하여 작성한다.

2. #### 여러 객체 사용하는 경우 페치 조인으로 성능을 최적화 한다.

3. #### 성능이 좋지 않다면 DTO로 직접 조회하는 방법을 사용한다.

4. #### 세가지 경우 모두 안좋을 경우 네이트브 SQL이나 스프링 JDBC Template을 사용하여 SQL을 직접 작성한다.



# Collection Optimization



## Order API



### Orders

- #### 이전까지는 xToOne 관계를 다뤘지만 이번에는 xToMany 관계를 다룰 것이다.



#### V1

```java
@RestController
@RequiredArgsConstructor
public class OrderApiController {

    // 1 : N 설명할 것이다
    private final OrderRepository orderRepository;
    private final OrderQueryRepository orderQueryRepository;

    @GetMapping("/api/v1/orders")
    public List<Order> ordersV1(){
        List<Order> all = orderRepository.findAllByString(new OrderSearch());
        for (Order order : all) {
          order.getMember().getName();
          order.getDelivery().getAddress();

            List<OrderItem> orderItems = order.getOrderItems();
//            for (OrderItem orderItem : orderItems) {
//                orderItem.getItem().getName();
//            }
            orderItems.stream().forEach(o -> o.getItem().getName());
        }
        return all;
    }
}
```

- #### 엔티티를 직접 노출하는 방식이다.

- #### 필요한 데이터를 모두 Lazy 로딩으로 초기화 한다.

- #### 초기화 하지 않은 데이터는 Hibernate5Module로 null 값을 반환한다.



#### V2

```java
    // V2, V3 코드 동일
  @GetMapping("/api/v2/orders")
    public List<OrderDto> orderV2(){
        List<Order> orders = orderRepository.findAllByString(new OrderSearch());
        List<OrderDto> collect = orders.stream()
                .map(o -> new OrderDto(o))
                .collect(toList());

        return collect;
    }

    @Getter
    static class OrderDto{

        private Long orderId;
        private String name;
        private LocalDateTime orderDate;
        private OrderStatus orderStatus;
        private Address address;
//        private List<OrderItem> orderItems;
        private List<OrderItemDto> orderItems;

        public OrderDto(Order order) {
            orderId = order.getId();
            name = order.getMember().getName();
            orderDate = order.getOrderDate();
            orderStatus = order.getStatus();
            address = order.getDelivery().getAddress();
//            order.getOrderItems().stream().forEach(o -> o.getItem().getName());
//            orderItems = order.getOrderItems();
            orderItems = order.getOrderItems().stream()
                    .map(orderItem -> new OrderItemDto(orderItem))
                    .collect(toList());
        }
    }

    @Getter
    static class OrderItemDto{

        private String itemName; // 상품 명
        private int orderPrice; // 주문 가격
        private int count; // 주문 수량

        public OrderItemDto(OrderItem orderItem) {
            itemName = orderItem.getItem().getName();
            orderPrice = orderItem.getOrderPrice();
            count = orderItem.getCount();
        }
    }
```

- #### 엔티티를 DTO로 변환하는 방법이다.

- #### @Data는 반환하거나 사용할 데이터를 편리하게 다룰 수 있게 하여 @Getter, @Setter 등이 포함되어 있다. 사용하지 않으면 서버 오류 500에 no property 에러가 발생한다.

- #### OrderDto 클래스에서 직접 order.getOrderItems() 메소드를 호출하면 orderItems가 엔티티이므로 null 값이 반환된다.

- #### DTO 내부에 엔티티 자체가 있으면 안돼기 때문에 OrderItem에 대한 DTO를 따로 생성하여 저장한다.

- #### 이 방식은 N + 1 문제에 취약하여 많은 양의 쿼리가 발생하는 문제가 있다.



#### V3

```java
    @GetMapping("/api/v3/orders")
    public List<OrderDto> orderV3(){
        List<Order> orders = orderRepository.findAllWithItem();
        List<OrderDto> collect = orders.stream()
                .map(o -> new OrderDto(o))
                .collect(toList());

        return collect;
    }
```

```java
/**
* OrderRepository
*/
	public List<Order> findAllWithItem() {
        // order는 2개지만 orderItem이 4개여서 데이터 뻥튀기가 일어나서 order 4개 인것처럼 join된다
        return em.createQuery(
                "select distinct o from Order o" +
                        " join fetch o.member m" +
                        " join fetch o.delivery d" +
                        " join fetch o.orderItems oi" +
                        " join fetch oi.item i", Order.class
        )		.getResultList();
    }
```

- #### OrdersV3는 페치 조인으로 성능을 최적화 시킨 방식이다.

- #### xToMany 방식으로 데이터 중복이 발생한다. 이러한 문제는 distinct로 어플리케이션 영역에서 방지한다.

- #### 페치 조인을 사용하면 페이징이 불가능하다. 쿼리를 보면 limit, offset이 없으면 warning이 발생한다. 즉, 메모리 영역에서 paging sort를 실행한다는 뜻이다. 메모리 영역에 중복된 데이터를 올려서  paging하므로 paging이 원하는 데이터를 가져오지 못하며 데이터가 많은 경우 OutOfMemory 오류가 발생할 수 있다.

- #### 컬렉션 페치 조인은 1개만 사용 가능하다. 즉, 1 : N : N 이 발생하여 데이터 뻥튀기를 방지하지 못하며 데이터가 부정합하게 조회될 수 있다.



#### V3.1

```java
    @GetMapping("/api/v3.1/orders")
    public List<OrderDto> orderV3_page(
            @RequestParam(value = "offset", defaultValue = "0") int offset,
            @RequestParam(value = "limit", defaultValue = "100") int limit
    ){
        List<Order> orders = orderRepository.findAllWithMemberDelivery(offset, limit);


        List<OrderDto> collect = orders.stream()
                .map(o -> new OrderDto(o))
                .collect(toList());

        return collect;
    }
```

```java
/**
* OrderRepository
*/
	public List<Order> findAllWithMemberDelivery(int offset, int limit) {
        return em.createQuery(
                "select o from Order o" +
                        " join fetch o.member m" +
                        " join fetch o.delivery d", Order.class
        )       .setFirstResult(offset)
                .setMaxResults(limit)
                .getResultList();
    }
```

```yaml
# application.yml
default_batch_fetch_size: 100
```

- #### OrdersV3.1은 xToOne 관계를 모두 페치 조인한 후 컬렉션은 Lazy 로딩으로 조회는 방식이다.

- #### Lazy 로딩 성능 최적화를 위해 hibernate.default_batch_fetch_size, @BatchSize를 적용하여 IN 쿼리로 조회한다.

  - #### Hibernate.default_batch_fetch_size : 글로벌 설정이다.

  - #### @BatchSize : 개별 최적화이다. XToMany 경우 컬렉션 위에 설정하며 xToOne 경우 One 쪽 클래스 위에 설정한다.

- #### @RequestParam은 쿼리 스트링으로 데이터를 받는 것이다.

- #### Offset 값이 0이면 hibernate가 default로 인지하여 쿼리에 추가하지 않는다.

- #### BatchSize를 이용하면 정규화된 상태의 데이터를 조회하는 것으로 데이터 중복이 없다.

- #### N + 1 문제를 보완하여 1 + 1로 쿼리가 최적화되며 조인보다 데이터베이스의 데이터 전송량이 최적화 된다. Order과 OrderItem을 조인하면 Order이 OrderItem 만큼 중복되서 조회 되었었다. 하지만 이 방식은 각각 조회하므로 전송해야할 중복 데이터가 없다.

- #### 이 방식은 페이징이 가능하여 이 방식을 이용하는 것이 xToMany에서 좋다.

- #### Default_batch_fetch_size는 최대 1000으로 설정할 수 있으며 100 ~ 1000으로 설정하는 것이 좋다. 1000으로 설정하는 것이 가장 좋겠지만 데이터베이스에서 한번에 1000개의 데이터를 불러오기 때문에 데이터베이스에 순간 부하를 증가할 수 있다. 하지만 어플리케이션은 100이나 1000이나 모두 결국 전체 데이터를 로딩해야하기 때문에 메모리 사용량은 같다. 즉, 너무 작은 사이즈로 설정하면 시간만 오래걸릴 수 있다. 초기에 100으로 설정한 후 WAS, 데이터베이스에 맞게 사이즈를 크게하는 것이 좋다.



#### V4

```java
    // 리스트의 각 데이터마다 쿼리 한번씩 날린다. N + 1 문제 발생
    @GetMapping("/api/v4/orders")
    public List<OrderQueryDto> orderV4(){
        return orderQueryRepository.findOrderQueryDtos();
    }

```

```java
/**
* OrderQueryRepository
*/
@Repository
@RequiredArgsConstructor
public class OrderQueryRepository {

    private final EntityManager em;

    public List<OrderQueryDto> findOrderQueryDtos() {
        List<OrderQueryDto> result =  findOrders(); // query 1번 -> N개 orderItem 나옴

        result.forEach(o -> {
            List<OrderItemQueryDto> orderItems = findOrderItems(o.getOrderId()); // query N번 // 즉, N + 1 발생
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
                        " join o.delivery d", OrderQueryDto.class
        ).getResultList();
    }
}
```

```java
/**
* OrderQueryDto
*/
@Data
@EqualsAndHashCode(of = "orderId")
public class OrderQueryDto {

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
```

```java
/**
* OrderItemQueryDto
*/
@Data
public class OrderItemQueryDto {

    @JsonIgnore
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
```

- #### OrdersV4는 DTO로 직접 조회하는 방식이다.

- #### 쿼리는 루트일 경우 페치 조인으로 1번, 컬렉션은 N번 실행되어 N + 1 문제가 발생한다. 루트는 최초로 전송하는 쿼리이다.

- #### 현재 컬렉션은 OrderItem을 기준으로 동작하는데 OrderItem 기준에서 Item은 xToOne 관계이므로 바로 조회하는 것이 가능하다.

- #### New로 DTO에 직접 입력할 때 페치 조인 대신 그냥 조인을 사용한다.



#### V5

```java
    // xToOne은 그대로 조인하여 가져오고 리스트는 in 쿼리로 한번에 모든 리스트를 가져오고 map으로 성능을 최적화한다
    @GetMapping("/api/v5/orders")
    public List<OrderQueryDto> orderV5(){
        return orderQueryRepository.findAllByDto_optimization();
    }

```

```java
/**
* OrderQueryRepository
*/
	public List<OrderQueryDto> findAllByDto_optimization() {
        List<OrderQueryDto> result = findOrders();

        List<Long> orderIds = toOrderIds(result);

        Map<Long, List<OrderItemQueryDto>> orderItemMap = findOrderItemMap(orderIds);

        result.forEach(o -> o.setOrderItems(orderItemMap.get(o.getOrderId())));

        return result;


    }

    private Map<Long, List<OrderItemQueryDto>> findOrderItemMap(List<Long> orderIds) {
        List<OrderItemQueryDto> orderItems = em.createQuery(
                        "select new jpabook.jpashop.repository.order.query.OrderItemQueryDto(oi.order.id, i.name, oi.orderPrice, oi.count)" +
                                " from OrderItem oi" +
                                " join oi.item i" +
                                " where oi.order.id in :orderIds", OrderItemQueryDto.class)
                .setParameter("orderIds", orderIds)
                .getResultList();

        Map<Long, List<OrderItemQueryDto>> orderItemMap = orderItems.stream()
                .collect(Collectors.groupingBy(orderItemQueryDto -> orderItemQueryDto.getOrderId()));
        return orderItemMap;
    }

    private List<Long> toOrderIds(List<OrderQueryDto> result) {
        List<Long> orderIds = result.stream()
                .map(o -> o.getOrderId())
                .collect(Collectors.toList());
        return orderIds;
    }
```

- #### OrdersV5는 IN 쿼리를 사용하여 한번에 컬렉션을 가져온다.

- #### OrderItems를 Map으로 변환하여 로직에 활용 시 O(1)로 성능을 향상한다(OrderId기준으로 매핑한다).

- #### 특정 데이터만 가져오므로 이전 fetch join보다 데이터 가져오는 양이 더 적다는 장점이 있다.

- #### OrdersV3.1, OrdersV5를 주로 이용하게 된다.



#### V6

```java
    // 쿼리 한번으로 xToOne, xToMany 모두 해결가능
    // Order, OrderItem, Item을 모두 조인하여 한번에 조회하는 것이다
    // OrderFlatDto로 반환했고 아래 V6는 OrderQueryDto 스펙으로 반환하는 api이다
    // 데이터 뻥튀기가 발생한다
//    @GetMapping("/api/v6/orders")
//    public List<OrderFlatDto> orderV6(){
//        return orderQueryRepository.findAllByDto_flat();
//    }

    // OrderFlatDto를 OrderQueryDto로 바꿔서 response한다
    // OrderQueryDto와 OrderItemQueryDto로 나눈 후 OrderQueryDto로 합친다
    // 객체에 @EqualsAndHashCode()로 Id를 기준으로 하여 데이터 뻥튀기를 방지한다.
    @GetMapping("/api/v6/orders")
    public List<OrderQueryDto> orderV6(){
        List<OrderFlatDto> flats = orderQueryRepository.findAllByDto_flat();

        return flats.stream()
                .collect(groupingBy(o -> new OrderQueryDto(o.getOrderId(),
                                o.getName(), o.getOrderDate(), o.getOrderStatus(), o.getAddress()),
                        mapping(o -> new OrderItemQueryDto(o.getOrderId(),
                                o.getItemName(), o.getOrderPrice(), o.getCount()), toList())
                )).entrySet().stream()
                .map(e -> new OrderQueryDto(e.getKey().getOrderId(), e.getKey().getName(), e.getKey().getOrderDate(), e.getKey().getOrderStatus(),e.getKey().getAddress(), e.getValue()))
                .collect(toList());
    }
```

```java
/**
* OrderQueryDto
*/
	public OrderQueryDto(Long orderId, String name, LocalDateTime orderDate, OrderStatus orderStatus, Address address, List<OrderItemQueryDto> orderItems) {
        this.orderId = orderId;
        this.name = name;
        this.orderDate = orderDate;
        this.orderStatus = orderStatus;
        this.address = address;
        this.orderItems = orderItems;
    }
```

```java
/**
* OrderQueryRepository
*/
	public List<OrderFlatDto> findAllByDto_flat() {

        return em.createQuery(
                "select new jpabook.jpashop.repository.order.query.OrderFlatDto(o.id, m.name, o.orderDate, o.status, d.address, i.name, oi.orderPrice, oi.count)" +
                        " from Order o" +
                        " join o.member m" +
                        " join o.delivery d" +
                        " join o.orderItems oi" +
                        " join oi.item i", OrderFlatDto.class)
                .getResultList();

    }
```

```java
/**
* OrderFlatDto
*/
// OrderQueryV6로 한번에 정보를 가져온다
@Data
public class OrderFlatDto {

    private Long orderId;
    private String name;
    private LocalDateTime orderDate;
    private OrderStatus orderStatus;
    private Address address;

    private String itemName;
    private int orderPrice;
    private int count;

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
}
```

- #### OrdersV6는 플랫 데이터로 최적화하는 방식이다. 즉, 미리 필요한 데이터를 모두 조인하여 가져오는 것이다.

- #### 쿼리가 1번만 발생한다는 장점이 있다.

- #### 플랫 DTO에는 필요한 데이터를 모두 저장한다.

- #### 데이터 중복이 발생하여 OrdersV5보다 느릴 수 있다. 즉, 데이터 양이 많을 때 부하가 커질 수 있다.

- #### 어플리케이션에서 플랫 DTO를 분해 후 orderQueryDto로 만드는 추가 작업이 필요하며 중복 데이터의 영향으로 Order 기준의 페이징이 불가능하다.



### Arrangement



1. #### 엔티티 조회 방식으로 접근한다.

   1. #### 페치 조인으로 쿼리 수를 최적화한다.

   2. #### 컬렉션 최적화한다.

      1. #### 페이징이 필요할 시 hibernate.default_batch_fetch_size, @BatchSize로 최적화한다.

      2. #### 페이징이 필요하지 않다면 페치 조인을 이용한다.

2. #### 엔티티 조회 방식으로 해결되지 않는다면 DTO 직접 조회 방식을 이용한다.

3. #### DTO 직접 조회 방식으로도 해결되지 않는다면 NativeSQL이나 스프링 JdbcTemplate을 이용한다.

- #### 엔티티 조회 방식은 코드를 거의 수정하지 않고 옵션만 약간 변경하여 성능 최적화를 할 수 있지만 DTO 직접 조회 방식은 많은 코드를 변경해야 성능 최적화를 할 수 있다.

- #### 사실 엔티티 조회 방식으로 해결되지 않는다면 traffic이 많은 것으로 DTO 직접 조회보다 캐시를 사용하여 문제를 해결해야 한다.

- #### 캐싱할 때에는 반드시 엔티티에 DTO로 저장해야 한다. 캐시로 인해 영속성이 사라져도 그대로 저장되어 있는 상황이 되면 영속성에 혼란이 오기 때문이다. 캐싱 시에는 redis나 local memory cache 방식을 이용한다.

- #### OrdersV3.1, OrderV5 방식을 이용하는 것이 권장된다.



# OSIV & Performance Optimization



- #### OSIV는 Open Session In View로 스프링에서는 spring.jpa.open-in-view로 되어 있으며 default 값은 true이다.



## OSIV ON



![OSIV ON](https://user-images.githubusercontent.com/79822924/152672177-d478c593-0929-4a7d-973e-1c6db4c57f4e.png)

- #### 어플리케이션 시작 시점에 OSIV default 사용 warn 로그가 발생한다.

- #### JPA는 데이터베이스에 트랜잭션 시작 시점에 연결을 맺고 API가 유저에게 반환되거나 화면이 렌더링 될 때까지 유지한다. 이러한 연결성으로 lazy 로딩이 프록시로 처리되는 것이 가능해진다. Lazy 로딩은 영속성 컨텍스트가 살이있어야 가능하며 영속성 컨텍스트는 기본적으로 데이터베이스와 연결이 되어 있어야 하기 때문이다.

- #### OSIV 전략은 너무 오랜시간 동안 데이터베이스 커넥션 리소스를 사용하기 때문에 실시간 트래픽이 중요한 어플리케이션에서는 커넥션이 모자랄 수 있다. 이러한 상황을 커넥션이 마른다고 한다. 즉, 외부 API가 blocking 되었을 때 thread pool이 모두 찰때까지 커넥션을 사용하여 커넥션이 마를 수 있는 것이다.



## OSIV OFF



![OSIV OFF](https://user-images.githubusercontent.com/79822924/152672182-b215a279-3ade-438d-8bd8-640ef1035316.png)

- #### 트랜잭션을 종료할 때 영속성 컨텍스트를 닫고 데이터베이스 커넥션도 반환한다.

- #### 데이터베이스 커넥션을 낭비하지 않는 장점이 있다.

- #### 모든 Lazy 로딩을 트랜잭션 내부에서 해결해야 하기 때문에 많은 코드를 트랜잭션 안에 넣어야 한다.

- #### View template에서 Lazy 로딩이 불가능하게 된다.

- #### 이 전략에서는 트랜잭션이 끝나기 전에 Lazy 로딩을 강제로 호출하거나 fetch join으로 모두 조회 후 로직을 사용해야 한다.



## OSIV OFF Strategy



- #### Command와 query를 분리하는 것이 복잡을 관리하기 좋은 방법이다. 즉, 조회와 등록, 수정 비즈니스 로직을 분리하는 것이다.

- #### 핵심 비즈니스 로직은 잘 변하지 않지만 화면 비즈니스 로직은 많이 변해서  lifecycle이 다르므로 command와 query를 분리해야 유지보수하기 좋다. 예를 들어, OrderService로 핵심 비즈니스 로직을 해결하며 OrderQueryService로 화면이나 API에 맞춘 조회 서비스를 해결한다(OrderQueryService는 transactional 전용 service 계층을 만드는 것이다).

- #### 보통 서비스 계층에서 트랜잭션을 유지하면서 lazy 로딩을 사용해야 한다.

- #### 보통 고객 서비스의 실시간 API는 OSIV OFF 하며 ADMIN처럼 커넥션을 많이 사용하지 않는 곳에서는 OSIV를 켠다. 한 프로젝트에서 ADMIN과 API 시스템은 배포를 따로 하게 된다.



# Keyboard Shortcut

- #### ctrl + alt + p : 변수를 파라미터로 즉시 생성하기 가능하다.

- #### shift + alt + insert : 여러줄을 한번에 수정하는 것이 가능하다.





Image 출처 : [김영한 JPA_활용2](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81%EB%B6%80%ED%8A%B8-JPA-API%EA%B0%9C%EB%B0%9C-%EC%84%B1%EB%8A%A5%EC%B5%9C%EC%A0%81%ED%99%94)
