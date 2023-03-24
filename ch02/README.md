## 2.1 네 개의 영역

표현 , 응용 , 도메인 , 인프라 스트럭처는 아키텍처 설계에서 사용하는 전형적인 네 가지 유형이다.

* 응용 서비스는 로직을 직접 수행하지 않고 도메인 모델에 로직 수행을 위임한다. 서비스 계층은 단순히 비즈니스 로직의 실행 순서 설정, 권한 위임의 역할을함.

* 인프라 스트럭처는 구현 기술에 대한 것을 다룬다. RDBMS 연동 처리, 메시징 큐에 메시시지 전송 수신, 몽고DB 레디스와 데이터 연동 처리 

## 2.2 계층 구조 아키텍처

계층 구조는 상위 계층에서 하위 계층으로의 의존만 존재하고 하위 계층은 상위 계층에 의존하지 않는다.

표현 -> 응용 -> 도메인 -> 인프라 스트럭처

응용 계층은 바로 아래 계층인 도메인 계층에 의존하지만 외부 시스템과의 연동을 위해 더 아래 계층인 인프라스트럭처에 의존하기도 한다.

(서비스가 리포지토리를 사용하고 호출함)

응용, 도메인 영역이 DB 나 외부 시스템을 사용하기 위해 인프라 영역을 사용하면 외부 시스템에 종속적인 코드가 되고 테스트 작성 및 기능확장이 어려워진다.

테스트를 할 경우 인프라 영역이 완벽하게 동작해야 하고, 기능확장시 기존의 (직,간접적) 의존적 코드를 모두 고쳐야 한다. 68p

```
public class CalculateDiscountService{
  private DroolsRuleEngine ruleEngine;
  
  public CalculateDiscountService(){
  ruleEngine = new DroolsRuleEngine(); // 저수준 모듈 구현체
  }
  
  public Money calculateDiscount(OrderLine orderLines, String customerId){
  Customer customer = findCustomer(customerId);
  
  MutableMoney money = new MutableMoney(0); // Drools 특화 코드, 연산 결과를 받기 위해 추가한 타입
  
  List<?> facts = Arrays.asList(customer, money); // 룰에 필요한 데이터(지식), Drools 에 특화된 코드
  facts.addAll(orderLines);
  
  ruleEngine.evalute("discountCalcuatlion",facts); // Drools 에 특화된 코드 세션 이름
  return money.toImmutableMoney();
  }

}
```

## 2.3 DIP

계층 구조 아키텍처를 적용해서 **도메인 및 응용(application) 영역**인 고수준 모듈(CalculateDiscountService)이 저수준 모듈(RDBMS, Drools 룰)에 의존하게 하면 

앞서 본 것 처럼 테스트 코드 작성 및 기능의 확장이 어려워진다. 70p

* 고수준 모듈이란? 의미있는 단일 기능을 제공하는 모듈을 말한다.

DIP 를 적용하면 고수준 모듈은 주입 받을 인터페이스에만 의존하게 되고 저수준 모듈이 고수준 모듈의 인터페이스를 구현함으로써 저수준 모듈이 고수준 모듈을 

의존하게 되는 의존 관계 역전이 일어나게 된다. 

**DIP 는 고수준(도메인) 과 저수준 인프라(인터페이스, 구현체) 를 분리하는 것이 아니다.** 이렇게 되면 결국 고수준인 도메인 영역이 인프라를 의존하는 관계가 될 뿐이다.

하위 기능을 추상화 하는 인터페이스를 **고수준 모듈의 관점에서 도출**하고 추상화하며 도메인 계층에 속한 인터페이스를 **저수준인 인프라 영역**에서 구현체로 만들어야 한다.

즉 고수준 모듈은 어떤 인프라가 구현체로 들어와도 상관없이 자신의 역할을 수행하면 된다. 고수준 모듈은 어떤 구현체가 주입되는지 알 필요가 없다.

* 표현 -> 응용 -> 도메인 <- 인프라(응용, 도메인의 인프라 인터페이스 구현 및 제공) 77p

DIP 를 적용하면 앞서 발생한 테스트 및 의존적 코드 문제를 해결할 수 있다.

```
@RequiredArgsConstructer
@Service
public class CalculateDiscountService{
  private final CustomerRepository customerRepositry;  //RDBMS JPA 로 고객 정보를 구한다.
  private final RuleDiscounter ruleDiscounter;  // Drools 로 할인 금액 룰 적용

...  스프링 부트를 사용하면 저수준의 빈 객체가 만들어지고 서비스에서 DI 받을 수 있다.
}
```
```
@Test
void 테스트 {

 // 테스트 목적 대역 객체
 CustomerRepository stubRepo = mock(CustomerRepository.class);
 when(stubRepo.findById("noCustId")).thenReturn(null);
 
 RuleDiscounter stubRule = (cust, lines) -> null;
 
 // 대용 객체 주입 받아 테스트 진행
 CalculateDiscountService cals = new CalculateDiscountService(stubRepo, stubRule);
 assertThrows(NoCustomerException.class, () -> cals.calculateDiscount(lines , "noCustId"));
}

Mock 프레임 워크를 이용한 대역 객체를 생성하고 대역을 주입받아서 테슷트를 진행할 수 있다, DIP 를 적용해서 고수준 모듈이 저수준 모듈에 의존하지 않기 떄문.

```

## 2.4 도메인 영역의 주요 구성 요소

* 엔티티 와 밸류

도메인 모델의 엔티티와 관계형 DB 의 엔티티는 다르다. 도메인 모델 엔티티는 데이터와 함께 도메인 기능을 제공한다. 

RDBMS 는 밸류 값을 표현하기 힘들다. 밸류는 불변으로 만들고 값을 바꿀 때는 새로운 객체를 생성해서 객체 자체를 교체해주자!

```
private void setShippingInfo(ShippingInfo newShippingInfo){
if(newShippingInfo == null) throw new IllegalArgumentException();

this.shippingInfo = newShippingInfo;
}
```

* 애그리거트

애그리거트란 관련 객체를 하나로 묶은 군집이다. 애그리거트는 루트를 통해서 간접적으로 애그리거트 내의 다른 엔티티나 밸류 객체에 접근할 수 있다.

애그리거트의 내부 구현을 숨기고 애그리거트 단위로 구현을 캡슐화.

ex) 주문(Order - Root) , 배송지 정보(ShippingInfo), 주문자(Orderer), 주문 목록(OrderLine) 이 녀석들이 하나로 묶여서 애그리거트가 되고 

애그리거트의 루트는 애그리거트의 상태를 관리하고 루트를 통해서 각각의 엔티티나 밸류 객체에 접근 할 수 있다.

* 리포지터리

리포지터리의 구현체는 저수준 인프라 모듈이지만 리포지토리의 인터페이스는 추상화된 고수준 모듈(도메인 객체를 영속화 하는 데 필요)이다. 

앞서 본 것 처럼 인프라를 추상화 할 때는 고수준 모듈의 관점에서 추상화 하자!

ex) 응용서비스(application) 의 관점에서 도메인 객체를 구하거나 저장해야 하는 로직이 필요한 경우 고수준 모듈의 관점에서 리포지토리를 추상화한다.

```
public interface SomeRepository { // 추상화
  void save(Some some);
  Some findById(Long id);
```

* 도메인 서비스

## 2.5 요청 처리 흐름

클라이언트 요청이 표현 영역을 통해서 들어오면 표현 영역에서는 데이터 형식이 올바른지 검사하고 응용 영역이 요구하는 데이터 형식으로 변환해서 전송한다.

## 2.6 인프라스트럭처 개요

인프라스트럭처는 표현, 응용, 도메인 영역을 지원한다. 응용, 도메인 영역에서 정의한 인터페이스가 인프라스트럭처로 구현된다.

무조건 인프라스트럭처에 대한 의존을 없앨 필요는 없다.

@Transacitonal 을 사용해서 트랜잭션을 시작한다던지, @Entity, @Table 과 같은 JPA 애너테이션을 이용해서 XML 매핑보다 편리하 사용할 수 있음.

이런 구현의 편리함은 DIP 가 주는 다른 장점 만큼 중요. DIP 의 장점을 해치지 않는 범위 내에서 응용, 도메인 영역에서 구현 기술에 대한 의존 관계를

가지고 가는 것도 나쁘지 않다 93p

## 2.7 모듈 구성

어플리케이션 아키텍처 4(ui, application, domain, infra) 가지 구성요소는 하나의 패키지 안에 존재한다.

* 도메인이 크다면 하위 도메인으로 분리해서 각각 패키지로 만들 수 있다.

    ex) catalog (ui , app, dom, infra) / order(ui, app, dom, infra ) / member (ui, app, dom, infra) / 95p

* 각각 하위 도메인 패키지 내의 도메인은 도메인에 속한 애그리거트를 기준으로 다시 패키지를 구성한다. 

    ex) 하위 도메인 catalog 내에서 (category , product) 로 나뉨 

* 애그리거트, 모델, 리포지터리는 같은 패키지에 위치한다. 

    ex) 주문과 관련된 Order, OrderLine, Orderer, OrderRepositor 등은 com.myshop.order.domain 패키지에 위치
     
* 도메인이 복잡하다면 도메인 모델과, 도메인 서비스(정책)를 별도의 패키지에 위치시킬 수 있다.

    ex) com.myshop.order.domain.order: 애그리거트 위치 / com.myshop.order.domain.service: 도메인 서비스 위치 

* 응용서비스(app)도 마찬가지로 도메인 별로 패키지를 구분할 수 있다.

    ex) com.myshop.catalog.application.product / com.myshop.application.category

TIP
하나의 패키지에 가능하면 10 ~ 15 개 미만으로 타입 개수를 유지하자.

