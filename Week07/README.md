## API 응답 통일
- 모든 API의 응답이 통일되지 않으면 프론트의 입장에서 사용하기 힘들다.
- 따라서 프로젝트마다 API를 통일하여야 한다.
- 보통의 경우 아래 4가지의 정보를 포함하도록 작성한다.
  - isSuccess : Boolean 타입의 성공 여부
  - code : HTTP 상태 코드 외에 더 세부적인 결과
  - message : code에 추가적으로 어떤 결과인지를 알려주기 위해 사용
  - result : 응답으로 필요한 또 다른 json 정보
- 프로젝트 파일에서 구현 시 아래와 같은 구조를 사용할 수 있다. 이때 (I)는 인터페이스 (E)는 이넘 클래스를 의미한다.

```
📁com.example.umc9th
├─ 📁domain
│   └─ 📁review
│        ├─ 📁controller
│        ├─ 📁converter
│        ├─ 📁dto
│        │   ├─ 📁req
│        │   │    └─ ReviewReqDto
│        │   └─ 📁res
│        │        └─ ReviewResDto
│        └─ 📁service
│            ├─ 📁command
│            └─ 📁query
└─ 📁global
    └─ 📁apiPayload
         ├─ 📁code
         │   ├─ (I) BaseErrorCode
         │   ├─ (I) BaseSuccessCode
         │   ├─ (E) GeneralErrorCode
         │   └─ (E) GeneralSuccessCode
         ├─ ApiResponse
         └─ ApiRequest
```

### 📁code
code는 응답 코드를 담는 역할을 수행한다. 특히 베이스 인터페이스에서는 최소한의 구현 메소드를 정한다.
```java
public interface BaseErrorCode {

    HttpStatus getStatus();
    String getCode();
    String getMessage();
}
```
베이스 인터페이스를 구현한 구현 클래스에서는 상태 코스, 상세 코드, 메시지를 정한다.
```java
@Getter
@AllArgsConstructor
public enum GeneralErrorCode implements BaseErrorCode{

  BAD_REQUEST(HttpStatus.BAD_REQUEST,
                "COMMON400_1",
                "잘못된 요청입니다."),
  UNAUTHORIZED(HttpStatus.UNAUTHORIZED,
                "AUTH401_1",
                "인증이 필요합니다."),
  FORBIDDEN(HttpStatus.FORBIDDEN,
                "AUTH403_1",
                "요청이 거부되었습니다."),
  NOT_FOUND(HttpStatus.NOT_FOUND,
                "COMMON404_1",
                "요청한 리소스를 찾을 수 없습니다."),
  ;

  private final HttpStatus status;
  private final String code;
  private final String message;
 }
```

### ApiResponse
응답 성공을 위한 API는 아래와 같은 방식으로 작성할 수 있다. 이때 result에 어떤 값을 담게 될 지 알 수 없으므로 제네릭 타입으로 작성한다.
```java
@Getter
@AllArgsConstructor
@JsonPropertyOrder({"isSuccess", "code", "message", "result"})
public class ApiResponse<T> {

    @JsonProperty("isSuccess")
    private final Boolean isSuccess;

    @JsonProperty("code")
    private final String code;

    @JsonProperty("message")
    private final String message;

    @JsonProperty("result")
    private T result;

    // 성공한 경우 (결과 포함)
    public static <T> ApiResponse<T> onSuccess(BaseSuccessCode code, T result) {
        return new ApiResponse<>(true, code.getCode(), code.getMessage(), result);
    }

    // 실패한 경우 (결과 포함)
    public static <T> ApiResponse<T> onFailure(BaseErrorCode code, T result) {
        return new ApiResponse<>(false, code.getCode(), code.getMessage(), result);
    }
}
```
해당 과정을 끝내면 API 응답을 위한 최소한의 통일이 이루어진다.

### TestResDTO
- DTO들은 MemberResDto, ReviewResDto 등등 큰 카테고리에서 public 클래스를 만들고 그 안에서 세부적으로 static 클래스를 정의해 사용하는 것이 좋다.
- DTO는 아주 많은 곳에서 사용되기 때문에 매번 클래스 
```java
public class MemberResDto {   // 큰 묶음으로 클래스 생성

    @Builder
    @Getter
    public static class Testing {   // 내부에서 public static으로 선언한 뒤 사용 
        private String testing;
    }
}
```
- 응답 DTO의 경우 빌터 패턴을 사용한다.
- 요청 DTO의 경우 프론트에서 만든 객체를 받아오는 것이므로 빌더 패턴을 사용하지 않는다.

### Converter
- 객체를 DTO로 바꾸는 클래스
```java
public class TestConverter {
    
    // 객체 -> DTO
    public static TestResDTO.Testing toTestingDTO(
            String testing
    ) {
        return TestResDTO.Testing.builder()
                .testString(testing)
                .build();
    }
}
```

### Controller
```java
@RestController
@RequiredArgsConstructor
@RequestMapping("/temp")    // 컨트롤러 전체 경로 설정
public class TestController {

    @GetMapping("/test")
    public ApiResponse<TestResDTO.Testing> test() throws Exception {
        // 응답 코드 정의
        GeneralSuccessCode code = GeneralSuccessCode.OK;
        return ApiResponse.onSuccess(
                code,
                TestConverter.toTestingDTO("This is Test!")
        );
    }
}
```

### API 요청
최종적으로 /temp/test 경로로 요청을 보내면 성공 메시지가 돌아오게 된다.

### +) Service
- 서비스를 작성할 때는 아래 두 가지 파일에 나눠 작성한다.
- Query : GET 요청에 대한 비즈니스 로직들
- Command : 이외 요청에 대한 비즈니스 로직들

<br/>

## 예외 처리
- 프로젝트 레벨의 예외와 각 도메인 레벨의 예외를 별개로 처리하는 것이 좋다.
- 응답이 통일된 예외를 만들기 위해서는 모든 도메인 예외를 모아줄 객체가 필요하다.
- 그것을 에러 헨들러라 부른다.
- 전체적인 패키지 구조는 아래와 같다.
```
📁com.example.umc9th
├─ 📁domain
│   └─ 📁review
│        ├─ 📁controller
│        ├─ 📁converter
│        └─ 📁dto
└─ 📁global
    └─ 📁apiPayload
         ├─ 📁code
         │   ├─ (I) BaseErrorCode
         │   ├─ (I) BaseSuccessCode
         │   ├─ (E) GeneralErrorCode
         │   └─ (E) GeneralSuccessCode
         ├─ 📁exception
         ├─ 📁handler
         │   └─ GeneralExceptionAdvice
         ├─ ApiResponse
         └─ ApiRequest
```

- 예외처리를 위한 에러 헨들러 코드는 아래와 같다.

```java
@RestControllerAdvice
public class GeneralExceptionAdvice {

    // 애플리케이션에서 발생하는 커스텀 예외를 처리
    @ExceptionHandler(GeneralException.class)
    public ResponseEntity<ApiResponse<Void>> handleException(
            GeneralException ex
    ) {

        return ResponseEntity.status(ex.getCode().getStatus())
                .body(ApiResponse.onFailure(
                                ex.getCode(),
                                null
                        )
                );
    }

    // 그 외의 정의되지 않은 모든 예외 처리
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiResponse<String>> handleException(
            Exception ex
    ) {

        BaseErrorCode code = GeneralErrorCode.INTERNAL_SERVER_ERROR;
        return ResponseEntity.status(code.getStatus())
                .body(ApiResponse.onFailure(
                                code,
                                ex.getMessage()
                        )
                );
    }
}
```
- 기본적인 원리는 정의한 예외를 감지하고 미리 정의해둔 에러 핸들어 로직을 실행하는 것이다.
- 따라서 도메인 예외는 프로젝트 예외를 상속하는 구조이고, 이러한 예외를 감지하는 것은 에러 핸들러의 역할이다.

### 에러 코드를 정의하는 Enum
- 에러 코드에 규칙을 정해두면 프론트엔드 개발자와 소통이 편해진다.
```java
_INTERNAL_SERVER_ERROR(HttpStatus.INTERNAL_SERVER_ERROR, "COMMON000", "서버 에러, 관리자에게 문의 바랍니다."),
```
- 위의 에러 코드에서 HttpStatus.INTERNAL_SERVER_ERROR만 보고도 오류의 큰 틀을 알 수 있다.
- 두 번째 인자인 code를 보고 더 자세한 오류를 알 수 있다.
- COMMON000 : common 에러
- MEMBER4001 : 멤버 관련 에러 + 400번대(서버의 잘못) + 그 중에서도 1번 에러
- 해당 규칙을 적용에 아래와 같은 Enum을 구성할 수 있다.
```java
// Member Error
MEMBER_NOT_FOUND(HttpStatus.BAD_REQUEST, "MEMBER4001", "사용자가 없습니다."),
NICKNAME_NOT_EXIST(HttpStatus.BAD_REQUEST, "MEMBER4002", "닉네임은 필수 입니다."),

// Article Error
ARTICLE_NOT_FOUND(HttpStatus.NOT_FOUND, "ARTICLE4001", "게시글이 없습니다.");
```


<br/>

## 🎯 핵심 키워드
### @RestControllerAdvice
- Spring에서 전역 예외를 처리하기 위한 어노테이션으로, 모든 컨트롤러의 예외를 한 곳에서 처리할 수 있다.
- @ExceptionHandler와 함께 사용되어 공통된 에러 응답 형식을 제공한다.
- @ControllerAdvice + @ResponseBody의 조합과 동일한 기능을 한다.

### Lombok
- 반복되는 코드(생성자, getter/setter, toString 등)를 자동으로 생성해주는 라이브러리다.
- @Getter, @Setter, @Builder, @AllArgsConstructor 등 어노테이션으로 코드 가독성을 높인다.
- 컴파일 시점에 실제 코드가 추가되므로 실행 속도에는 영향을 주지 않는다.

### DTO: public static class vs record
- public static class는 Lombok과 함께 쓰여 유연하고 계층적 구조를 만들기 좋다.
- record는 자바 16+에서 도입된 불변(immutable) 데이터 전달용 클래스이며, 코드가 매우 간결하다.
- 단, record는 필드 수정이나 상속이 불가능하므로 단순 DTO에만 적합하다.

<br/>

## ✅ 미션 기록

### 1️⃣ 
```java
```

### 2️⃣ 
```java
```

### 3️⃣ 
```java
```

