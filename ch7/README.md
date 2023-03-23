## 7.1 여러 애그리거트가 필요한 기능 

주문에서는 주문 가격을 구하고, 상품에서는 상품 가격을 보여준다. 그런데 쿠폰을 적용한다든지 할인 이벤트를 적용 한다면

쿠폰 가격, 할인 가격은 어디서 구해야 할까?

주문, 상품 애그리거트의 책임 범위를 넘어서 각 애그리거트에 부가적인 기능을 더하면 부가 기능에 대한 의존도가 높아지고

코드를 복잡하게 만들어 유지보수를 어렵게한다. 

또한 애그리거트의 책임 범위를 넘어서는 도메인 개념이 애그리거트에 숨어들어 명시적으로 드러나지 않게 된다.

도메인 기능을 별도의 서비스로 구현하면 위 문제를 해결할 수 있다.

## 7.2 도메인 서비스

도메인 서비스는 애그리거트에 부가적인 기능을 더하기 위해 사용된다. 

도메인 서비스는 상태 없이 로직만으로 구현된다. 도메인 서비스를 구현하는 데 필요한 상태는 다른 방법으로 전달 받는다.

도메인 서비스는 *도메인의 의미가 드러나는 용어*를 타입과 메서드 이름으로 갖는다.

* 계산 로직과 도메인 서비스

```
public class DiscountCalculateionService{

  public Money calculateDiscountAmounts(
      List<OrderLine> orderLines,
      List<Coupon> coupons,
      MemberGrade grade){
     
     Money couponDiscount = 
          coupons.stream()
                 .map(coupon -> calcuateDiscount(coupon))
                 .rdeuce(Money(0), (v1, v2) -> v1.add(v2)); // 별령 처리로 쿠폰 할인금액 구하기 
      
     Money membershipDiscount = 
           calculateDiscount(orderer.getMember().getGrade());
      
      return couponDiscount.add(membershipDiscount);
      }
      
      
  // 쿠폰 할인 로직
  private Money calculateDiscount(Coupon coupon){...}
  
  // 등급 할인 로직
  private Money calculateDiscount(MemberGrade grade){...}

}

```

```
할인 계산을 하는 주체는 애그리거트가 될 수도 있고 응용 서비스가 될 수도 있다.

// 애그리거트
public class Order

public void calculateAmounts(DiscountCalculationService disCal){
    Money discountAmounts = discal.calculateDiscountAmounts(this.orderLines, this.coupons, grade);
}

// 응용 서비스
public class OrderService 

@Autowired
private DiscountCaluculationService disCalService;

```

Tip

도메인 메서드를 애그리거트 모델에서 받지 말자. 애그리거트 모델 필드는 데이터베이스에 보관하는 값들이지만

도메인 메서드는 저장 대상이 아니다. 애그리거트의 모든 기능에서 도메인 메서드를 사용하지 않을 수 있다.

```
애그리거트 메서드를 실행할 때 위의 경우와 달리 도메인 서비스를 인자로 전달받지 않고 도메인 서비스 실행시

애그리거트를 전달하는 경우도 있다.

public class TransferService { // 도메인 서비스

    public void transfer(Account fromAcc, Account toAcc, Money amounts) {
      fromAcc.withdraw(amouts);
      toAcc.credit(amounts);
    }
}

```

Tip 

특정 기능이 응용 서비스인지 도메인 서비스인지 헷갈릴 때는 해당 로직이 애그리거트의 상태를 변경 하거나

애그리거트의 상태 값을 계산하는지 검사해보면 된다.

도메인 로지기면서 한 애그리거트에 넣기 적합하지 않다면 도메인 서비스로 구현! 


* 외부 시스템 연동과 도메인 서비스

외부 시스템과의 연동이 필요한 경우 도메인 관점에서 인터페이스를 작성하고 인프라 영역에 구현체를 만들고

응용 서비스에서 도메인 서비스를 이용해 권한 검사를 할 수 있다.

ex) 설문 조사 시스템을 외부 인프라로 연동해서 사용할 때 도메인은 사용자가 설문 조사 생성 권한을 가졌는지 확인하는 

도메인 로직을 만들어야 할 수 있다. 

```
public interface SurveyPermissionChecker{ // 권한 검사 도메인 서비스

    boolean hasUserCreationPermission(String userId);
}

public class CreateSurveyService{ // 도메인 서비스를 이용해 권한을 검사하는 응용 서비스

    private SurveyPermissionChecker permissionChecker;
    
    public Long createSurvey(CreateSurveyRequest req){
    validate(req);
    
    if(!permissionChecker.hasUserCreationPermission(req.getRequestorId())){
      throw new NoPermissionException();
      }
      ...
    }
```


* 도메인 서비스의 패키지 위치

도메인과 같은 곳에 둔다. 도메인 패키지의 클래스들이 많은 경우

domain.model, domain.service(도메인 서비스), domain.repository 와 같이 구분할 수 있다.


* 도메인 서비스의 인터페이스와 클래스

도메인 서비스의 구현이 특정 구현 기술에 의존하거나 외부 시스템의 API 를 실행한다면 도메인 서비스를 인터페이스로 추상화 하고

인프라 영역에 구현체를 만들자. 

도메인 영역이 특정 구현에 종속되는 것을 방지하고 도메인 영역에 대한 테스트가 쉬워진다.

OrdererService 참고 
