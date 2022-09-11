---
emoji: headers/topring-chapter1.png
title: '오브젝트와 의존관계 (토비의 스프링)'
date: '2022-09-11 22:20:00'
author: 써머
tags: Spring
categories: Spring
---

해당 포스팅은 [토비의 스프링 3.1](https://search.shopping.naver.com/book/catalog/32463364883?cat_id=50010881&frm=PBOKPRO&query=%ED%86%A0%EB%B9%84%EC%9D%98+%EC%8A%A4%ED%94%84%EB%A7%81&NaPm=ct%3Dl7xcl9rk%7Cci%3D2beb40c880925062f67b8f4c293f36caed990219%7Ctr%3Dboknx%7Csn%3D95694%7Chk%3De46d0a257b549c8d320d7074983959664f253dd5)을 읽고 책 내용과 실습 코드 정리 및 스터디에서 나온 의견을 정리한 포스팅이다.  

## 1장 오브젝트와 의존관계  

늘 `개발자인 나는 왜 스프링을 사용했는가`만 생각했기에, 1장 서두의 `스프링은 왜 자신을 쓰라고 하는가`가 중점인 스프링의 핵심 철학이 인상깊었다. 
그래서 포스팅을 인용으로 시작한다.     

> 자바 엔터프라이즈 기술의 혼란 속에서 잃어버렸던 객체지향 기술의 진정한 가치를 회복시키고, 그로부터 객체지향 프로그래밍이 제공하는 폭넓은 혜택을 누릴 수 있도록 기본으로 돌아가자는 것이 바로 스프링의 핵심 철학이다.  
> 그래서 스프링이 가장 관심을 많이 두는 대상은 오브젝트다.  
> p.53  


## 초난감 DAO 개선기  

JDBC API를 처음 배우며 만들었던 모습의 `DAO` 클래스를 만들었다. 
제일 기본적인 기능으로 User 정보를 단건 저장하는 `add()`, 단건 조회하는 `get()`이 있다.  

```java
public class UserDao {

    public static final String MYSQL_JDBC_DRIVER = "com.mysql.cj.jdbc.Driver";
    public static final String MYSQL_URL = "jdbc:mysql://localhost:3306/springbook";
    public static final String MYSQL_USER = "spring";
    public static final String MYSQL_PASSWORD = "book";

    public void add(final User user) throws ClassNotFoundException, SQLException {
        Class.forName(MYSQL_JDBC_DRIVER);
        Connection connection = DriverManager.getConnection(
                MYSQL_URL, MYSQL_USER, MYSQL_PASSWORD
        );

        PreparedStatement statement = connection.prepareStatement(
                "insert into users(id, name, password) values(?, ?, ?)"
        );
        statement.setString(1, user.getId());
        statement.setString(2, user.getName());
        statement.setString(3, user.getPassword());

        statement.executeUpdate();

        statement.close();
        connection.close();
    }

    public User get(final String id) throws ClassNotFoundException, SQLException {
        Class.forName(MYSQL_JDBC_DRIVER);
        Connection connection = DriverManager.getConnection(
                MYSQL_URL, MYSQL_USER, MYSQL_PASSWORD
        );

        PreparedStatement statement = connection.prepareStatement(
                "select id, name, password from users where id = ?"
        );
        statement.setString(1, id);

        ResultSet resultSet = statement.executeQuery();
        resultSet.next();

        User user = new User();
        user.setId(resultSet.getString("id"));
        user.setName(resultSet.getString("name"));
        user.setPassword(resultSet.getString("password"));

        resultSet.close();
        statement.close();
        connection.close();

        return user;
    }
}
```  

비록 동작은 잘 하지만 썩 잘 짠 코드 같진 않다. 
어느 부분이 문제일까? 
책에서는 **세 가지 관심사가 섞여있는 것**이 문제라고 한다.  

- DB 연결 커넥션 가져오기  
- SQL 문장을 담은 Statement를 만들고 실행  
- 자원 반납  

이 비대한 `DAO`를 멋진 `DAO`로 바꾸려면 **관심사를 분리**해야 한다. 
프로그램의 변경은 보통 한가지 관심사에 대해 일어난다. 
예를 들어 `연결할 DB를 다른 DB로 변경한다`, `실행할 SQL 문이 바뀌었다`가 있겠다. 
테이블이 10개, 20개로 늘어났을 때, 연결할 DB가 바뀌었다고 20개의 `DAO`클래스를 다 수정하게 된다면 곤란하다. 
**분리와 확장**을 고려한 설계로 작업을 최소화하고, 변경이 문제를 일으키지 않게 하나씩 개선해보자.  

💡 토프링에서 말한 관심사의 분리란?  

> 관심이 같은 것끼리는 하나의 객체 안으로 또는 친한 객체로 모이게 하고, 관심이 다른 것은 가능한 한 따로 떨어져서 서로 영향을 주지 않도록 분리하는 것 

<br>

### 메서드가 한가지 일을 하도록 분리    

`DB 연결 커넥션 가져오기` 관심사를 `메서드 추출`을 통해 분리할 수 있다.  

```java
public void add(final User user) throws ClassNotFoundException, SQLException {
    Connection connection = getConnection();
    
    PreparedStatement statement = connection.prepareStatement(
        "insert into users(id, name, password) values(?, ?, ?)"
    );
    ...
}

private Connection getConnection() throws ClassNotFoundException, SQLException {
    Class.forName("com.mysql.cj.jdbc.Driver");
    return DriverManager.getConnection(
        "jdbc:mysql://localhost:3306/springbook", "spring", "book"
    );
}
```

`Connection`을 가져오는 부분을 분리해 `add(), get()`의 중복 코드를 제거했다. 
이제 DB 접속 정보가 바뀌면 두 메서드의 코드를 수정하는 대신, `getConnection()`의 코드만 수정하면 된다. 
하지만 다른 DB에 접속하도록 **확장**하려면 이 코드는 어떻게 바뀔까? 
새로운 `DAO`를 만들어 `getConnection()`을 바꿔야 한다. 
`add(), get()`은 그대로 두 클래스 간의 중복 코드가 될 것이다.  

<br>

### 상속으로 중복을 공유하되 추상메서드 사용  

그렇다면 상속을 통해 `add(), get()` 기능을 공유하도록 하자.  

```java
public abstract class UserDao {

    public void add(final User user) throws ClassNotFoundException, SQLException {
        Connection connection = getConnection();
        ...
    }

    public User get(final String id) throws ClassNotFoundException, SQLException {
        Connection connection = getConnection();
        ...
    }

    public abstract Connection getConnection() throws ClassNotFoundException, SQLException;
}

public class GoogleUserDao extends UserDao {

    @Override
    public Connection getConnection() throws ClassNotFoundException, SQLException {
        Class.forName("com.mysql.cj.jdbc.Driver");
        return DriverManager.getConnection(
                "jdbc:mysql://localhost:3306/google", "user", "password"
        );
    } 
}
```

슈퍼클래스 `UserDao`에 기본적인 로직의 흐름을 만들었다. 
그리고 DB 연결 정보마다 달라질 `getConnection()`을 추상 메서드로 지정했다. 
이제 다른 DB를 쓸 거라면, 상속 받은 클래스에서 `getConnection()`만 추가 구현하면 된다. 
여기서 `GithubUserDao`가 추가로 생겨도 `add(), get()`은 슈퍼클래스의 메서드를 호출해 사용할 수 있다. 
이런 방식을 `템플릿 메소드 패턴`이라 부른다.  

동시에 `Connection getConnection()`을 구현해 각자 다른 객체를 리턴하게 둔 것을 `팩토리 메소드 패턴`이라 한다. 
처음 책을 읽었을 때는 이 둘의 차이가 헷갈렸다. 
`템플릿 메소드 패턴`은 변하지 않는 기능(=기본 기능 골격)을 상속으로 공유하는 것에, 
`팩토리 메소드 패턴`은 특정 오브젝트를 반환하는 메소드를 하위 클래스에서 구현하는 것에 초점을 맞춰 이해했다. 
후자의 경우 대개 인터페이스를 반환하도록 한다. 
그렇게 하면 슈퍼클래스는 하위 클래스가 어떤 구현체의 인스턴스를 반환할 지 숨길 수 있다.  

<br>

### 관심사를 클래스로 분리  

여기서 더 나아가, `DB 연결 커넥션 가져오기` 라는 관심사를 별도 클래스로 분리해보자.  

```java
public class DBConnector {

    public Connection makeConnection() throws ClassNotFoundException, SQLException {
        Class.forName("com.mysql.cj.jdbc.Driver");
        return DriverManager.getConnection(
                "jdbc:mysql://localhost:3306/springbook", "spring", "book"
        );
    }
}

public class UserDao {

    private DBConnector dbConnector;

    public UserDao() {
        this.dbConnector = new DBConnector();
    }

    public void add(final User user) throws ClassNotFoundException, SQLException {
        Connection connection = dbConnector.makeConnection();
        ...
    }
    ...
}
```  

이제 `DB 연결 커넥션 가져오기`라는 관심사가 `UserDao`와 클래스 차원에서 분리되었다. 
하지만 `DB 설정이 달라지면 코드를 수정해야 한다`는 단점이 되돌아왔다. 
이를 인터페이스로 분리하고, 구현체를 추가하는 식으로 해결할 수 있다.  

```java
public interface DBConnector {

    Connection makeConnection() throws ClassNotFoundException, SQLException;
}

public class GoogleDBConnector implements DBConnector {

    @Override
    public Connection makeConnection() throws ClassNotFoundException, SQLException {
        Class.forName("com.mysql.cj.jdbc.Driver");
        return DriverManager.getConnection(
                "jdbc:mysql://localhost:3306/google", "spring", "book"
        );
    }
}

public class UserDao {

    private DBConnector dbConnector;

    public UserDao() {
        this.dbConnector = new GoogleDBConnector();
    }
    ...
}
```  

이제 DB 설정이 달라진다면 `DBConnector`의 구현체만 추가되면 된다.  

<br>

### 관계설정 책임의 분리  

여기서 앞서 쓰지 않은 숨겨진 관심사가 있다. 
바로 `Dao`에서 `DBConnector`의 구현체를 정하는 문제다. 
어떤 `DBConnector`를 사용할지 정하는 관심사를 `Dao`와 분리하려면, **클라이언트 오브젝트**로 이동시키면 된다. 
실습에서는 간단히 `main()` 메서드 안으로 이동시켜봤다.  

```java
public static void main(String[] args) throws SQLException, ClassNotFoundException {
    
    UserDao userDao = new UserDao(new GoogleDBConnector());
    
    User user = new User("blackdog", "검은개", "password");
    userDao.add(user);
}

public class UserDao {

    private DBConnector dbConnector;

    public UserDao(final DBConnector dbConnector) {
        this.dbConnector = dbConnector;
    }
    ...
}
```

이제 `UserDao`는 `DBConnector`에 어떤 구현체가 올지 모른다. 
`UserDao`를 사용하는 **클라이언트**에서 필요에 따라 구현체를 주입해주면 된다.  

후에 스프링까지 적용한 전체 실습 코드는 [깃헙 topring 레포](https://github.com/hyewoncc/topring/tree/chapter1)에서 볼 수 있다.  

<br>

## Dao 개선으로 보는 객체지향  

`DB 연결 커넥션 가져오기`라는 기능 관심사 하나로 이렇게 많은 얘기를 할 줄 몰랐다. 
인상깊었던 내용을 중점으로 정리해봤다.   

### 클래스/오브젝트 의존과 DI    

> 클래스 사이에 관계가 만들어진다는 것은 한 클래스가 인터페이스 없이 다른 클래스를 직접 사용한다는 뜻이다. 따라서 클래스가 아니라 오브젝트와 오브젝트 사이의 관계를 설정해줘야 한다.  
> p.78

'A가 B를 의존하고 있다'라고 하면 이제까지는 'A가 B를 사용하고 있다'로 생각했다. 
그래서 클래스 사이의 관계와 **오브젝트** 사이의 관계를 구분해서 생각하지 않았다. 
앞선 예제에서 `UserDao`를 보면, 내부 코드 상으로 어떤 오브젝트(=구현체)를 사용할지 알 수 없다. 
이 구현체가 정의되는 곳은 외부의 **클라이언트 오브젝트**이며, 실제 오브젝트와의 관계는 **런타임 시점**에 맺어진다.  

DI(의존관계 주입)까지 같이 짚고 넘어가자면, 이 오브젝트간의 의존 관계를 외부(=클라이언트)에서 설정하는 것이다. 
예제 코드상 `Main` 클래스는 `main()` 메서드에서 `GoogleDBConnector` 인스턴스의 참조를 `UserDao`에 전달한다. 
이때 두 오브젝트간의 의존관계가 맺어진다.    

<br>

### 개방 폐쇄 원칙(Open-Closed Principle)  

우테코 레벨1 쯤에 `확장에는 열려있고 변경에는 닫혀있다`는 설명을 제대로 이해하지 못했었다. 
그냥 '코드를 추가해서 기능이 늘어남'을 '확장'이라 생각했다. 
예제로 보자면 `UserDao`에 `delete()` 메서드가 생기는 걸 확장이라 여겼다. 
다행히 이 큰 오해는 누군가 바로잡아줬었다.  

어쨌든 현재 [최종 예제 코드](https://github.com/hyewoncc/topring/tree/chapter1)에서 `DBConnector` 클래스는 `DB 연결`이라는 하나의 관심사에 집중하고 있다. 
이를 **응집도가 높다**고 한다. 
동시에 해당 DB연결에 어떤 변경이 생기더라도 `UserDao`에 영향을 끼치지 않는다. 
이렇게 변경 사항이 전파되지 않고 느슨하게 연결된 상태를 **결합도가 낮다**고 한다. 
`DBConnector`의 구현체를 만들어 다른 DB 연결이라는 `확장에 열려`있고, 이 변경에 다른 클래스가 영향을 받지 않기에 `변경에는 닫혀`있다.  

<br>

### 전략 패턴  

전략 패턴은 우테코의 기념비적인 첫 미션 자동차 경주에서 `랜덤값에 따라 달라지는 결과를 테스트하기`라는 문제를 해결하며 접했던 디자인 패턴이다. 
당시에는 `값 하나를 제공한다`는 관심사를 인터페이스로 분리한 것이 충격적으로 획기적이었다. 
공부보다 실전을 통해 체득한 개념이라 전략 패턴을 설명하라 하면 항상 실전 예제를 들었던 것 같은데... 
토프링에서 아주 우아하게 정의한 부분을 보니 신선했다.  

> 전략 패턴은 자신의 기능 맥락(context)에서, 필요에 따라 변경이 필요한 알고리즘을 인터페이스를 통해 통째로 외부로 분리시키고, 이를 구현한 구체적인 알고리즘 클래스를 필요에 따라 바꿔서 아요할 수 있게 하는 디자인 패턴이다.  
> 여기서 말하는 알고리즘이란... 독립적인 책임으로 분리가 가능한 기능을 뜻한다.  
> p.87  

<br>

### IoC(제어의 역전)  

빈출 면접 질문 중에 *라이브러리와 프레임워크의 차이*가 있는 걸로 안다. 
라이브러리는 자주 쓰는 코드를 모듈화 한 것... 프레임워크는 뼈대... 이런 답이 떠올랐던 것 같다. 
그런데 프레임워크도 `제어의 역전` 개념이 적용된 것이라 해서 큰 충격을 받았다... 

> 라이브러리를 사용하는 애플리케이션 코드는 애플리케이션 흐름을 직접 제어한다.  
> ...반면에 프레임워크는 거꾸로 애플리케이션 코드가 프레임워크에 의해 사용된다.  
> p.93  

<br>

이제까지 읽은 기술 책들은 1장이 제일 쉬웠다. 
그런데 토프링은 1장부터 밀도가 장난아니어서 한번 놀라고, 
스프링 공부 하려고 읽었는데 객체지향에 대해 더 흥미롭고 좋은 얘기가 나와서 두번 놀랐다. 
1장 후반부의 용어 정리나 스프링을 이용한 DI의 경우 포스팅에서 중요한 내용이 아니라 생각해 뺐을 정도다. 
객체지향 얘기 만으로 이 책을 산 값을 한다고 느꼈을 정도다. 
남은 두 개의 관심사는 앞으로 어떻게 해결할지, 뒤에는 어떤 얘기가 나올지 기대된다.  

<br>

```toc
```
