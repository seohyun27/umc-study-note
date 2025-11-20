# 회원가입 API 만들기
## DTO, Enum 생성
```java
📁com.example.umc9th
└─ 📁domain
    └─ 📁member
         ├─ 📁controller
         ├─ 📁converter
         ├─ 📁dto
         │   ├─ 📁req
         │   │    └─ MemberReqDto
         │   └─ 📁res
         │        └─ MemberResDto
         ├─ 📁exception
         │    ├─ 📁code
         │    │  ├─ (E) MemberErrorCode
         │    │  └─ (E) MemberSuccessCode
         │    └─ MemberException
         └─ 📁service
```

### DTO
#### 멤버 요청 DTO
```java
public class MemberReqDTO {

    public record JoinDTO(
            String name,
            Gender gender,
            LocalDate birth,
            Address address,
            String specAddress,
            List<Long> preferCategory
    ){}
}
```

#### 멤버 응답 DTO
응답 DTO의 경우 빌더와 컨버터 클래스를 사용하게 된다.
```java
public class MemberResDTO {

    @Builder
    public record JoinDTO(
            Long memberId,
            LocalDateTime createAt
    ){}
}
```

### 예외처리 Enum 생성
#### 성공 Enum
전역 성공 인터페이스를 상속받아 만들어진다.
```java
@Getter
@AllArgsConstructor
public enum MemberSuccessCode implements BaseSuccessCode {
    
    FOUND(HttpStatus.OK,
            "MEMBER200_1",
            "성공적으로 사용자를 조회했습니다."),
    ;
    
    // 회원가입 성공 시 201번 생성 코드로 돌려줌
    CREATED(HttpStatus.CREATED,
            "MEMBER201_1",
            "성공적으로 회원가입이 완료되었습니다."),
    ;
    
    private final HttpStatus status;
    private final String code;
    private final String message;
}
```

#### 실패 Enum
전역 실패 인터페이스를 상속받아 만들어진다.
```java
@Getter
@AllArgsConstructor
public enum MemberErrorCode implements BaseErrorCode {
    
    // 회원가입 실패는 여러 경우로 발생할 수 있음
    DUPLICATE_MEMBER(HttpStatus.BAD_REQUEST,
            "MEMBER400_1",
            "이미 존재하는 회원입니다."),

    INVALID_SIGNUP_REQUEST(HttpStatus.BAD_REQUEST,
            "MEMBER400_2",
            "회원가입 요청 값이 올바르지 않습니다."),
    ;
    
    NOT_FOUND(HttpStatus.NOT_FOUND,
            "MEMBER404_1",
            "해당 사용자를 찾지 못했습니다."),
    ;
    
    private final HttpStatus status;
    private final String code;
    private final String message;
}
```

#### 멤버 예외 클래스
전역 예외 클래스를 상속받아 만들어진다.
```java
public class MemberException extends GeneralException {
    public MemberException(BaseErrorCode code) {
        super(code);    // 생성자에서 전역 예외 처리로 code를 전달
    }
}
```

### 컨버터 클래스 작성
public static 메소드로 객체 → DTO 변환 메소드를 구현
```java
public class MemberConverter {

    // Entity → DTO
    public static MemberResDTO.JoinDTO toJoinDTO(Member member){
        return MemberResDTO.JoinDTO.builder()
                .memberId(member.getId())
                .createAt(member.getCreatedAt())
                .build();
    }

    // DTO → Entity
    public static Member toMember(MemberReqDTO.JoinDTO dto){
        return Member.builder()
                .name(dto.name())
                .birth(dto.birth())
                .address(dto.address())
                .detailAddress(dto.specAddress())
                .gender(dto.gender())
                .build();
    }
}
```

## 레파지토리, 서비스, 컨트롤러 작성
### 레파지토리
JPA 레파지토리를 상속받는 형태로 구현.
```java
public interface MemberRepository extends JpaRepository<Member, Long> {
}
```

### 서비스
서비스 인터페이스인 MemberCommandService를 상속받는 구현체로 작성한다.
#### 서비스 인터페이스
구현체에서 구현하고자 하는 메소드를 미리 정의한다.
```java
public interface MemberCommandService{
    MemberResDTO.JoinDTO singup(MemberReqDTO.Join dto);
}
```

#### 서비스 구현체
인터페이스에서 정의한 메소드를 @Override를 이용해 덮어쓴다.
```java
@Service
@RequiredArgsConstructor
public class MemberCommandServiceImpl implements MemberCommandService{

    private final MemberRepository memberRepository;

    @Override
    public MemberResDTO.JoinDTO signup(MemberReqDTO.JoinDTO dto){
        // 로직 생략
        return null;
    }
}
```

### 컨트롤러
예외 없이 성공했을 때의 경우를 컨트롤러에 정의한다. 예외의 경우 @RestControllerAdvice에서 공통적으로 처리된다.
```java
@RestController
@RequiredArgsConstructor
public class MemberController {

    private final MemberCommandService memberCommandService;

    @PostMapping("/sign-up")
    public ApiResponse<MemberResDTO.JoinDTO> signUp(
            @RequestBody MemberReqDTO.JoinDTO dto
    ){
        return ApiResponse.onSuccess(MemberSuccessCode.CREATED, memberCommandService.signup(dto));
    }
}
```
컨트롤러의 전체적인 흐름은 아래와 같다.
1. 회원가입을 위한 요청 DTO를 프론트의 요청 body에서 받아온다.
2. 해당 DTO를 서비스 클래스로 전달한다.
3. 서비스 클래스에서는 서비스 로직을 실행한 후 필요한 결과들을 응답 DTO에 넣어 컨트롤러로 리턴한다.
4. 컨트롤러는 서비스로부터 전달받은 응답 DTO를 응답 통일을 위한 API 클래스(ApiResponse)로 한 번 더 감싸 프론트로 전송한다. 

### 💡서비스 로직의 구성
앞서 생략했던 서비스 로직은 아래와 같이 구성할 수 있다. 이때 사용자 엔티티와 선호 음식 엔티티를 생성해 DB에 반영할 수 있다.
```java
@Service
@RequiredArgsConstructor
public class MemberCommandServiceImpl implements MemberCommandService{

    private final MemberRepository memberRepository;
    private final MemberFoodRepository memberFoodRepository;
    private final FoodRepository foodRepository;

    // 회원가입
    @Override
    @Transactional
    public MemberResDTO.JoinDTO signup(
            MemberReqDTO.JoinDTO dto
    ){
        // 사용자 생성
        Member member = MemberConverter.toMember(dto);
        // DB 적용
        memberRepository.save(member);
        
        // 선호 음식 존재 여부 확인
        if (dto.preferCategory().size() > 1){
            List<MemberFood> memberFoodList = new ArrayList<>();

            // 선호 음식 ID별 조회
            for (Long id : dto.preferCategory()){

                // 음식 존재 여부 검증
                Food food = foodRepository.findById(id)
                        .orElseThrow(() -> new FoodException(FoodErrorCode.NOT_FOUND));

                // MemberFood 엔티티 생성 (컨버터 사용해야 함)
                MemberFood memberFood = MemberFood.builder()
                        .member(member)
                        .food(food)
                        .build();

                // 사용자 - 음식 (선호 음식) 추가
                memberFoodList.add(memberFood);
            }

            // 모든 선호 음식 추가: DB 적용
            memberFoodRepository.saveAll(memberFoodList);
        }


        // 응답 DTO 생성
        return MemberConverter.toJoinDTO(member);
    }
}
```
- 여기서는 선호 음식 id별 조회를 for문을 이용해 구현했지만 stream()을 통해 구현하면 성능을 더 향상할 수 있다.
- DB에 조회 이외의 삽입, 수정, 삭제를 하는 경우 해당 메소드에 @Transactional을 붙여주는 것이 좋다.

<br/>

# Swagger와 어노테이션
## Swagger
### Swagger의 사용
- Swagger를 사용하면 매번 Postman을 사용할 필요 없이 개발한 API들을 확인하고 테스트할 수 있다.
- 프로젝트의 컨트롤러를 기반으로 API 명세 문서와 테스트를 위한 UI 화면을 제공해준다.
- 스프링 부트 프로젝트에 Swagger(OpenAPI) 라이브러리를 추가하면 해당 기능들을 사용할 수 있다.
- build.gradle에 라이브러리를 추가한 뒤 SwaggerConfig를 작성한다.
- 서버의 실행 수 브라우저에서 `http:\//localhost:8080/swagger-ui/index.html#/`를 열면 해당 기능들을 사용할 수 있다.

### SwaggerConfig
```java
@Configuration
public class SwaggerConfig {

    @Bean
    public OpenAPI swagger() {
        Info info = new Info().title("Project").description("Project Swagger").version("0.0.1");

        // JWT 토큰 헤더 방식
        String securityScheme = "JWT TOKEN";
        SecurityRequirement securityRequirement = new SecurityRequirement().addList(securityScheme);

        Components components = new Components()
                .addSecuritySchemes(securityScheme, new SecurityScheme()
                        .name(securityScheme)
                        .type(SecurityScheme.Type.HTTP)
                        .scheme("Bearer")
                        .bearerFormat("JWT"));

        return new OpenAPI()
                .info(info)
                .addServersItem(new Server().url("/"))
                .addSecurityItem(securityRequirement)
                .components(components);
    }
}
```

## 어노테이션을 통한 검증
```
📁com.example.umc9th
├─ 📁domain
│   └─ 📁food
└─ 📁global
    ├─ 📁annotaion
    │    └─ (I) ExistFoods
    ├─ 📁handler
    │   └─ handleMethodArgumentNotValidException
    └─ 📁validator
         └─ FoodExistValidator
```
+) `handleMethodArgumentNotValidException`의 위치는 임의로 작성된 위치이다. 일단 `GeneralExceptionAdvice`가 있는 파일에 같이 넣어두긴 했는데 조금 더 고민해보면 좋을 것 같다.

### 기존의 검증 방식
지금까지는 Food가 존재하는지 확인하는 과정을 서비스에서 직접 실행했다.
```java
Food food = foodRepository.findById(foodId)
                .orElseThrow(() -> new NotFoundException("해당 음식이 없습니다."));
```
그러나 이 경우 서비스 코드가 검증 코드로 복잡해지고 DTO와 역할이 뒤섞인다. 즉, DTO 레벨에서의 검증이 필요하다.

### 커스텀 어노테이션
- 이를 확인하기 위해 커스텀 어노테이션 기능을 활용할 수 있다.
- 입력값 자체가 올바른지 체크하는 과정은 컨트롤러 호출시 스프링이 자동으로 하도록 넘긴다.
- 서비스는 비즈니스 로직만을 담당할 수 있게 된다!

### build.gradle
build.gradle에 라이브러리를 추가한다.
```java
implementation 'org.springframework.boot:spring-boot-starter-validation'
```

### ExistFoods
@Constraint : 사용자가 커스텀 어노테이션을 통해 검증을 할 수 있도록 제공하는 어노테이션 
validatedBy 파라미터 : 해당 파라미터에 지정된 클래스를 통해 @ExistFoods가 붙은 대상의 검증이 이루어진다.
```java
@Documented
@Constraint(validatedBy = FoodExistValidator.class)
@Target( { ElementType.METHOD, ElementType.FIELD, ElementType.PARAMETER })
@Retention(RetentionPolicy.RUNTIME)
public @interface ExistFoods {
    //여기서 디폴트 메시지를 설정합니다.
    String message() default "해당 음식이 존재하지 않습니다.";  
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

### FoodExistValidator
@ExistFoods가 붙은 대상을 검증하기 위한 클래스
```java
@Component
@RequiredArgsConstructor
public class FoodExistValidator implements ConstraintValidator<ExistFoods, List<Long>> {

    private final FoodRepository foodRepository;
    
    @Override
    public boolean isValid(List<Long> values, ConstraintValidatorContext context) {
        boolean isValid = values.stream()
                .allMatch(value -> foodRepository.existsById(value));

        if (!isValid) {
		        // 이 부분에서 아까 디폴트 메시지를 초기화 시키고, 새로운 메시지로 덮어씌우게 됩니다.
            context.disableDefaultConstraintViolation();
            context.buildConstraintViolationWithTemplate(FoodErrorCode.NOT_FOUND.getMessage()).addConstraintViolation();
        }

        return isValid;

    }
}
```
만약 검증 실패 시 DTO에 붙인 `@ExistFoods`의 message를 그대로 쓰고 싶다면 `FoodErrorCode.NOT_FOUND.getMessage()` 대신 `constraintAnnotation.message()`처럼 에러 메시지를 가져올 수도 있다.

### 어노테이션을 통한 검증
#### DTO 처리
요청 DTO 안에서 검증이 필요한 어트리뷰트에 어노테이션을 작성한다.
```java
public class MemberReqDTO {

    public record JoinDTO(
            String name,
            Gender gender,
            LocalDate birth,
            Address address,
            String specAddress,
            @ExistFoods List<Long> preferCategory // Food가 존재하는지 검증
    ){}
}
```

#### 컨트롤러 처리
컨트롤러에서 받는 요청 DTO에 검증 어노테이션(@Vaild)을 붙인다.
```java
@PostMapping("/sign-up")
public ApiResponse<MemberResDTO.JoinDTO> signUp(
        @RequestBody @Valid MemberReqDTO.JoinDTO dto
){
    return ApiResponse.onSuccess(MemberSuccessCode.CREATED, memberCommandService.signup(dto));
}
```

### 검증에서 발생한 예외처리
검증이 실패하면 MethodArgumentNotValidException을 발생시키고 그 안에 어떤 요소가 어떤 검증에 실패했는지를 담아준다. 이걸 이용해서 HandlerAdvice를 작성할 수 있다.
```java
// 컨트롤러 메서드에서 @Valid 어노테이션을 사용하여 DTO의 유효성 검사를 수행
@ExceptionHandler(MethodArgumentNotValidException.class)
protected ResponseEntity<ApiResponse<Map<String, String>>> handleMethodArgumentNotValidException(
        MethodArgumentNotValidException ex
) {
    // 검사에 실패한 필드와 그에 대한 메시지를 저장하는 Map
    Map<String, String> errors = new HashMap<>();
    ex.getBindingResult().getFieldErrors().forEach(error ->
            errors.put(error.getField(), error.getDefaultMessage())
    );

    GeneralErrorCode code = GeneralErrorCode.VALID_FAIL;
    ApiResponse<Map<String, String>> errorResponse = ApiResponse.onFailure(code, errors);

    // 에러 코드, 메시지와 함께 errors를 반환
    return ResponseEntity.status(code.getStatus()).body(errorResponse);
}
```

<br>

# 🎯핵심 키워드
## 1️⃣ Java Exception 종류

Java에서 예외는 프로그램 실행 중 발생하는 예기치 못한 상황을 처리하기 위해 사용된다. Checked Exception과 Unchecked Exception으로 나눌 수 있다.

### ① Checked Exception
- 컴파일 시점에 반드시 처리해야 하는 예외
- try-catch 또는 throws 키워드로 처리하지 않으면 컴파일 에러 발생
- 핸들러에서 변환하면 Checked도 프론트로 전달이 가능하다
- 대표적인 예:
  - IOException – 파일 입출력, 네트워크 등 I/O 작업 중 발생
  - SQLException – DB 작업 중 발생
  - ClassNotFoundException – 동적으로 클래스를 로드할 때 발생

```java
try {
    FileReader reader = new FileReader("file.txt");
} catch (IOException e) {
    e.printStackTrace();
}
```

### ② Unchecked Exception
- 런타임에 발생 
- 처리하지 않아도 컴파일을 통과한다 
- 주로 프로그래밍 오류를 나타낸다
- RuntimeException을 상속한다 
- 프론트로 보내는 예외 대부분은 RuntimeException 계열이다
- 대표적인 예:
  - `NullPointerException` – null 객체 참조 시
  - `IllegalArgumentException` – 잘못된 매개변수 전달 시
  - `ArrayIndexOutOfBoundsException` – 배열 범위 초과 접근 시

```java
String str = null;
str.length(); // NullPointerException 발생
```

### ③ Error
- Exception과 달리 핸들러에서 처리하지 않는다
- 일반적으로 try-catch로 처리하지 않고 애플리케이션이 종료된다


## 2️⃣ @Valid
- DTO나 메서드 파라미터에 붙여서 유효성 검사를 수행하는 어노테이션
- 스프링에서는 @Valid와 함께 @RequestBody를 쓰면, 컨트롤러로 들어오는 요청 객체를 검증할 수 있음
- DTO 클래스 안에는 `@NotNull`, `@Size`, `@Min`, `@Max` 같은 제약 어노테이션이 붙어 있어야 함
- `@Validated`와 `@Valid`의 차이
  - `@Valid` : 단일 DTO 검증
  - `@Validated` : 그룹별 검증, 파라미터 검증 가능

```java
public class UserDTO {
    @NotNull(message = "이름은 필수입니다")
    private String name;ㄴㄴ

    @Min(value = 18, message = "나이는 18세 이상이어야 합니다")
    private int age;
}
```

```java
@PostMapping("/users")
public ResponseEntity<String> createUser(@RequestBody @Valid UserDTO dto) {
    return ResponseEntity.ok("성공");
}
```

- 요청값이 제약 조건(NotNull, Min 등)을 만족하지 않으면 스프링은 MethodArgumentNotValidException을 발생시킨다.
- 이를 다루는 핸들러 클래스를 작성할 수 있다.