## 9.1 도메인 모델과 경계

도메인 규칙과 요구사항에 따라서 하위 도메인 모델이 만들어진다. 이때 유사하거나 동일한 라이프 사이클을 가지는 

모델들이 모여서 애그리거트가 된다.

상품이라는 하위 도메인 모델은 상품 상세에서는 **주문자가 구매한 상품**이 되고, 카탈로그에서의 **상품을 지칭하는 것은 상세 설명**이 담긴 

상품 정보를 의미하게 된다.

같은 것처럼 보일 수 있지만 주문 애그리거트에서 클라이언트는 **주문자**가 되고, 회원 애그리거트에서 클라이언트는 **회원**이, 

배송 애그리거트에서 클라이언트는 **보내는 사람**이 된다.

이렇게 ***각각의 하위 도메인은 같은 용어를 사용해도 의미가 달라지고, 의미가 같더라도 다른 용어로 지칭될 수 있다.***

하위 도메인 마다 사용하는 용어, 의미가 다르기 때문에 하위 도메인 마다 모델을 만들어야 하고 각 모델은 명시적으로 구분되는 경계를 가져서

섞이지 않게 해야 한다 (바운디드 컨텍스트)

## 9.2 바운디드 컨텍스트

바운디드 컨텍스트는 각 애그리거트의 하위 모델들별 경계를 결정한다. 논리적으로 한 개의 모델을 가지게 되며 앞서 말한 용어의 사용에 따라서

바운디드 컨텍스트가 구분된다.

주문 애그리거트에서의 Orderer 와 회원 애그리거트의 Member 는 바운디드 컨텍스트별로 구분된다.

애그리거트 하위 도메인을 여러 개의 바운디드 컨텍스트로 구분할 수 도 있고, 규모가 작다면 하위 모델을 모아서 하나의 컨텍스트로 묶을 수도 있음 279p

이때 주의할 점은 애그리거트 간 하위 모델을 섞어버리면 안된다는 것이다. 애그리거트를 하나로 합치지 말자.

> 각 도메인 모델별 컨텍스트 구분 예시 280p
>
>* 회원 바운디드 컨텍스트 - Member<root>
>
>* 주문 바운디드 컨텍스트 - Orderer<value>  
>  
>* 카탈로그 바운디드 컨텍스트 - Product<root> , Category<root> // 용어를 명확하게 구분하지 못한 경우 두 하위 도메인을 하나로 구현하기도 한다.
>  
>* 재고 바운디드 컨텍스트 - Product<root>

  
## 9.3 바운디드 컨텍스트 구현
  
바운디드 컨텍스트별로 하위 도메인을 구분한다고해서 바운디드 컨텍스트가 도메인 모델만 가지고 있는 것은 아니다.
  
주문 바운디드 컨텍스트를 구현한다면 (표현, 응용, 도메인, 인프라) 의 모든 요소가 다 들어가게 된다.

모든 바운디드 컨텍스트를 DDD 로 구현하지 않아도 된다.

간단한 리뷰 같은 경우 복잡한 도메인 로직이 필요 없기 때문에(표현, 서비스, DAO) 순으로 구현해도 된다.
  
CQRS 를 이용해서 한 바운디드 컨텍스트에서 두 가지 방식을 혼합해서 사용할 수도 있음

도메인 로직을 실행해야하는 부분은 DDD 를 사용하고, 간단한 조회같은 경우 서비스에서 바로 DAO 를 호출하면 된다. 283p

상품 상세 화면의 경우 카탈로그 컨텍스트와 리뷰 컨텍스트를 보여줘야하는데
  
UI 계층을 파사드 처럼 이용해서 한번에 모아서 값을 보여줄 수도 있고 각각 따로따로 보여줄 수도 있다. 283P
  
  
## 9.4 바운디드 컨텍스트 간 통합
  
온라인 쇼핑 사이트에서 매출 증대를 위해 카탈로그 하위 도메인에 상품 추천 기능을 도입하기로 한다면 
  
바운디드 컨텍스트간 통합이 발생할 수 있다. 상품 카탈로그를 조회했을 때 추천 상품 정보를 추천 도메인에서 읽어와야 하기 때문이다.  
  
카탈로그 시스템이 추천 시스템으로부터 추천 데이터를 받아올 때 카탈로그에서 추천 도메인 모델을 사용하기보다는 카탈로그 도메인 모델을
  
  사용해서 응용 서비스에서 추천 상품을 표현해야 한다.
  
  ```
  // 상품 추천 기능을 표현하는 도메인 서비스
  public interface ProductRecommendationService{
    List<Product> getRecommendationOf(ProductId id); // 상품 추천에서 받은 추천 상품 아이디.
  }
  
  카탈로그 관점에서 도메인 서비스를 만들고 인프라에서 외부 상품 추천 시스템 기능과 통신을 수행한다. 286p
  ```
 
  ```
  infra 구현체
  
  public class RecSystemClient implements ProductRecommendationService{
  
  private ProductRepository productRepository; // ProductRepository 에 의존.
  
  @Override
  private List<Product> getRecommendationsOf(ProductId id){
  List<RecommendationItem> items = getRecItems(id.getValue());
                          return toProducts(items);
  }
  
  private List<RecommendationItem> getRecItems(String itemId){
  
  //externalRecClient 를 외부 추천 시스템을 위한 클라이언트라고 가정한다.
  return externalRecClient.getRecs(itemId); 
  
  }
  
  private List<Product> toProducts(List<RecommendationItem> items){
  return items.stream().map(item -> toProductId(item.getItemId()).map(prodId -> productRepository.findById(prodId))
              .collect(toList());
  
  private ProductId toProductId(String itemId){
    return new ProductId(itemId);
  }
  }
  
  
  응용 서비스에서 카탈로그에 있는 프로덕트 아이디를 이용해서 외부 추천 시스템의 추천 상품 로직을 조회한다.
  
  (외부 추천 시스템은 인프라 영역에 해당한다) 
  
  추천 상품의 아이디 값으로 추천 상품들을 ProductRepository 에서 조회해서 추천 상품 목록을 반환하는 구조.
  
  ```
                                          
                                          
                                          
                                          
                                          
                                          
                                          
                                          
                                          
                                          
                                          
                                          
                                          
                                          
                                          
                                          
                                          
                                          
                                          
                                          
                                          
                                          
                                          
                                          
                                          
                                          



                                          
                                          
                                          
                                          
                                          
                                          
                                          
                                          
                                          
                                          
                                          
                                          
                                          
                                          
                                          
                                          
                                           
                                           
