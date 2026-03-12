---
title: Spring Annotation
date: 2026-03-12-19:00
categories:
- Spring
tags:
- Annotation
---

> "**주석의 의미를 가지고, 자바에서는 코드 사이에 특별한 의미나 기능을 수행해준다.**"

---
## 목표
> 스프링의 어노테이션의 종류와 각각의 역할이 무엇인지 알아봅니다.

---

### Annotation

#### @SpringBootApplication

- Spring Boot를 자동으로 실행시켜주는 어노테이션이다.
- Bean 등록은 두 단계로 진행된다.
  - `@ComponentScan`을 통해 Component들을 Bean으로 등록한다.
  - `@EnableAutoConfiguration`을 통해 미리 정의해둔 자바 설정 파일들을 Bean으로 등록한다.(Bean은 스프링 IoC 컨테이너에 의해 인스턴스화 되어 조립되거나 관리되는 객체)

#### @Configuration

- 스프링 IoC Container에게 해당 클래스가 Bean 구성 클래스임을 알려주는 어노테이션이다.
- `@Bean`을 해당 클래스의 메서드에 적용하면 `@Autowired`로 빈을 부를 수 있다.

#### @EnableAutoConfiguration

- Spring Application Context를 만들 때 자동으로 설정하는 기능을 켠다.
- 만약 `tomcat-embed-core.jar`가 존재하면 톰캣 서버가 setting된다.

#### @ComponentScan
- `@Component`, `@Service`, `@Repository`, `@Controller`, `@Configuration`이 붙은 빈들을 찾아서 Context에 빈을 등록해 주는 어노테이션이다.
- bean 인스턴스를 생성한다.
- `@Component`는 아래에 포함되지 않는 요소 즉, 유틸이나 외부 API 호출 클래스 같은 곳에 사용된다.
- `@Service`는 의미상 다르게 쓰는 것이고, `@Repository` 같은 경우 DB 예외가 터지면 `DataAccessException`을 던지고, `@Controller`는 `HandlerMapping`에 등록하기에 일부 MVC 기능이 동작하지 않을 수 있고, `@Configuration`은 설정값을 정의한다.

#### @Bean
- 개발자가 직접 제어가 불가능한 외부라이브러리 등을 Bean으로 만들 때 사용되는 어노테이션이다.
```java
@Configuration
public class AppConfig {
    
    @Bean
    public MessageSender messageSender() {
        if (isProduction()) {
            return new SmsSender();
        }
        return new EmailSender();
    }
}
```

- 보통 외부 라이브러리에 많이 사용되고 메서드 기준으로 사용된다.

#### @Autowired
- 필드, setter 메서드, 생성자에 사용하며 타입에 따라 알아서 Bean을 주입해주는 역할을 한다.
- 객체에 대한 **의존성을 주입**시킨다.

##### 주입 방식에 따른 특징
```java
// 필드 주입 방식
@Service
public class UserService {

    @Autowired
    private UserRepository userRepository;
}

// setter 주입 방식
@Service
public class UserService {

    private UserRepository userRepository;

    @Autowired
    public void setUserRepository(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}

// 생성자 주입 방식
@Service
public class UserService {

    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```
- 필드 방식은 간단하지만, 테스트가 어렵고 의존성이 숨겨져 있어 요즘에는 권장하지 않는다.
- setter 방식은 선택적 의존성일 경우만 사용한다.
- 생성자 방식은 테스트도 쉽고, 안전해서 대부분 이렇게 사용한다.

#### @Controller
- MVC의 Controller로 사용되고, 클래스 선언을 단순화 시켜준다.

#### @RestController
- Controller 중 View로 응답하지 않는 컨트롤러를 말한다.
- 반환 결과를 JSON으로 반환한다.
- `@ResponseBody` 역할을 자동으로 해준다.

> API + VIEW = `@Controller`
> API = `@RestController`

#### @Service
- 비즈니스 로직을 수행하는 클래스이다.

#### @Repository
- DAO 클래스에 쓰인다.
- DB에 접근하는 클래스이다.

#### @Resource
- Bean 객체를 주입해주는데 `@Autowired`는 타입으로, `@Resource`는 이름으로 연결해준다.

```java
// @Autowired의 경우 NoUniqueBeanDefinitionException 에러 발생
@Repository
public class JpaUserRepository implements UserRepository {
}

@Repository
public class JdbcUserRepository implements UserRepository {
}

// 사용 예제
@Service
public class UserService {

    @Resource(name = "jpaUserRepository")
    private UserRepository userRepository;

}
```

- 하지만, 요즘은 `@Qualifier`나 `Primary`를 사용합니다.

#### @PreConstruct, @PostConstruct
- 의존하는 객체 생성 이후, 초기화 작업을 위해 실행해야할 메서드 앞에 붙인다.

#### @PreDestroy
- 객체를 제거하기 전, 작업을 수행해야 하는 메서드 앞에 붙인다.

#### @PropertySource
- 프로퍼티 파일을 환경변수로 로딩하게 해준다.

#### @Lazy
- **지연 로딩**을 지원한다.
- 실제로 사용될 때 스프링에서 Bean을 생성한다.

#### @Value
- 프로퍼티에서 값을 가져와 적용할 때 사용한다.

#### @RequestMapping
- URL 요청이 들어오면 어떤 메서드가 처리할 지 매핑해주는 어노테이션이다.

#### @CookieValue
- 쿠키 값을 파라미터로 전달 받고, 없으면 `500` 에러를 발생시킨다.

#### @CrossOrigin
- CORS 보안상의 문제로 다른 **origind의 AJAX 요청을 방지하기 위해** 사용한다.

```java
@CrossOrigin(origins = "http://localhost:3000")
@GetMapping("/users")
public List<User> getUsers() {
    return userService.findAll();
}
```
- 개발 환경에서는 보통 전역 설정을 하지만, 운영 환경에서는 외부 파트너 api 호출 같은 경우 자주 사용된다.

#### @ModelAttirbute
- VIEW에서 전달해 준 파라미터를 DTO의 멤버 변수로 매핑해주는 어노테이션이다.
- 태그의 `name` 값과 변수명이 일치해야 한다.

#### @GetMapping
```java
@RequestMapping(value = "/users", method = RequestMethod.GET)
public List<User> getUsers() {
}
```
- 사실 위처럼 사용이 가능하지만, 가독성을 위해 `GET`,`POST`, `PUT`, `DELETE`로 사용한다.

#### @SessionAttributes
- 세션에 데이터를 넣을 때 사용하는 어노테이션이다.

```java
@SessionAttributes("name")
```
- 위는 Key 값이 name으로 있는 값은 자동으로 세션에 저장되게 한다.

#### @Valid
- 유효성 검증이 필요한 객체임을 지정한다.

#### @RequestAttribute
- Request에 설정되어 있는 속성 값을 가져올 수 있다.

#### @RequestBody
- 요청이 온 데이터를 바로 클래스나 모델로 매핑하기 위한 어노테이션이다.
- 클라이언트가 보낸 HTTP BODY(JSON)를 객체로 변환한다.

#### @RequestHeader
- Request의 헤더 값을 가져오고, 메서드의 파라미터에 사용한다.

#### @RequestParam
- `@PathVariable`과 유사하다.
- `name=taehyun`과 같은 쿼리 파라미터를 파싱해준다.

#### @RequestPart
- Request로 온 멀티파일을 바인딩해준다.

#### @ResponseBody
- `HttpMessageConverter`를 이용하여 JSON으로 요청에 응답할 수 있게 해주는 어노테이션이다.

#### @PathVariable
- 메서드 파라미터 앞에 사용하면서 해당 URL에서 `{특정 값}`을 변수로 받아올 수 있다.

> `@RequestParam`은 특정 리소스를, `@PathVariable`은 리소스 목록이나 필터 조건이 필요할 때 사용한다.

#### @ExceptionHandler
- 해당 클래스의 예외를 캐치해서 처리한다.

#### @ControllerAdvice
- 클래스 위에 붙이고 어떤 예외를 잡아낼 것인지는 각 메서드 상단에 `@ExceptionHandler`를 붙여서 사용한다.
- 전역 예외 처리 클래스를 만든다.

#### @RestControllerAdvice
- `@ControllerAdvice` + `@ResponseBody`

#### ResponseStatus
- 사용자에게 원하는 응답 코드와 이유를 반환해주는 어노테이션이다.
- 예외처리 함수 앞에 사용한다.

#### @Transactional
- DB 트랜잭션을 설정하고 싶은 메서드에 어노테이션을 적용하면 me메서드thod 내부에서 일어나는 DB 로직이 전부 성공하게되거나 DB 접근중 하나라도 실패하면, 다시 롤백할 수 있게 해주는 어노테이션이다.
- 주로 Service에서 관리한다.

#### @Cacheable
- 메서드 앞에 지정하면 해당 메서드를 처음 호출하면 캐시에 적재하고, 그 다음 부터는 캐시에서 결과를 가져와 메서드 호출을 줄여주는 어노테이션이다.
- 보통 입력이 같으면 출력이 항상 같은 메서드에 적용한다.

#### @CachePut
- 캐시를 업데이트 하기 위해 메서드를 항상 실행하게 강제하는 어노테이션이다.

#### @CacheEvict
- 캐시에서 데이터를 제거하는 트리거로 동작하는 메서드에 붙이는 어노테이션이다.
- 캐시 설정에 만료시간을 줄 수 있다.

#### @CacheConfig
- 클래스 레벨에서 공통의 캐시 설정을 공유하는 기능이다.

#### @Scheduled
- 정해진 시간에 메서드를 실행하게 하는 기능이다.

---

### Lombok Annotation

#### @NoArgsConstructor
- 기본 생성자를 자동으로 추가한다.
- 엔티티 객체를 아무데서나 생성하지 못하게 막고, JPA만 생성할 수 있게 하기 위해 사용한다.

#### @AllArgsConstructor
- 모든 필드 값을 파라미터로 받는 생성자를 추가한다.

#### @RequiredArgsConstructor
- `final`이나 `@NonNull`인 필드 값만 파라미터로 받는 생성자를 추가한다.

#### @Getter
- 클래스 내 모든 필드의 Getter 메서드를 자동으로 생성해준다.

#### @Setter
- 클래스 내 모든 필드의 Setter 메서드를 자동으로 생성해준다.
- 엔티티는 DTO를 생성해서 DTO타입으로 받는다.(레이어 책임을 분리하고 통제 지점을 만들기 위해서)

#### @ToString
- 클래스 내 모든 필드의 `toString` 메서드를 자동 생성한다.

#### @EqualsAndHashCode
- `equals`와 `hashCode` 메서드를 오버라이딩 해주는 어노테이션이다.

#### @Builder
- 어느 필드에 어떤 값을 채워야 할지 명확하게 정하여 객체 생성 시점에 값을 채워준다.
- 필드 이름이 보이기에 가독성이 좋다.

#### @Data
- `@Getter`,  `@Setter`,  `@EqualsAndHashCode`,  `@AllArgsConstructor`을 포함한 Lombok에서 제공하는 필드와 관련된 모든 코드를 생성한다.
- 보통, DTO에 많이 사용된다.

---

### JPA Annotation

#### @Entity
- 실제 DB 테이블과 매핑될 클래스임을 나타낸다.

#### @Table
- 엔티티 클래스에 매핑할 테이블 정보를 알려준다.
- 해당 어노테이션 생략 시 클래스 명을 테이블 명으로 자동 매핑한다.

#### @Id
- 해당 테이블의 PK 필드를 의미한다.

#### @GeneratedValue
- PK의 생성 규칙을 나타낸다.

#### @Column
- 테이블의 컬럼을 나타내며 선언하지 않아도 클래스의 필드는 모두 컬럼이 된다.
- 해당 어노테이션 자동으로 컬럼명을 매핑한다.

#### @EnableJpaAuditing
- JPA를 사용하여 엔티티와 DB 테이블을 매핑할 대 공통적으로 도메인들이 가지고 있는 필드나 컬럼(생성일자, 수정일자, 식별자)를 자동으로 매핑해주는 어노테이션이다.
- SpringBootApplication 위에 적어준다.
- 그리고 `@Entitylisteners(AuditingEntityListener.class)`를 해당 엔티티에 써준다.
- 보통 BaseEntity를 만들어 상속하여 사용한다.(중복 코드 제거)