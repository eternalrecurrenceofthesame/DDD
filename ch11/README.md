## 11.1 단일 모델의 단점

단일 모델로 도메인 상태를 변경하는 것은 어렵지 않은데 조회 문제가 있다. 조회의 경우 여러 애그리거트의 값을 가져와야 하는 경우가 많은데 

(주문 내역 조회시 Order, Product, Member 를 가지고 주문 상세 화면을 만든다)

DDD 는 객체 참조가 아닌 식별자 참조를 사용하기 때문에 즉시 로딩으로 최적화를 할 수 없다. 참조를 이용하더라도 즉시 로딩 지연 로딩을 처리해야 하는 고민은 있다.

##  11.2 CQRS

명령과 조회를 분리해서 관리하는 것을 CQRS 라고 한다. 어플리케이션에서 조회 호출이 명령 호출보다 훨씬 많다. 명령과 조회를 분리해서 조회에 최적화된 기능을 제공하여 성능 최적화를 유지한다.

조회를 따로 분리해서 사용할 때 간단한 데이터 요청은 응용 서비스를 만드는 대신 컨트롤러에서 바로 DAO 를 통해 값을 조회해도 된다.

각 모델에 맞게 데이터베이스를 사용할 수도 있다 명령의 경우 RDBMS 를 조회의 경우 NoSQL 을 

서로 다른 데이터베이스를 사용하면 이벤트를 통해서 데이터 정합성을 유지한다.

* 동기로 이벤트를 처리하는 경우

명령 모델에서 값이 바뀔 때 조회 모델에 바로 반영해야 한다면 **동기 이벤트와 글로벌 트랜잭션**을 사용해서 실시간으로 동기화 할 수 있다 (성능이 떨어지는 단점이 있다.)

* 비동기로 이벤트를 처리하는 경우

명령 모델에서 바뀐 값을 실시간으로 동기화 하지 않아도 되는 경우에는 비동기 이벤트를 메시징이나 이벤트 저장소를 이용해 처리하면 된다. 특정 시간 단위 통계 처리 같은 것

CQRS 를 적용하기 위한 필수 기술이 있는 것은 아니다. 명령은 JPA 로 처리하고, 조회는 QDS, JDBC template 으로 처리할 수 있다 필요한 것을 맞게 사용하면 된다.

#### + CQRS 장단점
```
* 장점

복잡한 도메인 모델을 유지보수하는 데 도움을 준다. 복잡한 도메인의 경우 명령 과정이 복잡하다 조회를 분리함으로써 도메인 로직에 집중할 수 있다.

조회단위 캐시 적용, 조회에 특화된 쿼리 사용, 조회 전용 저장소 사용, 조회 성능을 높이기 위한 코드가 명령 모델에 영향을 주지 않음
```
```
* 단점

구현해야할 코드가 많다. 도메인이 단순하거나 트래픽이 많지 않다면 굳이 CQRS 를 사용하지 않아도 된다.

CQRS 를 사용하면 다양한 구현기술이 필요하다. 명령과 조회에 따른 다른 기술 적용, 메시징 적용 등등
```

위 장단점을 고려해 CQRS 도입을 결정하자.

CQRS 구현에 관해서는 https://github.com/eternalrecurrenceofthesame/DDD/tree/main/ch05 를 참고한다. 





