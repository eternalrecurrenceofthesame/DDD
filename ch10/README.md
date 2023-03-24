## 10.1 시스템 간 강결합 문제

쇼핑몰에서 구매를 취소하면 환불 처리를 해야 한다. 환불 처리를 도메인 서비스로 만들고 도메인 객체에서 도메인 서비스를 이용해 직접 처리하거나 

응용 서비스에서 도메인 서비스를 이용해서 환불 처리를 할 있다.

* 도메인 에서 도메인 서비스를 직접 처리하는 경우

```
public class Order{

public void cancel(RefundService refundService){

verifyNotYetShipped();
this.state = OrderState.CANCELED; // 주문 관련 로직

this.refundStatus = Status.REFUND_STARTED; // 결제 로직
try{
refundSvc.refund(getPaymendId());
this.refundStatus = Status.REFUND_COMPLETED;

} catch(Exception ex){
...
}
}
```

도메인 객체에서 도메인 서비스를 직접 이용하는 경우 주문 로직과 결제 로직이 서로 섞이게 된다.

환불 기능이 바뀌게 되면 Order 도 같이 영향을 받게되고 

환불 취소기능에 환불 취소 메시지 전송 같은 부가 기능이 추가될수록 로직이 꼬이게되고 더 복잡하게 된다.


* 응용 서비스에서 도메인 서비스를 직접 처리하는 경우

```
public class CancelOrderService{
  private RefundService refundService;
  
  @Transactional
  public void cancel(OrderNo orderNo){
  
  Order order = findOrder(orderNo);
  order.cancel();
  
  order.refundStarted();
  
  try{
  refundService.refund(order.getPaymendId());
  order.refundCompleted();
  }catch(Exception ex){...}

}
```

트랜잭션이 문제될 수 있다. 주문을 찾고 주문을 취소하면서 환불 도메인 서비스를 이용해서 환불을 호출했는데

환불 도메인 서비스에서 오류가 발생한다면 주문 취소 서비스를 롤백하거나 주문을 취소 상태로 변경하고 환불만 

나중에 다시 시도해야할 수 있다.

환불 요청에 대한 대기시간이 길어진다면 외부 서비스로 인한 성능 문제까지 발생한다.


## 10.2 이벤트 개요

앞서 살펴본 강 결합 문제를 해결할 수 있는 것으로 이벤트를 사용하는 방법이 있다.

이벤트란? 과거에 벌어진 어떤 것을 의미한다. '암호 변경 했음 이벤트', '주문을 취소 했음 이벤트' 등등

~할 때, ~가 발생하면, 만약 ~하면 과 같은 요구 사항은 도메인의 상태 변경과 관련된 경우가 많고 이런 요구 사항을

이벤트로 구현할 수 있다.

이벤트는 네 개의 구성요소로 구성된다.

이벤트 생성 주체 -> 이벤트 디스패처(퍼블리셔) -> 이벤트 핸들러(구독자)

이벤트 생성주체가 이벤트를 생성하고 이벤트 디스패처가 중간에서 이벤트를 받고 이벤트 핸들러를 호출해서 이벤트를 전달한다.

> 이벤트 구성 요소
> * 이벤트 종류: 클래스 이름으로 이벤트 종류를 표현
> * 이벤트 발생 시간
> * 추가 데이터: 주문 번호, 신규 배송지 정보 등 이벤트 관련 정보 

```
이벤트를 위한 클래스

public class ShippingInfoChangedEvent { // 배송지를 변경했음 이벤트, 이벤트는 현재 기준으로 과거에 벌어진 일이다! 과거 시제 사용

  private String orderNumber;
  private long timestamp; // 발생 시간
  private ShippingInfo new ShippingInfo;

}
```

```
public class Order{ // 이벤트 발행 주체인 Order 애그리거트 

  public void changeShippingInfo(ShippingInfo newShippingInfo){
  verifyNotYetShipped();
  setShippinginfo(newShippingInfo);
  Events.raise(new ShippingInfoChangedEvent(number, newShippingInfo)); // 디스패처를 통한 이벤트 전파 기능 제공
  }
}
```

```
핸들러는 이벤트에 담긴 내용을 이용해 원하는 기능을 수행한다.

public class ShippingInfoChangedHandler { // 핸들러는 보통 도메인의 인프라에 속한다

  @EventListner(ShippingInfoChangedEvent.class)
  public void handle(ShippingInfoChangedEvent evt){
    shippingInfoSynchronizer.sync(
    evt.getOrderNumber(),
    evt.getNewShippingInfo());
  
  }
}

이벤트는 핸들러에서 처리하는 스펙에 맞게 보내야한다. 핸들러에서 처리할 수 없는 데이터를 전송하면 관련 API 를 호출하거나

DB 에서 데이터를 직접 호출해야한다. 또한 이벤트 자체와 관련 없는 데이터를 포함할 필요는 없다.
```



