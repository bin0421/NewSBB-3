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

- **링크**
- **원인** : 아무것도 없는 상태에서 시작하므로 빌드·실행할 수 있는 스프링부트 프로젝트 뼈대가 필요하다.
- **작업**
  - 추가 : Gradle 래퍼(9.8.2), `build.gradle.kts`, `settings.gradle.kts`, `.gitignore`, `.gitattributes`
  - 추가 : 의존성 `spring-boot-starter-webmvc`, `lombok`, `spring-boot-devtools`, `spring-boot-starter-webmvc-test`
  - 추가 : `BackApplication`, `application.yaml`, `README.md`, `NewSBB-3.txt`
- **설명**
  - 강사 저장소 첫 커밋과 같은 Spring Initializr 구성이다. 루트 프로젝트명 `back`, 패키지 `com.back`, Java 툴체인 25.
  - `.gitattributes` 로 `gradlew` 는 LF, `*.bat` 는 CRLF, `*.jar` 는 바이너리로 고정해 Windows 에서도 줄바꿈이 섞이지 않게 한다.
  - Initializr 가 만드는 `contextLoads` 테스트는 넣지 않는다. 이후 테스트는 모두 기능 단위로 먼저 작성한다.
