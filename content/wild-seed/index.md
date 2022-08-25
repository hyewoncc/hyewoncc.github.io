---
emoji: headers/wild-seed.png
title: '야생에서 사과씨 심기 (@Bean)'
date: '2022-08-02 23:50:00'
author: 써머
tags: Spring 회고 우아한테크코스
categories: Spring 회고 우아한테크코스
---

야생 학습을 하면 자연스레 지식이 실전을 뒤따르게 된다. 우테코에서 처음 제대로 경험한 야생 학습은 낯설었으나 *너무 잘* 적응한 탓인지 어느 순간 당장 쓰이지 않을 지식을 공부하는 게 마땅찮았다.  
당장 터지는 테스트를 고치고 싶을 뿐인데 왜 `Configuration`을 알아야 하지? 이걸 쓸 때가 있을까? 좋게 말하면 야생, 나쁘게 말하면 땜빵 마인드로, 그래도 연계되는 지식을 우겨넣고는 있었다. 언젠가는 쓸모가 있겠지 하고.  <!--more--> 
이런 *알지만 알지 못하는 죽은 지식*이 레벨3 프로젝트를 하다 *죽은 줄 알았지만 다 때가 있구나!* 로 바뀌는 순간을 겪었다.  


## @Bean은 외부 라이브러리를 등록하고...  

나에겐 `@Bean`이 *알지만 알지 못하는 죽은 지식*이었다. 
[@Bean과 @Component의 차이](https://hyewoncc.github.io/2022/04/25/spring-study-1.html#bean%EA%B3%BC-component%EC%9D%98-%EC%B0%A8%EC%9D%B4) 포스팅을 쓰기도 했고, `@Bean`의 쓰임새가 뭐냐고 하면 줄줄 얘기할 수 있었다. *개발자가 직접 제어할 수 없는 외부 라이브러리 클래스를 빈에 등록할 때 사용됩니다.* 하고. 그런데 좀처럼 `@Bean`을 사용할 일이 없었다.    

그러다 레벨3 프로젝트인 줍줍에서 슬랙 api를 자바 라이브러리를 통해 호출하게 되었다.  

```java
MethodsClient slackClient = Slack.getInstance().methods();
ConversationsInfoResponse response = slackClient.conversationsInfo(
    request -> request.token(slackBotToken));
```

해당 코드를 살펴보자. 우선 `Slack.getInstance()`부분이다. `getInstance()`라는 정적 팩토리 메서드 명에서 추측 가능하듯이 `Slack`의 싱글턴 객체를 얻게 된다. `methods()`는 `MethodsClient`라는 api 호출용 인스턴스를 반환한다. 이 `methods()`안에서는 http 요청을 보낼 때 필요값을 설정하는 작업이 일어난다.  

얻게 되는 최종 인스턴스는 `MethodsClientImpl`이라는 `MethodsClient`의 구현체이다. 변경되는 상태값이 없는데다 여러 객체에서 이 인스턴스를 이용해 슬랙 api를 호출한다. 따라서 필요한 곳에서 중복 생성하기 보다 싱글턴 스프링 빈으로 만들어 관리하고 주입하는게 좋을 것 같았다. 그런데 이걸 어떻게 빈으로 만들까? 생각하다가 *바로 이 때 `@Bean`을 쓰는 거구나!* 하는 깨달음이 왔다.  

```java  
@Configuration
public class SlackConfig {

    @Bean
    public MethodsClient methodsClient() {
        return Slack.getInstance().methods();
    }
}
```  

그리고 이렇게 생성한 빈을 해당 객체를 사용하는 곳에서 주입받게 개선했다.  

```java
@Component
public class MemberInitializer {

    private final MethodsClient slackClient;
    private final MemberRepository memberRepository;

    public MemberInitializer(MethodsClient slackClient, MemberRepository memberRepository) {
        this.slackClient = slackClient;
        this.memberRepository = memberRepository;
    }
    ...
```  

<br>

## 야생 적응 완료 기념비  

어찌보면 그냥 *`@Bean`을 사용했다* 로 요약할 수 있는 하찮은 내용을 굳이 회고 포스팅으로 남긴 두 가지 이유가 있다. 먼저 이 일을 통해 내가 완전히 야생 학습에 적응했다는 것을 기념하고 싶어서다.  

[우테코 레벨2 글쓰기 미션](https://github.com/hyewoncc/woowa-writing-4/blob/hyewoncc/level2.md)에서 야생 학습이 익숙하지 않았음을 얘기했었다. 주어진 인터넷 강의를 차례로 들으며 따라 치는게 익숙했었고, 책을 목차대로 읽는게 마음 편했다. 그런데 이번 일로 야생에서 익히지 않은, 그러니까 실전과 일치하지 않는 지식은 사실 나에게 *진짜* 와닿은 지식이 아니었음을 실감했다. 우테코에 와서 바뀐 나의 학습 모습이 스스로도 놀랍다.  

두번째로, 다람쥐가 묻어두고 잊어버린 도토리처럼 죽은 줄 알았던 지식이 우연히 실전이라는 비를 만나 싹틔우는 짜릿함에 눈뜬 것을 기념하기 위함이다. 부끄럽지만 깊은 학습을 해야 한다고 되뇌이면서도, 야생 학습이라는 핑계로 늘 빠져나갈 궁리를 했던 것 같다. 그런데 `@Bean`이라는 복선이 회수되는 순간 진심으로 `와... 이걸 위해 공부했구나!` 싶어 정말 기뻤다. 야생 학습과 모순되는 말이겠지만 이 즐거움을 더 느끼고 싶어서라도 지식을 더 더 많이, 미리 확장시켜 둬야겠다 생각했다.  

하여간 `@Bean`하나 적절히 쓴 걸로 유난이다 싶을 수도 있겠지만... 이 일이 있어 더 열심히 할 수 있을 것 같다. 또 다시 *얻어 걸리기* 위해서.  레벨3, 진~짜 힘들지만 재밌다.  

<br>  

```toc
```
