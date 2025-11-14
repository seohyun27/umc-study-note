# API 응답 통일

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
│   └─ 📁member
│        ├─ 📁controller
│        ├─ 📁converter
│        ├─ 📁dto
│        │   ├─ 📁req
│        │   │    └─ MemberReqDto
│        │   └─ 📁res
│        │        └─ MemberResDto
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
         └─ ApiResponse
```
- 추가적으로 서비스를 작성할 때는 아래 두 가지 파일에 나눠 작성한다.
    - Query : GET 요청에 대한 비즈니스 로직들
    - Command : 이외 요청에 대한 비즈니스 로직들

### 📁code
code는 응답 코드를 담는 역할을 수행한다. 특히 베이스 인터페이스에서는 최소한의 구현 메소드를 정한다.
```java
public interface BaseErrorCode {

    HttpStatus getStatus();
    String getCode();
    String getMessage();
}
```
베이스 인터페이스를 구현한 이넘 구현 클래스에서는 상태 코스, 상세 코드, 메시지를 정한다.
```java
@Getter // 롬복을 통해 인터페이스에서 정의한 게터들을 실제로 구현함
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
- 응답을 위한 공통 API 클래스는 아래와 같이 작성할 수 있다.
- 이때 result에 어떤 값이 담기게 될 지 알 수 없으므로 제네릭 타입으로 작성한다.
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
해당 과정들을 끝내면 API 응답을 위한 최소한의 통일이 마무리 된다.

<br/>

## DTO
- 응답 API의 결과로 담기 위한 정보들과 프론트에서 전송받는 정보들을 포장하기 위해 사용되는 DTO들을 정의한다.

### TestResDTO
- DTO들은 MemberResDto, ReviewResDto 등등 큰 카테고리에서 public 클래스를 만들고 그 안에서 세부적으로 static 클래스를 정의해 사용하는 것이 좋다.
- DTO는 아주 많은 곳에서 사용되기 때문에 매번 클래스를 작성하는 것보다 하나의 클래스 내에서 static 클래스를 작성하는 것이 매우 효율적이다.
- 또한 MemberResDto 클래스 안에서 선언되는 것이므로 모든 DTO에 일일이 Member라는 도메인의 이름을 입력해줄 필요가 없다.
- 사용 시에도 MemberResDto.changeNickname처럼 외부 클래스 이름과 내부 클래스 이름을 함께 적어 사용이 가능하다.
```java
public class MemberResDto {   // 큰 묶음으로 클래스 생성

    @Builder
    @Getter
    public static class ChangeNickname {   // 내부에서 public static으로 선언한 뒤 사용 
        private String nickname;
    }
}
```
- 응답 DTO의 경우, 백앤드 측에서 내용을 채워야 하므로 빌터 패턴을 사용한다.
- 요청 DTO의 경우, 프론트에서 측에서 만들어진 객체의 정보를 한 번에 받아오는 것이므로 빌더 패턴을 거의 사용하지 않는다.

### Converter
- 객체를 DTO로 바꾸는 클래스.
- 여러 객체와 필요한 정보를 받아 필요한 정보만을 걸러 DTO로 포장해준다.
```java
public class MemberConverter {
    
    // 멤버 객체를 받아 관련 DTO로 변환해줌
    public static MemberResDto.ChangeNickname toNicknameDTO(Member member) {
        return MemberResDto.ChangeNickname.builder()
                .nickname(member.getNickname())
                .build();
    }
}
```

<br/>

## 컨트롤러

### Controller
- `@Controller` : 전통적인 MVC에서 View(화면, HTML)를 반환하기 위해 사용
- `@RestController`
  - RESTful API를 구축할 때 사용한다.
  - `@Controller` + `@ResponseBody`가 합쳐진 어노테이션이다.
  - 데이터(JSON)를 반환하는 것이 주 목적이다.
```java
@RestController
@RequiredArgsConstructor
@RequestMapping("/members")    // 컨트롤러 전체 경로 설정
public class TestController {

    @PatchMapping("/me")
    public ApiResponse<MemberResDto.ChangeNickname> changeNickname(
        @RequestBody @Valid MemberReqDto.ChangeNickname req) throws Exception {
        /**
        * 서비스 클래스에서 닉네임 변경 로직이 이루어짐
        * MemberConverter.toNicknameDTO(member)를 반환하게 됨!!
        */
        MemberResDto.ChangeNickname res = memberService.changeNickname(req);    // 서비스 클래스 실행 후 DTO 반환
        GeneralSuccessCode code = GeneralSuccessCode.OK;                        // 응답 코드 정의
        return ApiResponse.onSuccess(
                code,
                res
        );
    }
}
```
최종적으로 프론트에서 PATCH /members/me 경로로 요청을 보내면 성공 메시지와 함께 변경된 닉네임이 돌아오게 된다.

### 💡 실제 반환 타입은?
- 실무에서는 HTTP 응답 자체를 제어하는(HTTP 상태 코드, 헤더 등) ResponseEntity<T>를 함께 사용하는 경우가 많다.
- 가장 보편적인 반환 타입은 ResponseEntity<ApiResponse<T>>이다.
- 즉, `Entity` → `DTO` → `ApiResponse` → `ResponseEntity`로 이어지는 흐름이 된다!
- 귀찮은 듯 보여도 이 모든 과정을 거쳐야 예측 가능하고 안정적인 API를 만들 수 있다.
  - HTTP 상태 코드(200, 400 등) : 기계/인프라를 위한 것이다. 브라우저, 캐시, 모니터링 툴, 네트워크 장비 등이 사용한다.
  - ApiResponse 내부 코드 : 사람/어플리케이션의 로직을 위한 것이다. 프론트엔드 개발자, 사용자에게 보여지게 될 메시지에서 사용된다.

<br/>

# 예외 처리
## 예외 처리
- 프로젝트 레벨의 예외와 도메인 레벨의 예외를 별개로 처리하는 것이 좋다.
- 응답이 통일된 예외를 만들기 위해서는 모든 도메인 예외를 모아줄 객체가 필요하다.
- 그것을 에러 헨들러라 부른다.
- api와 예외처리에 관련된 전체적인 패키지 구조는 아래와 같다.
```
📁com.example.umc9th
├─ 📁domain
│   └─ 📁review
│        ├─ 📁controller
│        ├─ 📁converter
│        ├─ 📁dto
│        ├─ 📁exception
│        └─ 📁service
│            ├─ 📁command
│            │    ├─ (I) ReviewCommandService
│            │    └─ ReviewCommandServiceImpl
│            └─ 📁query
│                 ├─ (I) ReviewQueryService
│                 └─ ReviewQueryServiceImpl
└─ 📁global
    └─ 📁apiPayload
         ├─ 📁code
         │   ├─ (I) BaseErrorCode
         │   └─ (E) GeneralErrorCode
         ├─ 📁exception
         ├─ 📁handler
         │   └─ GeneralExceptionAdvice
         └─ ApiResponse
```

<br/>

## 에러 핸들러
- 수많은 위치에서 터진 예외들의 응답을 통일할 통일 객체가 필요하다
- 이를 예외 핸들러라 부른다.
- 에러 핸들러 코드의 작성은 아래와 같다.

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
- 기본적인 원리는 정의한 예외를 감지하고 미리 정의해둔 에러 핸들러 로직을 실행하는 것이다.
- 따라서 도메인 예외는 프로젝트 예외를 상속하는 구조이고, 이러한 예외를 감지하는 것은 에러 핸들러의 역할이다.

### 에러 코드를 정의하는 Enum
- 에러 코드에 규칙을 정해두면 프론트엔드 개발자와 소통이 편해진다.
```java
_INTERNAL_SERVER_ERROR(HttpStatus.INTERNAL_SERVER_ERROR, "COMMON000", "서버 에러, 관리자에게 문의 바랍니다."),
```
- 위의 에러 코드에서 HttpStatus.INTERNAL_SERVER_ERROR만 보고도 오류의 큰 틀을 알 수 있다.
- 두 번째 인자인 code를 보고 더 자세한 오류를 알 수 있다.
- COMMON000 : common 에러
- MEMBER4001 : 멤버 관련 에러 + 400번대(클라이언트 오류) + 그 중에서도 1번 에러
- 해당 규칙을 적용에 아래와 같은 Enum을 구성할 수 있다.
```java
// Member Error
MEMBER_NOT_FOUND(HttpStatus.BAD_REQUEST, "MEMBER4001", "사용자가 없습니다."),
NICKNAME_NOT_EXIST(HttpStatus.BAD_REQUEST, "MEMBER4002", "닉네임은 필수 입니다."),

// Article Error
ARTICLE_NOT_FOUND(HttpStatus.NOT_FOUND, "ARTICLE4001", "게시글이 없습니다.");
```

<br/>

## 예외 처리의 테스트
- 컨트롤러 내에 전역 예외 처리기(@RestControllerAdvice)를 테스트하기 위한 경로를 하나 생성한다.
  - GET /temp/exception
  - 이는 내가 작성한 전역 예외 처리기가 제대로 작동하는지를 확인해보기 위한 테스트 경로이다.
- 즉, 예외처리가 제대로 되는지를 확인하고 싶다면 GET /temp/exception로 요청을 보내보면 된다.
- 이때 쿼리스트링 flag를 사용해 오로지 flag=1일 때만 예외가 터지도록 한다.
- 이는 해당 예외 처리 경로가 원래는 정상적인 경로이며 예외를 터트리면 해당 예외가 제대로 실패 api에 담겨 전달되는지를 확인하기 위함이다.


### 컨트롤러
`@RequestParam`는 쿼리 스트링을 받아오기 위한 어노테이션이다.
```java
@RestController
@RequestMapping("/temp")
@RequiredArgsConstructor
public class TempRestController {

		...
		
    // 전역 예외 처리기의 테스트 경로
    @GetMapping("/exception")
    public ApiResponse<TestResDTO.Exception> exception(@RequestParam Long flag) {
        return null;
    }
}
```

### 서비스
Service를 작성 할 때는 아래와 같은 규칙을 따른다.
1. 조회에 대한 요청은 query 폴더로, 이외의 요청은 command 폴더로 구분한다.
2. 서비스를 만들 경우 인터페이스를 먼저 두고 이를 구체화 한다. 
  - ex) TempQueryService 인터페이스를 먼저 만들고 이에 대한 Impl 구체화 클래스를 만든다.
3. 컨트롤러는 인터페이스를 의존하며 실제 인터페이스에 대한 구체화 클래스는 스프링부트의 의존성 주입을 이용한다!

#### 💡 인터페이스를 사용하는 이유
위의 경우 인터페이스와 구현체는 일대일로 매핑된다. 그럼 왜 인터페이스를 사용해야 할까?
1. 의존성 역전 원칙
  - 컨트롤러는 구체적인 구현 클래스가 어떤 식으로 동작하는지 알 필요가 없다.
  - 인터페이스에 정의된 인자값과 반환 타입에 대한 정보만을 알고 있다.
2. 테스트 용이성
  - 만약 컨트롤러가 구체적인 구현체에 직접 의존한다면 컨트롤러를 테스트할 때 Service가 의존하는 레파지토리나 DB가 모두 필요하게 된다.
  - 그러나 컨트롤러가 인터페이스에 의존하게 되면 테스트 시 가짜 구현체를 쉽게 주입할 수 있다.

#### 💡 하나의 인터페이스, 여러 구현체를 사용하는 경우
- 만약 PaymentService라는 결제 서비스 인터페이스가 있는 경우 결재 방식에 따라 구현체가 나뉠 수 있다.
  - KakaoPaymentService
  - TossPaymentService
  - NaverPaymentService
- 이때 컨트롤러에서 사용자의 결재 수단에 대한 정보가 들어오면 적절한 구현체를 동적으로 선택하여 사용할 수 있다.

#### 서비스 인터페이스
```java
public interface TestQueryService {
    void checkFlag(Long flag);
}
```

#### 서비스 실제 구현
checkFlag 메소드는 flag가 1인 경우에만 예외를 터트린다.
```java
@Service
@RequiredArgsConstructor
public class TestQueryServiceImpl implements TestQueryService {

    @Override
    public void checkFlag(Long flag){
        if (flag == 1){
            throw new TestException(TestErrorCode.TEST_EXCEPTION);
        }
    }
}
```

### 최종적으로 완성된 Get /temp/exception 경로 메소드
```java
@GetMapping("/exception")
public ApiResponse<TestResDTO.Exception> exception(
        @RequestParam Long flag
) {

    testQueryService.checkFlag(flag);   // flag가 1이라면 이 부분에서 예외가 발생한다!!

    GeneralSuccessCode code = GeneralSuccessCode.OK;
    return ApiResponse.onSuccess(code, TestConverter.toExceptionDTO("This is Test!"));
}
```

<br/>

## 🎯 핵심 키워드
### @RestControllerAdvice
- Spring에서 전역 예외를 처리하기 위한 어노테이션으로 모든 컨트롤러의 예외를 한 곳에서 처리할 수 있다.
- @ExceptionHandler와 함께 사용되어 공통된 에러 응답 형식을 제공한다.
- @ControllerAdvice + @ResponseBody의 조합과 동일한 기능을 한다.

### Lombok
- 반복되는 코드(생성자, getter/setter, toString 등)를 자동으로 생성해주는 라이브러리이다.
- @Getter, @Setter, @Builder, @AllArgsConstructor 등 어노테이션으로 코드 가독성을 높인다.
- 컴파일 시점에 실제 코드가 추가되므로 실행 속도에는 영향을 주지 않는다.

### DTO: public static class vs record
- public static class는 Lombok과 함께 쓰여 유연하고 계층적 구조를 만들기 좋다.
- record는 자바 16+에서 도입된 불변(immutable) 데이터 전달용 클래스이며 코드가 매우 간결하다.
- 그러나 record는 필드 수정이나 상속이 불가능하므로 단순 DTO에만 적합하다.

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

