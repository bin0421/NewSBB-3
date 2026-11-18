# NewSBB-3

스프링부트 REST API로 글·댓글의 CRUD(다건조회, 단건조회, 작성, 수정, 삭제)를 TDD 로 아주 작은 단계마다 커밋하며 구현하는 저장소.

- 참고 : [강의 설명 1~56강](https://www.slog.gg/p/14172), 강사 저장소 [jhs512/p-14172-2](https://github.com/jhs512/p-14172-2)
- 진행 방식 : 강사 저장소 56강 시점의 구조와 응답 형식을 따르되, 기능마다 테스트를 먼저 커밋(실패)하고 구현을 커밋(통과)한다.
- 단계 번호 : 세 자리 번호 001, 002 ... / 한 단계 안의 커밋은 하위 번호 -1, -2 ... (`-1` 테스트 추가, `-2` 구현, 필요하면 `-3`)
  - 제목 끝의 `(실패)` 는 테스트가 실패하는 상태로 커밋했다는 뜻이다.
- 진행 기록·결정 사항 : [NewSBB-3.txt](NewSBB-3.txt)

## 개발 환경

| 항목 | 버전 |
|---|---|
| Java | 25 (Temurin) |
| Spring Boot | 4.1.2 |
| Gradle | 9.8.2 (Kotlin DSL) |
| DB | H2 (개발 : 파일, 테스트 : 메모리) |
| 테스트 | JUnit 5, MockMvc |

## 개발 일지

각 단계는 다음 형식으로 적는다.

- **링크** : 단어 "링크"를 누르면 해당 커밋 페이지로 이동해 직전 단계와의 코드 변경(diff)을 볼 수 있다
- **원인** : 이 단계가 필요한 이유 (발견한 문제나 개선점)
- **작업** : 추가·수정·삭제한 기능, 객체, 클래스, 메서드
- **설명** : 작업의 구현 방식과 동작 원리 (작업에 적은 내용은 반복하지 않는다)

### 001 프로젝트 생성

- **[링크](https://github.com/bin0421/NewSBB-3/commit/df3721f377b9c25868fa7c12a77c123ddbecfb61)**
- **원인** : 아무것도 없는 상태에서 시작하므로 빌드·실행할 수 있는 스프링부트 프로젝트 뼈대가 필요하다.
- **작업**
  - 추가 : Gradle 래퍼(9.8.2), `build.gradle.kts`, `settings.gradle.kts`, `.gitignore`, `.gitattributes`
  - 추가 : 의존성 `spring-boot-starter-webmvc`, `lombok`, `spring-boot-devtools`, `spring-boot-starter-webmvc-test`
  - 추가 : `BackApplication`, `application.yaml`, `README.md`, `NewSBB-3.txt`
- **설명**
  - 강사 저장소 첫 커밋과 같은 Spring Initializr 구성이다. 루트 프로젝트명 `back`, 패키지 `com.back`, Java 툴체인 25.
  - `.gitattributes` 로 `gradlew` 는 LF, `*.bat` 는 CRLF, `*.jar` 는 바이너리로 고정해 Windows 에서도 줄바꿈이 섞이지 않게 한다.
  - Initializr 가 만드는 `contextLoads` 테스트는 넣지 않는다. 이후 테스트는 모두 기능 단위로 먼저 작성한다.

### 002 JPA·H2 설정

- **[링크](https://github.com/bin0421/NewSBB-3/commit/036a384012899fefefc2bd0fd14ec386e12ab782)**
- **원인** : 글과 댓글을 저장할 DB 와 JPA 가 없다. 또 개발용 DB 와 테스트용 DB 를 나눠야 테스트가 개발 데이터를 건드리지 않는다.
- **작업**
  - 추가 : 의존성 `spring-boot-starter-data-jpa`, `spring-boot-h2console`, `h2`, `spring-boot-starter-data-jpa-test`
  - 추가 : `application-dev.yaml` (파일 DB `./db_dev`), `application-test.yaml` (메모리 DB `db_test`)
  - 추가 : `.gitignore` 에 `db_dev.*.db` 파일
  - 수정 : `application.yaml`
    - 이전 : 애플리케이션 이름만 설정
    - 이후 : 기본 프로필 `dev`, `ddl-auto: update`, SQL 출력·바인딩 값 로그, `hikari.auto-commit: false`
- **설명**
  - `spring.profiles.active: dev` 로 평소에는 `application-dev.yaml` 의 파일 DB 를 쓴다. 테스트 클래스에 `@ActiveProfiles("test")` 를 붙이면 `application-test.yaml` 이 덮어써 메모리 DB 로 바뀌고, 테스트가 끝나면 DB 가 사라진다.
  - `ddl-auto: update` 는 엔티티를 보고 테이블을 만들거나 컬럼을 추가한다. 아직 엔티티가 없으므로 이 단계에서 만들어지는 테이블은 없다.
  - `auto-commit: false` 는 커넥션을 트랜잭션 단위로만 커밋하게 해, `@Transactional` 밖에서 쓰기가 자동으로 확정되지 않게 한다.
  - `org.hibernate.orm.jdbc.bind: TRACE` 로 SQL 의 `?` 에 들어간 값까지 로그에 찍어 테스트 실패 원인을 추적하기 쉽게 한다.

### 003 공통 엔티티와 글 엔티티

- **[링크](https://github.com/bin0421/NewSBB-3/commit/abd9722b871f364bc7b2134a108aec417e4c7f58)**
- **원인** : 글을 저장할 테이블과 그 테이블을 다룰 리포지터리가 없다. 앞으로 만들 엔티티마다 id·작성일·수정일을 반복해서 적지 않도록 공통 부모도 필요하다.
- **작업**
  - 추가 : `BaseEntity` — `id`, `createDate`, `modifyDate`, id 기준 `equals()`·`hashCode()`
  - 추가 : `Post` 엔티티 — `title`, `content`, 생성자 `Post(title, content)`
  - 추가 : `PostRepository` — `JpaRepository<Post, Integer>`
  - 수정 : `BackApplication`
    - 이전 : `@SpringBootApplication`
    - 이후 : `@SpringBootApplication` + `@EnableJpaAuditing`
- **설명**
  - `BaseEntity` 는 `@MappedSuperclass` 라 자기 테이블은 없고, 필드만 자식 엔티티 테이블의 컬럼으로 물려준다. `abstract` 로 두어 직접 만들 수 없게 한다.
  - `@GeneratedValue(strategy = IDENTITY)` 는 id 를 DB 의 자동 증가 컬럼에 맡긴다. 그래서 INSERT 가 실행된 뒤에야 id 를 알 수 있다.
  - `@CreatedDate`·`@LastModifiedDate` 는 `AuditingEntityListener` 가 저장·수정 직전에 채운다. 이 리스너는 `@EnableJpaAuditing` 이 있어야 동작한다.
  - `@NoArgsConstructor` 는 JPA 가 조회 결과로 객체를 만들 때 쓰는 기본 생성자다. `ddl-auto: update` 에 의해 앱이 뜰 때 `post` 테이블이 만들어진다.

### 004 댓글 엔티티와 글-댓글 연관관계

- **[링크](https://github.com/bin0421/NewSBB-3/commit/2a647bbec3cce67f60e82ea24fe30c11ad669156)**
- **원인** : 댓글을 저장할 곳이 없고, 어떤 글의 댓글인지 연결할 방법도 없다.
- **작업**
  - 추가 : `PostComment` 엔티티 — `post`(`@ManyToOne`), `content`, 생성자 `PostComment(post, content)`
  - 추가 : `Post.comments` — `@OneToMany(mappedBy = "post", cascade = {PERSIST})`
  - 추가 : `Post.addComment(content)` — 댓글을 만들어 글의 댓글 리스트에 넣고 돌려준다
- **설명**
  - 외래키 `post_id` 는 `post_comment` 테이블에 생기고, 관계의 주인은 `PostComment.post` 다. `Post.comments` 는 `mappedBy` 로 그 관계를 거꾸로 읽기만 하는 쪽이라 컬럼을 만들지 않는다.
  - `fetch = LAZY` 는 댓글을 읽을 때 글을 바로 조회하지 않고, 실제로 글에 접근할 때 조회한다.
  - 댓글을 별도 리포지터리로 저장하지 않고 `addComment()` 로 글의 리스트에 넣는다. `cascade = PERSIST` 이므로 글이 영속 상태면 리스트에 들어간 새 댓글도 함께 저장(INSERT)된다.
  - 이 단계는 구조만 만든다. 연관관계가 실제로 동작하는지는 다음 단계의 샘플 데이터 테스트로 확인한다.

### 005-1 샘플 데이터 테스트 추가 (실패)

- **[링크](https://github.com/bin0421/NewSBB-3/commit/429feff2777a7f3fbc89a75a6b892a7d18cc15c6)**
- **원인** : 앱이 뜰 때 글 3건·댓글 5건이 들어 있어야 한다. TDD 로 진행하므로 구현보다 "무엇이 되어야 하는지"를 테스트로 먼저 적는다.
- **작업**
  - 추가 : 테스트 `BaseInitDataTest.t1` — 전체 글이 3건이고, 글마다 댓글 수를 더하면 5건인지 확인
- **설명**
  - `@SpringBootTest` 는 앱 전체 컨텍스트를 띄우므로, 앱이 뜰 때 실행되는 초기 데이터 코드도 테스트 전에 실행된다. 테스트는 그 결과가 DB 에 남아 있는지 본다.
  - `@ActiveProfiles("test")` 로 메모리 DB 를 쓰고, `@Transactional` 로 지연 로딩되는 `comments` 를 테스트 메서드 안에서 읽을 수 있게 한다.
  - 실패 확인 : 아직 초기 데이터를 넣는 코드가 없어 `Expected size: 3 but was: 0` 으로 실패한다.

### 005-2 샘플 글 3건·댓글 5건 생성

- **링크**
- **원인** : 005-1 테스트가 실패한다. 앱이 뜰 때 샘플 데이터를 넣는 코드가 없다.
- **작업**
  - 추가 : `PostService.count()`, `PostService.write(title, content)`
  - 추가 : `BaseInitData` — 앱 구동 직후 글 3건(제목 1~3)과 댓글 5건(1번 글 3건, 2번 글 2건)을 만든다
- **설명**
  - `@Bean ApplicationRunner` 는 컨텍스트가 다 뜬 뒤 한 번 실행된다. 테스트 컨텍스트에서도 실행되므로 005-1 테스트가 통과한다.
  - `work1()` 의 `@Transactional` 은 스프링 프록시를 거쳐 호출될 때만 적용된다. 같은 객체 안에서 `this.work1()` 로 부르면 프록시를 건너뛰므로, `@Lazy` 로 주입한 자기 자신(`self`)을 통해 호출한다. `@Lazy` 는 자기 자신을 주입할 때 생기는 순환 참조를 프록시로 미룬다.
  - 글은 `save()` 로 바로 영속 상태가 되고, 댓글은 `addComment()` 로 리스트에 넣기만 한다. 트랜잭션이 끝날 때 `cascade = PERSIST` 에 의해 댓글이 INSERT 된다.
  - 글이 이미 있으면(`count() > 0`) 건너뛰어, 파일 DB 를 쓰는 개발 환경에서 재시작해도 데이터가 중복되지 않는다.
  - 결과 : `BaseInitDataTest.t1` 통과.
