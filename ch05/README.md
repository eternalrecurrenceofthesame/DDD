## 5.1 시작에 앞서

CQRS 란? 명령(Command) 모델과 조회(Query) 모델을 분리하는 패턴!

회원 가입, 암호 변경, 주문 취소와 같이 데이터를 변경하는 것은 command 패키지에서 명령 모델을 구현하고 

주문 목록 , 주문 상세 같이 데이터를 조회하는 기능은 query 패키지에서 조회 모델로 구현한다. 

명령(Command) 모델에서는 도메인을 관리하는 리포지토리를 사용하고 조회(Query) 모델에서는 DAO 라는 이름의 

저장소를 사용한다. (DAO 란 데이터 접근을 의미함)

## 5.2 검색을 위한 스펙

검색 조건을 다양하게 가져가야 할 때 스펙을 사용한다.

## 5.3 스프링 데이터 JPA 를 이용한 스펙 구현

OrderSummarySpecs 참고 검색 조건을 구현했다. 

#### + JPA 정적 메타 모델이란?

@StaticMetamodel 은 정적 메타모델로 지정하는 애노테이션 역할을 한다 정적 메타모델은 클래스 뒤에 _ 를 붙여준다

정적 메타모델을 사용하면 스펙을 조건을 지정할 때 문자열이 아닌 코드를 사용하기 때문에 컴파일시 오류를 방지할 수 있다.

http://egloos.zum.com/kwon37xi/v/4452159 참고
```
* OrderSummarySpecs 참고 

public static Specification<OrderSummary> orderDateBetween(LocalDateTime from, LocalDateTime to){
        return (Root<OrderSummary> root, CriteriaQuery<?> query, CriteriaBuilder cb) ->
                cb.between(root.get(OrderSummary_.orderDate), from, to);
    }
```
## 5.4 리포지터리/DAO 에서 스펙 사용하기 
```
* OrderSummaryDaoTest 참고 

1. Specification<OrderSummary> spec = new OrderSummarySpecs.ordererId("orderId");
 - 스펙 객체를 통해서 스펙(검색 조건)을 만든다

2. List<OrderSummary> results = orderSummaryDao.findAll(spec);
 - 스펙 조건으로 데이터 리포지토리 (Dao) 를 호출해서 검색!

3. 리포지토리를 호출하면 OrderSummary DTO 에 정의한 @Subselect 의 서브 쿼리를 사용해서 데이터베이스
 테이블을 조회할 수 있다.
```
```
findOrderView 

OrderSummaryDao 에서 view 전용으로 만든 DTO 로 JPQL 을 사용해서 값을 가져오는 방법도 있다.
```
## 5.5 스펙 조합 

스프링 데이터 JPA 스펙 인터페이스는 스펙을 조합할 수 있는 여러가지 제공한다

```
* and, or 을 사용한 스펙 조건 조합 OrderSummaryDaoTest 참고

Specification<OrderSummary> spec1 = OrderSummarySpecs.ordererId("user1");
Specification<OrderSummary> spec2 = OrderSummarySpecs.orderDateBetween(
          LocalDateTime.of(2022,1,1,0,0,0),
          LocalDateTime.of(2022,1,2,0,0,0));
          
Specification<OrderSummary> spec3 = spe1.and(spec2); // 스펙 2 개를 조합해서 3 을만들었다.

Specification<OrderSummary> spec = OrderSummarySpecs.ordererId("user1")
                      .and(OrderSummarySpecs.orderDateBetween(from, to)); // 간소화 버전
```
```
* not 을 사용한 필터링

Specification<OrderSummary> spec = Specification
                                   .not(OrderSummarySpecs.ordererId("user1"));
```
```
* null 여부를 판단해서 NPE 방지, where 로 간소화

Specification<OrderSummary> nullableSpec = createNullableSpec(); // null 일 가능성 있는 조건
Specification<OrderSummary> otherSpec = createOtherSpec();

Specification<OrderSummary> spec = 
          nullableSpec == null ? otherSpec : nullableSpec.and(otherSpec);


where 메서드를 이용한 null 판단
Specification<OrderSummary> spec =
            Specification.where(createNullableSpect()).and(createOtherSpec());
```

## 5.6 정렬 지정하기
```
Sort sort1 = Sort.by("number").ascending();
Sort sort2 = Sort.by("orderDate").descending();

Sort sort = sort1.and(sort2);

Sort osrt = Sort.by("number").ascending().and(Sort.by("orderDate").descending()); // 간소화

List<OrderSummary> results = orderSummaryDao.findByOrderId("use1", sort);
```

## 5.7 페이징 처리하기
```
Pageable pageable = PageRequest.of(2,3);

Page<MemberData> page = memberDataDao.findByBlocked(false, pageReq);

page.getContent(); // 조회 결과 목록
page.getTotalElements(); // 조건에 해당하는 전체 개수
page.getTotalPages(); // 전체 페이지 번호
page.getNumber(); // 현재 페이지 번호
page.getNumberOfElements(); // 조회 결과 개수
page.getSize(); // 페이지 크기 

```
```
Page<MemberData> findByBlocked(boolean blocked, Pageable pageable); 
List<MemberData> findByNameLike(String name, Pageable pageable);

List 로 값을 받으면 count 쿼리를 실행하지 않는다.

List<MemberData> findAll(Specification<MemberData> spec, Pageable pageable);

스펙을 사용하면 List 로 값을 받아도 Count 쿼리를 실행한다.
```
```
처음부터 N 개의 데이터가 필요한 경우

List<MemberData> findFirst3ByNameLikeOrderByName(String name); // 프로 퍼티 기준 3 개 조회

MemberData findFirstByBlockedOrderById(boolean blocked); // 개수를 지정하지 않으면 한 개 결과만 리턴

First 대신 Top 을 사용해도 된다.
```
## 5.8 스펙 조합을 위한 스펙 빌더 클래스

스펙 빌더를 만들고 검색 요청 값을 넘기면 스펙을 조합할 수 있다.  SpecBuilder, SearchRequest 참고. 

```
* MemberDataDaoTest 참고 

SearchRequest sr = new SearchRequest();

sr.set("name");
sr.setOnlyNotBlocked(true);

Specification<MemberData> spec = SpecBuilder.builder(MemberData.class)
   . ifTrue(searchRequest.isOnlyNotBlocked(), 
           () -> MemberDataSpecs.nonBlocked())
   .ifHasText(searchRequest.getName(),
             name -> MemberDataSpecs.nameLike(searchRequest.getName())
    .toSpec();         

List<MemberData> result = memberDataDao.findAll(spec, PageRequest.of(0,5)); 
```

## 5.9 동적 인스턴스 생성

OrderSummaryDao 참고

## 5.10 하이버네이트 @Subselect 사용

데이터를 조회할 때 @Table("view_entitiy") 을 사용해서 뷰 전용 가상 테이블을 엔티티와 매핑시켜 사용할 수 

있지만

@Subselect 쿼리를 사용하면 데이터베이스에 테이블을 생성하지 않고도 뷰의 역할을 할 수 있다. 이 애노테이션에 

쿼리를 지정하면 지정된 쿼리를 데이터베이스 테이블 조회시 from 절의 서브 쿼리에 포함시켜서 데이터를 조회하는

기능을 제공한다. [참고](https://velog.io/@sierra9707/TIP-View-%EB%AA%A9%EC%A0%81%EC%9D%98-Entity-%EA%B0%9D%EC%B2%B4%EB%A5%BC-%EB%A7%8C%EB%93%9C%EB%8A%94-%EB%B2%95) 
``` 
* @Subselect 쿼리 예시 197 p

select osm.number as number1_0, ...
from(Subquery)
```
```
* @Subselect - sql 뷰를 사용하는 것과 같은 용도, 뷰를 수정할 수 없듯이 이녀석도 수정할 수 없다. 

* @Immutable - 실수로 @Subselect 뷰를 수정하면 변경감지가 일어나는데 매핑할 테이블이 없으니 오류가 발생한다. 
이 애노테이션을 사용하면 엔티티의 매핑 필드/프로퍼티가 변경되어도 DB 에반영되지 않고 무시된다.
    
* @Synchronzie - 데이터의 정합성을 맞추기 위해 사용한다. 테이블에서 변경한 값이 아직 반영되지 않았을 때 뷰 조회를 시도하면 
값이 반영되어 있지 않음, 뷰 조회 로딩전 지정한 엔티티의 값들을 먼저 flush 로 반영하고 값을 가져오는 기능.
```    
Subselect 를 사용해도 일반 @Entity 와 같기 때문에 EntityManager , JPQL, Criteria 를 사용해서 조회 할 수 있다는 장점이 있다.

OrderSummary 참고
