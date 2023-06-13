## 4.1 JPA 를 이용한 리포지토리 구현

리포지토리 인터페이스는 애그리거트 루트를 기준으로 만들어진다. 리포지토리 인터페이스는 도메인 영역에 속하는 고수준 모듈, 

리포지토리 구현체는 인프라 영역에 속하는 저수준 모듈로 설계하자! 

Tip
> 삭제 요구사항이 있더라도 실제 삭제 기능을 만들지 않고 화면에서 보여줄지 여부를 결정하는 방식으로 만들자. 
> 
> 삭제 후 일정 기간 데이터를 보관해야 할 때도 있기 때문이다.


## 4.2 스프링 데이터 JPA 를 통한 리포지터리 구현 

## 4.3 매핑 구현 

애그리거트와 JPA 매핑을 위한 기본 규칙

* 애그리거트 루트는 엔티티이므로 @Entity 매핑

한 테이블에 엔티티와 밸류 데이터가 같이 있다면

* 밸류는 @Embeddable (값 타입을 정의하는 곳) 로 매핑

* 밸류 타입 프로퍼티는 @Embedded (값 타입을 사용하는 곳) 로 매핑

JPA 에서 @Entity, @Embeddable 로 클래스를 매핑하려면 **기본 생성자**를 제공해야 한다. DB 에서 데이터를 읽어와 매핑된 객체를 

생성할 때 기본 생성자를 사용해서 객체를 생성하기 때문 값 타입은 불변 객체로 만들어야 한다, 다른 곳에서 사용하지 못하도록 프로텍티드 생성자를 만들어주자.


Tip
> JPA 에서 식별자 타입을 사용하는 경우 Serializable 타입이어야 하므로 식별자로 사용할 밸류 타입은 
> 
> Serializable 인터페이스를 상속 받아야 한다.
>
>  밸류 타입으로 식별자를 구현하면 식별자에 기능을 추가할 수 있는 장점이 있다. OrderNo 참고
>
>  JPA 는 엔티티를 비교할 목적으로 equals() , hashcode() 를 사용하기 때문에 
>  
>  식별자로 사용할 밸류 타입은 이 두 메서드를 알맞게 구현해야 한다.


Tip 

> 엔티티 프로퍼티 접근법@Access(AccessType.PROPERTY) 을 사용하게 되면 get/set 메서드를 열어야 됨 get/set 메서드를 추가하면 
>
> 도메인의 의도가 사라지고, 내부의 데이터를 외부에서 변경할 수 있기 때문에 캡슐화를 깨는 원인이 된다.
>
> set 을 사용하는 경우는 private 으로 클래스 내부 변경만을 목적으로 사용한다, 그리고 set 메서드 대신 의도가 잘 드러나는 메서드 이름을 사용해야 한다.
>
> ex) setShippingInfo() 보다 changeShippingInfo() 를 사용해서 배송지를 변경한다는 의미를 명확하게 표현


애그리거트에서 루트 엔티티는 대부분 하나, 루트 엔티티는 PK 값을 가지고 있으며 애그리거트는 루트를 통해서 간접적으로 애그리거트
 
내의 다른 엔티티나 밸류 객체에 접근할 수 있다. 2.4

자신만의 독자적인 사이클을 갖는 경우 다른 애그리거트일 수 있다. 상품과 상품 리뷰는 서로 독자적인 관계로 함께 생성되지 않고 함께 변경되지 않는다.

리뷰의 변경이 상품에 영향을 주지 않고 반대로 상품의 변경이 리뷰에 영향을 주지 않기 떄문이다. 

애그리거트에 속한 객체가 밸류인지 엔티티인지 구분하는 방법은 고유 식별자를 갖는지 확인하는 것이지만! 식별자를 갖고 있다고 해서 엔티티가 되는 것은 아님.

@SecondaryTable 을 사용해서 엔티티의 내용(content)을 담을 수 있는 밸류로 만들 수 있음 - ex) Article, ArticleContent 

밸류를 컬렉션으로 매핑할 때는 2 가지 방법이 있다. 

* @ElementCollection 사용 - ex) Order, OrderLine 

* @Entity / @OneToMany , @ManyToOne 관계 만들기. - ex) Image

각각 상황에 맞게 필요한 것을 사용하면 됨. 단순 애그리거트 관계에서는 루트 엔티티를 중심으로 @ElementCollection 을 만들면 되고

값 타입간 상속 계층 구조가 필요하다면 @Entity 를 사용하면 된다. 예제 클래스 참고! 


## 4.4 애그리거트 로딩 전략

애그리거트에 속한 객체가 모두 모여야 완전한 하나가 된다. 실무에서는 지연로딩으로 깔고가자 165P

## 4.5 애그리거트의 영속성 전파

애그리거트가 완전한 상태여야 한다는 것은 루트를 조회할 때 뿐만 아니라 저장하고 삭제할 때도 하나로 처리해야 한다는 의미다.

참고로 값 타입 컬렉션은 Cascade + Orphan Remove 를 필수적으로 가진다. @Entity 를 사용하면 따로 설정해주면 됨.

https://github.com/eternalrecurrenceofthesame/DDD/tree/main/ch03 참고

## 4.6 식별자 생성 기능

* 사용자가 직접 생성
* 도메인 로직으로 생성
* DB 를 이용한 생성

식별자 생성 규칙은 도메인 규칙이므로 도메인 영역에 식별자 생성 기능을 위치시키고 응용 서비스에서 도메인 서비스를 이용해서 식별자를 

구하고 엔티티를 생성할 수 있다.

```
//도메인 서비스
public class PrductIdService{
  public ProductId nextId(){
  // 식별자 생성
  }

// 응용 서비스
public class CreateProductService{
 @Autowired private ProductIdService idService;
 @Autowired private ProductRepository productRepository;
}
```

리포지토리에서 식별자를 따로 생성하거나 
``` 
OrderRepository 

default OrderNo nextOrderNo(){
        int randomNo = ThreadLocalRandom.current().nextInt(900000) + 100000;
        String number = String.format("%tY%<tm%<td%<tH-%d", new Date(), randomNo);
        return new OrderNo(number);
    }

```
DB 자동 키 생성 방식으로 식별자를 생성하고 조회해 올수도 있음. 아이덴티티 전략.

## 4.7 도메인 구현과 DIP

저수준의 기술이 변경되어도 고수준의 로직이 영향을 받지 않기 위해 DIP 를 적용하지만 지금 도메인 모델에는 JPA 관련 애노테이션들이 많음.

JPA 애너테이션을 도메인 모델에 사용하면서 기술에 따른 구현 제약이 낮다면 합리적인 선택이다. 

그 외 필요한 것들은 예제 코드를 확인한다. 
