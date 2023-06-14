# spring-data-jpa-basic
[인프런] 실전! 스프링 데이터 JPA 공부

## 1. 공통 인터페이스 기능

- 인터페이스만으로 repository가 동작하는 이유는 스프링 데이터 JPA가 구현 클래스를 대신 만들어서 injection 해줬기 때문
- `@Repository` 애노테이션 생략 가능 ⇒ 컴포넌트 스캔을 스프링 데이터 JPA가 자동으로 처리
- (적용방법)

    ```java
    public interface MemberRepository extends JpaRepository<Member, Long> {
    }
    ```

- (주요 메서드)
    - `save(S)` : 새로운 엔티티는 저장하고 이미 있는 엔티티는 병합한다.
    - `delete(T)` : 엔티티 하나를 삭제한다. 내부에서 EntityManager.remove() 호출
    - `findById(ID)` : 엔티티 하나를 조회한다. 내부에서 EntityManager.find() 호출
    - `getOne(ID)` : 엔티티를 프록시로 조회한다. 내부에서 EntityManager.getReference()
    - `findAll(…)` : 모든 엔티티를 조회한다. 정렬( Sort )이나 페이징( Pageable ) 조건을 파라미터로 제공할 수 있다.

## 2. 쿼리 메소드 기능

### 2.1 메소드 이름으로 쿼리 생성

- 쿼리 메서드 이름을 규칙에 맞게 정의하여 원하는 쿼리를 생성할 수 있다.
  도큐먼트 Query Creation 참고 - [Spring Data JPA - Reference Documentation](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#jpa.query-methods.query-creation)
- (예시) find…ByUsername ⇒ username을 기준 equal 쿼리 질의 함. 가운데 …은 아무거나 적어도 된다.

### 2.2 @Query, 리포지토리 메소드에 쿼리 정의하기

- (예시)

    ```java
    @Query("select m from Member m where m.username = :username and m.age = :age")
    List<Member> findUser(@Param("username")String username, @Param("age") int age);
    ```

- 실행할 메서드에 정적 쿼리를 직접 작성하므로 이름 없는 Named 쿼리라 할 수 있음
- JPA Named 쿼리처럼 애플리케이션 실행 시점에 문법 오류를 발견할 수 있음(매우 큰 장점!)

### 2.3 파라미터 바인딩

- 컬렉션 파라미터 바인딩

    ```java
    @Query("select m from Member m where m.username in :names")
    List<Member> findByNames(@Param("names") List<String> names);
    ```


### 2.4 반환 타입

- 다양한 반환 타입 지원

    ```java
    List<Member> findByUsername(String name); //컬렉션
    Member findByUsername(String name); //단건
    Optional<Member> findByUsername(String name); //단건 Optional
    ```

    <aside>
    📌 **조회 결과가 많거나 없으면? (중요)**

  컬렉션
  ⇒ 결과 없음: 빈 컬렉션 반환 (null체크 필요X)
  단건 조회
  ⇒ 결과 없음: null 반환
  ⇒ 결과가 2건 이상: javax.persistence.NonUniqueResultException 예외 발생

    </aside>


### 2.5 스프링 데이터 JPA 페이징과 정렬

- 페이징과 정렬 파라미터
    - `org.springframework.data.domain.Sort` : 정렬 기능
    - `org.springframework.data.domain.Pageable` : 페이징 기능 (내부에 Sort 포함)
- 특별환 반환 타입
    - `org.springframework.data.domain.Page` : 추가 count 쿼리 결과를 포함하는 페이징
    - `org.springframework.data.domain.Slice` : **추가 count 쿼리 없이** 다음 페이지만 확인 가능
      (내부적으로 limit + 1조회)
    - List (자바 컬렉션): 추가 count 쿼리 없이 결과만 반환
- (예시)

    ```java
    // 페이징
    Page<Member> findByAge(int age, Pageable pageable);
    
    @Test
    public void paging() {
    
        // given
        memberRepository.save(new Member("member1", 10));
        memberRepository.save(new Member("member2", 10));
        memberRepository.save(new Member("member3", 10));
        memberRepository.save(new Member("member4", 10));
        memberRepository.save(new Member("member5", 10));
    
        int age = 10;
        PageRequest pageRequest = PageRequest.of(0, 3, Sort.by(Sort.Direction.DESC, "username"));
    
        // when
        Page<Member> page = memberRepository.findByAge(age, pageRequest);
    
        // then
        List<Member> content = page.getContent();
        long totalElements = page.getTotalElements();
        for (Member member : content) {
            System.out.println("member = " + member);
        }
        System.out.println("totalElemts = " + totalElements);
    
        assertThat(content.size()).isEqualTo(3);
        assertThat(page.getTotalElements()).isEqualTo(5); // 전체 데이터 수
        assertThat(page.getNumber()).isEqualTo(0);        // 현재 페이지 번호
        assertThat(page.getTotalPages()).isEqualTo(2);    // 전체 페이지 수
        assertThat(page.isFirst()).isTrue();              // 첫 페이지 여부
        assertThat(page.hasNext()).isTrue();              // 다음 페이지 존재 여부
    
    }
    ```

- count 쿼리 분리 방법
  Page 반환타입 사용할 경우 count 쿼리가 자동으로 쿼리되는데 데이터를 left join한다면 count까지 굳이 할 필요 없음 ⇒ 카운트 쿼리 분리. **실무에서 매우 중요!!**

    ```java
    @Query(value = “select m from Member m”,
    countQuery = “select count(m.username) from Member m”)
    Page<Member> findMemberAllCountBy(Pageable pageable);
    ```

- 엔티티를 DTO로 쉽게 변환하기 위해 map 사용
  `Page<MemberDto> toMap = page.map(member -> new MemberDto(member.getId(), member.getUsername(), null));`

### 2.6 벌크성 수정쿼리

- `@Modifying` 어노테이션을 추가하여 JPA `.executeUpdate()` 를 실행

    ```java
    @Modifying
    @Query("update Member m set m.age = m.age + 1 where m.age >= :age")
    int bulkAgePlus(@Param("age") int age);
    ```

- 벌크성 쿼리를 실행하고 나서 영속성 컨텍스트 초기화: `@Modifying(clearAutomatically = true)`

    <aside>
    📌  벌크 연산은 영속성 컨텍스트를 무시하고 실행하기 때문에, 영속성 컨텍스트에 있는 엔티티의 상태와 DB에 엔티티 상태가 달라질 수 있다.

  > 권장하는 방안
  >
  > 1. 영속성 컨텍스트에 엔티티가 없는 상태에서 벌크 연산을 먼저 실행한다.
  > 2. 부득이하게 영속성 컨텍스트에 엔티티가 있으면 벌크 연산 직후 영속성 컨텍스트를 초기화 한다.
    </aside>


### 2.7 @EntityGraph

- 사실상 페치 조인(FETCH JOIN)의 간편 버전. JPQL 작성 필요 없음
- LEFT OUTER JOIN 사용
- (예시)

    ```java
    //(1) 공통 메서드 오버라이드
    @Override
    @EntityGraph(attributePaths = {"team"})
    List<Member> findAll();
    //(2) JPQL + 엔티티 그래프
    @EntityGraph(attributePaths = {"team"})
    @Query("select m from Member m")
    List<Member> findMemberEntityGraph();
    //(3) 메서드 이름으로 쿼리에서 특히 편리하다.
    @EntityGraph(attributePaths = {"team"})
    List<Member> findByUsername(String username)
    ```


### 2.8 JPA Hint & Lock

- JPA 구현체에게 제공하는 힌트

```java
@QueryHints(value = {@QueryHint(name = "org.hibernate.readOnly", value="true")}, forCounting = true)
Member findReadOnlyByUsername(String username);
```

## 3. 확장 기능

### 3.1 사용자 정의 리포지토리 구현

- 다양한 이유로 인터페이스의 메서드를 직접 구현하고 싶을 때 사용
    - JPA 직접 사용( EntityManager )
    - 스프링 JDBC Template 사용
    - MyBatis 사용
    - 데이터베이스 커넥션 직접 사용 등등...
    - Querydsl 사용
- 사용자 정의 인터페이스 + 사용자 정의 인터페이스 구현클래스 + JpaRepository 상속한 인터페이스에 사용자 정의 인터페이스 상속 추가 (**클래스명 규칙에 맞게 작성 필요)**
  MemberRepository 인터페이스
  MemberRepositoryCustom 인터페이스
  MemberRepositoryImpl 클래스 ⇒ 이름 규칙 맞춰줘야 함

### 3.2 Auditing

- 엔티티를 생성, 변경할 때 변경한 사람과 시간을 추적하고자 할 때 사용

<aside>
📌 실무에서 대부분의 엔티티는 등록시간, 수정시간이 필요하지만, 등록자, 수정자는 없을 수도 있다. 그래서 다음과 같이 Base 타입을 분리하고, 원하는 타입을 선택해서 상속한다.

</aside>

### 3.3 Web확장 - 페이징과 정렬

- (예시)

    ```java
    @GetMapping("/members")
    public Page<MemberDto> list(@PageableDefault(size = 5) Pageable pageable) {
        return memberRepository.findAll(pageable).map(MemberDto::new);
    }
    ```

- 요청 파라미터
    - 예) /members?page=0&size=3&sort=id,desc&sort=username,desc
    - page: 현재 페이지, 0부터 시작한다.
    - size: 한 페이지에 노출할 데이터 건수
    - sort: 정렬 조건을 정의한다. 예) 정렬 속성,정렬 속성...(ASC | DESC), 정렬 방향을 변경하고 싶으면 sort 파라미터 추가 (asc 생략 가능)
- 개별 설정 `@PageableDefault` 어노테이션 사용

## 4. 스프링 데이터 JPA 분석

- 스프링데이터 JPA가 제공하는 공통 인터페이스의 구현체인 SimpleJpaRepository 안에 `@Transactional`이 적용 돼 있기에 서비스 계층에서 트랜잭션 시작하지 않아도 리파지토리 계층에서 시작
- JPA 식별자 생성 전략이 @Id 만 사용해서 직접 할당이면 이미 식별자 값이 있는 상태로 save() 를 호출. 따라서 이 경우 merge() 가 호출된다. merge() 는 우선DB를 호출해서 값을 확인하고, DB에 값이 없으면 새로운 엔티티로 인지하므로 매우 비효율 적
  ⇒ `Persistable` 를 사용해서 새로운 엔티티 확인 여부를 직접 구현하게는 효과적
  ⇒ 참고로 등록시간( @CreatedDate )을 조합해서 사용하면 이 필드로 새로운 엔티티 여부를 편리하게 확인할 수 있음

## 5. 나머지 기능들

### 5.1 Specifications(명세)

- 동적쿼리를 사용하기 위한 Specification 기술 있으나 복잡. **⇒ QueryDSL을 사용**

### 5.2 Query By Example

- Example을 이용하여 도메인 객체를 검색 조건으로 사용할 수 있음. 단, 실무에서 사용하기에는 매칭 조건이 너무 단순하고, LEFT 조인이 안됨 **⇒ QueryDSL을 사용**

### 5.3 Projections

- 엔티티 대신에 DTO를 편리하게 조회할 때 사용
- `인터페이스 기반 Closed Projections` / `인터페이스 기반 Open Proejctions`
  `클래스 기반 Projection` / `동적 Projections`
- 프로젝션 대상이 root 엔티티를 넘어가면 JPQL SELECT 최적화가 안된다!
  **⇒ 복잡한 쿼리는 QueryDSL을 사용**

### 5.4 네이티브 쿼리

- (예시)

    ```java
    @Query(value = "select * from member where username = ?", nativeQuery = true)
    Member findByNativeQuery(String username);
    ```

- **네이티브 SQL을 DTO로 조회할 때는 JdbcTemplate or myBatis 권장**
- 스프링 데이터 JPA 네이티브 쿼리 + 인터페이스 기반 Projections 활용 : 반환타입 DTO로 가능

    ```java
    @Query(value = "select m.member_id as id, m.username, t.name as teamName " +
            "from member m left join team t",
            countQuery = "select count(*) from member",
            nativeQuery = true)
    Page<MemberProjection> findByNativeProjection(Pageable pageable);
    ```