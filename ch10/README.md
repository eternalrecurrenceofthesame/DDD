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


