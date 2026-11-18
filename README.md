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

- **[링크](https://github.com/bin0421/NewSBB-3/commit/4cdf391205f06ef3f450fb9d59891e3e03c97219)**
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

### 006-1 글 다건조회 테스트 추가 (실패)

- **[링크](https://github.com/bin0421/NewSBB-3/commit/c7c2b7c7fea247429ad062b57b606e0cec3eb984)**
- **원인** : 글 목록을 JSON 으로 돌려주는 API 가 필요하다. 먼저 `GET /api/v1/posts` 가 어떻게 응답해야 하는지 테스트로 정한다.
- **작업**
  - 추가 : 테스트 `ApiV1PostControllerTest.t1` — `GET /api/v1/posts` 를 처리한 컨트롤러·메서드(`ApiV1PostController.getItems`), 200, 응답 배열 길이, 글마다 `id`·`createDate`·`modifyDate`·`title`·`content` 확인
- **설명**
  - `@AutoConfigureMockMvc` 의 `MockMvc` 는 서버를 띄우지 않고 스프링 MVC 에 가짜 요청을 보낸다. `perform()` 으로 요청하고 `andExpect()` 로 응답을 검사한다.
  - 기대값은 하드코딩하지 않고 `postService.findAll()` 로 DB 에서 읽어 응답과 하나씩 대조한다.
  - 날짜는 DB 저장 시 나노초 자릿수가 달라질 수 있어 앞 20자리(`yyyy-MM-ddTHH:mm:ss.`)만 `startsWith` 로 비교한다.
  - 실패 확인 : 아직 없는 `ApiV1PostController` 와 `PostService.findAll()` 을 쓰므로 테스트 컴파일 단계에서 `cannot find symbol` 로 실패한다. TDD 에서는 컴파일 실패도 "실패"로 본다.

### 006-2 글 다건조회

- **[링크](https://github.com/bin0421/NewSBB-3/commit/d1c6dc4525ecaf0650985ed8d760ce5d0257bc28)**
- **원인** : 006-1 테스트가 실패한다. 글 목록을 응답할 컨트롤러가 없다.
- **작업**
  - 추가 : `ApiV1PostController.getItems()` — `GET /api/v1/posts`, 응답 `List<PostDto>`
  - 추가 : `PostDto` — `id`, `createDate`, `modifyDate`, `title`, `content`
  - 추가 : `PostService.findAll()`
- **설명**
  - `@RestController` 는 메서드가 돌려준 객체를 Jackson 으로 JSON 으로 바꿔 응답 본문에 쓴다. 클래스의 `@RequestMapping("/api/v1/posts")` 가 이 컨트롤러 모든 주소의 앞부분이 된다.
  - 엔티티를 그대로 응답하면 `comments` 같은 연관관계까지 직렬화하다 지연 로딩·순환 참조 문제가 생기고, 엔티티 구조가 API 에 그대로 드러난다. 응답에 필요한 필드만 담은 `PostDto` 레코드로 바꿔서 돌려준다.
  - `@Transactional(readOnly = true)` 는 조회 전용 트랜잭션이다. 변경 감지를 하지 않아 가볍다.
  - 결과 : `ApiV1PostControllerTest.t1` 통과.

### 007-1 글 단건조회 테스트 추가 (실패)

- **[링크](https://github.com/bin0421/NewSBB-3/commit/d3f7d5c07aa505c9e712b4e3dd3805cf256f0237)**
- **원인** : 글 하나를 id 로 조회하는 API 가 필요하다. 먼저 `GET /api/v1/posts/{id}` 의 응답을 테스트로 정한다.
- **작업**
  - 추가 : 테스트 `ApiV1PostControllerTest.t2` — `GET /api/v1/posts/1` 을 처리한 메서드(`getItem`), 200, 1번 글의 `id`·`createDate`·`modifyDate`·`title`·`content` 확인
- **설명**
  - 샘플 데이터가 항상 먼저 들어가므로 1번 글은 반드시 있다. 기대값은 `postService.findById(1)` 로 읽는다.
  - 다건조회와 달리 응답이 배열이 아니라 객체 하나이므로 `$.title` 처럼 루트에서 바로 필드를 꺼낸다.
  - 실패 확인 : 아직 없는 `PostService.findById()` 를 써서 테스트 컴파일 단계에서 `cannot find symbol` 로 실패한다.

### 007-2 글 단건조회

- **[링크](https://github.com/bin0421/NewSBB-3/commit/56be133589669291cdb096b550ba3ab916c7b5ee)**
- **원인** : 007-1 테스트가 실패한다. id 로 글을 찾는 서비스 메서드와 단건조회 액션이 없다.
- **작업**
  - 추가 : `ApiV1PostController.getItem(id)` — `GET /api/v1/posts/{id}`, 응답 `PostDto`
  - 추가 : `PostService.findById(id)` — `Optional<Post>` 를 돌려준다
- **설명**
  - `@GetMapping("/{id}")` 의 `{id}` 자리 값을 `@PathVariable int id` 가 받아 정수로 바꾼다.
  - `findById()` 는 글이 없을 수도 있음을 `Optional` 로 드러낸다. 지금은 `.get()` 으로 바로 꺼내므로, 없는 id 를 요청하면 `NoSuchElementException` 이 난다. 이 문제는 008 에서 테스트로 드러내고 고친다.
  - 결과 : `ApiV1PostControllerTest.t2` 통과.

### 008-1 없는 글 단건조회 테스트 추가 (실패)

- **[링크](https://github.com/bin0421/NewSBB-3/commit/90b8f80515bef5dd509e338545930d9e04ebf24c)**
- **원인** : 없는 글을 조회하면 서버 내부 오류가 난다. 클라이언트에게는 "없다"는 뜻의 404 와 정해진 형식의 본문을 돌려줘야 한다.
- **작업**
  - 추가 : 테스트 `ApiV1PostControllerTest.t3` — `GET /api/v1/posts/{Integer.MAX_VALUE}` 가 404, `resultCode` `"404-1"`, `msg` `"해당 데이터가 존재하지 않습니다."` 로 응답하는지 확인
- **설명**
  - 절대 만들어질 수 없는 id 로 `Integer.MAX_VALUE` 를 쓴다.
  - 오류 응답도 `resultCode`·`msg` 를 갖는 같은 형식으로 정한다. 이 형식은 다음 단계에서 `RsData` 로 만든다.
  - 실패 확인 : `findById(id).get()` 에서 난 `NoSuchElementException` 을 처리하는 곳이 없어 `ServletException: Request processing failed: java.util.NoSuchElementException: No value present` 로 실패한다.

### 008-2 없는 데이터 404 응답

- **[링크](https://github.com/bin0421/NewSBB-3/commit/903f103fbaa36db7964786663495cbfbf7d19d93)**
- **원인** : 008-1 테스트가 실패한다. 컨트롤러에서 난 `NoSuchElementException` 을 받아 404 응답으로 바꿔 주는 곳이 없다.
- **작업**
  - 추가 : `RsData<T>` — 응답 공통 형식 `resultCode`, `msg`, `data` (+ JSON 에서 빠지는 `statusCode`)
  - 추가 : `GlobalExceptionHandler.handle(NoSuchElementException)` — 404, `RsData("404-1", "해당 데이터가 존재하지 않습니다.")`
- **설명**
  - `@ControllerAdvice` 클래스의 `@ExceptionHandler` 는 모든 컨트롤러에서 던져진 해당 예외를 가로챈다. 액션 메서드마다 `try-catch` 를 쓰지 않고 한 곳에서 처리한다.
  - `ResponseEntity` 로 본문(`RsData`)과 HTTP 상태코드(`NOT_FOUND`)를 함께 정한다. 예외 처리 메서드는 컨트롤러가 아니라서 본문을 JSON 으로 쓰려면 `ResponseEntity` 가 필요하다.
  - `RsData` 는 `resultCode` 의 `-` 앞 숫자를 `statusCode` 로 계산해 둔다. `@JsonIgnore` 로 JSON 에는 나가지 않는다. `data` 가 없는 응답은 `RsData<Void>` 로 표현한다.
  - 예외가 나도 `handler()` 는 원래 요청을 맡았던 `getItem` 으로 남으므로, 테스트의 컨트롤러·메서드 검사도 통과한다.
  - 결과 : `ApiV1PostControllerTest.t3` 통과.

### 009-1 글 작성 테스트 추가 (실패)

- **[링크](https://github.com/bin0421/NewSBB-3/commit/ee5a490145a94a1fe0963d0216fd4a0eeeb40b30)**
- **원인** : 글을 새로 작성하는 API 가 필요하다. 먼저 `POST /api/v1/posts` 의 요청 본문과 응답을 테스트로 정한다.
- **작업**
  - 추가 : 테스트 `ApiV1PostControllerTest.t4` — JSON 본문 `{"title", "content"}` 로 `POST /api/v1/posts` 요청 후 처리한 메서드(`write`), 201, `resultCode` `"201-1"`, `msg` `"%d번 글이 작성되었습니다."`, `data` 에 담긴 새 글의 `id`·날짜·`title`·`content` 확인
- **설명**
  - 생성 요청의 성공은 200 이 아니라 201(Created)로 정한다. 응답은 `RsData` 형식으로 감싸고 작성된 글을 `data` 에 담는다.
  - 새 글의 id 는 미리 알 수 없으므로, 요청 뒤 가장 최근 글(`findLatest()`)을 DB 에서 꺼내 응답의 id·메시지와 대조한다.
  - `@Transactional` 테스트라 끝나면 작성한 글이 롤백되어 다른 테스트에 영향을 주지 않는다.
  - 실패 확인 : 아직 없는 `PostService.findLatest()` 를 써서 테스트 컴파일 단계에서 `cannot find symbol` 로 실패한다.

### 009-2 글 작성

- **[링크](https://github.com/bin0421/NewSBB-3/commit/87eb361db01779d6e20c4136ca1986397bba0aca)**
- **원인** : 009-1 테스트가 실패한다. 글을 작성하는 액션이 없고, `RsData` 의 `resultCode` 에 맞춰 HTTP 상태코드를 201 로 바꿔 줄 방법도 없다.
- **작업**
  - 추가 : `ApiV1PostController.write()` — `POST /api/v1/posts`, 요청 본문 `PostWriteReqBody(title, content)`, 응답 `RsData<PostDto>`
  - 추가 : `ResponseAspect` — 컨트롤러가 돌려준 `RsData` 의 `statusCode` 를 HTTP 응답코드로 설정
  - 추가 : `PostService.findLatest()`, `PostRepository.findFirstByOrderByIdDesc()`
- **설명**
  - `@RequestBody` 는 JSON 본문을 레코드 `PostWriteReqBody` 로 바꾼다. 레코드 필드는 같은 클래스 안이라 `reqBody.title` 처럼 바로 읽는다.
  - `save()` 는 `IDENTITY` 전략이라 INSERT 를 바로 실행하므로, 저장 직후 `post.getId()` 로 새 id 를 응답 메시지에 쓸 수 있다.
  - `ResponseAspect` 는 `@Around` 로 `@RestController` 의 매핑 메서드 중 `RsData` 를 돌려주는 것만 감싼다. 메서드 실행 뒤 `"201-1"` 에서 계산된 201 을 `HttpServletResponse` 에 설정한다. 그래서 액션은 `ResponseEntity` 없이 `resultCode` 만으로 상태코드를 정한다. 주입된 `HttpServletResponse` 는 요청마다 현재 응답을 가리키는 프록시다.
  - `findFirstByOrderByIdDesc()` 는 메서드 이름으로 쿼리를 만드는 Spring Data JPA 기능으로, id 내림차순 첫 번째 글을 찾는다.
  - 아직 요청 값 검증은 없다. 빈 제목도 저장되는 문제는 010 에서 테스트로 드러내고 고친다.
  - 결과 : `ApiV1PostControllerTest.t4` 통과.

### 010-1 제목 없는 글 작성 테스트 추가 (실패)

- **[링크](https://github.com/bin0421/NewSBB-3/commit/4e572b10505d1a89bce70d99f8df05047ea90938)**
- **원인** : 지금은 제목이 빈 글도 저장된다. 잘못된 요청은 400 과 무엇이 잘못됐는지 알려 주는 메시지로 거절해야 한다.
- **작업**
  - 추가 : 테스트 `ApiV1PostControllerTest.t5` — 제목이 `""` 인 작성 요청이 400, `resultCode` `"400-1"`, `msg` 에 제목의 `NotBlank`·`Size` 위반 두 줄이 담기는지 확인
- **설명**
  - 메시지 형식은 `필드-제약이름-기본메시지` 한 줄씩, 정렬해서 줄바꿈으로 잇는다. 빈 문자열은 "공백이면 안 됨"과 "2~100자" 두 제약을 동시에 어긴다.
  - 텍스트 블록(`"""`)의 들여쓰기와 끝 줄바꿈은 `stripIndent().trim()` 으로 지워 비교한다.
  - 실패 확인 : 검증이 없어 그대로 저장되므로 `Status expected:<400> but was:<201>` 로 실패한다.

### 010-2 제목 검증

- **[링크](https://github.com/bin0421/NewSBB-3/commit/936a31388ab7de719bbcc81d2439798f6a78d535)**
- **원인** : 010-1 테스트가 실패한다. 요청 본문의 제목을 검사하지 않고, 검사에 실패했을 때 400 으로 응답하는 곳도 없다.
- **작업**
  - 추가 : 의존성 `spring-boot-starter-validation`, `spring-boot-starter-validation-test`
  - 추가 : `GlobalExceptionHandler.handle(MethodArgumentNotValidException)` — 400, `RsData("400-1", 위반 목록)`
  - 수정 : `PostWriteReqBody.title`
    - 이전 : 제약 없음
    - 이후 : `@NotBlank`, `@Size(min = 2, max = 100)`
  - 수정 : `ApiV1PostController.write()` 의 요청 본문
    - 이전 : `@RequestBody PostWriteReqBody reqBody`
    - 이후 : `@RequestBody @Valid PostWriteReqBody reqBody`
- **설명**
  - `@Valid` 가 붙은 파라미터는 JSON 을 레코드로 바꾼 직후 Bean Validation 으로 검사된다. 위반이 있으면 액션 메서드를 실행하지 않고 `MethodArgumentNotValidException` 을 던진다.
  - 예외의 `BindingResult` 에서 필드 오류마다 `필드-제약이름-기본메시지` 를 만들고, 순서가 매번 같도록 정렬해 줄바꿈으로 잇는다.
  - 기본 메시지는 요청 로케일을 따른다. MockMvc 요청의 기본 로케일이 영어라 `must not be blank` 같은 영어 메시지가 나온다.
  - 내용(`content`)에는 아직 제약을 걸지 않았다. 010-3 에서 테스트로 먼저 드러낸다.
  - 결과 : `ApiV1PostControllerTest.t5` 통과.

### 010-3 내용 없는 글 작성 테스트 추가 (실패)

- **[링크](https://github.com/bin0421/NewSBB-3/commit/d55e18687aa23127a0b4509f31c43da0f4e65233)**
- **원인** : 제목은 검사하지만 내용이 빈 글은 여전히 저장된다.
- **작업**
  - 추가 : 테스트 `ApiV1PostControllerTest.t6` — 내용이 `""` 인 작성 요청이 400, `resultCode` `"400-1"`, `msg` 에 내용의 `NotBlank`·`Size`(2~5000자) 위반 두 줄이 담기는지 확인
- **설명**
  - 내용은 제목보다 길 수 있으므로 최대 길이를 5000 자로 정한다.
  - 실패 확인 : 내용에는 제약이 없어 그대로 저장되므로 `Status expected:<400> but was:<201>` 로 실패한다.

### 010-4 내용 검증

- **[링크](https://github.com/bin0421/NewSBB-3/commit/a7b036a1f2878b58f98b4913eacaf10d78aa6f69)**
- **원인** : 010-3 테스트가 실패한다. 내용에 검증 제약이 없다.
- **작업**
  - 수정 : `PostWriteReqBody.content`
    - 이전 : 제약 없음
    - 이후 : `@NotBlank`, `@Size(min = 2, max = 5000)`
- **설명**
  - 010-2 에서 만든 `@Valid` 와 `MethodArgumentNotValidException` 처리가 그대로 쓰이므로 제약 애너테이션만 추가하면 된다. 위반 메시지 형식도 제목과 같다.
  - 결과 : `ApiV1PostControllerTest.t6` 통과.

### 010-5 잘못된 JSON 글 작성 테스트 추가 (실패)

- **[링크](https://github.com/bin0421/NewSBB-3/commit/2959a4e2e35e3c1c4a76f718701429d16d2c9768)**
- **원인** : 문법이 깨진 JSON 을 보내면 스프링 기본 처리로 400 은 나가지만 본문이 비어 있다. 다른 오류와 같은 `RsData` 형식으로 알려 줘야 한다.
- **작업**
  - 추가 : 테스트 `ApiV1PostControllerTest.t7` — 따옴표가 빠진 JSON 으로 작성 요청 시 400, `resultCode` `"400-1"`, `msg` `"요청 본문이 올바르지 않습니다."` 확인
- **설명**
  - JSON 을 레코드로 바꾸는 단계에서 실패하므로 검증(`@Valid`)까지 가지 못한다. 그래서 010-2 의 처리로는 잡히지 않는다.
  - 실패 확인 : 상태코드는 기본 처리(`DefaultHandlerExceptionResolver`)로 400 이 되지만 본문이 없어 `No value at JSON path "$.resultCode"` 로 실패한다.

### 010-6 잘못된 요청 본문 400 응답

- **[링크](https://github.com/bin0421/NewSBB-3/commit/1fa2f865c3ed2204791a63e29585a76d3b545750)**
- **원인** : 010-5 테스트가 실패한다. JSON 을 읽지 못했을 때 나는 예외를 `RsData` 형식으로 바꿔 주는 곳이 없다.
- **작업**
  - 추가 : `GlobalExceptionHandler.handle(HttpMessageNotReadableException)` — 400, `RsData("400-1", "요청 본문이 올바르지 않습니다.")`
- **설명**
  - 요청 본문을 객체로 바꾸는 `HttpMessageConverter` 가 JSON 을 해석하지 못하면 `HttpMessageNotReadableException` 을 던진다. `@ExceptionHandler` 가 스프링 기본 처리보다 먼저 이 예외를 잡아 본문을 채운다.
  - 파싱 오류의 상세 내용(줄·열 위치, 내부 클래스 이름)은 클라이언트에게 쓸모가 적고 내부 구조를 드러내므로, 고정된 메시지만 돌려준다.
  - 결과 : `ApiV1PostControllerTest.t7` 통과. 글 작성의 성공·실패 응답이 모두 `RsData` 형식이 됐다.

### 011-1 글 수정 테스트 추가 (실패)

- **[링크](https://github.com/bin0421/NewSBB-3/commit/fdcd6b4de089a0de2fad09b8915895e3b0914870)**
- **원인** : 작성한 글의 제목·내용을 고치는 API 가 필요하다. 먼저 `PUT /api/v1/posts/{id}` 의 응답과 결과를 테스트로 정한다.
- **작업**
  - 추가 : 테스트 `ApiV1PostControllerTest.t8` — 1번 글을 `{"title": "제목 new", "content": "내용 new"}` 로 수정 요청 후 처리한 메서드(`modify`), 200, `resultCode` `"200-1"`, `msg` `"1번 글이 수정되었습니다."` 확인, 다시 조회한 1번 글의 제목·내용이 바뀌었는지 확인
- **설명**
  - 수정은 기존 자원을 통째로 바꾸므로 `PUT` 을 쓴다. 돌려줄 데이터가 없어 `data` 는 비운다.
  - 참조 테스트는 응답만 검사한다. 응답 메시지만 만들고 실제로 고치지 않는 구현도 통과하지 않도록, 마지막에 글의 제목·내용을 직접 확인하는 검사를 더했다.
  - 실패 확인 : `PUT` 을 받는 매핑이 없어 요청을 맡을 핸들러가 없으므로 `No handler` 로 실패한다. (응답은 405 Method Not Allowed)

### 011-2 글 수정

- **[링크](https://github.com/bin0421/NewSBB-3/commit/e19b3f579062a12d86f46596b68da1f5c2dafcb6)**
- **원인** : 011-1 테스트가 실패한다. `PUT /api/v1/posts/{id}` 를 받는 액션과 글을 고치는 메서드가 없다.
- **작업**
  - 추가 : `ApiV1PostController.modify(id, reqBody)` — `PUT /api/v1/posts/{id}`, 요청 본문 `PostModifyReqBody(title, content)`, 응답 `RsData<Void>`
  - 추가 : `Post.modify(title, content)`, `PostService.modify(post, title, content)`
- **설명**
  - 수정 요청 본문도 작성과 같은 제약(`@NotBlank`, `@Size`)을 건다. 요청 레코드를 작성용과 나눠 두어 나중에 한쪽 규칙만 바뀌어도 서로 영향을 주지 않는다.
  - `save()` 를 다시 부르지 않는다. 트랜잭션 안에서 조회한 엔티티는 영속성 컨텍스트가 관리하므로, 필드만 바꾸면 트랜잭션이 끝날 때 변경 감지(dirty checking)로 UPDATE 가 나간다. 이때 `@LastModifiedDate` 로 `modifyDate` 도 갱신된다.
  - 없는 id 로 수정을 요청하면 `findById().get()` 에서 `NoSuchElementException` 이 나고 008-2 의 처리로 404 가 된다.
  - 결과 : `ApiV1PostControllerTest.t8` 통과.

### 012-1 글 삭제 테스트 추가 (실패)

- **[링크](https://github.com/bin0421/NewSBB-3/commit/0bb38186e9ee6f6b39fa34245f8dc4edc9b6e95a)**
- **원인** : 글을 지우는 API 가 필요하다. 먼저 `DELETE /api/v1/posts/{id}` 의 응답과 결과를 테스트로 정한다.
- **작업**
  - 추가 : 테스트 `ApiV1PostControllerTest.t9` — 1번 글 삭제 요청 후 처리한 메서드(`delete`), 200, `resultCode` `"200-1"`, `msg` `"1번 글이 삭제되었습니다."` 확인, 전체 글 수가 2건인지 확인
- **설명**
  - 1번 글에는 샘플 댓글 3건이 달려 있다. 댓글이 있는 글을 지우는 경우를 일부러 고른 것이다.
  - 참조 테스트는 응답만 검사한다. 여기서는 마지막에 `postService.count()` 를 더했다. JPQL 조회 전에 JPA 가 쌓아 둔 DELETE 를 먼저 DB 로 보내므로(자동 flush), 삭제가 DB 제약 조건까지 통과하는지 테스트 안에서 드러난다.
  - 실패 확인 : `DELETE` 를 받는 매핑이 없어 `No handler` 로 실패한다.

### 012-2 글 삭제 (외래키 위반 실패)

- **[링크](https://github.com/bin0421/NewSBB-3/commit/4033549905a1fb1c157f29b0d789101de1311ca1)**
- **원인** : 012-1 테스트가 실패한다. `DELETE /api/v1/posts/{id}` 를 받는 액션이 없다.
- **작업**
  - 추가 : `ApiV1PostController.delete(id)` — `DELETE /api/v1/posts/{id}`, 응답 `RsData<Void>`
  - 추가 : `PostService.delete(post)`
- **설명**
  - 삭제는 조회와 같은 주소에 HTTP 메서드만 `DELETE` 로 바꿔 표현한다. `"200-1"` 이므로 `ResponseAspect` 가 200 으로 응답한다.
  - 액션 응답까지는 기대대로 나온다. 하지만 `post_comment.post_id` 가 1번 글을 가리키는 댓글 3건이 남아 있어, 글만 지우는 DELETE 는 외래키 제약을 어긴다.
  - 실패 확인 : 테스트의 `count()` 가 DELETE 를 DB 로 보내는 순간 `DataIntegrityViolationException: Referential integrity constraint violation ... POST_COMMENT FOREIGN KEY(POST_ID) REFERENCES PUBLIC.POST(ID)` 로 실패한다. 실제 서버였다면 트랜잭션 커밋 때 같은 오류로 500 이 났을 문제를 테스트가 미리 잡은 것이다.

### 012-3 글 삭제 시 댓글 함께 삭제

- **[링크](https://github.com/bin0421/NewSBB-3/commit/930aa0aa806337a98427cbf2ee14067ef03e4a6a)**
- **원인** : 012-2 에서 댓글이 달린 글을 지우면 외래키 제약 위반으로 실패한다.
- **작업**
  - 수정 : `Post.comments` 의 cascade
    - 이전 : `cascade = {PERSIST}`
    - 이후 : `cascade = {PERSIST, REMOVE}`
- **설명**
  - `CascadeType.REMOVE` 는 부모 엔티티를 `remove` 할 때 `comments` 리스트의 자식들에게도 `remove` 를 전파한다. JPA 는 자식(댓글) DELETE 를 먼저, 부모(글) DELETE 를 나중에 보내므로 외래키 제약을 어기지 않는다.
  - 댓글 수만큼 DELETE 가 나가지만, 이 저장소의 댓글 규모에서는 문제가 되지 않는다.
  - 결과 : `ApiV1PostControllerTest.t9` 통과. 글 CRUD(다건조회·단건조회·작성·수정·삭제)가 모두 테스트로 확인됐다.

### 013-1 댓글 다건조회 테스트 추가 (실패)

- **[링크](https://github.com/bin0421/NewSBB-3/commit/a64803838f82353629edb8c2f70818167d64834b)**
- **원인** : 이제 댓글 API 가 필요하다. 첫 번째로 한 글의 댓글 목록을 주는 `GET /api/v1/posts/{postId}/comments` 의 응답을 테스트로 정한다.
- **작업**
  - 추가 : 테스트 `ApiV1PostCommentControllerTest.t1` — 1번 글의 댓글 목록 요청을 처리한 컨트롤러·메서드(`ApiV1PostCommentController.getItems`), 200, 배열 길이, 댓글마다 `id`·`createDate`·`modifyDate`·`content` 확인
- **설명**
  - 댓글은 글에 딸린 자원이므로 주소에 글 id 를 넣는다. 기대값도 `post.getComments()` 로 글을 통해 읽는다.
  - 테스트 클래스 설정(`test` 프로필, MockMvc, `@Transactional`)은 글 테스트와 같다.
  - 실패 확인 : 아직 없는 `ApiV1PostCommentController` 를 써서 테스트 컴파일 단계에서 `cannot find symbol` 로 실패한다.

### 013-2 댓글 다건조회

- **[링크](https://github.com/bin0421/NewSBB-3/commit/361258854a48769b193299c61985ba9d5d622a1a)**
- **원인** : 013-1 테스트가 실패한다. 댓글 API 를 맡을 컨트롤러가 없다.
- **작업**
  - 추가 : `ApiV1PostCommentController.getItems(postId)` — `GET /api/v1/posts/{postId}/comments`, 응답 `List<PostCommentDto>`
  - 추가 : `PostCommentDto` — `id`, `createDate`, `modifyDate`, `content`
- **설명**
  - 클래스 레벨 `@RequestMapping("/api/v1/posts/{postId}/comments")` 의 `{postId}` 도 메서드 파라미터의 `@PathVariable` 로 받을 수 있다.
  - 댓글 전용 리포지터리·서비스 없이 `PostService` 로 글을 찾고 `post.getComments()` 로 댓글을 꺼낸다. 댓글은 항상 글을 통해 다룬다는 참조의 설계를 따른다.
  - `comments` 는 지연 로딩이라 `@Transactional(readOnly = true)` 안에서 읽어야 한다. 트랜잭션 밖에서 읽으면 `LazyInitializationException` 이 난다.
  - `PostCommentDto` 에는 `post` 를 넣지 않는다. 댓글이 글을, 글이 다시 댓글을 직렬화하는 순환을 막는다.
  - 결과 : `ApiV1PostCommentControllerTest.t1` 통과.

### 014-1 댓글 단건조회 테스트 추가 (실패)

- **링크**
- **원인** : 댓글 하나를 조회하는 `GET /api/v1/posts/{postId}/comments/{id}` 가 필요하다. 먼저 응답을 테스트로 정한다.
- **작업**
  - 추가 : 테스트 `ApiV1PostCommentControllerTest.t2` — 1번 글의 1번 댓글 요청을 처리한 메서드(`getItem`), 200, 댓글의 `id`·`createDate`·`modifyDate`·`content` 확인
- **설명**
  - 기대값은 글의 댓글 리스트에서 id 로 찾는 `post.findCommentById(id)` 로 읽는다. 댓글을 글을 통해서 찾는다는 규칙을 테스트에서부터 쓴다.
  - 실패 확인 : 아직 없는 `Post.findCommentById()` 를 써서 테스트 컴파일 단계에서 `cannot find symbol` 로 실패한다.
