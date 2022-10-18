---
emoji: headers/sql-no-wildcard.png
title: '[SQL] SELECT * 쓰지 말자'
date: '2022-05-25 17:30:00'
author: 써머
tags: etc
categories: etc
---

지금까지 DB를 사용하는 미션에서 컬럼 수가 적기에 `select * from table` 문을 자주 썼다. 그런데 `*`로 모든 컬럼을 가져오는 것 보다 컬럼을 명시해 가져오는 게 낫다는 영문 포스팅을 제이슨이 공유해주셔서 읽었다.  
[7 Reasons Why Using SELECT * FROM TABLE in SQL Query Is a Bad Idea](https://dzone.com/articles/why-you-should-not-use-select-in-sql-query-1) 다음은 해당 포스팅을 전체 번역한 것이다.  

## SELECT * FROM TABLE 쿼리문을 쓰는 게 좋지 않은 7가지 이유  

인터넷에서 `SELECT *` 쿼리문을 쓰는 게 나쁘니 피하라고 하는 많은 글을 봐왔다. 그 대신 정확한 컬럼을 나열해야 한다. 이는 내가 주니어 개발자에게 가르치기도 하는 좋은 제안이자 좋은 SQL 작성 습관 중 하나인데, 많은 사람들이 그 이유는 설명하지 않는다.  

<!--more-->

왜 쿼리문에서 `SELECT *`을 쓰면 안되는지 이유를 설명하지 않고 SQL 개발자들을 설득하기란 어려운데, 대개 Oracle database에서 `SELECT * from EMP`를 실행하는 것으로 SQL 학습을 시작하기 때문이다.  

이 글에서 왜 `SELECT *` 쿼리문을 쓰면 안되는지 실질적인 이유를 제시해 인식의 차이를 메우고자 한다. 아래는 `SELECT * from a table`을 쓰지 말아야 할 설득력 있는 몇 가지 이유이다.  

### 1. 불필요한 I/O (인풋, 아웃풋)  
`SELECT *`를 사용하면 사용되지 않을 불필요한 데이터를 반환할 수 있다. 하지만 그 데이터를 불러오는 데는 비용이 들고, 페이지나 인덱스에서 데이터를 읽어오는 동안 IO 사이클이 낭비된다. 이는 쿼리 속도를 낮출 수도 있다. 먄악 쿼리가 실제로 어떻게 실행되는지, [쿼리 엔진이 쿼리문을 어떤 순서로 어떻게 처리](https://dzone.com/refcardz/event-stream-processing-essentials)하는지... 등을 모른다면 [SQL Performance Explained(번역서 없음)](https://www.amazon.com/Performance-Explained-Everything-Developers-about/dp/3950307826/?tag=javamysqlanta-20) 같은 책이나 유데미의 [The Complete SQL BootCamp(한국어 지원 없음)](https://www.udemy.com/course/the-complete-sql-bootcamp/?LSNPUBID=JVFxdTr9V80&ranEAID=JVFxdTr9V80&ranMID=39197&ranSiteID=JVFxdTr9V80-cg.Nnfk_77u8fZ5YccVnaw&utm_medium=udemyads&utm_source=aff-campaign) 강의를 들어보길 바란다.  

### 2. 네트워크 트래픽 증가  
`SELECT *`는 명백하게 클라이언트에서 필요한 것 보다 많은 데이터를 반환하므로 네트워크 대역폭을 더 많이 사용한다. 이는 클라이언트 어플리케이션에 전송되는 데 더 큰 시간이 든다는 말도 된다. 여기서 말한 클라이언트에는 당신이 쿼리를 실행하는 `SQL Server Maganement, Toad, SQL Developer for Oracle`같은 쿼리 에디터와 `Java 어플리케이션`등이 포함된다.  

### 3. 어플리케이션 메모리 낭비  
사용하지 않을 데이터를 저장하고 있느라 어플리케이션의 메모리가 낭비된다.    

### 4. ResultSet의 컬럼 순서에 의존하는 결과  
그래서는 안되지만 어플리케이션이 `SELECT *`쿼리를 사용하며 컬럼 순서에 의존하고 있다면, 컬럼 추가나 순서 변경이 result set에 영향을 미친다.  

### 5. 테이블에 새로운 컬럼을 추가하면 뷰에 오류 발생  
`SELECT *`를 뷰([select 문을 실행한 가상 테이블](https://www.java67.com/2012/11/what-is-difference-between-view-vs-materialized-view-database-sql.html))에 사용하면 새로운 컬럼을 추가하거나 기존 컬럼을 삭제했을 때 알아채기 힘든 오류를 발생시킨다. 뷰가 깨지는 대신에 틀린 결과를 반환하기 때문이다. 이를 피하려면 스키마바인딩을 사용해라. `SELECT *`을 사용하는 것도 막아준다.  

### 6. JOIN 쿼리 충돌  
`SELECT *`를 `JOIN` 쿼리에서 사용하면 같은 컬럼 이름을 가진 여러 테이블이 충돌을 일으킬 수 있다. `straight query(서브쿼리가 없는 쿼리)`를 사용하면 충돌은 없겠지만, `Common Table Expression`이나 `파생 테이블`에서 한 컬럼을 기준으로 정렬하려고 할때 추가적인 조치가 필요해진다.  

### 7. 테이블 간에 데이터를 복사할 때 문제 발생  
한 테이블에서 다른 테이블로 데이터를 복사할 때 `SELECT * into INSERT .. SELECT`문을 흔히 사용하는데, 두 테이블 간에 컬럼 순서가 다르면 잠재적으로 틀린 데이터를 복사할 수 있다. 몇 개발자들은 `EXISTS clause`에서 `SELECT * vs. SELECT 1`사용하는 게 *정적 값을 검사하는 추가 작업을 하지 않아(DB를 잘 몰라 의역합니다)* 더 빠르다 생각하는데, 예전에는 맞는 얘기였으나, 요새는 `EXISTS`문에 대한 처리가 잘 되어 있어 `select`문과 완전히 무관하다.   

## 결론  
위와 같은 이유로 `SELECT *`문의 사용을 지양해야 한다. `*` 와일드카드를 사용하는 것 보다 정확한 컬럼명을 쓰는 것이 언제나 낫다. 쿼리문 수행이 개선될 뿐 아니라 코드도 명확해진다. 거기다 특히 뷰가 있는 상황에서 테이블에 새로운 컬럼을 추가했을 때도 문제가 없기에, 코드의 유지보수 측면에서도 도움이 된다.   

<br/>

컬럼이 2개인 테이블이 있을 만큼 대부분 컬럼 수가 많지 않아, 별 생각 없이 `*`을 많이 사용했는데 지양해야 할 분명한 이유를 알았다.   

```toc
```
