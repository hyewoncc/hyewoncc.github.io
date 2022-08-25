---
emoji: headers/spring-study-1.png
title: '[Spring] DB 연결 없는 컨트롤러 테스트에서 시작한 스프링 공식 문서 공부'
date: '2022-04-25 11:40:00'
author: 써머
tags: Spring 우아한테크코스
categories: Spring 우아한테크코스
---

## 사건의 발단 

우테코 레벨1 마지막 미션인 웹 체스 강의에서 [테스트 더블](https://tecoble.techcourse.co.kr/post/2020-09-19-what-is-test-double/)을 이용해 DB 연결 없이 DAO 객체를 테스트 하는 걸 배웠다. 웹 체스에 스프링 프레임워크를 적용하고 컨트롤러를 짜는데, **컨트롤러도 DB 연결 없이 테스트 할 수 있지 않을까?** 하는 생각이 들었다. 체스방의 정보(id, name)을 담은 `room` 테이블을 대신 할 `FakeRoomRepository`와 `RoomRepository`를 주입받는 `RoomService`를 만든 것 까진 아주 좋았다. 

<!--more--> 

```java
public class FakeRoomRepository implements RoomRepository {

    private final Map<Integer, String> database = new HashMap<>();
    private int autoIncrementId = 0;

    @Override
    public int save(String name) {
        autoIncrementId++;
        database.put(autoIncrementId, name);
        return autoIncrementId;
    }

    @Override
    public Optional<RoomDto> findById(int roomId) {
        return Optional.ofNullable(database.get(roomId))
                .map(name -> new RoomDto(roomId, name));
    }
}
```  

```java
@Service
public class RoomService {

    private final RoomRepository roomRepository;

    public RoomService(RoomRepository roomRepository) {
        this.roomRepository = roomRepository;
    }
    ...
}
``` 

문제는 컨트롤러를 짜며 생겼다. 당초 계획은 `FakeRoomRepository`를 주입받은 `RoomService`를 테스트 환경에서만 컨트롤러에 주입하기였다. 하지만 스프링에서는 사용자가 직접 new로 객체를 생성하지 않고 `@Component`로 등록된 클래스를 스캔해 빈(Bean)으로 생성한다. 그래서 테스트에서만 필요한 객체를 주입 할 수가 없었다.  

### 일단 해결은 했으나...
 
페어 더즈와 방법을 고민하다 지나가던 제이슨이 도와줘서 문제를 해결할 수 있었다. 먼저 test 폴더 하위에 `application.properties`를 추가로 작성해 프로덕션과 다른 설정이 적용되게 한다.  

```properties
spring.main.allow-bean-definition-overriding=true
```

[공식 문서](https://docs.spring.io/spring-boot/docs/current/reference/html/application-properties.html#application-properties.core.spring.main.allow-bean-definition-overriding)에 따르면, 해당 설정은 이미 존재하는 빈 정의(definition)와 같은 이름을 다시 정의했을 때, 존재하던 정의가 덮어 씌워짐(overriding)을 허용하는 설정이다. 이를 허용하고 테스트 코드에서 `RoomRepository`를 재정의 하였다.  

```java
@TestConfiguration
class TestConfig {
    @Bean
    public RoomRepository roomRepository() {
        return new FakeRoomRepository();
    }
}

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Import(TestConfig.class)
class SpringChessControllerTest {

    @LocalServerPort
    int port;

    @Autowired
    ApplicationContext context;

    @BeforeEach
    void setUp() {
        RestAssured.port = port;
    }

    @DisplayName("부적절한 이름이 입력되면 400 에러 발생")
    @ParameterizedTest
    @ValueSource(strings = {"", "16자를넘는방이름은안되니까돌아가"})
    void nameException(String name) {
        RestAssured.given().log().all()
                .formParam("name", name)
                .when().post("/rooms")
                .then().log().all()
                .statusCode(HttpStatus.BAD_REQUEST.value());
    }

}
```

테스트 코드 실행 시, 프로덕션 코드의 실제 DB를 사용하는 `RoomRepositoryImpl`빈이 `FakeRoomRepository`로 덮어 씌워지고, 컨트롤러를 DB 연결 없이 사용할 수 있게 된다.  
스프링을 처음 써본 건 아니지만, 내부 작동 원리는 잘 모르고 있었음을 위 테스트를 시도하다 인지하게 되었다. 그래서 공식 문서 탐방을 했다. 각잡고 공식 문서를 꼼꼼히 읽어 본 건 처음이다.  

## Spring

### @Bean과 @Component의 차이  

@Component 어노테이션이 붙은 클래스는 컴포넌트 스캔의 대상이 되어 스프링 어플리케이션 실행 시, 빈으로 생성된다는 것은 알았다. 하지만 @Bean 어노테이션과 무슨 차이가 있는 지는 몰랐다. @Bean이 @Component의 상위 어노테이션이라 생각했는데 틀렸다.  

[스프링 공식 문서](https://docs.spring.io/spring-framework/docs/4.3.12.RELEASE/spring-framework-reference/htmlsingle/#beans-java-bean-annotation)의 `7.12.3 Using the @Bean annotation`에는 이렇게 나온다.  

> @Bean은 메서드 레벨의 어노테이션이며 XML \<bean/> 엘리먼트의 직접적인 유사체입니다. 이 어노테이션은 [init-method](https://docs.spring.io/spring-framework/docs/4.3.12.RELEASE/spring-framework-reference/htmlsingle/#beans-factory-lifecycle-initializingbean), [destroy-method](https://docs.spring.io/spring-framework/docs/4.3.12.RELEASE/spring-framework-reference/htmlsingle/#beans-factory-lifecycle-disposablebean),[autowiring](https://docs.spring.io/spring-framework/docs/4.3.12.RELEASE/spring-framework-reference/htmlsingle/#beans-factory-autowire),name 같이 \<bean/>에서 제공하는 일부 속성을 지원합니다.  
> @Bean 어노테이션은 @Configuration와 @Component가 붙은 클래스 내부에서 사용할 수 있습니다.   

그리고 빈 선언에 앞선 컨트롤러 테스트에서 작성한 코드와 유사한 예시 코드가 보여 이 섹션도 마저 읽었다.  

> 빈은 선언하려면 메서드에 @Bean만 붙이면 됩니다. ApplicationContext 내에서 메서드의 리턴값으로 특정된 빈 정의를 등록하게 됩니다. 기본적으로 메서드 이름이 빈 이름이 됩니다. @Bean 메서드 선언의 간단한 예는 다음과 같습니다.  

```java
@Configuration
public class AppConfig {

    @Bean
    public TransferServiceImpl transferService() {
        return new TransferServiceImpl();
    }
}
```  

> 이 configuration은 다음 Spring XmL과 완전히 동일하게 작동합니다.   

```xml
<beans>
    <bean id="transferService" class="com.acme.TransferServiceImpl"/>
</beans>
```  

> 둘 다 `ApplicationContext`내에서 유효한 `TransferServiceImpl`의 인스턴트를 주입받은 `transferService`라는 이름의 빈을 만듭니다. @Bean으로 인터페이스(또는 베이스 클래스)를 리턴하는 메서드 역시 선언 가능합니다.  

```java
@Configuration
public class AppConfig {

    @Bean
    public TransferService transferService() {
        return new TransferServiceImpl();
    }
}
```  

> 그러나 타입 예측에 대한 가시성이 특정한 인터페이스 타입(TransferService)으로 제한되고, 전체 유형(TransferServiceImpl)은 컨테이너에 이 클래스에 영향받는 싱글턴 빈이 인스턴스화 될 때 알려집니다. 이른 초기화 되는 싱글턴 빈들은 선언 순서에 따라 인스턴스화 되기에, 타입 매칭이 다르게 될 수도 있습니다. 이는 다른 컴포넌트가 "transferService"가 인스턴스화 된 뒤에야 생성되는 선언되지 않은 타입(예: @Autowired TransferServiceImpl)과 매치하려 할 때 일어납니다.  
> 서비스 인터페이스로 선언된 유형을 일관되게 참조하는 경우, @Bean 반환 타입은 디자인 의도에 맞게 동작할 것입니다. 하지만 여러 인터페이스를 구현하거나, 잠재적으로 구현 유형에 따라 참조되는 컴포넌트의 경우에는, 가능한 제일 특정한 리턴 타입을 사용하는 것이 안정합니다. (적어도 빈을 참조하는 주입 시점에서 필요한 만큼은 구체적이어야 합니다.)  

결국 @Bean은 메서드에, @Component는 클래스에 쓰인다고 축약할 수 있다. 실제로 둘을 반대로 써보면 컴파일 에러가 나는 것을 처음 눈으로 확인했다. 둘의 차이는 [개발자 이동욱님 블로그 포스팅](https://jojoldu.tistory.com/27)에도 서치하니 나왔다. ~~문서를 읽기 전에 봤다면 좋았을텐데...~~ [DevAndy - Bean과 Component 차이](https://youngjinmo.github.io/2021/06/bean-component/)에 차이에 좀 더 중점을 둔 상세한 설명이 있다.  

### @Repository, @Service, @Controller는 내부가 같다  

이 세가지 어노테이션을 쓰면서 나는 이 셋에게 각각 이름값 하는 기능이 있을 거라 생각했다. 그런데 클래스 파일을 보니 내부 구조가 **동일** 했다.  

```java
// Repository, Controller도 동일  
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Service {
	@AliasFor(annotation = Component.class)
	String value() default "";
}
```

모두 동일하게 @Component를 써도 동작했고, 내용물과 다른 어노테이션을 붙여도 당연하게도 문제가 없었다. 어노테이션과 코드는 상관이 없으니 생각해보면 이상할 게 없는 일이다. 그런데 이제까지 한 번도 의심없이 `그냥 컨트롤러에는 @Controller 붙이는 거다~` 했기에 충격이었다.  

[스프링 공식 문서](https://docs.spring.io/spring-framework/docs/4.3.12.RELEASE/spring-framework-reference/htmlsingle/#beans-stereotype-annotations)의 `7.10.1 @Component and further stereotype annotations`에는 이렇게 나온다.  

> @Repository 어노테이션은 (Data Access Object 또는 DAO 로도 불리는) 레포지토리의 역할 또는 *전형* 을 만족하는 클래스에 사용하는 마커입니다. 이 마커의 기능 중에는 [Section 20.2.2, "Exception Translation"](https://docs.spring.io/spring-framework/docs/4.3.12.RELEASE/spring-framework-reference/htmlsingle/#orm-exception-translation) 에 나오는 예외 자동 변환이 있습니다.  
> 스프링은 @Component, @Service, @Controller 등의 어노테이션을 지원합니다. @Component는 스프링이 관리하는 모든 컴포넌트의 일반형입니다. @Repository, @Service, @Controller는 좀 더 특정한 사용에 맞춘 @Component의 특정한 유형입니다. 예를 들어 각각 퍼시스던트, 서비스, 표현 계층에 사용합니다. 따라서 작성한 컴포넌트 클래스에 @Component를 붙일 수 있지만, @Repository, @Service, @Controller를 사용하면 툴 사용이나 분석에 좀 더 적합해집니다. 예를 들어 이 어노테이션들이 pointcuts(스프링의 또 다른 기능이군요)의 대상으로 맞춰집니다. 또한 추후 있을 스프링 릴리즈에서 @Repository, @Service, @Controller에 추가 시멘틱이 생길 가능성이 있습니다. 결론적으로 서비스 계층에 @Component와 @Service중 무얼 붙일지 골라야 한다면, @Service가 명백하게 나은 선택입니다. 비슷하게 위에서 말했듯이, 퍼시스턴트 계층에 붙이는 @Repository는 이미 예외 자동 변환 지원 마커로 쓰이고 있습니다.  

그럼 `코드에 있는 모든 @Repository, @Service, @Controller를 뒤죽박죽 써도 된다는 말인가?`하고 해보았더니 컨트롤러에 @Component를 포함한 다른 어노테이션을 붙일 경우, 경로 매핑이 작동하지 않았다. 서비스에 @Repository를 붙여도 정상 작동 하는 걸 보고 이제까지 어노테이션을 구분한 이유가 뭔가... 하고 트루먼쇼에 빠진 느낌이었는데, 그건 아니어서 다행이다. 어떤 원리로 동작하는 것일까? 참으로 신기하다... 더 파고들어 공부해야겠다.  

### @Configuration의 사용  

짧은 공식 문서 번역으로 대체한다.  

> @Configuration은 해당 오브젝트가 빈 정의를 담고 있음을 뜻하는 클래스 레벨 어노테이션입니다. @Configuration 클레스는 @Bean이 붙은 퍼블릭 메서드로 빈을 선언합니다. @Configuration 클래스에서 @Bean 메서드의 사용은 내부 빈 의존성 정의에 사용될 수 있습니다. 좀 더 넓은 설명에 대해서는 [Section 7.12.1, “Basic concepts: @Bean and @Configuration”](https://docs.spring.io/spring-framework/docs/4.3.12.RELEASE/spring-framework-reference/htmlsingle/#beans-java-basic-concepts)을 참고하십시오.  

### @TestConfiguration과 @Import  

이 쯤 되니 @TestConfiguration과 @Configuration도 똑같은 거 아닐까 하는 의심이 들었으나 달랐다. @TestConfiguration은 @Configuration과 @TestComponent가 붙어있고, @Configuration은 @Component가 붙은 구조였다. 자바 문서의 @TestConfiguraion은 이렇게 쓰여있다.  

> 테스트를 위한 추가 빈 정의나 커스터마이징에 쓰는 @Configuration 입니다. 일반 @Configuration과 달리 @TestConfiguration은 @SpringBootConfiguration에 의한 자동 탐지(auto-detection)를 방지하지 않습니다.  

auto-detection이 뭔지 알기 위해 [7.10.3 Automatically detecting classes and registering bean definitions](https://docs.spring.io/spring-framework/docs/4.3.12.RELEASE/spring-framework-reference/htmlsingle/#beans-scanning-autodetection)읽어 보았는데, 그래서 왜 프로덕션과 테스트에 따라 이걸 방지하고 방지하지 않는지에 대한 해답은 찾지 못했다. 추후 공부가 더 필요하다.  

@Import는 자바 코드 상 configuration을 설정하는데 쓰이는 것이었고, 공식 문서 설명은 아래와 같다.  

> 스프링 xml 파일에서 configuration들의 모듈화를 위해 \<import/>를 사용하는 것 처럼, @Import 어노테이션은 다른 configuration 클래스에서 @Bean 정의를 불러오는 것을 허용합니다.  

```java
@Configuration
public class ConfigA {

     @Bean
    public A a() {
        return new A();
    }

}

@Configuration
@Import(ConfigA.class)
public class ConfigB {

    @Bean
    public B b() {
        return new B();
    }
}
```

> 이제 context가 초기화 될 때, `ConfigA.class`와 `ConfigB.class`를 둘 다 특정할 필요 없이, `ConfigB`만 명시적으로 설정하면 됩니다.  

```java
public static void main(String[] args) {
    ApplicationContext ctx = new AnnotationConfigApplicationContext(ConfigB.class);

    // 이제 A와 B 빈을 둘 다 사용 가능  
    A a = ctx.getBean(A.class);
    B b = ctx.getBean(B.class);
}
```  

> 이제 개발자는 수많은 @Configuration 클래스들을 기억할 필요 없이 한 클래스만 신경쓸 수 있으므로, 컨테이너 인스턴스화를 간단화합니다.  

> 스프링 4.2에서 @Import는 `AnnotationConfigApplicationContext.register`메서드와 유사하게 일반 컴포넌트 클래스도 지원하게 되었습니다. 이 기능은 당신이 모든 컴포넌트를 명시적으로 정의하기 위한 시작점으로 몇몇 configuration 클래스를 사용함으로서, 전체 컴포넌트 스캔을 피하고 싶을 때 특히 유용합니다.  

또 다시 하나를 알고 둘을 모르는 상황에 빠졌다. 공부할 거리가 복사가 되고 있다.  

## 꼬리에 꼬리를 무는 스프링  

모르는 하나를 알기 위해 공식 문서를 들어가니 모르는 것이 두 개가 된다. 원래는 다른 한국 개발자들이 포스팅 한 글 등을 보고 사용법만 익혔다. 그래서 내가 스프링을 좀 아는 줄 알았는데, 공식 문서를 들어가니 처음 보는 것 투성이었다. 영어로 읽으니 한글과 달리 모르는 것을 추측으로 떼우지 못해서 모르는 것에 대한 **메타 인지**가 정확히 이루어졌다. 공식 문서를 보지 않았다면 모든 걸 두루뭉술하게 아는 채로 그냥 스프링을 사용했을 것이다. 앞으로는 공식 문서를 최우선으로 삼고 공부해야겠다. 그리고 스프링을 개발 한 사람은 천재다.  

그리고 테스트는 그냥 테스트 한정 인메모리 h2를 써서 하면 비용 지출을 아끼며 할 수 있었다. ㅎㅎ 하지만 이런 고민이 없었다면 공식 문서 읽어 볼 생각을 늦게 했을테니 오히려 좋아.  

<br/>

```toc
```
