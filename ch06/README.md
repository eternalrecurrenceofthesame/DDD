## 6.1 표현 영역과 응용 영역

표현 영역에서 사용자의 요청을 분석하고 사용자가 원하는 기능을 제공하는 응용 서비스를 실행한다.

이때 표현 영역에서 응용 영역이 필요로하는 데이터로 값을 변환해서 넘겨준다

## 6.2 응용 서비스의 역할

사용자의 요청을 처리하기 위해 리포지토리에서 도메인 객체를 가져와서 사용한다. 응용 서비스는 도메인 객체간의 흐름을 제어 한다.

**응용 서비스에 도메인 로직을 넣으면 안 된다!**

도메인 응집도가 떨어진다, 여러 계층에 로직이 분산되어 있기때문에 여러 영역을 분석해야하고 코드를 변경할 때 여러 영역을 동시에 

수정해야 한다.

또한 여러 응용 서비스에서 동일한 도메인 로직을 구현할 가능성이 높아진다, 도메인 로직은 도메인에서 만들고 사용하자


## 6.3 응용 서비스의 구현 

* 응용 서비스의 크기
  
  한 응용 서비스에서 도메인 기능을 모두 처리할 수도 있지만 서비스가 커지면 유지보수가 어려워진다.
  
  구분되는 기능별로 서비스 클래스를 구현하는 방식을 사용하면 2 ~ 3 개의 기능을 한곳에 구현할 수 있고 
  
  각 클래스 기능별로 필요한 의존 객체만 포함하므로 유지 보수에 용이하다.
  
  응용 서비스마다 공통으로 사용되는 로직을 별도 클래스로 만들어서 사용하는 방법도 있다.

```
// 응용 서비스에서 공통으로 사용하는 단건 조회 로직
public final class MemberServiceHelper {

   public static Member findExistingMember(MemberRepository repo, String memberId){
   
   Member member = memberRepository.findById(memberId);
   if(member == null)
   throw new NoMemberException(memberId);
   
   return member;
   }
}


// 공통로직을 사용하는 응용 서비스 구현

public class ChangePasswordService{

  private MemberRepository memberRepository;
  
  public void changePassword(String memberId, String curPw, String newPw{
  
  Member member = findExistingMember(memberRepository, memberId);
  member.changePassword(curPw, newPw);
  }

}

```

* 응용 서비스의 인터페이스 클래스를 만들어야 하는가?

굳이 만들 필요는 없다. 인터페이스를 만드는 이유는 구현 클래스가 여러 개 일때 다형성을 이용해서 구현 클래스를

쉽게 교체하기 위함이지만 응용 서비스마다 공통으로 사용하는 로직을 묶어서 만들게 되면 여러 개의 구현 클래스를 

만들 이유가 없기 때문이다. 응용 서비스의 구현 클래스가 2 개인 경우는 드물다.


* 메서드 파라미터와 값 리턴

응용 서비스는 사용자 요청 표현 영역에서 넘어온 값을 이용해서 도메인 로직을 호출하는 역할을 수행한다. 

표현 영역에서 값을 하나하나 받을 수도 있고 하나의 요청 클래스로 받을 수도 있음

```
public class ChangePasswordService

public void changePassword(String memberId, String curPw, String newPw) {}
or
public void changePassword(ChangePasswordRequest req) { // 리퀘스트 클래스로 파라미터 값 3개 받아서 꺼내서 쓰기 

Member member = findExistingMember(req.getMemberId());
member.chnagePassword(req.getCurrentPassword(), req.getNewPassword());

}
```

표현 영역에서 응용서비스의 결과 값을 사용해야 할때 응용 서비스에서 값을 리턴하는데 애그리거트 자체를 리턴하면 안 된다.

이렇게되면 도메인의 로직 실행을 표현 영역에서 할 수 있기 때문에 응집성이 떨어지게 된다.

응용서비스는 표현영역에서 필요한 값만 리턴해주자.

```
OrderNo orderNo = orderService.placeOrder(orderReq);
model.setAttribute("orderNo", orderNo.toString());

return "order/success";
```

* 표현 영역에 의존하지 않기

표현 영역에서 응용 서비스에서 사용하는 자료형으로 바꾸지 않고 파라미터를 넘겨주게 되면 응용 서비스가 표현 영역에 의존하게 된다. 

(표현 영역에서 사용하는 HttpServletRequest request 의 값을 그대로 넘겨주면 안 된다.)

* 트랜잭션 처리 

트랜잭션은 응용 서비스 영역에서 시작한다. 스프링 프레임워크는 @Transactional 이 적용된 곳에서 RuntimeException 이 발생하면 

트랜잭션을 롤백한다.

## 6.4 표현 영역
```
* 표현 영역이 제공하는 기능

사용자가 시스템을 사용할 수 있는 화면을 제공하고 제어한다.
- 단순 컨트롤러 뿐 아닌 입력 폼을 포함

사용자의 요청을 알맞은 응용 서비스에 전달하고 결과를 사용자에게 제공한다.
- 응용 서비스에 요청을 전달할 때 요청 데이터를 응용 서비스가 요구하는 형식으로 변환하고
  반환된 응용 서비스의 결과를 사용자에게 응답할 수 있는 형식으로 변환한다. 

사용자의 세션을 관리한다.
- 웹은 쿠키나 서버 세션을 이용해서 사용자의 연결 상태를 관리한다. 
```
## 6.5 값 검증

값에 대한 검증은 표현 영역과 응용 서비스 두 곳에서 모두 수행할 수 있다. 원칙적으로 모든 값에 대한 검증은 응용 서비스에서 처리한다.
```
표현 영역: 필수 값, 값의 형식, 범위등에 대한 검증
응용 서비스: 데이터의 존재 유무와 같은 논리적 오류의 검증

가능하면 응용 서비스에서 필수 값 검증과 논리적인 검증을 모두 하는 것이 좋다. 프레임워크가 제공하는
검증 기능을 사용할 때보다 작성 코드가 늘어나는 불편함은 있지만 응용 서비스의 완성도를 높일 수 있다.
```

## 6.6 권한 검사 

표현 영역, 응용 서비스, 도메인 세 곳에서 스프링 프레임워크로 권한 검사를 수행할 수 있다.
```
* 표현 영역에서 권한 검사

스프링 프레임워크의 시큐리티 필터 체인을 사용해서 인증 여부와 권한에 따른 엔드 포인트 호출을 제어할 수 있다. 
```
```
* 응용 서비스에서 권한 검사

스프링 시큐리티가 제공하는 전역 메서드 보안을 사용해서 애플리케이션 전 계층에서 권한을 검사할 수 있다.
https://github.com/eternalrecurrenceofthesame/Spring-security-in-action/tree/main/part4/ch16 참고
```
```
* 개별 도메인 객체 단위 검사 하기 

public void delete(String userId, Long articleId){

  Article article = articleRepository.findById(articleId);
  checkArticleExistence(article);
  permissionService.checkDeletePermission(userId, article); // 권한 체크 
  article.markDeleted(); // 도메인 규칙 호출
  ...
}

permissionService.checkDeletePermission 메서드는 전달받은 사용자 아이디와 게시글을 이용해서
삭제 권한이 있는지 검사할 수 있다. 그리고 도메인 규칙을 호출한다.

위 방법은 앞서 언급한 전역 메서드 보안을 사용해서 검사하는 방향으로 만들 수도 있다.
```

## 6.7 조회 전용 기능과 응용 서비스

단순 조회를 위한 기능은 트랜잭션이 필요 없으므로 표현 영역에서 바로 조회할 수도 있다. 





  

