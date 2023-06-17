## 8.1 애그리거트와 트랜잭션

애그리거트에 대해 사용할 수 있는 대표적인 트랜잭션 처리 방식 - 선점 잠금, 비선점 잠금

## 8.2 선점 잠금

먼저 애그리거트를 구한 스레드에서 작업이 끝날 때까지 다른 스레드가 해당 애그리거트를 수정하지 못하는 것을 말한다.

```
JPA em 사용시

Order order = entityManager.find(Order.class, 
                                 orderNo, LockModeType.PESSIMISTIC_WRITE);

하이버네이트에서는 PESSIMISTIC_WRITE 를 잠금모드로 사용하면 'for update' 쿼리를 이용해 선점 잠금을 구현한다.

```
```
스프링 데이터 JPA 사용시 

@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("select m form Member m where m.id = :id")
Optional<Member> findByIdForUpdate(@Param("id") MemberId memberId);
```

**선점 잠금을 사용할 때는 잠금 순서에 따른 교착 상태가 발생하지 않도록 주의하자**
```
* 데드락 예시

스레드 1 이 먼저 A 애그리거트에 대한 선점 잠금, 스레드 2 가 B 애그리거트에 대한 선점 잠금을 각각 구함

스레드 1 이 작업 중 B 애그리거트에 대한 락 획득 시도, 스레드 2 도 A 애그리거트 락 획득을 시도하면 교착 상태에 빠지게 된다.  
```

이런 문제가 발생하지 않게 하기 위해서는 잠금을 구할 때 최대 대기 시간을 지정해야 한다.

```
JPA 에서 대기 시간 지정하기

Map<String, Object> hints = new HashMap<>();

hints.put("javax.persistence.lock.timeout", 2000);
Order order = em.find(Order.class, orderNo,
                         LockModeType.PESSIMISTIC_WRITE, hints);
```

```
스프링 데이터 JPA

@Lock(LockModeType.PESSIMISTIC_WRITE)
@QueryHints({
       @QueryHint(name = "javax.persistence.lock.timeout", value = "2000")
})
@Query("select m from Member m where m.id = :id")
Optional<Member> findByIdForUpdate(@Param("id") MemberId memberId);
```

힌트를 밀리초 시간 동안 작업을 완료하지 못하면 익셉션을 발생시킨다.

Tip

DBMS 에 따라 교착 상태에 빠진 커넥션을 처리하는 방식이 다르다. 쿼리별로 대기 시간을 지정할 수 있는 DBMS 가 있고

커넥션 단위로만 대기 시간을 지정할 수 있는 DBMS 도 있다. 따라서 선점 잠금을 사용하려면 사용하는 DBMS 에 대해

JPA 가 어떤 식으로 대기 시간을 처리하는지 반드시 확인해야 한다.

또한 DBMS 에서 힌트를 지원하지 않을 수도 있기 때문에 관련 기능을 지원하는지 확인해야 한다.


## 8.3 비선점 잠금

선점 잠금으로 해결할 수 없는 상황이 발생한다면?

운영자가 배송을 위해 배송지 정보를 포함한 주문 정보를 조회한다. 그 사이 고객이 자신의 배송지 정보를 수정한경우

운영자는 배송상태를 변경할 때 수정전의 배송지 정보를 토대로 배송 상태를 변경할 수 있다.

문제는 선점 해서 값을 순차적으로 바꾸더라도 잘못된 정보를 토대로 값이 바뀔 수 있다는 것이다.

비선점 잠금을 사용하면 애그리거트를 선점하는 대신 , 변경한 데이터를 실제 DBMS 에 반영하는 시점에 변경 가능 여부를 확인한다.

비선점 잠금을 사용하려면 애그리거트에 버전으로 사용할 숫자 타입 프로퍼티를 추가해야 한다.

```
스레드 1 , 2가 각각 애그리거트를 구하고 수정한다.

이때 먼저 수정한 스레드 1 의 작업이 커밋되면 DBMS 에 애그리거트 버전이 변경된다.

스레드 2 는 자신이 조회한 애그리거트 버전과 저장된 애그리거트의 버전이 다르기 때문에 값을 변경할 수 없다.
(즉 새로 조회해서 값을 수정해야 함)
```

```
JPA 는 버전을 이용한 비선점 잠금 기능을 지원한다.

@Entity
@Table(name = "purchase_order")
@Access(AccessType.FIELD)
public class Order{

  @EmbeddedId
  private OrderNo number;

  @Version
  private long version;

업데이트 쿼리 실행시 @Version 필드에 명시한 필드를 이용해서 비선점 잠금 쿼리를 실행!

UPDATE purchase_order SET ... , version = + 1
WHERE number = ? and version = 10

JPA 버전 관리는 https://github.com/eternalrecurrenceofthesame/JPA/tree/main/JPA-advance 를 참고한다. 
```

응용 서비스는 버전에 대해서 알 필요가 없다.

비선점 잠금을 위한 쿼리의 수행 결과 수정된 행의 개수가 0 이면 누군가 앞서 데이터를 수정한 것이고 예외가 발생한다. 256p

예외는 서블릿을 거쳐서 예외 화면을 보여줄 수도 있고 컨트롤러에서 처리하거나 익셉션 리졸버로 필요한 값을 보여주면 된다.

비선점 잠금 방식을 여러 트랜잭션으로 확장하고 싶다면 애그리거트 정보를 뷰로 보여줄 때 버전 정보도 함께 전달해야 한다.

```
히든 타입으로 인풋 태그를 만들고 폼 전송시 버전 값이 서버에 함께 전달되도록 하자.

조회할때 버전 정보도 같이 조회한 다음에 배송 상태 변경 요청시 버전값도 같이 보내줘야 한다는 의미.

<form th:action="@{/startShiping}" method = "post">
<input type = "hidden" name = "version" th:value = ${orderDto.version}">
<input type = "hideen" name = "orderNumber" th:value = "${orderDto.orderNumber}">
...

<input type = "submit" value = "배송 상태로 변경">
</form>
```
```
표현 계층에서 응용 서비스로 값을 요청할 때도 버전 타입을 넣어줘야 함.

public class StartShippingRequest{
  private String orderNumber;
  private long version;
}


응용 서비스에서 버전 값을 이용해 애그리거트 버전이 일치하는지 확인하고 일치하는 경우에만 기능 수행!

public class StartShippingService{

  @PreAuthorize("hasRole('ADMIN')")
  @Transactional
  public void startShipping(StartShippingRequest req){
    Order order = orderRepository.findById(new OrderNo(req.getOrderNumber()));
    checkOrder(order); //??
    
    if(!order.matchVersion(req.getVersion()) {
       throw new VersionConflictException();  // 예외 발생시 표현 영역을 거쳐서 처리하게 됨. 
    }
  
  order.startShipping();
  }
}

```

```
표현 영역에서 오류 처리

@PostMapping("/startShipping")
public String startShipping(StartShippingRequest startReq){
    try{
      startShippingService.startShipping(startReq);
      return "shippingStarted";
      }catch(OptimisticLockingFailureException | VersionConflictException ex){
      
      return "startShippingTxConflict";

      }
    }
    
    * OptimisticLockingFailureException : 거의 동시에 애그리거트 수정시 발생하는 익셉션
    
    버전 충돌에 대한 구분이 명시적으로 필요 없다면 응용 서비스에서 프레임워크 용 익셉션을 발생시킬 수도 있다.
```

애그리거트에 엔티티가 두 개 있을 때 루트 엔티티가 아닌 다른 엔티티에서 값을 수정 한다고 해서 DBMS 의 버전 값은 변경되지 않음.

하지만 비선점 잠금 방식을 사용하려면 다른 엔티티 값이 수정된 경우 루트 엔티티의 버전을 갱신해줘야 올바르게 비선점 잠금이 동작한다.

```
JPA 의 강제 버전 증가

public Order findByIdOptimisticLockMode(OrderNo id){

return em.find(Order.class, id, LockModeType.OPTIMISTIC_FORCE_INCREMENT);
}

스프링 데이터 JPA 사용시 @Lock 으로 해주면 된다.
```

## 8.4 오프라인 선점 잠금 

선점 잠금은 하나의 트랜잭션에서 적용되는 개념이다. 

수정화면을 요청하고 수정화면을 폼으로 보여주는 것을 하나의 트랜잭션 개념으로 끊을 수 있다. 이렇게하면 

누군가 수정화면을 보고 있을 때 , 수정화면을 다른 스레드에서 수정해버릴 수 있음.

수정화면을 보고 있을 때 수정하지 못하게 하는 것은 선점 잠금이나 , 비선점 잠금 방식으로는 구현할 수 없다. 

선점 잠금의 경우 트랜잭션이 끝나버리고, 비선점 잠금은 단순 조회로 버전이 변경되지 않기 때문임 

오프라인 선점 잠금을 이용해서 여러 트랜잭션에 걸쳐서 동시에 변경되는것을 막을 수 있다.

```
lock package

LockManager.class // 오프라인 선점 잠금을 지원하는 인터페이스 

LockId.class // 오프라인 선점 잠금에 필요한 락 아이디 (고유 식별자)


응용 서비스에서 데이터와 락 아이디를 리턴하는 메서드
public DataAndLockId getDataWithLock(Long id){
   
   // 오프라인 선점 잠금 시도
   LockId lockId =  lockManager.tryLock("락 대상 타입",id);
   
   // 기능 실행 아이디 값으로 데이터를 DBMS 에서 가지고 온다.
   Data data = someDao.select(id); 
   
   // 데이터와 락 아이디 값을 반환한다.
   return new DataAndLockId(data, lockId);
   
}
```

```
표현 영역

@RequestMapping("/some/edit/{id}")
public String editForm(@PathVariable("id") Long id, ModelMap model) {

  DataAndLockId dl = dataService.getDataWithLock(id); // 앞서 만든 응용 서비스
  model.addAttribute("data", dl.getData());
  
  // 잠금 해제에 사용하는 락 아이디
  model.addAttribute("lockId", dl.getLockId());
  return "editForm"
}

여기까지의 흐름을 정리하자면 수정 폼을 요청할 때 락아이디와 데이터 값을 리턴으로 받고 

폼 화면을 보여준다. 이때 잠금 선점에 실패시 오류 발생, 다른 사용자가 데이터를 수정중인 것이니

오류 화면을 보여주면 된다.

<form th:action={@/some/edit/{id}(id=${data.id})}" method = "post">
<input type = "hidden" name = "lid" th:value = "${lockId.value}">
...
</form>

수정폼에서 post 호출시 할당받은 락 아이디를 히든필드로 같이 넘겨준다.
```


```
@RequestMapping(value = "/some/edit/{id}", method = RequestMethod.Post)
public String edit(@PathVariable("id") Long id, @ModelAttribute("editReq") EditRequest editReq,
                                                @RequestParam("lid") String lockIdValue){
editReq.setId(id);
someEditService.edit(editReq, new LockId(lockIdValue)); // 처음 받은 락 아이디 값을 넘겨주고 응용 영역에서 작업 수행

model.addAttribute("data",data);  // 데이터를 다시 받을 수 있는듯?? 
return "editSuccess";  // 화면 출력.
            }
            

응용 영역
public void edit(EditRequest editReq, LockId lockId){

lockManager.checkLock(lockId); // 잠금 상태 확인

                               // 기능 실행

lockManager.releaseLock(lockId);// 잠금 해제
}

```


