---
emoji: headers/test-annotations.png
title: '[Spring] @JdbcTest, @WebMvcTest, @SpringBootTest'
date: '2022-05-29 23:00:00'
author: 써머
tags: Spring
categories: Spring
---

DB를 연동하고 스프링을 쓰기 시작하면서 어플리케이션의 구조가 복잡해졌다. 그에 따라 테스트도 다양한 어노테이션을 활용하여 작성해야 했는데, 처음에는 예제를 보며 따라했다. 그러다 각 어노테이션의 정확한 역할과 적용되는 방식이 궁금해 미션을 진행하며 사용한 세 테스트용 어노테이션을 학습했다.  

## @JdbcTest  

> JDBC 기반 컴포넌트만 테스트하는 JDBC 테스트를 위한 어노테이션이다. 이 어노테이션을 사용하면 auto-configuration이 비활성화 되고, JDBC 테스트와 관련된 configuration만 적용된다. 기본적으로 `@JdbcTest`가 붙은 테스트는 트랜젝션 처리가 되고, 각 테스트가 죵로된 후 롤백을 시행한다. 또한 명시됐거나 자동 설정된 DataSource 대신 인메모리 DB를 사용하게 된다.`@AutoConfigureTestDatabase`를 사용하면 이런 설정을 덮어쓸 수 있다.  <!--more-->
> 전체 어플리케이션 설정을 사용하면서 인메모리 DB를 사용하고 싶다면, 이 어노테이션 대신 `@SpringBootTest`와 `@AutoConfigureTestDatabase`를 조합해 사용하는 게 낫다. `JUnit 4`를 사용한다면 `@RunWith(SpringRunner.class)`를 함께 사용해야 한다.  

미션에서 `JdbcTemplate`만 주입 받으면 되는 `Dao` 객체들을 테스트하는 데 사용했다.   

```java
@JdbcTest
public class JdbcLineDaoTest {

    private final LineDao lineDao;

    @Autowired
    public JdbcLineDaoTest(JdbcTemplate jdbcTemplate) {
        this.lineDao = new JdbcLineDao(jdbcTemplate);
    }
    ...
}

@Repository
public class JdbcLineDao implements LineDao {

    private final NamedParameterJdbcTemplate jdbcTemplate;

    public JdbcLineDao(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = new NamedParameterJdbcTemplate(jdbcTemplate);
    }
    ...
}
```  

자바독에서 볼 수 있듯이 해당 어노테이션을 적용한 테스트는 인메모리 DB로 테스트를 시행하며, 트랜젝션과 자동 롤백을 지원하게 된다. 또한, 스프링 컨테이너에 전체 컴포넌트를 등록하지 않아 테스트 비용을 절감할 수 있다.  


## @WebMvcTest  

> `스프링 MVC` 컴포넌트만 테스트하기 위한 어노테이션이다. 이 어노테이션을 사용하면 auto-configuration이 비활성화 되고, MVC 테스트와 관련된 configuration들만 적용된다. (ex. @Controller, @ControllerAdvice, @JsonComponent, Converter/GenericConverter, Filter, WebMvcConfigurer and HandlerMethodArgumentResolver 빈들, 다만 **@Component, @Service, @Repository 빈들 제외**)  
> 기본적으로 `@WebMvcTest`가 사용된 테스트는 스프링 시큐리티와 MockMvc를 자동 설정한다. (HtmlUnit WebClient와 Selenium WebDriver 지원 포함) MockMvc에 대한 더욱 세밀한 설정을 위해 `@AutoConfigureMockMvc`를 사용할 수 있다. 일반적으로 `@WebMvcTest`를 사용할 때, `@Controller` 빈에 필요한 객체를 `@MockBean`이나 `@Import`를 통해 생성한다.  
> 전체 어플리케이션 configuration을 로드하면서 `MockMvc`를 사용하고 싶다면, 이 어노테이션 보다 `@SpringBootTest`와 `@AutoConfigureMockMvc`를 함께 사용하는 게 낫다. `JUnit 4`를 사용한다면 `@RunWith(SpringRunner.class)`를 함께 사용해야 한다.  

미션에서 *컨트롤러만 단위 테스트*할 때 사용했다. 괄호 안에 테스트 할 특정 클래스를 적어주면 해당 클래스만 생성된다. MVC에 관련 된 빈만 생성되고 컨트롤러에 주입되는 빈이 생성되지 않기 떄문에, `@MockBean`으로 필요한 빈을 생성해야 한다. 그렇지 않으면 `Failed to load ApplicationContext` 메시지와 함께 실행에 실패한다.  


```java
@WebMvcTest(LineController.class)
class LineControllerUnitTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private LineService lineService;
    ...
}
```

### 번외?  

테스트 할 컨트롤러를 특정하지 않으면 모든 `@Controller` 빈이 등록된다. 처음에는 테스트 대상 클래스를 표기해야만 실행되는 줄 알았다. 그런데 공식 문서로 공부하고 보니 특정하지 않아 실패한 것이 아니라, 모든 컨트롤러에 필요한 모든 서비스(테스트 내에서 사용하지 않더라도)를 생성하지 않아서 실행되지 않은 것이었다. 따라서 그럴 일은 없겠지만, 컨트롤러를 특정하지 않으면 다음과 같이 `@MockBean`으로 몽땅 등록하면 된다.  

```java
@WebMvcTest
class LineControllerUnitTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private LineService lineService;
    @MockBean
    private PathService pathService;
    @MockBean
    private StationService stationService;
    ...
    // 외의 모든 컨트롤러에서 사용되는 모든 주입받을 객체  
}  
```

## @SpringBootTest  

> 스프링부트 기반 테스트를 위한 어노테이션이다. `Spring TestContext Framework`에 더해 다음과 같은 기능을 제공한다.  
> - `@ContextConfiguration(loader=...)`가 정의되어 있지 않다면, 기본 `ContextLoader`로 `SpringBootContextLoader`를 사용    
> - 내부에 `@Configuration`이 없고 `classes`가 정의되어 있지 않다면, 자동으로 `@SpringBootConfiguration`을 탐색   
> - `properties attribute`를 이용한 `Environment` 설정  
> - `args attribute`를 통한 어플리케이션 arguments를 정의  
> - 웹 서버를 `defined` 또는 `random` 포트에서 구동 시킬지를 포함해, 여러 `webEnviroment` 모드를 설정  
> - 웹 서버를 구동하는 웹 테스트에서 `TestRestTemplate, WebTestClient` 빈을 등록  

모든 객체를 실제 빈으로 등록하고 사용하는 컨트롤러 통합 테스트와 인수 테스트에서 해당 어노테이션을 사용했다.  

```java
// 컨트롤러 통합 테스트  
@SpringBootTest
@AutoConfigureMockMvc
@Transactional
@Sql("/testSchema.sql")
public class LineControllerIntegrationTest {

    @Autowired
    private MockMvc mockMvc;
    ...
}
``` 

```java
// 인수 테스트
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@DirtiesContext(classMode = DirtiesContext.ClassMode.BEFORE_EACH_TEST_METHOD)
@Sql("/testSchema.sql")
public class AcceptanceTest {
    @LocalServerPort
    int port;

    @BeforeEach
    public void setUp() {
        RestAssured.port = port;
    }
    ...
}
```

모든 빈을 생성하기 때문에 비용과 실행 시간이 제일 많이 든다. 특히 `@DirtiesContext`로 매 테스트 마다 새로운 컨테이너를 생성하면 정말 끝내주는 실행 속도를 보여준다.  

<br/>

어플리케이션 규모가 커지고 테스트 수가 늘어나면서 각 경우에 적절한 테스트를 설정하는 것이 중요해졌다. 사실 성격이 느긋해서 테스트에 시간이 많이 걸려도 신경쓰지 않는 편이었는데... 인수테스트를 필요에 따라 `@DirtiesContext`와 함께 쓰면서 테스트에 소모되는 비용과 시간 절약을 자연스레 고려하게 되었다. 이 외에도 다양한 테스트용 어노테이션과 추가 설정 가능한 사항들이 있는데, 아는 것만 쓰지 말고 더 넓게 학습해 더 나은 효율의 테스트를 작성하고 싶다.  

```toc
```
