## QueryDSL

### DSL, Fluent API와 QueryDSL 

#### 1. DSL
- Domain-Specific Language. 도메인 특화 언어.
- 가장 큰 상위 개념.
- Java나 Python 같은 범용 언어와 달리 특정 목적에만 사용되도록 설계된 언어이다.
- 높은 가독성과 간결함을 간결함을 가진다.
- 아래 2가지의 종류가 있다.
  - External DSL: 별도의 문법을 가진 완전한 독립 언어 (예: SQL, HTML, CSS, 정규표현식 등등)
  - Internal DSL: Java나 Kotlin 같은 기존 언어의 문법 내에서, 특정 라이브러리나 API를 통해 DSL처럼 보이게 만든 것 (예: QueryDSL, Gradle Groovy 등등)

#### 2. Fluent API (Fluent Interface)
- Internal DSL을 구현하는 데 자주 사용되는 디자인 패턴.
- **메소드 체이닝**을 사용한다.
- 대부분의 메서드가 자기 자신(this)이나 다음 단계에서 사용할 객체를 리턴한다.
- 이를 통해 여러 메서드를 . 으로 연결하여 마치 하나의 문장처럼 코드를 작성할 수 있다.
- 동일한 내용의 코드를 간결하고 읽기 쉽게 작성할 수 있다.

#### 3. QueryDSL
- QueryDSL은 위 두 가지 개념을 합친 특정 자바/코틀린 라이브러리.
- Java 코드 기반으로 타입에 안전한(type-safe) 데이터베이스 쿼리를 작성할 수 있게 해주는 Internal DSL이다.
- 즉, 기본적인 문법은 모두 JPQL을 따라간다.
- QueryDSL은 완전히 새로운 쿼리 언어가 아니라 문자열로 직접 작성하던 JPQL 쿼리문을 Java 객체와 메소드를 이용해 대신 조립해주는 도구이다!
- Fluent API 패턴(메소드 체이닝)을 사용하여 쿼리를 구축한다.
- QueryDSL은 Q-Type이라는 컴파일 시점에 생성되는 클래스를 사용하여, 자바 코드(객체와 메서드)로 쿼리를 작성한다.
- 만약 쿼리 내에서 존재하지 않는 객체나 메소드를 사용하면 즉시 컴파일 에러가 발생한다.

### Fluent API를 사용할 때의 장점

#### 1. IDE의 코드 자동 완성 기능 사용
코드를 작성할 때 IED가 다음에 입력할 수 있는 코드 목록을 자동으로 보여준다.

#### 2. 문법적으로 잘못된 쿼리를 허용하지 않음
쿼리 문법의 오류를 컴파일 시점에 잡아내준다. 문자열 쿼리문(JPQL)의 경우 컴파일 시점에 문법 오류를 잡지 못하고 런타임이 되어 해당 쿼리가 DB에 전송할 때가 되어서야 예외가 발생한다.

### 3. 도메인 타입과 프로퍼티를 안전하게 참조할 수 있음
도메닝 객체 필드명의 오류나 데이터 타입의 오류를 컴파일 시점에서 잡아낼 수 있다.

#### 4. 도메인 타입의 리팩토링을 더 잘 할 수 있음
코드의 구조를 변경할 때 쿼리 코드까지 함께 변경이 가능하다. IDE의 리펙토리 기능을 사용해 필드명을 바꾸면 쿼리 코드의 필드명 역시 함께 변경된다.

> 즉, 런타임에서나 잡을 수 있었던 오류를 컴파일 시점에서 잡을 수 있게 되었다!

### 동적 쿼리
동적 쿼리는 런타임시의 입력에 따라 WHERE / JOIN / ORDER BY 등의 조건을 동적으로 구성해 SQL을 생성하고 실행할 수 있다. 단, 사용자의 입력을 그대로 문자열에 결합할 시 SQL 인젝션을 당할 수 있다.

#### Q클래스
Q클래스는 QueryDSL에서 엔티티(Entity)를 기반으로 생성되는 쿼리 전용 클래스이다. Q클래스를 사용할 때의 이점은 아래와 같다.
- 타입 안정성 : 컴파일 단계에서 엔티티 기반으로 쿼리를 생성함으로 잘못된 필드나 타입을 사용할 수 없다. 런타임 시 오류를 발생시키지 않는다.
- 자동 완성 : IDE가 자동완성 기능을 제공한다.
- Fluent Interface 스타일 : 메소드 체이닝을 사용하여 읽기 쉬운 쿼리를 작성할 수 있다.
- Internal DSL 효과 : 자바의 코드 안에서 도메인 특화 언어처럼 표현할 수 있다.
그러나 Q클래스를 그대로 깃허브에 올려서는 안 된다!!

#### Q클래스를 깃허브에 올리면 안 되는 이유
- Q클래스는 객체 구조가 변경될 때마다 자바 버전마다 다르게 생성된다.
- 따라서 팀원마다 Q클래스를 공유할 수 없다!
- 이를 해결하기 위해 깃허브에 무시할 파일을 설정하는 .gitignore 파일에 Q클래스가 생성되는 폴더(예: build/generated/ 또는 target/generated-sources/)를 추가한다.


<br/>


## 핵심 키워드

### QueryDSL에서 FetchJoin 하는 법
- 앞서 설명했듯이 QueryDSL는 JPQL의 문법을 따른다!
- 필터링을 위한 join 메소드와 실제 테이블을 결합하기 위한 fetch join이 별개로 존재한다. 
- 이 중 fetch join를 사용해야 N+1 문제 없이 원하는 데이터를 가져올 수 있다.

### DTO 매핑 방식
- DTO 생성자 @QueryProjection 어노테이션을 추가한다.
- 이 방식은 Projections.constructor와 달리 컴파일 시점에 DTO 생성자의 타입이나 인자 개수가 잘못되면 컴파일 오류를 발생시켜서 안전하다.
- @QueryProjection 사용 후 Gradle/Maven으로 컴파일하면 QMemberDto 클래스가 자동 생성된다.
- @QueryProjection를 사용하면 어떠한 DTO가 다른 DTO를 필드로 가지는 구조(DTO안의 DTO)를 쿼리 한 번으로 쉽게 매핑할 수 있다.
```java
// QMember가 QTeamDto를 필드로 가질 때

queryFactory.select(
        new QMemberDTO(
                member.name,
                new QTeamDto(team.name, team.location)))
```


### 커스텀 페이지네이션
- JPA의 Pageable 객체를 직접 사용하기 어렵거나 복잡한 join이 필요할 때 Pageable 객체를 사용하지 않고 수동으로 페이지네이션을 구현하는 방식이다.
- 아래 2개의 쿼리가 필요하다.
- 내용 조회 쿼리: offset()과 limit()을 사용
- 전체 개수 조회 쿼리: select(member.count())를 사용
```java
// Predicate 객체, Pageable 객체가 파라미터로 넘어왔다고 가정

Pageable pageable = PageRequest.of(0, 10); // 0번 페이지, 10개씩

// 1. 내용 조회 쿼리
List<Member> content = queryFactory
        .selectFrom(member)
        .orderBy(member.name.desc())
        .offset(pageable.getOffset())   // 페이징 과정을 수동으로 작성
        .limit(pageable.getPageSize())
        .fetch();

// 2. 전체 개수 조회 쿼리
Long total = queryFactory
        .select(member.count())
        .from(member)
        .where(predicate) // content 쿼리의 where 조건과 동일해야 함
        .fetchOne();

// 3. PageImpl 객체로 조합하여 반환
return new PageImpl<>(content, pageable, total);
```
- 원하는 쿼리문에 페이징 시스템을 한 번 더 덧씌우는 느낌으로 생각하면 된다.
- PageImpl 객체는 페이징을 통해 DB에서 잘라 온 10개의 데이터(content)와 전체 개수가 100만 몇 개인지를 알려주는 쿼리 결과(count)를 합쳐서 포장해주는 역할을 한다.
- 이를 통해 List<T>만으로는 알 수 없는 페이지에 관한 정보를 프론트에게 넘길 수 있게 된다.

### transform - groupBy
- 조회 결과를 Map 같은 컬렉션으로 변환할 때, 특히 1:N 관계를 DTO로 매핑할 때 유용한 기능이다
- 일반적인 JPQL은 SELECT 절에서 컬렉션을 반환할 수 없다.
- 표준 JPQL의 SELECT 절은 엔티티나 DTO 같은 단일 객체만 반환할 수 있다.
- `SELECT new TeamDto(..., List<MemberDto>)`처럼 DTO 생성자에 컬렉션(List)을 직접 넣을 수 없다는 뜻이다.
- 그러나 QueryDSL은 transform으로 이를 가능하게 해준다.
- transform은 map으로 반환된 쿼리 결과를 일단 메모리로 가져온다. 이후 QueryDSL이 이 데이터를 GroupBy.groupBy(...).as(...)에 맞춰 원하는 구조로 재조립해준다.
- 단, transform은 DB에서 모든 데이터를 일단 가져와서 애플리케이션 메모리에서 그룹핑하므로 데이터가 수십만 건 이상으로 매우 많을 경우 성능 저하의 원인이 될 수 있다.

### order by null
- 정렬 메소드로 데이터를 정렬하려 할 때 해당 정렬 기준이 NULL인 값들을 모든 값의 제일 앞에 둘지, 제일 뒤에 줄 지를 지정하는 기능이다.
- 맨 앞에이라면 nullsFirst(), 맨 뒤라면 nullsLast()를 사용한다.
- DB마다 NULL의 기본 정렬 순서가 다르기 때문에(ex. MySQL/H2는 맨 앞, Oracle은 맨 뒤) 일관된 정렬을 위해 사용하게 된다.


<br/>


## ✅ 미션 기록
- 내가 작성한 리뷰 보기 API를 QueryDSL로 구현하기
- 필터링 조건 : 가게별, 별점별
- 제약 조건 : 하나의 API로 설계할 것

### 1️⃣ reviewRepositoryQueryDsl 인터페이스 생성
```java
public interface reviewRepositoryQueryDsl {
    
    List<ShopReview> searchShopReview(Predicate predicate);
}
```
- QueryDSL로 구현하고 싶은 메소드들을 인터페이스 안에서 미리 정의한다.
- 인터페이스에 정의하는 모든 메서드는 자동으로 public이고 abstract가 된다. 
- 따라서 public이고 abstract등의 키워드 작성은 필요하지 않다. 
- 인자로 받게 되는 Predicate 타입의 경우 queryDSL에서 제공하는 것인지 아닌지 잘 보고 import하여야 한다.

### 2️⃣ reviewRepositoryQueryDsl를 구현하는 구현 클래스 작성
```java
@Repository
@RequiredArgsConstructor
public class reviewRepositoryQueryDslImpl implements reviewRepositoryQueryDsl {

    private final EntityManager em;

    @Override
    public List<ShopReview> searchShopReview(Predicate predicate){

        // JPA 세팅
        JPAQueryFactory queryFactory = new JPAQueryFactory(em);

        // Q클래스 선언
        QShopReview review = QShopReview.shopReview;
        QShop shop = QShop.shop;
        QMember member = QMember.member;

        return queryFactory
                .selectFrom(review)
                .join(review.shop, shop).fetchJoin()
                .join(review.member, member)
                .where(predicate)
                .fetch();
    }
}
```
- 현재 별점은 ShopReview 엔티티 안에 존재 → Q클래스를 통한 join이 필요하지 않다.
- join이 필요한 shop과 member의 경우 Q클래스를 통해 join한다.
- 이때 join은 필터링(where절)을 위한 것이고 fetchJoin을 사용해야 N+1 문제 없이 테이블을 결합해서 가져온다.
- 모든 검색의 조건은 Service에서 전달받은 Predicate를 통해 결정된다.
- 어떤 member의 리뷰인지, 어떤 가게의 리뷰인지, 어떤 별점을 가진 리뷰인지는 서비스에서 결정되는 것.

### 3️⃣ reviewRepository에 JPA와 QueryDsl 인터페이스를 둘 다 상속
```java
public interface reviewRepository extends JpaRepository<ShopReview, Long>, reviewRepositoryQueryDsl {...}
```
- 메인 인터페이스인 reviewRepository가 JpaRepository와 직접 만든 reviewRepositoryQueryDsl를 둘 다 상속
- 서비스 계층에서는 ReviewRepository 하나만 주입받으면 reviewRepository.save() (JPA 기본 기능)와 reviewRepository.searchShopReview() (내가 만든 QueryDSL 기능)를 모두 사용할 수 있게 된다.

