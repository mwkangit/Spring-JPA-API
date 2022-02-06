# Spring-JPA-API


## Description

본 프로젝트는 [Spring-JPA-Shop](https://github.com/mwkangit/Spring-JPA-Shop)의 결과물을 API로 데이터를 전송을 구현한 것이다. 각 로직에 대해 네트워크, 쿼리의 성능과  N + 1 문제를 refactoring 과정을 거치면서 버전을 나눠서 구현하였다. 로직으로는 회원, 주문에 대한 등록, 조회를 구현하였다. 응답 데이터 형식은 JSON을 이용하였다.



------



## Environment

<img alt="framework" src ="https://img.shields.io/badge/Framework-SpringBoot-green"/><img alt="framework" src ="https://img.shields.io/badge/Language-java-b07219"/> 

Framework: `Spring Boot` 2.5.4

Project: `Gradle`

Packaging: `Jar`

IDE: `Intellij`

ORM: `JPA`(Hibernate, Spring Data JPA)

DBMS: `H2-Database`

Template Engine: `Thymeleaf`

Dependencies: `Spring Web`, `Spring Data JPA`, `Lombok`, `Validation`



------



## Installation



![Linux](https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black) 

```
./gradlew build
cd build/lib
java -jar hello-spring-0.0.1-SNAPSHOT.jar
```



![Windows](https://img.shields.io/badge/Windows-0078D6?style=for-the-badge&logo=windows&logoColor=white) 

```
gradlew build
cd build/lib
java -jar hello-spring-0.0.1-SNAPSHOT.jar
```



------



## Core Feature

- Entity 조회 후 컬렉션 Lazy 로딩

```java
// OrderApiController

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

xToOne 관계를 모두 Fetch Join 후 컬렉션은 Lazy 로딩으로 조회하는 것이다. Batch_Size를 조절하여 쿼리 발생 수를 감소할 수 있어서 실행 속도가 빠르다. 또한, Fetch Join한 Entity를 기준으로 페이징 하는 것이 가능하다.



- DTO 조회 후 IN 쿼리 컬렉션 로딩

```java
// OrderApiController

	@GetMapping("/api/v5/orders")
    public List<OrderQueryDto> orderV5(){
        return orderQueryRepository.findAllByDto_optimization();
    }

// OrderQueryRepository

	private List<OrderQueryDto> findOrders() {
        return em.createQuery(
                "select new jpabook.jpashop.repository.order.query.OrderQueryDto(o.id, m.name, o.orderDate, o.status, d.address)" +
                        " from Order o" +
                        " join o.member m" +
                        " join o.delivery d", OrderQueryDto.class
        ).getResultList();
    }

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

DTO로 바로 조회한 후 IN 쿼리로 컬렉션을 로딩하는 방법이다. 엔티티를 모두 조회하는 것이 아닌 필요한 데이터만 조회하기 때문에 Join을 사용했으며 이후 컬렉션은 IN 쿼리를 사용하여 N + 1 문제를 방지한다. 필요한 데이터만 조회로 엔티티 직접 조회보다 데이터의 낭비가 적을 수 있다.



-----



## Demonstration Video

- Entity 조회 후 컬렉션 Lazy 로딩

![Spring-JPA-API-Entity](https://user-images.githubusercontent.com/79822924/152672153-b9bb7d35-9bae-45e0-927c-d60eca4be31a.gif)



- DTO 조회 후 IN 쿼리 컬렉션 로딩

![Spring-JPA-API-DTO](https://user-images.githubusercontent.com/79822924/152672160-9c3a85a2-f76a-492a-922e-3d1cea660b16.gif)





------



## More Explanation

[Spring-JPA-API-Note.md](https://github.com/mwkangit/Spring-JPA-API/blob/master/Spring-JPA-API-Note.md)
