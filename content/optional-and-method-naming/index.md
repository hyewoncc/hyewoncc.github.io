---
emoji: headers/optional-and-method-naming.png
title: 'Optional과 메서드 네이밍 find vs get'
date: '2022-05-23 22:15:00'
author: 써머
tags: Java
categories: Java
---

우테코 레벨1 웹체스 미션에서 일급 컬렉션에서 특정 요소를 `Optional<T>`로 꺼내는 메서드를 작성했다. 그런데 해당 요소를 그냥 꺼내는 메서드가 따로 있었고, 이름이 겹치고 싶지 않아 생각하다 `find`를 붙였다. 레벨2에서 DB를 본격적으로 사용하며 `Dao`객체에서 데이터 조회를 할 때, `find`와 `get` 중 무엇을 쓸지 고민했다. `get`은 `getter`처럼 느껴져서 `find`를 사용했는데, `DDD`의 `Repository` 패턴을 보니 `get`을 쓰는 경우가 많았다. 그래서 둘의 차이를 찾아봤다.  

<!--more-->

## find vs get

[스프링 Data의 `CRUD repository`](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/repository/CrudRepository.html) 인터페이스는 `Optional<T>`를 반환한다.  

```java
public interface CrudRepository<T,ID> {
    ...
    Optional<T> findById(ID id);
    Iterable<T> findAll();
    ...
}
```  

반면 같은 스프링 Data의 [JPA repository](https://docs.spring.io/spring-data/jpa/docs/current/api/org/springframework/data/jpa/repository/JpaRepository.html)에서는 T를 반환한다.  

```java
public interface JpaRepository<T,ID> {
    ...
    T getOne(ID id);
    List<T> findAll();
    ...
}
```

`CrudRepository`는 찾는 값이 없을 경우 `빈 Optional`을, `JpaRepository`는 찾는 값이 없을 경우 `예외를 발생`한다는 차이가 있다. 여기서 볼 수 있듯이 일반적으로 반환 값이 없을 수 있는 경우 `find`을, 항상 있어야 하는 경우 `get`을 붙인다. `find`를 쓰면 api 사용자에게 이름을 통해 반환값이 없을 수 있음을 알려 줄 수 있는 것이다.  
반면, 반환형이 `List`같은 컬렉션이라면 값이 없어도 `null`이 아닌 `빈 컬렉션`을 반환하면 된다.  


## Optional  

![cat](img.png)  
<div style="text-align:center; font-style:italic; color:grey;">대표적 Optional인 슈뢰딩거의 고양이</div>

`Optional`에 대해 한 번 정리하려 했었는데, `find~` 메서드에서 `Optional`을 쓰는 김에 이펙티브 자바를 참고해 작성했다.  

### Optional이란  

`Optional<T>`은 null이 아닌 T 참조를 하나 담거나 아무것도 담지 않을 수 있는 클래스이다. 보통은 `T`를 반환해야 하지만 특정 조건에서는 아무것도 반환하지 않아야 할 때 `T`대신 `Optional<T>`를 반환하도록 선언하면 된다. `Optional`은 정적 팩토리 메서드로 생성할 수 있다.  
- `Optional.empty()` : 빈 옵셔널 생성  
- `Optional.of(T)` : T가 든 옵셔널 생성  
- `Optional.ofNullable(T)` : T가 든 옵셔널 생성, `null`값이 들어갈 수도 있을 때 사용  

그냥 `of(T)`와 `ofNullable(T)`를 구분해 사용하는 것이 중요하다. `Optional`을 쓰는데는 `NullPointerException` 체크의 번거로움을 생략하는 목적이 있는데, `of(T)`에 `null`을 넣으면 `NullPointerException`이 발생해 하나마나가 되기 때문이다.  

### 효과적으로 사용하기  

처음 `Optional`을 알게 되었을 때는 `isPresent()`나 `isEmpty()`로 분기문을 만들어 처리했다.  

```java
if (optionalPiece.isEmpty()) {
    throw new IllegalArgumentException("해당 말은 존재하지 않습니다.");
}
```

그런데 `Optional`이 제공하는 메서드를 이용해서 효율적으로 처리할 수 있는 방법이 있었다. 위 예시처럼 분기문으로 처리하면 사실 try~catch로 예외처리 하는 것과 다를 바가 없다고 생각한다. 

- `orElse`로 기본값 정하기  
```java
String name = optionalName.orElse("익명");
```

- `orElseGet`으로 기본값 정하기  
```java
String name = optionalName.orElseGet(generateRandomName());
```

`orElse`에 메서드를 넣으면 `Optional`안에 실제로 값이 있건 없건 일단 실행되는 것에 유의해서 사용하자. `orElseGet`은 `Optional`에 값이 없어야만 실행된다.  

- `orElseThrow`로 예외 던지기  
```java
String name = optionalName.orElseThrow(IllegalArgumentException::new);
```

### 주의사항  

- 컬렉션, 스트림, 배열, 옵셔널 같은 컨테이너 타입을 옵셔널로 감싸기 금지  
그냥 빈 컬렉션이나 배열을 사용하는 것이 낫다.   

- 박싱된 기본 타입을 옵셔널로 다시 감싸기 금지  
값을 두 겹이나 감싸기 보다 `OptionalInt, OptionalLong, OptionalDouble`을 사용하자.  

- 컬렉션의 키, 값, 원소나 배열의 원소로 사용 금지  
복잡성만 높여서 혼란과 오류 가능성만 키운다.

- 메서드 파라미터나 인스턴스 필드로 사용 금지  
메서드 내에서 옵셔널 분기 처리를 하는 건 생산성이 저하되며, 필드로 쓰기엔 선택형 반환값으로 설계되었기에 직렬화가 구현되지 않았다.  

## 참고자료  

- 이펙티브 자바 3판 : 아이템 55. 옵셔널 반환은 신중히 하라  
- [stackoverflow : Why shoud java 8's Optional not be used in arguments](https://stackoverflow.com/questions/31922866/why-should-java-8s-optional-not-be-used-in-arguments)   

```toc
```
