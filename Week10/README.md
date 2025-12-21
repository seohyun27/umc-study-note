# 로그인 및 회원 가입
## 로그인 및 회원 가입의 구현
- 로그인 및 회원 가입을 위한 기본적인 보안 설정은 Spring Security를 활용한다.
- 의존성 추가 방식은 아래와 같다.
```java
dependencies {
...
    // Security
    implementation 'org.springframework.boot:spring-boot-starter-security'
	testImplementation 'org.springframework.security:spring-security-test'
...
}
```
- 해당 의존성을 추가하면 Spring Security는 Auto-Configuration을 통해 프로젝트에 기본적인 보안 기능을 적용한다. 별도의 설정이나 코드의 작성이 없더라도 기본적인 보안이 활성화된다.
- 실행 로그에서 제공되는 기본 계정을 통해 로그인할 수 있다.
- 실제 어플리케이션에서는 사용자 정의 계정과 권한 부여(Authorization), 추가적인 보안 기능이 필요하다.
- 이를 프로젝트 요구에 맞게 커스터마이징하는 작업이 필수적이다.

<br>

## Spring Security
- Spring Security는 Spring 기반 애플리케이션의 보안을 담당하는 프레임워크이다.
- 주로 인증과 권한 부여를 제공한다.
  - 누가 들어오는지를 확인 : 인증
  - 들어온 사람이 어디에 갈 수 있는지를 결정 : 인가
  - 위험한 상황으로부터 보호 : 보안 위협 방어

### 1. 인증
- 인증은 사용자가 제공한 크리텐셜(아이디와 비밀번호)를 통해 신원을 검증하는 과정이다.
- interface `Authentication` : 인증된 사용자의 정보를 담고 있다.
- `getPrincipal()` : 사용자의 식별 정보를 반환
- `getAuthorities()` : 사용자의 권한 정보를 반환

### 2. 인가
- 어플리케이션 접근 → 인증 → 인가의 절차로 진행된다.

### 3. 주요 컴포넌트
#### 1) AuthenticationManager
- 인증 과정을 관리하는 중심 컴포넌트
- 사용자가 로그인을 시도할 때 AuthenticationManager가 사용자의 자격 증명(이메일/비밀번호)을 받아 이를 인증할 수 있는 프로세스를 호출

#### 2) AuthenticationProvider
- 여러 AuthenticationProvider가 존재할 수 있으며 각각은 특정 인증 방식을 처리
- ex) 기본적인 이메일/비밀번호 인증은 하나의 AuthenticationProvider처리, 소셜 로그인의 경우 또 다른 AuthenticationProvider가 담당

#### 3) UserDetailsService
- 사용자 정보를 불러오고 검증하는 서비스
- 이 서비스는 DB 혹은 다른 저장소에서 사용자 정보를 가져오고 이를 UserDetails 객체로 반환한다.
- UserDetails 객체 : 사용자의 아이디, 비밀번호, 권한 등 다양한 정보를 포함한다.

#### 4) SecurityContext
- 인증이 완료된 사용자 정보를 저장하는 컨텍스트
- 이 정보는 애플리케이션 전반에서 공유되며 SecurityContextHolder를 통해 접근할 수 있다.

### 4. SecurityContextHolder
- SecurityContextHolder는 현재 보안 컨텍스트에 대한 세부 정보를 보관한다.
- 기본적으로 ThreadLocal을 사용하여 동일 스레드 내에서 각 사용자의 인증 정보를 개별적으로 유지한다.
- 즉, 요청마다 인증된 사용자의 정보를 보존하고 다른 요청에서는 다른 사용자의 정보를 처리할 수 있도록 한다.
- 어플리케이션의 어디서아 `SecurityContextHolder.getContext()` 메소드를 이용해 인증 정보에 접근할 수 있다.

### 5. Filter Chain
-  Spring Security에서 HTTP 요청을 처리할 때 사용하는 일련의 필터들
- 각 필터는 특정 보안 기능을 담당 → 요청이 어플리케이션에 도달하기 전 이 필터들을 순차적으로 통과한다.
- 모든 필터를 통과한 요청만이 실제 어플리케이션 로직에 도달할 수 있다.

#### 주요 필터들
**SecurityContextPersistenceFilter**
- 요청 간 SecurityContext를 유지
- 새 요청이 들어올 때 이전에 인증된 사용자의 정보를 복원

**UsernamePasswordAuthenticationFilter**
- 폼 기반 로그인을 처리
- 사용자가 제출한 username과 password를 확인하여 인증을 시도

**AnonymousAuthenticationFilter**
- 이전 필터에서 인증되지 않은 요청에 대해 익명 사용자 인증을 제공

**ExceptionTranslationFilter**
- Spring Security 예외를 HTTP 응답으로 변환
- 인증 실패 시 로그인 페이지로 리다이렉트하거나, 인가 실패 시 403 오류를 반환

**FilterSecurityInterceptor**
- 접근 제어 결정을 내리는 마지막 필터
- 현재 인증된 사용자가 요청한 리소스에 접근할 권한이 있는지 확인

### 6. 인증과 인가 흐름
#### 인증 흐름
- 사용자 로그인 요청
- UsernamePasswordAuthenticationFilter가 요청을 가로채고 Authentication 객체를 생성
- AuthenticationManager는 적절한 AuthenticationProvider를 선택하여 인증을 위임
- 선택된 AuthenticationProvider는 UserDetailsService를 사용하여 사용자 정보를 로드 → 로드된 정보를 바탕으로 비밀번호를 검증
- 인증이 성공하면 Authentication 객체가 SecurityContext에 저장

#### 인가 흐름
- 인증된 사용자가 보호된 리소스에 접근을 시도
- FilterSecurityInterceptor가 요청을 가로채고 권한 검사를 시작
- AccessDecisionManager는 현재 사용자의 권한과 요청된 리소스의 필요 권한을 비교
- SecurityContext에서 현재 인증된 사용자의 권한 정보를 조회
- 사용자의 권한이 충분하면 리소스 접근이 허용
- 권한이 부족하면 AccessDeniedException이 발생하고 접근이 거부

<br>

## 세션 방식 구현
### 1. Security 설정
- Spring Security를 커스텀 설정하기 위해서 SecurityConfig 클래스를 생성
- 어떤 URL에 어떤 권한을 가진 사람을 접근하게 할 것인지를 지정
- Filter Chain을 사용하여 들어오는 HTTP 요청을 필터링하고 인증 또는 인가 로직을 처리 → 이때 SecurityConfig는 그 필터 체인과 보안 정책을 설정하는 역할을 한다
```java
@EnableWebSecurity      // Spring Security 설정을 활성화
@Configuration
public class SecurityConfig {

    /// 허용할 Url을 따로 빼서 관리
    private final String[] allowUris = {
			// Swagger 허용
            "/swagger-ui/**",
            "/swagger-resources/**",
            "/v3/api-docs/**",
    };


    /// SecurityFilterChain을 정의
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
                .authorizeHttpRequests(requests -> requests         // HTTP 요청에 대한 접근 제어를 설정
                        .requestMatchers(allowUris)                 // 특정 URL 패턴에 대한 접근 권한을 설정
                        .permitAll()                                // 인증 없이 접근 가능한 경로를 지정
                        .requestMatchers("/admin/**").hasRole("ADMIN")      //  'ADMIN' 역할을 가진 사용자만 접근 가능
                        .anyRequest().authenticated()               // 이 외 모든 요청에 대해 인증을 요구
                )
                .formLogin(form -> form                                     // 폼 기반 로그인에서의 처리
                        .defaultSuccessUrl("/swagger-ui/index.html", true)  // 성공시 /swagger-ui/index.html 으로 리다이렉트
                        .permitAll()
                )
                .logout(logout -> logout
                        .logoutUrl("/logout")                   // /logout 경로로 로그아웃을 처리
                        .logoutSuccessUrl("/login?logout")      // 로그아웃 성공 시 /login?logout으로 리다이렉트
                        .permitAll()
                );

        return http.build();
    }
}
```

### 2. 회원가입에 보안 추가하기
#### 사용자 역할 정의
- global → auth → enums 폴더에 사용자 역할을 정의하는 Enum Role을 추가
- 현재는 관리자와 일반 사용자의 역할을 정의함
```java
public enum Role {
    ROLE_ADMIN, ROLE_USER
}
```
- 추가적으로 SecurityConfig의 hasRole 메서드를 사용할때 DB에 ROLE_ 접두사가 붙어 있어야 한다

#### 멤버 엔티티 수정
- 이후 member 엔티티에 아이디로 사용할 이메일과 비밀번호 속성을 추가
```java
@Entity
public class Member extends BaseEntity {
    // 기존 필드들...

    @Column(nullable = false, unique = true)
    private String email;

    @Column(nullable = false)
    private String password;

    @Enumerated(EnumType.STRING)
    private Role role;

    // 기존 필드들...
}
```
- 여기에 맞춰 MemberReqDTO, MemberConverter, MemberCommandServiceImpl(비밀번호를 암호화하여 저장하기 위해 PasswordEncoder를 사용), SecurityConfig를 순서대로 수정한다.
```java
		...
		
	// Password Encoder
    private final PasswordEncoder passwordEncoder;

    // 회원가입
    @Override
    public MemberResDTO.JoinDTO signup(
            MemberReqDTO.JoinDTO dto
    ){

        // 솔트된 비밀번호 생성
        String salt = passwordEncoder.encode(dto.password());

        // 사용자 생성: 유저 / 관리자는 따로 API 만들어서 관리
        Member member = MemberConverter.toMember(dto, salt, Role.ROLE_USER);
				
				...
```

```java
@EnableWebSecurity
@Configuration
public class SecurityConfig {

    private final String[] allowUris = {
				    "/sign-up",
            "/swagger-ui/**",
            "/swagger-resources/**",
            "/v3/api-docs/**",
    };

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
                .authorizeHttpRequests(requests -> requests
                        .requestMatchers(allowUris).permitAll()
                        .requestMatchers("/admin/**").hasRole("ADMIN")
                        .anyRequest().authenticated()
                )
                .formLogin(form -> form
                        .defaultSuccessUrl("/swagger-ui/index.html", true)
                        .permitAll()
                )
                .csrf(AbstractHttpConfigurer::disable)
                .logout(logout -> logout
                        .logoutUrl("/logout")
                        .logoutSuccessUrl("/login?logout")
                        .permitAll()
                );

        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### 3. 로그인 구현
인증 객체인 UserDetails와 인증 서비스인 UserDetailsService를 상속받는 클래스들을 작성해야 한다.

#### CustomUserDetails
```java
@RequiredArgsConstructor
public class CustomUserDetails implements UserDetails {

    private final Member member;

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {    // 권한을 리스트 형태로 반환 (관리자 or 유저)
        return List.of(() -> member.getRole().toString());
    }

    @Override
    public String getPassword() {       // 비밀번호를 반환
        return member.getPassword();
    }

    @Override
    public String getUsername() {       // 아이디를 반환 (해당 시스템에서는 이메일을 아이디로 사용하고 있음)
        return member.getEmail();
    }
}
```

#### CustomUserDetailsService
보안을 강화하고 싶다면 로그인 시도 횟수 제안, 2단계 인증 등의 추가적인 기능을 구현할 수 있다.
```java
@Service
@RequiredArgsConstructor
public class CustomUserDetailsService implements UserDetailsService {

    private final MemberRepository memberRepository;

    @Override
    public UserDetails loadUserByUsername(
            String username
    ) throws UsernameNotFoundException {
        // 검증할 Member 조회
        Member member = memberRepository.findByEmail(username)
                .orElseThrow(() -> new MemberException(MemberErrorCode.NOT_FOUND));
        // CustomUserDetails 반환
        return new CustomUserDetails(member);
    }
}
```

<br>

## 세션 기반 로그인 vs 토큰 기반 로그인
### 세션 기반 로그인
- 로그인 시 서버가 세션 객체를 생성.
- 클라이언트에게 쿠키를 전달.
- 이후 클라이언트의 요청마다 쿠키를 통해 사용자를 식별한다.
- 사용자의 정보가 서버의 메모리에 저장된다.
- 클라이언트는 쿠키로 인증을 유지한다.
- 로그아운 시에는 세션 삭제가 필요하다.

#### 실습에서의 로그인 과정
- 로그인 과정은 formLogin()을 통해 진행
- 인증 정보는 서버 세션에 저장 → 브라우저는 JSESSIONID 쿠키를 전송

### 토큰 기반 로그인
- 로그인 시 서버가 토큰을 발급
- 클라이언트가 이를 저장
- 이후 모든 요청에 Authorization 헤더로 전송
- 사용자의 정보는 토큰에 포함되고 서버는 이를 저장하지 않는다.
- 로그아웃 시에는 토큰을 삭제하거나 서버에서 블랙리스트 관리가 필요하다

### 비교
- 둘의 가장 큰 차이점은 서버가 사용자의 정보를 기억하고 있느냐 or 기억하지 않느냐
- 최근에는 서버와 클라이언트가 분리된 구조의 서비스가 많아지며 토큰 기반 인증(JWT 등)이 범용적으로 더 많이 사용된다.

<br>

## JWT
- 토큰의 한 종류
- JSON 형식과 서명을 통해 인증 정보를 안전하게 전달하는 표준화된 방식
- Header.Payload.Signature 세 부분으로 나뉜 긴 문자열이며 각 부분은 .으로 구분된다.
  - Header : 토큰 타입(여기서는 JWT)과 서명 알고리즘 정보
  - Payload : 사용자 식벽자, 권한, 만료 시간 등등 (민감한 정보는 넣지 않는다)
  - Signature : 서버의 시크릿 키로 서명하여 위조 여부를 검증한다.
- 모든 부분은 인코딩 기법으로 처리되어 누구든지 디코딩이 가능하다. 따라서 페이로드에는 민감한 정보를 넣지 않는다.
- 해당 방식은 stateless한 특성을 가진다.
- 서버가 클라이언트의 상태를 저장하지 않고 클라이언트가 매 요청마다 헤더에 JWT를 전달하여 검증한다 → 수평 확장에 유리
- 서버 메모리에 상태가 저장되지 않으므로 성능 상의 이점이 있다.
- 단, JWT는 토큰 자체가 인증의 수단이 되므로 탈취 시 곧바로 권한이 넘어간다는 위험이 있다.

### Access Token과 Refresh Token
#### Access Token
- 사용자가 실제로 서버에 요청을 보낼 때 본인임을 증명하는 진짜 토큰
- 안전을 위해 유효 기간이 아주 짧다.
- JWT 토큰이 이 방식에 해당한다.

#### Refresh Token
- Access Token의 유효 기간이 끝났을 때 새 출입증을 발급받기 위한 용도로만 쓰는 토큰
- 유효 기간이 길다.
- 보통 DB나 보안이 강력한 곳에 따로 저장한다.
- 만약 어플리케이션의 사용 중 Access Token이 만료되었다면 클라이언트는 가지고 있던 Refresh Token을 서버에 보내며 새 Access Token을 요청한다.
- 이때 사용자는 번거롭게 재로그인을 할 필요가 없다.

### 블랙리스트
- 사용자가 로그아웃을 했지만 서버의 Access Token 유효 기간이 남아 있을 경우 서버는 해당 토큰을 블랙리스트로 올리게 된다.
- 그러나 해당 기능을 위해 사용자의 요청이 들어올 때마다 블랙리스트를 뒤져봐야 한다.
- 즉, 매 요청마다 서버의 저장소를 뒤져봐야 하니 서버 자체에 세션 상태를 두고 확인하는 기존의 세션 방식과의 차이가 없어질 수 있다!

### 블랙리스트와 세션의 딜레마 완화법
#### 1. Access Token의 수명을 극단적으로 줄이기
- 토큰의 유효 기간을 5~10분 정도로 아주 짧게 설정한다.
- 블랙리스트를 만들지 않는 대신 토큰이 탈취되더라도 해커가 사용할 수 있는 시간을 최대 5분으로 제한한다.
- 서버가 블랙 리스트를 사용하지 않으므로서 성능이 빨라진다.
- 5분 동안의 해킹 위험을 감수해야 한다.
- 5분마다 Refresh Token으로 갱신하는 요청이 잦아져 트래픽이 조금 늘어난다.

#### 2. Redis를 활용한 블랙리스트
- 데이터베이스(Disk)가 아닌 Redis 같은 인메모리 DB(RAM)에 블랙리스트를 저장
- RAM은 디스크보다 수백에서 수십배 속도가 빠르다. 매 요청마다 블랙리스트를 조회해도 딜레이가 거의 느껴지지 않는다.
- 보통은 로그아웃된 Access Token의 남은 유효 시간(TTL) 동안만 Redis에 저장하고 시간이 지나면 자동 삭제되게 설정한다 → 메모리 낭비도 거의 일어나지 않는다
- 실무에서 가장 많이 채택하는 방식이다

#### 3. Refresh Token Rotation (RTR)
- Refresh Token이 탈취되는 것까지를 고려한 보안 강화 기법
- 클라이언트가 Refresh Token을 사용해 Access Token을 재발급받을 때 기존 Refresh Token을 버리고 새로운 Refresh Token을 함께 발급한다
- 만약 이미 폐기된(한 번 사용된) Refresh Token이 또 들어온다면 서버는 해당 사용자의 모든 토큰을 무효화 시키고 강제 로그아웃 한다.
- 토큰이 탈취된 사실을 서버가 알 수 있고 이를 통해 즉각적인 차단이 가능하다.

<br>

## JWT 토큰 방식 구현
리프레시 토큰이 없는 가장 간단한 방식으로 구현한다.

### 1. 설정 파일 추가
build.gradle에 의존성 추가
```java
implementation 'io.jsonwebtoken:jjwt-api:0.12.3'
implementation 'io.jsonwebtoken:jjwt-impl:0.12.3'
implementation 'io.jsonwebtoken:jjwt-jackson:0.12.3'
implementation 'org.springframework.boot:spring-boot-configuration-processor'
```

application.yml 파일에도 jwt 관련 설정을 추가
```java
spring:
...
jwt:
  token:
    secretKey: ZGh3YWlkc2F2ZXdhZXZ3b2EgMTM5ZXUgMDMxdWMyIHEyMiBAIDAgKTJFVio=
    expiration:
      access: 14400000
```
- 이때 시크릿 키에는 외부에는 공개되지 않을 아무 값을 넣어주면 된다.
- 보안을 위해 깃허브에 올리는 대신 환경 변수를 사용한다.
- expiration(만료) 시간 설정은 ms 단위이다.
- 14400000의 경우 4시간으로 설정된 것.

### 2. 로그인 컨트롤러 제작
Member 도메인 혹은 auth 도메인 안에 요청, 응답을 위한 DTO를 만든다.
```java
public class MemberReqDTO {
		
		...
				
	// 로그인
    public record LoginDTO(
            @NotBlank
            String email,
            @NotBlank
            String password
    ){}
}
```

```java
public class MemberResDTO {

    ...

    // 로그인
    @Builder
    public record LoginDTO(
            Long memberId,
            String accessToken
    ){}
}
```

이후 위의 DTO들을 바탕으로 컨트롤러 코드를 작성. 인터페이스를 생성해 스웨거 문서를 함께 만들어준다.
```java
@RestController
@RequiredArgsConstructor
public class MemberController {

    private final MemberCommandService memberCommandService;
    private final MemberQueryService memberQueryService;

    // 회원가입
    @PostMapping("/sign-up")
    public ApiResponse<MemberResDTO.JoinDTO> signUp(
            @RequestBody @Valid MemberReqDTO.JoinDTO dto
    ){
        return ApiResponse.onSuccess(MemberSuccessCode.FOUND, memberCommandService.signup(dto));
    }

    // 로그인
    @PostMapping("/login")
    public ApiResponse<MemberResDTO.LoginDTO> login(
            @RequestBody @Valid MemberReqDTO.LoginDTO dto
    ){
        return ApiResponse.onSuccess(MemberSuccessCode.FOUND, memberQueryService.login(dto));
    }
}
```

### 3. JWT 유틸 생성
Spring Security와 연동하여 JWT를 실제로 생성(발급), 검증, 정보 추출을 하게 되는 핵심 유틸리티 클래스

```java
@Component
public class JwtUtil {

    private final SecretKey secretKey;          // 시크릿 키
    private final Duration accessExpiration;    // 접근 만료 시간

    /// 생성자
    public JwtUtil(
            @Value("${jwt.token.secretKey}") String secret,                 // application.yml에서 비밀 키 값을 가져옴
            @Value("${jwt.token.expiration.access}") Long accessExpiration  // 유효 기간 값을 가져옴
    ) {
        // 가져온 문자열 비밀 키를 암호화 알고리즘에서 쓸 수 있는 객체 형태로 변환
        this.secretKey = Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
        this.accessExpiration = Duration.ofMillis(accessExpiration);
    }

    /// 외부에서 호출하는 메소드
    public String createAccessToken(CustomUserDetails user) {
        // 실제로 토큰이 만들어지는 내부 메소드
        return createToken(user, accessExpiration);
    }

    /// 토큰에서 정보를 추출하는 메소드
    public String getEmail(String token) {
        try {
            return getClaims(token).getPayload().getSubject(); // 전달받은 토큰의 주인(여기서는 이메일)을 가져온다
        } catch (JwtException e) {
            return null;
        }
    }

    /// 외부에서 호출하는 메소드
    /// 토큰의 유효성을 검증한다
    public boolean isValid(String token) {
        try {
            getClaims(token);   // 토큰 검증 메소드 호출
            return true;
        } catch (JwtException e) {  // 에러(만료, 위조 등) 발생 시 false 리턴
            return false;
        }
    }

    /// 실제로 토큰이 만들어지는 내부 메소드
    private String createToken(CustomUserDetails user, Duration expiration) {
        Instant now = Instant.now();    // 현재 시간

        // 권한 정보 추출 (USER 또는 ADMIN)
        String authorities = user.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority)
                .collect(Collectors.joining(","));

        return Jwts.builder()
                .subject(user.getUsername())        // 토큰의 주인 설정 (여기서는 이메일)
                .claim("role", authorities)         // 토큰의 권한
                .claim("email", user.getUsername()) // 사용자 이메일
                .issuedAt(Date.from(now))           // 발급 시간
                .expiration(Date.from(now.plus(expiration))) // 만료 시간
                .signWith(secretKey)    // 서버의 비밀 키로 서명
                .compact();             // .으로 구분된 문자열로 압축하여 반환
    }

    /// 토큰의 내부 정보를 열어보는 메소드
    /// 클라이언트가 가져온 토큰의 진위여부와 유효기간을 확인함
    private Jws<Claims> getClaims(String token) throws JwtException {
        return Jwts.parser()
                .verifyWith(secretKey)      // 비밀키로 서명한 게 맞는지
                .clockSkewSeconds(60)       // 서버 간 차이 60초 고려
                .build()
                .parseSignedClaims(token);  // 토큰 파싱
    }
}
```

### 4. JwtAuthFilter
- 요청 한 번당 딱 한 번만 실행되는 필터
- 사용자가 API를 호출할 때마다 이 필터가 무조건 실행되어 검사를 진행

```java
@RequiredArgsConstructor
public class JwtAuthFilter extends OncePerRequestFilter {

    private final JwtUtil jwtUtil;
    private final CustomUserDetailsService customUserDetailsService;

    /// 토큰 검사 메소드
    @Override
    protected void doFilterInternal(
            @NonNull HttpServletRequest request,
            @NonNull HttpServletResponse response,
            @NonNull FilterChain filterChain
    ) throws ServletException, IOException {

        try {
            // 헤더에서 Authorization 찾기
            String token = request.getHeader("Authorization");
            // token이 없거나 Bearer가 아니면 넘기기
            if (token == null || !token.startsWith("Bearer ")) {
                filterChain.doFilter(request, response);
                return;
            }
            // Bearer이면 추출
            token = token.replace("Bearer ", "");
            // 유효성 검사 : jwtUtil.isValid() 이용
            if (jwtUtil.isValid(token)) {    // 유효성 검사를 통과했다면
                // 토큰에서 이메일 꺼내기
                String email = jwtUtil.getEmail(token);     
                // DB에서 해당 유저의 정보 전체를 가져오기 (비밀번호, 권한 등)
                UserDetails user = customUserDetailsService.loadUserByUsername(email);
                // 인증 티켓(Authentication) 만들기
                Authentication auth = new UsernamePasswordAuthenticationToken(
                        user,
                        null,
                        user.getAuthorities()
                );
                // 스프링 시큐리티 저장소(SecurityContextHolder)에 넣기
                SecurityContextHolder.getContext().setAuthentication(auth);
            }
            // 검사가 모두 끝났으니 다음 단계(컨트롤러 등)로 넘어가기
            filterChain.doFilter(request, response);
        } catch (Exception e) {     // 검증 도중 에러가 발생했다면 아래 코드를 실행
            response.setContentType("application/json;charset=UTF-8");
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);

            ApiResponse<Void> errorResponse = ApiResponse.onFailure(
                    GeneralErrorCode.UNAUTHORIZED,
                    null
            );

            ObjectMapper mapper = new ObjectMapper();
            mapper.writeValue(response.getOutputStream(), errorResponse);
        }
    }
}
```

### 5. SecurityConfig 설정
```java
@EnableWebSecurity
@Configuration
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtUtil jwtUtil;
    private final CustomUserDetailsService customUserDetailsService;

    private final String[] allowUris = {
            "/login"
            "/swagger-ui/**",
            "/swagger-resources/**",
            "/v3/api-docs/**",
    };

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
                .authorizeHttpRequests(requests -> requests
                        .requestMatchers(allowUris).permitAll()
                        .requestMatchers("/admin/**").hasRole("ADMIN")
                        .anyRequest().authenticated()
                )
                // 폼로그인 비활성화
                .formLogin(AbstractHttpConfigurer::disable)
                // JwtAuthFilter를 UsernamePasswordAuthenticationFilter 앞에 추가
                .addFilterBefore(jwtAuthFilter(), UsernamePasswordAuthenticationFilter.class)
                .csrf(AbstractHttpConfigurer::disable)
                .logout(logout -> logout
                        .logoutUrl("/logout")
                        .logoutSuccessUrl("/login?logout")
                        .permitAll()
                );

        return http.build();
    }

    @Bean
    public JwtAuthFilter jwtAuthFilter() {
        return new JwtAuthFilter(jwtUtil, customUserDetailsService);
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### 6. 로그인 Service
```java
@Service
@RequiredArgsConstructor
public class MemberQueryServiceImpl implements MemberQueryService{

    private final MemberRepository memberRepository;
    private final JwtUtil jwtUtil;
    private final PasswordEncoder encoder;

    @Override
    public MemberResDTO.LoginDTO login(
            MemberReqDTO.@Valid LoginDTO dto
    ) {

        // Member 조회
        Member member = memberRepository.findByEmail(dto.email())
                .orElseThrow(() -> new MemberException(MemberErrorCode.NOT_FOUND));

        // 비밀번호 검증
        if (!encoder.matches(dto.password(), member.getPassword())){
            throw new MemberException(MemberErrorCode.INVALID);
        }

        // JWT 토큰 발급용 UserDetails
        CustomUserDetails userDetails = new CustomUserDetails(member);

        // 엑세스 토큰 발급
        String accessToken = jwtUtil.createAccessToken(userDetails);

        // DTO 조립
        return MemberConverter.toLoginDTO(member, accessToken);
    }
}
```

### 7. 오류 처리
#### AuthenticationEntryPoint 작성
- 필터에서 발생하는 에러는 에러 핸들러가 잡지 못한다.
- 이를 위해 필터 전용 에러 처리기인 AuthenticationEntryPoint를 작성해야 한다.
```java
public class AuthenticationEntryPointImpl implements AuthenticationEntryPoint {
    private final ObjectMapper objectMapper = new ObjectMapper();


    @Override
    public void commence(
            HttpServletRequest request,
            HttpServletResponse response,
            AuthenticationException authException
    ) throws IOException {
        response.setContentType("application/json;charset=UTF-8");
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);

        ApiResponse<Void> errorResponse = ApiResponse.onFailure(
                GeneralErrorCode.UNAUTHORIZED,
                null
        );

        objectMapper.writeValue(response.getOutputStream(), errorResponse);
    }
}
```
- 만약 JwtAuthFilter에서 에러가 발생하면 try-catch문이 실행되며 다음 단계를 마저 실행한다.
- 이후 스프링 시큐리티 필터에서 인증/인가를 확인하고 이때 인증이 없다는 것을 확인한 뒤 AccessDeniedException를 발생시킨다.
- 발생한 AccessDeniedException은 AuthenticationEntryPoint에서 처리한다.
- AuthenticationEntryPoint를 구현한 AuthenticationEntryPointImpl에서 commence()를 실행한다.

#### 필터 체인에 등록
```java
@EnableWebSecurity
@Configuration
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtUtil jwtUtil;
    private final CustomUserDetailsService customUserDetailsService;

    private final String[] allowUris = {
            "/login",
            "/sign-up",
            "/swagger-ui/**",
            "/swagger-resources/**",
            "/v3/api-docs/**",
    };

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
                .authorizeHttpRequests(requests -> requests
                        .requestMatchers(allowUris).permitAll()
                        .requestMatchers("/admin/**").hasRole("ADMIN")
                        .anyRequest().authenticated()
                )
                // 폼로그인 비활성화
                .formLogin(AbstractHttpConfigurer::disable)
                // JwtAuthFilter를 UsernamePasswordAuthenticationFilter
                .addFilterBefore(jwtAuthFilter(), UsernamePasswordAuthenticationFilter.class)
                .csrf(AbstractHttpConfigurer::disable)
                .logout(logout -> logout
                        .logoutUrl("/logout")
                        .logoutSuccessUrl("/login?logout")
                        .permitAll()
                )
                .exceptionHandling(exception -> exception.authenticationEntryPoint(authenticationEntryPoint()))

        ;

        return http.build();
    }

    @Bean
    public JwtAuthFilter jwtAuthFilter() {
        return new JwtAuthFilter(jwtUtil, customUserDetailsService);
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }


    @Bean
    public AuthenticationEntryPoint authenticationEntryPoint() {
        return new AuthenticationEntryPointImpl();
    }

}
```

#### JwtAuthFilter 수정
try-catch 문을 아래와 같이 수정한다.
```java
    try {
        if (token != null && jwtUtil.isValid(token)) {
            // 토큰이 유효할 때만 인증 객체 저장
            String email = jwtUtil.getEmail(token);
            UserDetails user = customUserDetailsService.loadUserByUsername(email);
            Authentication auth = new UsernamePasswordAuthenticationToken(user, null, user.getAuthorities());
            SecurityContextHolder.getContext().setAuthentication(auth);
        }
    } catch (JwtException e) {
        // 예외가 발생해도 여기서 response를 직접 건드리지 않음!
        // 그냥 로그만 찍고 넘어감 (=인증 객체가 null인 상태로 넘어감)
        request.setAttribute("exception", e); // 필요하면 에러 이유를 request에 담아둠
    }
    
    // 예외가 났든 안 났든 무조건 다음 필터로 진행
    filterChain.doFilter(request, response);
```
