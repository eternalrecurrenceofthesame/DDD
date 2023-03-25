## 3.1 애그리거트

애그리거트가 한 개의 엔티티 객체만 갖는 경우가 많다. 두 개이상의 엔티티로 구성되는 경우는 드물다. 

* 애그리거트는 관련된 모델을 하나로 모았기 때문에 한 애그리거트에 속한 객체는 유사하거나 동일한 라이프 사이클을 갖는다.

* 애그리거트의 경계를 설정할 때 기본이 되는 것은 도메인 규칙과 요구사항이다. 도메인 규칙에 따라 함께 생성되는 구성 요소는 한 애그리거트에 속할 가능성이 높다.

* A가 B를 갖는다로 설계할 수 있는 요구사항이 항상 애그리거트로 묶이는 것은 아니다. ex) 상품과 리뷰는 함께 생성되고 변경되지 않는다. 상품의 변경 주체는 담당자, 리뷰의 변경 주체는 고객이다.



## 3.2 애그리거트 루트

애그리거트에 속한 모든 객체의 일관성을 유지시켜주는 루트 엔티티, 애그리거트의 루트 엔티티가 애그리거트의 대표 엔티티가 된다. 애그리거트에 속한 객체는

애그리거트 루트 엔티티에 직접 또는 간접적으로 속하게 된다.

애그리거트 루트는 도메인 규칙을 구현한 기능을 제공한다 이때 애그리거트 외부에서 애그리거트에 속한 객체를 직접 변경하면 안 된다. 루트 엔티티에서 

도메인 규칙을 적용하면서 일관성을 책임지기 때문이다.

```
public class Order{

public void changeShippingInfo(ShippingInfo newShippingInfo){
  verifyNotYetShipped();   // 배송 여부 체크
  setShippingInfo(newShippingInfo);
}

}
```

애그리거트 루트를 통해서만 도메인 로직을 구현하게 하려면 도메인 모델에 대해 두 가지를 습관적으로 적용해야 한다.

* 단순히 필드를 변경하는 set 메서드를 공개 범위로 만들지 않는다. (private 으로)

* 밸류 타입은 불변으로 구현하고 변경시 새로운 인스턴스를 넘겨준다. 
    
    * 밸류 타입을 불변으로 만들 수 없는 경우에는 변경 기능을 패키지나 protected 범위로 한정해서 외부에서 실행할 수 없도록 제한해주자.

애그리거트의 외부에서 내부의 상태를 함부로 변경하지 못하게 함으로써 일관성이 깨질 가능성이 줄어든다.


* 하나의 트랜잭션에서는 한 개의 애그리거트만 수정해야 한다. (성능)
    
```
잘못된 예

public class Order{

  private Orderer orderer;
  
  public void shipTo(ShippingInfo newShippingInfo,
                     boolean useNewShippingAddrAsMemberAddr){
   verifyNotYetShipped();
   setShippingInfo(newShippingInfo);
   
   if(useNewShipingAddrAsMemberAddr){
   orderer.getMember().change() <- order 애그리거트에서 Member 애그리거트의 값을 바꾸지 말자!!
   }
                     
   }

}

```

두 개 이상의 애그리거트 변경시 응용 서비스에서 각 애그리거트의 상태를 변경한다.
```
public class ChangeOrderService{
@Transactianl
public void chagneShippingInfo(OrderId id, ShippingInfo newShippingInfo, boolean useNewShippingAddrAsMemberAddr){

  Order order = orderRepository.findById(id);
  if(order == null) throw new OrderNotFoundException();
  
  order.shipTo(newShippingInfo);
  if(useNewShippingAsMemberAddr){
  Member member = findMember(order.getOrderer());
  member.changeAddress(newShippingInfo.getAddress());
}

}
```
트랜잭션 전파 합류를 통해서 하나의 트랜잭션으로 묶고 하나의 물리 트랜잭션으로 관리할 수 있다.

보통 하나의 트랜잭션에서 한 개의 애그리거트 수정을 권장하지만, 다음의 경우 두 개 이상의 애그리거트 변경을 고려할 수 있다.
``` 
팀 표준

기술 제약

UI 구현의 편리 111P
```
    
## 3.3 리포지터리와 애그리거트

애그리거트는 개념상 완전한 한 개의 도메인 모델을 표현하므로 객체의 영속성을 처리하는 리포지터리는 애그리거트 단위로 존재한다.

애그리거트는 개념적으로 하나이므로 리포지토리는 애그리거트 전체를 저장소에 영속화 해야 한다.

Order 애그리거트와 관련된 테이블이 3 개라면 Order 애그리거트를 저장할 때 애그리거트 루트와 매핑되는 테이블 뿐만 아니라 

애그리거트에 속한 모든 구성요소에 매핑된 테이블에 데이터를 저장해야 한다.

반대로 조회할 때도 애그리거트에 속한 모든 구성요소에 매핑된 테이블 데이터를 가지고 올 수 있어야 한다.



## 3.4 ID 를 이용한 애그리거트 참조

애그리거트가 관리하는 범위는 자기 자신으로 한정해야 한다. 다른 애그리거트를 참조해서 값을 얻어야 하는 경우 필드로 참조 값을 받아서 객체를 

탐색할 수도 있지만 단점도 존재한다.

* 외부 애그리거트에 쉽게 접근할 수 있기 때문에 값을 함부로 변경할 위험이 있다. (한 애그리거트가 관리하는 범위는 자기 자신으로 한정해야 함! 3.2)

* JPA 사용시 로딩 성능 관리에 대한 고민을 해야한다. 116p

* 마이크로 서비스를 이용해서 서비스를 분리해 다양한 데이터베이스를 사용할 경우 애그리거트 루트를 참조하기 위한 JPA 와 같은 단일 기술을 사용할 수 없게 된다.

이런 단점들을 해결할 수 있는 방법은 객체를 직접 참조하는 것이 아닌 ID 를 이용해서 참조하는 것이다. 

다른 애그리거트를 참조할 때 애그리거트 루트의 ID 값을 참조하는 것.

```
public  class Orderer {
  private MemberId memberId;  // 멤버 애그리거트의 루트 ID
  private String name; 
}
```

ID 참조를 사용해서 애그리거트 간 물리적 연결을 제거하고 애그리거트의 경계를 명확하게 할 수 있으며 애그리거트 간 의존을 제거해서 응집도를 높여준다.

JPA 를 사용한다고 하면 직접 참조값을 사용하지 않으므로 로딩에 대한 고민을 할 필요가 없다. 참조하는 애그리거트가 필요하면 

응용 서비스에서 ID 값을 이용해서 로딩하면 된다! 

```
public class ChangeOrderService{
  
  @Transactional
  public void changeShippingInfo(OrderId id, ShippingInfo newShippingInfo, boolean useNewShippingAddrAsMemberAddr){
  
  Order order = orderRepository.findById(id);
  if(order == null) throw new OrderNotFoundException();
  
  if(useNewShipingAsMemberAddr){
  
  Member member = memberRepository.findById(order.getOrderer().getMemberId()); <- 다른 애그리거트 루트의 ID 값 사용! 
  
  member.changeAddress(newShippingInfo.getAddress();

}
}

}
```
응용 서비스에서 필요한 애그리거트(Member)를 로딩하므로 애그리거트 수준에서 지연 로딩을 하는 것과 동일한 결과를 만든다.

ID 를 이용해서 여러 애그리거트를 조회하면 조회성능에서 문제가 생길 수 있다.

주문 목록을 보여주려면 상품 애그리거트와 회원 애그리거트를 함께 읽어야 하는데 한번에 값을 가져오는 게 아니라

각각 쿼리를 날려서 N+1 의문제를 발생시킴 먼저 회원을 조회했는데 회원이 주문한 상품이 10 개면 10 번의 쿼리를 추가로 실행함. 119p

ID 쿼리 참조 방식을 사용하면서 N+1 문제를 방지하려면 조회를 위한 별도의 DAO 를 만들고 조회 전용 쿼리로 한번에 값을 가져오면 된다.

```
@Repository
public class JpaOrderViewDao implements OrderViewDao{

@PersistenceContext
private EntityManger em;

@Override
public List<OrderView> selectByOrderer(String ordererId){

String selectQuery = 
    "select new com.myshop.order.application.dto.OrderView(o,m,p) " + 
     "from Order o join o.orderLines ol, Member m, Product p " + 
     "where o.orderer.memberId.id = :ordererId " +
     ... 
}

}
```

애그리거트마다 서로 다른 저장소를 사용하면 한번의 쿼리로 관련 애그리거트를 조회할 수 없다. 이때는 조회 성능을 높이기 위해 캐시를 적용하거나

조회 전용 저장소를 따로 구성해야 한다 (CQRS 패턴 사용)


## 3.5 애그리거트 간 집합 연관

1-N 연관관계인 카테고리와 상품에서 특정 카테고리에 속한 상품을 조회할 때 페치조인을 사용하면 쿼리한번으로 값을 가지고 올 수 있다.

하지만 컬렉션과 페치조인을 사용할 경우 페이징을 하지 못한다. 모든 값을 다 메모리에 올린 다음에 페이징을 시도하기 때문이다 

**(@BatchSize 로 최적화를 하는 방법이 있지만 교재에서 설명하는 범주가 아니므로 위 상황을 가정하고 설명한다.)**

카테고리에 속한 상품을 구할 필요가 있다면 상품 입장에서 자신이 속한 카테고리를 N-1 연관으로 구하면 된다.

```
public class Product{
...
private CategoryId categoryId; // 아이디 값만 받아옴
...
}
```

```
product 만든 후 리팩토링
```

#### + M-N 다대 다 연관 관계 해결하기

다대다는 양쪽 애그리거트에 컬렉션으로 연관 관계를 만든다. 

상품이 여러 카테고리에 속할 수도 있고 카테고리가 여러 상품을 가질 수도 있다.

앞서 만든 것들 처럼 값 타입으로 아이디를 사용하는 경우 다대 다 관계를 만들 때는 요구사항을 고려해서 @CollectionTable 을 만들면 된다.

보통 특정 카테고리에 속한 상품 목록을 보여줄 때 상품 목록 화면에서 각 상품이 속한 모든 카테고리 정보를 표시하지 않는다.

제품이 속한 모든 카테고리가 필요한 화면은 상품 상세 화면이 된다.

이러한 요구 사항을 고려할 때 카테고리에서 상품으로의 집합 연관은 필요하지 않다. 상품에서 상품상세를 보여줄 때 카테고리가 필요하므로

상품에서 카테고리를 알고 있으면 된다는 의미.



## 3.6 애그리거트를 팩토리로 사용하기

Store 에서 Product 를 등록하지 못하도록 차단하는 로직을 구현할 때 Product 를 등록할 수 있는지 없는지 판단하고 애그리거트를 생성하는 로직은 도메인의 기능이다.

```
<잘못된 예시>

public class RegisterProductService{

public ProductId registerNewProduct(NewProductRequest req){

Store store = storeRepository.findById(req.getStoredId());  // Store 
checkNull(store);

if(store.isBlocked()){                                 // Store 에서 등록 여부 판단
 throw new StoreBlockedException();
}

ProductId id = productRepository.nextId();
Product product = new Product(id, store.getId(), ...); // 애그리거트 생성 

productRepository.save(product);

return id;
}

}
```

Product 처럼 다른 애그리거트 Store 의 식별자를 이용해서 생성해야 하는 경우 팩토리 메서드를 고려하자 , Store 의 상태에 따라서 Product 를 생성할 수 있다

Product 를 생성할 떄 필요한 데이터의 일부를 직접 제공하면서 동시에 중요한 도메인 로직도 함께 구현할 수 있다.


```
public class Store{

public Product createProduct(ProductId newProductId, ...){  // 애그리거트 생성 팩터리
  if(isBlocked()) throw new StoreBlockedException();
  return new Product(newProductId, getId(), ...);
}
}


public class RegisterProductService{

public ProductId registerNewProduct(NewProductRequest req){

Store store = storeRepository.findById(req.getStoreId());
checkNull(store);

ProductId id = productRepository.nextId();
Product product = store.createProduct(id, ...);

productRepository.save(product);
return id;
}

}

```
위 예시처럼 애그리거트 생성 팩터리를 만들고 응용 서비스에서 팩터리 메서드를 호출해서 애그리거트를 생성하도록 해주자.

Store 애그리거트가 Product 애그리거트를 생성할 때 많은 정보를 알아야 한다면, Store 애그리거트에서 Product 애그리거트를 직접 생성하지 않고

다른 팩토리에 위임하는 방법도 있다. 다른 팩토리에 위임하더라도 차단 상태의 상점은 상품을 만들 수 없다는 도메인 로직은 Store 에 계속 위치하게 된다.

```
다른 팩토리에 위임

public class Store{

public Product createProduct(ProductId newProductId, ...){  // 애그리거트 생성 팩터리
  if(isBlocked()) throw new StoreBlockedException();
  return ProductFactory.create(newProductId, getId(), pi);
}
}
```







