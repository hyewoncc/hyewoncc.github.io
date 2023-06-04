---
emoji: headers/jpa-is-new.png
title: '[JPA]는 새로운 엔티티를 어떻게 알아볼까?'
date: '2022-08-24 15:55:00'
author: 써머
tags: JPA kotlin
categories: JPA kotlin
---

## Long id = 0L ?    

코틀린을 공부하다 참고하려고 [우테코 지원 플랫폼](https://github.com/woowacourse/service-apply)의 코드를 봤다. 
그 중 흥미로운 점을 발견했는데, 엔티티를 생성할 때 `id`를 `0L`로 초기화 하는 것이다.  

```kotlin
@Entity
class Member(
  ...
  id: Long = 0L
)...
```  

이게 왜 흥미로웠냐면 [상수 픽스쳐 사용 주의](https://hyewoncc.github.io/2022/08/13/jpa-static-fixture-trouble.html) 포스팅에서 썼듯이 `id`가 `null`이 아니면 `merge()`를 시행한다고 생각했기 때문이다. 
굳이 `0L`로 초기화하면 `select` 비용만 추가로 들 것 같았다.  <!--more-->

그래서 자바 스프링에서 직접 간단한 엔티티를 만들고, id를 `0L`로 지정해 저장해봤다.    

```java
@Entity
public class Member {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(length = 20, nullable = false)
    private String name;
    ...
}

@SpringBootTest
public class MemberJPATest {

    @Autowired
    private MemberRepository members;
    
    @DisplayName("Long id = 0L 이라면")
    @Test
    void test() {
        Member member = new Member(0L, "dog");
        members.save(member);
    }
}
```  

```
Hibernate: 
    select
        member0_.id as id1_0_0_,
        member0_.name as name2_0_0_ 
    from
        member member0_ 
    where
        member0_.id=?
Hibernate: 
    insert 
    into
        member
        (id, name) 
    values
        (default, ?)

```

실행된 SQL문을 보면, 역시나 `select`문이 `insert`문에 선행되었다. 
`persist()`대신 `merge()`가 수행된 것이다.  

<br>

## JPA의 새로운 엔티티 식별법  

`JPARepository` 상속 시 쓰게 되는 구현체인 `SimpleRepository`의 `save`를 다시 살펴보자.  

```java
@Transactional
@Override
public <S extends T> S save(S entity) {

    Assert.notNull(entity, "Entity must not be null.");
    
    if (entityInformation.isNew(entity)) {
        em.persist(entity);
        return entity;
    } else {
        return em.merge(entity);
    }
}
```  

참 명료한 언어로 잘 짜여있다. 
`if (entityInformation.isNew(entity))`니까 **엔티티가 새로운 엔티티라면** `persist()`를, 아니라면 `merge()`를 수행한다. 
그렇다면 이 `isNew()`는 어디서 가져오는 걸까?  

`isNew()`를 구현한 몇 개의 클래스에 디버깅을 걸며 찾아보니 `AbstractEntityInformation`라는 낯선 이름의 클래스에 걸렸다.   

```java
public abstract class AbstractEntityInformation<T, ID> implements EntityInformation<T, ID> {
    ...
    public boolean isNew(T entity) {
      
        ID id = getId(entity);
        Class<ID> idType = getIdType();
        
        if (!idType.isPrimitive()) {
            return id == null;
        }
        
        if (id instanceof Number) {
            return ((Number) id).longValue() == 0L;
        }
        
        throw new IllegalArgumentException(String.format("Unsupported primitive id type %s", idType));
    }
}
```

코드 상 `isNew()`를 판단하는 조건은 다음과 같다.  

- 엔티티의 `id`가 원시 타입이 아니라면  
  - `null`일 때 새로운 엔티티다  
  - 값이 있을 때 새로운 엔티티가 아니다  
- 엔티티의 `id`가 원시 타입이고, `getId`한 `id`가 래퍼 클래스의 인스턴스라면  
  - 값이 `0`일 때 새로운 엔티티다
  - 값이 `0`이 아니면 새로운 엔티티가 아니다  

처음 읽었을 때 `id instanceof Number`라는 부분이 이상했다. 
위에서 원시값이 아닌 경우를 한번 분기처리 했는데, 왜 다시 `Number`의 인스턴스임을 확인하는 걸까? 
애초에 `id`가 원시값이면 `Number`의 인스턴스가 될 수 없지 않나? 
코드를 읽다 혼란이 와서 `Number`에 대한 정보도 찾아보고 왔다.  

이 의문의 답은 `Id id = getId(entity)`에 있었다. `getId`를 하면서 원시타입의 경우 래퍼 클래스로 바꿔준다. 

```java
public ID getId(T entity) {
    return (ID) persistentEntity.getIdentifierAccessor(entity).getIdentifier();
}
```

굳이 이렇게 번거롭게 처리한 이유는 찾아봤지만 사실 아직 잘 모르겠다... 
어쨌든 여기의 `ID`형은 `JPARepository`를 상속하면서 지정한 형태다. 
따라서 엔티티에서 `long`으로 정의된 `id`는, `getID`를 거쳐 자동으로 `Long`이 된다.  

그럼 `Long`이 아닌 `long id`를 `0L`로 초기화하면 `persist()`가 호출될까?  

```java
@Entity
public class Member {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;

    @Column(length = 20, nullable = false)
    private String name;
    ...
}
```  

```
Hibernate: 
    insert 
    into
        member
        (id, name) 
    values
        (default, ?)

```

실행 결과 그런 것을 볼 수 있다.  
코틀린 코드는 위 예제처럼 자바에서 `id`의 자료형을 `long`으로 하고, `0L`로 초기화 한 것과 동일하다.  


### 숫자가 아닌 값을 id로 쓰려면?  

줍줍 서비스를 개발하다 슬랙은 식별값으로 문자열을 쓴다는 사실이 기억났다. 
(물론 내부적으로는 숫자 `id`를 쓸 수도 있겠으나...) 
메시지 같은 경우는 수가 많아서 그런지 [UUID](https://ko.wikipedia.org/wiki/%EB%B2%94%EC%9A%A9_%EA%B3%A0%EC%9C%A0_%EC%8B%9D%EB%B3%84%EC%9E%90)를 사용한다. 
해당 코드를 본다면 `id`로 UUID를 사용할 경우 `persist`가 호출되지 않을 것 같았다.  

```java
@Entity
public class Member {

    @Id
    @Column(columnDefinition = "BINARY(16)")
    private UUID id = UUID.randomUUID();

    @Column(length = 20, nullable = false)
    private String name;
    ...
}

@SpringBootTest
public class MemberJPATest {

    @Autowired
    private MemberRepository members;
    
    @DisplayName("UUID id = UUID 랜덤값 이라면")
    @Test
    void test() {
        Member member = new Member("dog");
        members.save(member);
    }
}
``` 

```
Hibernate: 
    select
        member0_.id as id1_0_0_,
        member0_.name as name2_0_0_ 
    from
        member member0_ 
    where
        member0_.id=?
Hibernate: 
    insert 
    into
        member
        (name, id) 
    values
        (?, ?)
```

역시나 `merge`가 호출되어 `select` 후 `insert`가 나갔다. 
그럼 숫자가 아닌 값을 `id`로 사용하려면 성능 저하를 필연적으로 안고 가야 하는 것일까?  

<br>

## Persistable<ID> 구현으로 내 맘대로 식별시키기  

당연히 그럴 리는 없다. 
JPA는 엔티티의 `isNew()`를 개발자 입맛대로 바꿀 수 있도록 `Persistable<ID>` 인터페이스를 제공한다.  

```java
@Entity
public class Member implements Persistable<UUID> {

    @Override
    public boolean isNew() {
        return true;
    }
    ...
}
```

이제 모든 `Member`의 인스턴스는 `save` 될 때 마다 새 엔티티로 인식된다. 
실제로 테스트 코드 내에서 `save`를 두 번 호출하면 `PK` 중복으로 에러가 난다.  

```java
@DisplayName("String id = UUID 랜덤값 이라면")
@Test
void test() {
    Member member = new Member("dog");
    members.save(member);
    members.save(member);
}
```

> org.springframework.dao.DataIntegrityViolationException: could not execute statement; SQL [n/a]; constraint ["PUBLIC.PRIMARY_KEY_8 ON PUBLIC.MEMBER(ID) VALUES ( /* 1 */ CAST(X'c63865c5bb6e4f16a3d44aa75244dc1a' AS BINARY(16)) )"; SQL statement:
insert into member (name, id) values (?, ?) [23505-214]]; nested exception is org.hibernate.exception.ConstraintViolationException: could not execute statement  

다른 조건인 `생성 일자`로 `isNew()`를 판단하도록 수정하자.  

```java
@Entity
public class Member implements Persistable<UUID> {

    @CreationTimestamp
    private LocalDateTime createdAt;

    @Override
    public boolean isNew() {
        return createdAt == null;
    }
    ...
}
```

이러면 엔티티 최초 생성 시 `createdAt` 값이 생길 것이고, 이 값의 여부로 `isNew()`를 판단한다. 
해당 인터페이스를 구현하면 이제 `AbstractEntityInformation`의 `isNew()`를 완전히 우선해 대체함에 주의해야 한다.  
`id`가 `null`이면 어쩌고... 는 더이상 적용되지 않는다는 뜻이다.  


### 또는, 조금 더 간단한 버전  

`isNew()` 구현을 위해 얘기하지 않았지만, 사실 UUID도 자동 생성 처리가 가능하다.  

```java
@Entity
public class Member {

    @Id
    @GeneratedValue(generator = "uuid2")
    @GenericGenerator(name = "uuid2", strategy = "uuid2")
    @Column(columnDefinition = "BINARY(16)")
    private UUID id = UUID.randomUUID();
    ...
}
```

역시 있을 건 거의 이미 다 있다. 바퀴를 새로 발명하지 말자.  

<br>

## 그 외에도 식별에 영향을 주는 방법  

### @Version  

첫번쨰로 `@Version`이 있다. 이 어노테이션은 `isNew()`를 판별하는 용도로 붙이는 건 아니다. 
여기서 다른 클래스가 또 끼어들게 된다. `JpaMetamodelEntityInformation`라는 클래스다.  

```java
public class JpaMetamodelEntityInformation<T, ID> extends JpaEntityInformationSupport<T, ID> {
    ...
    @Override
    public boolean isNew(T entity) {
      
        if (!versionAttribute.isPresent()
            || versionAttribute.map(Attribute::getJavaType).map(Class::isPrimitive).orElse(false)) {
            return super.isNew(entity);
        }

        BeanWrapper wrapper = new DirectFieldAccessFallbackBeanWrapper(entity);

        return versionAttribute.map(it -> wrapper.getPropertyValue(it.getName()) == null).orElse(true);
    }
}
```

이 클래스의 `if`문을 보자. `@Version`이 붙은 값이 존재하지 않거나, 이 값들이 원시 타입이라면 상위의 `isNew()`를 호출한다. 
이 상위 클래스가 `AbstractEntityInformation`이다. 앞서 `@Version`이 없던 경우에도 사실 이 메서드를 한 번 타고 왔던 것이다.  

`@Version` 필드가 있고 원시 타입이 아니라면, 이제 이 값으로 `isNew()`를 판단하게 된다. 
이 때도 역시 `id`가 `null`이면 어쩌고... 는 더이상 적용되지 않는다. 
`@Version`이 붙은 필드 값이 `null`인가의 여부로 판단한다.  

```java
@Entity
public class Member {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Version
    private Integer version;
    ...
}

@SpringBootTest
public class MemberJPATest {
    
    @Autowired
    private MemberRepository members;

    @DisplayName("id 값이 있어도 @Version이 null 이라면")
    @Test
    void test() {
        Member member = new Member(1L, "dog");
        members.save(member);
    }
}
```

```
Hibernate: 
    insert 
    into
        member
        (id, name, version) 
    values
        (default, ?, ?)

```

이 `@Version`은 원래 `LOCK`을 위해 사용된다고 한다. 
[JPA 잠금(Lock) 이해하기](https://reiphiel.tistory.com/entry/understanding-jpa-lock) 라는 포스팅에서 상세한 설명을 볼 수 있다.  


### EntityInformation 구현    

앞서 `isNew()`가 있었던 구현체들의 상위 인터페이스인 `EntityInformation`을 직접 구현해 `SimpleJpaRepository`에 주입하는 방법이다. 
이 방식은 아직 이해를 제대로 못했다. 
`SimpleJpaRepository`의 생성자에 어떤 방식으로 주입되는지 제대로 파악하고 나서 더 공부해보려 한다.  

```java
public SimpleJpaRepository(JpaEntityInformation<T, ?> entityInformation, EntityManager entityManager) {
  
    Assert.notNull(entityInformation, "JpaEntityInformation must not be null!");
    Assert.notNull(entityManager, "EntityManager must not be null!");
    
    // 이 부분을 자신의 구현체로 대체하라는 의미일까? 어떻게...?  
    this.entityInformation = entityInformation;
    this.em = entityManager;
    this.provider = PersistenceProvider.fromEntityManager(entityManager);
}
```  

참고한 다른 블로그 포스팅인 [spring-data-jpa save 동작 원리](https://insanelysimple.tistory.com/300)에 나와있다.  

<br>

## 정리  

실제 코드를 탐색한 순서대로 포스팅을 했더니 글이 길어졌다. 
`EntityInformation` 구현을 제외하고, 알아낸 `isNew()`결정 흐름을 정리하자면 다음과 같다.  

- 엔티티 클래스에서 **Persistable\<ID\>** 를 구현했다면, 엔티티 클래스 내의 **isNew()** 로 판단  
- **JpaMetamodelEntityInformation**의 **isNew()** 가 호출  
  - **@Version**필드가 있고, 해당 값이 원시 타입이 아니라면  
    - 필드가 **null**이라면 **true**, 아니라면 **false** 반환  
  - **@Version**필드가 없거나, 해당 값이 원시 타입이라면
    - **AbstractEntityInformation**의 **isNew()** 가 호출  
      - **id**가 원시 타입이 아니라면
        - 값이 **null**이라면 **true**, 아니라면 **false** 반환  
      - **id**가 숫자값이라면  
        - 값이 **0**이라면  **true**, 아니라면 **false** 반환  

참으로 복잡한 과정이 숨겨져 있었다.  

### 그 외의 호기심?  

코드를 열어보면서 `IsNewStrategy`라는 인터페이스와 구현체들을 발견했는데, 정작 어디 쓰이는지는 알아내지 못했다. 
구현체에는 `AbstractEntityInformation`와 비슷한 로직이 들어있다.  

```java
public interface IsNewStrategy {
  
    /**
    * 새로운 엔티티인지 여부를 반환합니다. 다시 말해, 실제로 저장 됐었는지와는 별개입니다.
	  * @param entity must not be {@literal null}.
	  * @return
	  */

    boolean isNew(Object entity);
}

class PersistentEntityIsNewStrategy implements IsNewStrategy {

    @Override
    public boolean isNew(Object entity) {
      
        Object value = valueLookup.apply(entity);

        if (value == null) {
            return true;
        }

        if (valueType != null && !valueType.isPrimitive()) {
            return false;
        }

        if (value instanceof Number) {
            return ((Number) value).longValue() == 0;
        }

        throw new IllegalArgumentException(
              String.format("Could not determine whether %s is new; Unsupported identifier or version property", entity));
    }
}
```

이 친구들은 또 언제 쓰이는 것인지 더 알아봐야겠다.  

<br>

## 도움받은 곳들  

- [Simple is best - spring-data-jpa save 동작 원리](https://insanelysimple.tistory.com/300)  

<br>

```toc
```
