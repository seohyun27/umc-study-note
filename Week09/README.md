# 목록 조회 API 만들기
- 대부분 서비스는 CRUD 중에 read의 비중이 크다.
- 또한 조회 API를 만들 때 정보의 처리와 페이징의 적용을 항상 함께 고민해야 한다.
- 목록 조회 API를 만들기 위해 필요한 정보는 아래와 같다.
  - 닉네임
  - 리뷰의 점수
  - 리뷰가 작성된 날짜
  - 리뷰의 상세 내용
  - 사진 (S3의 사용)
  - 사장님의 답글 (사진과 함께 이번 API에서는 생략한다)
- API를 만드는 것은 아래의 순서를 따르면 편하다
  1. API 시그니처를 만든다 
  2. API 시그니처를 바탕으로 API 명세서에 명세를 해준다 
  3. 비즈니스 로직을 만든다 
  4. 컨트롤러를 완성한다 
  5. validation 처리를 한다

## 1. API 시그니처 만들기
- 어떠한 요청을 응답하기 위한 DTO를 만드는 과정이다.
- 쿼리 파라미터, 리쿼스트 바디가 필요한 경우 함께 정의한다.

### DTO 작성
여러 리뷰 DTO를 ReviewResDTO라는 하나의 클래스로 묶은 뒤 그 하위에서 record나 static class를 만들어 사용함으로서 코드의 가독성을 높인다.
```java
public class ReviewResDTO {

    @Builder
    public record ReviewPreViewListDTO(
            List<ReviewPreViewDTO> reviewList,
            Integer listSize,
            Integer totalPage,
            Long totalElements,
            Boolean isFirst,
            Boolean isLast
    ){}

    @Builder
    public record ReviewPreViewDTO(
            String ownerNickname,
            Float score,
            String body,
            LocalDate createdAt
    ){}
}
```

### 성공/실패 코드 만들기
- 리뷰 도메인의 성공/실패 코드는 리뷰 → 예외 → 코드 아래에 작성된다
```java
@Getter
@RequiredArgsConstructor
public enum ReviewErrorCode implements BaseErrorCode {

    NOT_FOUND(HttpStatus.NOT_FOUND,
            "REVIEW404_1",
            "해당 리뷰를 찾을 수 없습니다."),
    ;

    private final HttpStatus status;
    private final String code;
    private final String message;
}
```

```java
@Getter
@RequiredArgsConstructor
public enum ReviewSuccessCode implements BaseSuccessCode {

    FOUND(HttpStatus.OK,
            "REVIEW200_1",
            "해당하는 리뷰를 성공적으로 찾았습니다."),
    ;

    private final HttpStatus status;
    private final String code;
    private final String message;
}
```

### 예외 만들기
- 리뷰 도메인의 예외는 리뷰 → 예외 아래에 작성
- 전역 예외 클래스를 상속하여 만들어진다
```java
public class ReviewException extends GeneralException {
    public ReviewException(BaseErrorCode code) {
        super(code);
    }
}
```

### 컨트롤러 만들기
```java
// 가게의 리뷰 목록 조회
@GetMapping("/reviews")
public ApiResponse<ReviewResDTO.ReviewPreViewListDTO> getReviews(){
    ReviewSuccessCode code = ReviewSuccessCode.FOUND;
    return ApiResponse.onSuccess(code, null);
}
```

## 2. API 명세서 작성하기
- 아직 모든 API가 개발되지 않더라도 명세서를 작성한다.
- 이를 통해 프론트의 병목을 최소로 할 수 있다.
- 만약 API 명세서를 스웨거로 끝내고 싶다면 빠른 배포가 필요하다.
- 스웨거를 사용하여 API 명세 문서를 자동으로 생성하고 싶다면 원하는 컨트롤러 메소드 위에 어노테이션을 사용할 수 있다.
```java
// 가게의 리뷰 목록 조회
@Operation(
        summary = "가게의 리뷰 목록 조회 API By 마크 (개발 중)",
        description = "특정 가게의 리뷰를 모두 조회합니다. 페이지네이션으로 제공합니다."
)
@ApiResponses({
        @io.swagger.v3.oas.annotations.responses.ApiResponse(responseCode = "200", description = "성공"),
        @io.swagger.v3.oas.annotations.responses.ApiResponse(responseCode = "400", description = "실패")
})
@GetMapping("/reviews")
public ApiResponse<ReviewResDTO.ReviewPreViewListDTO> getReviews(){
    ReviewSuccessCode code = ReviewSuccessCode.FOUND;
    return ApiResponse.onSuccess(code, null);
}
```
- `@Operation` : 해당 API 메소드(엔드포인트)에 대한 요약 정보와 상세 설명을 제공
- `@ApiResponse` : 해당 API 호출이 발생시킬 수 있는 응답 코드(성공/실패)와 메시지를 명시
- `@ApiResponses` : 위의 @ApiResponse를 묶음

### 로직과 명세서 설정을 분리
- 위의 경우 실제 컨트롤러의 구현과 명세서 설정이 뒤섞여 컨트롤러의 로직을 알아보기 힘들어진다.
- 따라서 ReviewControllerDocs라는 인터페스에서 스웨어 관련 어노테이션들을 정의하고 이를 상속받은 ReviewController를 만들어 이곳에서 로직을 구현한다.

## 3. 서비스 로직 작성
가장 우선적으로는 구현하려는 메소드의 흐름을 생각한다. 리뷰 조회 로직의 경우 아래와 같이 구상할 수 있다.
1. 가게를 가져온다 → 가게가 존재하지 않는다면 예외를 터트린다
2. 가게에 맞는 리뷰를 가져온다 → offset 페이징을 사용한다
3. 결과를 응답 DTO로 반환한다 → 이때 DTO의 생성은 컨버터를 이용한다

### 서비스 인터페이스 정의
- 조회를 위한 것이므로 리뷰 → 서비스 → 쿼리 아래에 작성한다
- 인터페이스 ReviewQueryService를 생성한다
- 위 인터페이스를 상속받는 구현 클래스 ReviewQueryServiceImpl를 생성한다
- 리뷰 조회를 위한 메소드 `List<Review> searchReview(String filter, String type) thows Exception;`을 정의한다.

### 컨트롤러와 연결
- 컨트롤러는 프론트로부터 받은 정보를 서비스 메소드로 넘겨준다.
- 서비스는 서비스 로직을 실행 후 해당 결과를 DTO로 묶어 리턴한다.
- 컨트롤러는 해당 결과를 성공 ApiResponese로 감싸 프론트로 전달한다.

### 서비스 작성
- 레포지토리 클래스를 이용하여 필요한 서비스 로직을 수행한다.
- 이때 컨트롤러에서 가게의 이름과 페이징 정보를 받아온다.

#### JPA의 페이지와 슬라이스
- 우리가 사용하는 `JpaRepository<T,ID>` 인터페이스는 `ListCrudRepository<T,ID>` 및 `ListPagingAndSortingRepository<T,ID>`를 상속한다.
- 즉 기본적으로 리스트 반환 및 페이징/정렬 기능을 포함하는 것이다.
- `ListPagingAndSortingRepository<T,ID>`는 정렬 조건(Sort)을 인자로 받아 `List<T>` 형태로 반환받는 것이 가능합니다.
- 또한 `ListPagingAndSortingRepository<T,ID>`은 `PagingAndSortingRepository<T,ID>`를 상속하고 있으므로 `Page<T>`를 반환하는 `findAll(Pageable pageable)` 등의 메서드를 사용할 수 있다.
- `Page<T>`는 `Slice<T>`를 상속하므로 Slice에서 제공하는 메서드를 모두 사용할 수 있다. 여기에 전체 요소 수(`getTotalElements()`), 전체 페이지 수(`getTotalPages()`) 등의 추가 메서드를 제공
- `PageRequest`는 페이징이나 슬라이싱 요청을 위해 필요한 정보(페이지 번호(page), 한 페이지 크기(size), 정렬 조건(Sort) 등)를 담는 `Pageable` 구현 객체이다.

#### Page
페이지 인터페이스는 아래와 같이 구현된다.
```java
public interface Page<T> extends Slice<T> {
    // 전체 페이지 개수
    int getTotalPages();
    // 전체 요소 개수
    long getTotalElements();
    // / 변환기
    <U> Page<U> map(Function<? super T, ? extends U> converter); // 변환기
}
```
페이징에서는 총 페이지 수가 중요하다. Page<T> 타입은 count 쿼리를 포함하는 페이징으로 카운트 쿼리가 자동으로 생성되어 함께 나간다.

#### Slice
- 페이지에서는 전체  데이터의 개수 즉, count가 필요하다.
- 그러나 슬라이스를 사용하는 경우 별도로 카운트 쿼리가 실행되지 않는다.
- 따라서 전체 페이지의 개수와 전체 엔티티의 개수를 알 수 없지만 불필요한 쿼리를 날리지 않아 성능 낭비가 발행하지 않는다.
- 모바일 UI등에서 사용되는 무한 스크롤을 구현할 때는 전체 페이지의 개수가 굳이 필요하지 않다. 다음 페이지가 존재하는지만을 확인하면 된다. 이럴 때 슬라이스를 사용하면 적합하다.

#### Slice의 다음 존재 여부 판단
- 그럼 슬라이스는 어떻게 다음 페이지가 존재하는지를 판단할까?
- 반환타입이 슬라이스인 레포지토리 메소드에 페이지 사이즈를 10으로 설정한 Pageable 객체를 전달하면 JPA는 거기에 1을 더해 11개의 데이터를 쿼리한다.
- 이를 통해 다음 쿼리의 존재 여부를 알 수 있다.

#### JPA에서의 페이징 사용
- 서비스 클래스에서 PageRequest를 생성한다 → 페이징할 정보 Offset, Limit을 정의
- Repository로 생성한 PageRequest을 넘겨준다.

#### 페이지 인덱스
- PA의 PageRequest.of(page, size)에서 page는 0부터 시작한다.
- 그러나 보통 프론트엔드나 클라이언트는 페이지를 1부터 요청하는 경우가 많다.
- 이를 위해 컨트롤러에서 page에 -1을 해준 후 서비스로 넘겨주는 것이 좋다.

### 컨버터 제작
- DB에서 조회한 객체를 DTO로 변환하기 위한 컨버터를 작성한다. 
- 컨버터는 레포지토리에서 전달받은 Page<Review>을 형식에 맞춰 DTO로 변환한다.
- 단 이 경우 review.getMember().getName() 부분에서 N+1 문제가 발생할 가능성이 높다 → 이는 객체 그래프 탐색으로 이어진다
```java
public class ReviewConverter {

    // result -> DTO
    // 아래의 ReviewPreViewDTO를 이용해 Page<Review> 전체를 DTO로 변환한다
    public static ReviewResDTO.ReviewPreViewListDTO toReviewPreviewListDTO(
            Page<Review> result
    ){
        return ReviewResDTO.ReviewPreViewListDTO.builder()
                .reviewList(result.getContent().stream()
                        .map(ReviewConverter::toReviewPreviewDTO)
                        .toList()
                )
                .listSize(result.getSize())
                .totalPage(result.getTotalPages())
                .totalElements(result.getTotalElements())
                .isFirst(result.isFirst())
                .isLast(result.isLast())
                .build();
    }
   
    // 리뷰를 받아 ReviewPreViewDTO를 생성한다
    public static ReviewResDTO.ReviewPreViewDTO toReviewPreviewDTO(
            Review review
    ){
        return ReviewResDTO.ReviewPreViewDTO.builder()
                .ownerNickname(review.getMember().getName())
                .score(review.getStar())
                .body(review.getContent())
                .createdAt(LocalDate.from(review.getCreatedAt()))
                .build();
    }
}
```

### 최종 서비스 메소드
```java
@Override
public ReviewResDTO.ReviewPreViewListDTO findReview(
        String storeName,
        Integer page
){
    // 가게 가져오기 (없다면 예외)
    Store store = storeRepository.findByName(storeName)
            .orElseThrow(() -> new StoreException(StoreErrorCode.NOT_FOUND));

    // 가게에 맞는 리뷰 가져오기 (Offset 페이징)
    PageRequest pageRequest = PageRequest.of(page, 5);
    Page<Review> result = reviewRepository.findAllByStore(store, pageRequest);
    
    // 결과를 응답 DTO로 변환 (컨버터 이용)
    return ReviewConverter.toReviewPreviewListDTO(result);
}
```

<br>

# 🎯핵심 키워드
## 객체 그래프 탐색
객체 그래프 탐색은 어떤 객체를 기준으로 그 객체와 연관된(참조된) 다른 객체들을 찾아가는 과정을 말한다

### 객체 그래프 탐색의 개념
객체 지향 프로그래밍에서 객체들은 서로 참조(Reference)를 통해 연결되어 있습니다. 이 연결 구조가 마치 거미줄(Graph) 같다고 하여 이를 객체 그래프라고 부른다.
- 점(Node) : 객체 (예: `Review`, `Member`, `Store`)
- 선(Edge) : 참조 관계 (예: `Review`는 `Member`를 알고 있음)
위의 코드에서 객체 그래프 탐색에 해당하는 부분은 아래와 같다.
```java
review.getMember().getName() 

// 1. review 객체에서 (시작)
// 2. getMember()를 통해 연결된 Member 객체로 이동 (탐색)
// 3. Member 객체의 getName()을 호출 (값 획득)
```
이처럼 `.`(점)을 찍어서 연관된 객체로 계속 들어가는 행위를 객체 그래프 탐색이라 할 수 있다.

### ⚠️ JPA에서의 주의점
객체 그래프 탐색을 주의하여야 하는 이유는 성능 때문이다.
1.  지연 로딩 (Lazy Loading): 보통 JPA에서 `Review`와 `Member`는 `ManyToOne` 관계로 성능을 위해 명시적으로 지연 로딩(LAZY)을 설정한다.
2.  탐색 시 쿼리 발생 : `ReviewConverter`에서 `review.getMember().getName()`을 호출하며 객체 그래프를 탐색하는 순간, JPA는 비워뒀던 멤버 정보를 채우기 위해 DB에 SELECT 쿼리를 추가로 날린다.
3.  N+1 문제 : 바로 이 지점에서 N+1 문제가 발생한다. 만약 10개의 리뷰가 존재한다고 가정해보자.
  - 리뷰 10개를 가져오기 위한 리뷰 쿼리 1개 
  - 각 리뷰마다 작성자의 이름을 알아내기 위해 추가적인 쿼리가 발생 → 10개 
  - 하나의 쿼리를 위해 10개의 쿼리가 추가적으로 발생한다
이를 해결하기 위해 `@Query` 어노테이션을 붙여 패치 조인을 위한 JPQL문을 직접 작성하거나 @EntityGraph 어노테이션을 사용할 수 있다.

