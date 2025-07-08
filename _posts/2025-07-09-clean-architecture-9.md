---
title: "[만들면서 배우는 클린 아키텍처] 09. 애플리케이션 조립하기"
date: 2025-07-09 00:24:12 +0900
categories: [book, 만들면서 배우는 클린 아키텍처]
tags: [architecture, study]
image: /assets/img/study/clean_architecture/thumbnail.jpg
---

> 유스케이스, 웹 어댑터, 영속성 어댑터를 구현했으니 애플리케이션으로 조립해보자. 
애플리케이션이 시작될 때 클래스를 인스턴스화하고 묶기 위해서 의존성 주입을 활용한다.

---

## 왜 조립까지 신경써야 할까?
모든 의존성은 안쪽으로, 애플리케이션의 도메인 코드 방향으로 향해야 도메인 코드가 바깥 계층의 변경으로부터 안전하다.

유스케이스는 인터페이스만 알아야 하고, 런타임에 해당 인터페이스의 구현을 제공받아야 한다.
또한 클래스가 필요로 하는 모든 객체를 생성자로 전달할 수 있다면 실제 객체 대신 mock으로 전달할 수 있어 단위 테스트가 쉬워진다.
이러한 장점을 살려, 인스턴스 생성을 위해 모든 클래스에 대한 의존성을 가지는 설정 컴포넌트(configuration component)가 있어야 한다.
![그림9.1](/assets/img/study/clean_architecture/example_image_9_1.png)

설정 컴포넌트는 애플리케이션의 조립을 책임지며, 다음과 같은 역할들을 수행한다.
- 웹 어댑터 인스턴스 생성
- HTTP 요청이 실제로 웹 어댑터로 전달되도록 보장
- 유스케이스 인스턴스 생성
- 웹 어댑터에 유스케이스 인스턴스 제공
- 영속성 어댑터 인스턴스 생성
- 유스케이스에 영속성 어댑터 인스턴스 제공
- 영속성 어댑터가 실제로 데이터베이스에 접근할 수 있도록 보장

설정 파라미터의 소스에도 접근할 수 있어야 하며, 이를 애플리케이션 컴포넌트에 제공해 어떤 데이터베이스에 접근하고 어떤 서버를 메일 전송에 사용할지 등의 행동 양식을 제어한다.

설정 컴포넌트는 책임(변경할 이유)이 굉장히 많다. 이는 단일 책임 원칙을 위반하기는 하지만, 애플리케이션의 나머지 부분을 깔끔하게 유지하기 위해서 필요하다.

---

## 평범한 코드로 조립하기
```java
class Application {
	public static void main(String args[]) {

		AccountRepository accountRepository = new AccountRepository();
		ActivityRepository activityRepository = new ActivityRepository();

		AccountPersistenceAdapter accountPersistenceAdapter = 
            new AccountPersistenceAdapter(accountRepository, activityRepository);

		SendMoneyUseCase sendMoneyUseCase =
            new SendMoneyUseService(
                accountPersistenceAdapter,  // LoadAccountPort
                accountPersistenceAdapter); // UpdateAccountStatePort

		SendMoneyController sendMoneyController =
            new SendMoneyController(sendMoneyUseCase);
		
		startProcessingWebRequests(sendMoneyController);

	}
}
```
프레임워크의 도움 없이 평범한 자바 코드로 설정 컴포넌트를 간단히 구현하면 이와 같다.
자바에서는 애플리케이션이 main 메서드로부터 시작된다.
main 메서드 안에서 웹 컨트롤러부터 영속성 어댑터까지 필요한 모든 클래스의 인스턴스를 생성한 후 함께 연결한다.
마지막으로 웹 컨트롤러를 HTTP로 노출하는 메서드인 `startProcessingWebRequests()`를 호출한다.

이러한 방식은 가장 기본적이지만 몇 가지 단점이 존재한다.
1. 이 코드는 웹 컨트롤러, 유스케이스, 영속성 어댑터가 단 하나씩만 있는 애플리케이션을 예로 든 것이고 실제 애플리케이션에서는 굉장히 많은 코드 구현이 필요하다.
2. 각 클래스가 속한 패키지 외부에서 인스턴스를 생성하기 때문에 접근자가 모두 `public`이어야 한다. 이렇게 구현하면 유스케이스가 영속성 어댑터에 직접 접근하는 것을 막지 못한다. `package-private` 접근 제한자를 이용해서 이러한 상황을 피할 수 있었다면 더 좋았을 것이다.

---

## 스프링의 클래스패스 스캐닝으로 조립하기
스프링 프레임워크를 이용해서 애플리케이션을 조립한 결과물을 애플리케이션 컨텍스트(application context)라고 하고, 이는 애플리케이션을 구성하는 모든 객체(자바 용어로는 bean)를 포함한다.
스프링은 애플리케이션 컨텍스트를 조립하기 위한 몇 가지 방법을 제공하는데, 가장 인기있고 편리한 방법인 클래스패스 스캐닝(classpath scanning)을 사용해보자.

스프링은 스캐닝으로 클래스패스에서 접근 가능한 클래스 중 `@Component` 애너테이션이 붙은 클래스를 찾고 이 애너테이션이 붙은 각 클래스의 객체를 생성한다.
이때 클래스는 필요한 모든 필드를 인자로 받는 생성자를 가지고 있어야 한다.
```java
@RequiredArgsConstructor
@Component
public class AccountPersistenceAdapter implements 
      LoadAccountPort,
      UpdateAccountStatePort {

    private final AccountRepository accountRepository;
    private final ActivityRepository activityRepository;
    private final AccountMapper accountMapper;

    @Override
    public Account loadAccount(
            AccountId accountId,
            LocalDateTime baselineDate) {
		// ...
    }


    @Override
    public void updateActivities(Account account) {
		// ...
	}

}
```
이 코드에서는 생성자를 직접 만들지 않고 Lombok 라이브러리의 `@RequiredArgsConstructor` 애너테이션을 이용해 모든 `final` 필드를 인자로 받는 생성자를 자동으로 생성했다.

그럼 스프링은 이 생성자를 찾아서 생성자의 인자로 사용된 `@Component`가 붙은 클래스들을 찾고, 이 클래스들의 인스턴스를 생성하여 애플리케이션 컨텍스트에 추가한다.
필요한 객체들이 모두 생성되면 `AccountPersistenceAdapter`의 생성자를 호출하고 생성된 객체도 마찬가지로 애플리케이션 컨텍스트에 추가한다.

클래스패스 스캐닝 방식을 이용하면 아주 편리하게 애플리케이션을 조립할 수 있다. 적절한 곳에 `@Component` 애너테이션을 붙이고 생성자만 잘 만들어주면 된다.

스프링이 인식할 수 있는 애너테이션을 직접 만들 수도 있다. 예를 들어, 다음과 같이 `@PersistenceAdapter`라는 애너테이션을 만들 수 있다.
```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface PersistenceAdapter {

    @AliasFor(annotation = Component.class)
    String value() default "";

}
```
이 애너테이션은 메타-애너테이션으로 `@Component`를 포함하고 있어서 스프링이 클래스패스 스캐닝을 할 때 인스턴스를 생성할 수 있도록 한다.
`@Component` 대신 `@PersistenceAdapter`를 이용해 영속성 어댑터 클래스들이 애플리케이션의 일부임을 표시해 아키텍처를 더 쉽게 파악할 수 있다.

클래스패스 스캐닝 방식에도 단점은 있다.
1. 클래스에 프레임워크에 특화된 애너테이션을 붙여야 하기 때문에 특정한 프레임워크와 결합이 생긴다.
2. 단순히 `@Component`를 찾아서 사용하는 방식으로 동작하기 때문에 추적하기 어려운 에러를 발생시킬 수 있다.

---

## 스프링의 자바 컨피그로 조립하기
이 방식에서는 애플리케이션 컨텍스트에 추가할 빈을 생성하는 설정 클래스를 만든다.
```java
@Configuration
@EnableJpaRepositories
public class PersistenceAdapterConfiguration {

    @Bean
    AccountPersistenceAdapter accountPersistenceAdapter(
            AccountRepository accountRepository,
            ActivityRepository activityRepository,
            AccountMapper accountMapper) {

        return new AccountPersistenceAdapter(
                accountRepository,
                activityRepository,
                accountMapper);
    }

    @Bean
    AccountMapper accountMapper() {
        return new AccountMapper();
    }

}
```
모든 영속성 어댑터들의 인스턴스 생성을 담당하는 설정 클래스를 하나 만들었다.
`@Configuration` 애너테이션을 통해 이 클래스가 스프링의 클래스패스 스캐닝에서 발견해야 할 설정 클래스임을 표시한다.
여전히 클래스패스 스캐닝을 사용하고 있는 것이긴 하지만, 모든 빈을 가져오는 대신, 설정 클래스만 선택하기 때문에 추적하기 어려운 에러를 발생시킬 확률이 줄어든다.

빈은 설정 클래스 내의 `@Bean` 애너테이션이 붙은 팩터리 메서드를 통해 생성된다.
예제 코드에서는 영속성 어댑터를 애플리케이션 컨텍스트에 추가했다. 영속성 어댑터는 2개의 리포지토리와 한 개의 매퍼를 생성자 입력으로 받는다.
스프링은 이 객체들을 자동으로 팩터리 메서드에 입력으로 제공한다.

`@EnableJpaRepositories` 애너테이션을 통해 리포지토리를 스프링이 직접 생성해서 제공한다.
스프링 부트가 이 애너테이션을 발견하면 정의한 모든 스프링 데이터 리포지토리 인터페이스의 구현체를 제공한다.

이렇게 Config 클래스를 활용하면 클래스패스 스캐닝을 활용하면서 어떤 빈이 애플리케이션 컨텍스트에 등록될지 제어할 수 있다.

비슷한 방법으로 웹 어댑터, 애플리케이션 계층의 특정 모듈을 위한 설정 클래스를 만들 수도 있다.
그러면 특정 모듈만 포함하고, 다른 모듈의 빈은 mocking해서 애플리케이션 컨텍스트를 만들 수 있기 때문에 테스트 유연성이 생긴다.

또한 이 방식은 클래스패스 스캐닝과 다르게 `@Component` 애너테이션에 의존하지 않아 프레임워크 독립성을 유지할 수 있다.
하지만 이 방법에도 문제점은 존재한다. 설정 클래스가 생성하는 빈이 설정 클래스와 다른 패키지에 존재한다면, `public`으로 만들어야 한다.

---

## 유지보수 가능한 소프트웨어를 만드는 데 어떻게 도움이 될까?
스프링과 스프링 부트는 클래스패스 스캐닝을 통해 조립을 간편하게 만들어 준다.
클래스패스 스캐닝만을 사용하는 것 보다는 전용 설정 컴포넌트를 만들어 애플리케이션을 변경할 이유를 줄이자.