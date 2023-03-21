## 5.1 시작에 앞서

CQRS 란? 명령(Command) 모델과 조회(Query) 모델을 분리하는 패턴!

ex) 회원 가입, 암호 변경, 주문 취소 처럼 데이터를 변경 시 명령 모델을, 주문 목록 주문 상세 같이 데이터를 보여주는 기능은 조회 모델을!

도메인 모델은 보통 명령 모델, 정렬 페이징 검색 조건 지정과 같은 기능은 조회 기능에 사용

조회 기능을 사용할 때 스펙 예제모델을 활용해보자.

## 5.2 검색을 위한 스펙

검색 조건을 다양하게 가져가야 할 때 스펙을 사용한다.

## 5.3 스프링 데이터 JPA 를 이용한 스펙 구현



## 5.4 리포지터리/DAO 에서 스펙 사용하기

```
1. Specification<OrderSummary> spec = new OrderSummarySpecs.ordererId("orderId");
 - 스펙 객체를 통해서 스펙을 만든다

2. List<OrderSummary> results = orderSummaryDao.findAll(spec);
 - 스펙 조건으로 데이터 JPA 를 호출해서 검색!

3. 스펙을 만들 때 사용한 정적 메타모델 OrderSummary_ 를 사용

4. 정적 메타모델이 OrderSummary 에 매핑을 하고 , DTO 내에 있는 서브쿼리로 사용해서 값을 가져오고

  보여주고 싶은 값을 DTO 로 보여주게 된다.
```

```
findOrderView 

OrderSummaryDao 에서 view 전용으로 만든 DTO 로 JPQL 을 사용해서 값을 가져오는 방법도 있다.

```


## 5.9 동적 인스턴스 생성

OrderSummaryDao 참고

## 5.10 하이버네이트 @Subselect 사용

Subselect 는 쿼리 결과를 **@Entity 로 매핑**할 수 있는 기능이다. 이때 여러가지 애노테이션을 함께 사용한다.

* @Subselect - sql 뷰를 사용하는 것과 같은 용도, 뷰를 수정할 수 없듯이 이녀석도 수정할 수 없음 

* @Immutable - 실수로 @Subselect 뷰를 수정하면 변경감지가 일어나는데 매핑할 테이블이 없으니 오류가 발생함 

    이 애노테이션을 사용하면 엔티티의 매핑 필드/프로퍼티가 변경되어도 DB 에반영되지 않고 무시된다.
    
* @Synchronzie - 데이터의 정합성을 맞추기 위해 사용한다. 테이블에서 변경한 값이 아직 반영되지 않았을 때 

    뷰 조회를 시도하면 값이 반영되어 있지 않음, 뷰 조회 로딩전 지정한 엔티티의 값들을 먼저 flush 로 반영하고
    
    값을 가져오는 기능.
    
Subselect 를 사용해도 일반 @Entity 와 같기 때문에 EntityManager , JPQL, Criteria 를 사용해서 조회 할 수 있다는 장점이 있다.

Subselect 는 Subselect 의 값으로 지정한 쿼리를 from 절의 서브 쿼리로 사용한다.

OrderSummary 참고