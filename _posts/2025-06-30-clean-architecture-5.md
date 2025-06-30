---
title: "[만들면서 배우는 클린 아키텍처] 05. 웹 어댑터 구현하기"
date: 2025-06-30 14:18:03 +0900
categories: [book, 만들면서 배우는 클린 아키텍처]
tags: [architecture, study]
image: /assets/img/study/clean_architecture/thumbnail.jpg
---

> 애플리케이션은 대부분 웹 인터페이스 같은 것을 제공한다. 웹 브라우저를 통해 상호작용할 수 있는 UI나 다른 시스템에서 우리 애플리케이션으로 호출하는 방식으로 상호작용하는 HTTP API가 여기에 해당한다. 우리가 목표로 하는 아키텍처에서 외부 세계와의 모든 커뮤니케이션은 어댑터를 통해 이뤄지기 때문에 웹 인터페이스를 제공하는 웹 어댑터 구현 방법을 살펴보자.

---

## 의존성 역전
![그림5.1](/assets/img/study/clean_architecture/example_image_5_1.png)

웹 어댑터는 '주도하는' 혹은 '인커밍' 어댑터다. 외부로부터 요청을 받아 애플리케이션 코어를 호출하고 무슨 일을 해야 할지 알려준다. 이때 제어 흐름은 웹 어댑터에 있는 컨트롤러에서 애플리케이션 계층에 있는 서비스로 흐른다.

애플리케이션 계층은 웹 어댑터가 통신할 수 있는 특정 포트를 제공한다. 서비스는 이 포트를 구현하고, 웹 어댑터는 이 포트를 호출할 수 있다.

의존성 역전 원칙이 적용될 것도 확인할 수 있다. 아래 그림과 같이 제어 흐름이 왼쪽에서 오른쪽으로 흐르기 때문에 웹 어댑터가 유스케이스를 직접 호출할 수 있다.

![그림5.2](/assets/img/study/clean_architecture/example_image_5_2.png)

그럼 어댑터와 유스케이스 사이에 또 다른 간접 계층이 왜 필요한 걸까? 애플리케이션 코어가 외부 세계와 통신할 수 있는 곳에 대한 명세가 포트이기 때문이다. 적절한 곳에 포트가 위치하면 외부와 어떤 통신이 일어나고 있는지 정확히 알 수 있고, 이는 유지보수에 도움이 된다.

웹 소켓을 통해 실시간 데이터를 사용자의 브라우저로 보내는 경우는 어떨까? 애플리케이션 코어에서는 이러한 실시간 데이터를 어떻게 웹 데이터로 보내고, 웹 어댑터는 이 데이터를 어떻게 사용자의 브라우저로 전송하는 것일까?

이 경우에는 반드시 포트가 필요하다. 아래 그림과 같이 이 포트는 웹 어댑터에서 구현하고 애플리케이션 코어에서 호출해야 한다.

![그림5.3](/assets/img/study/clean_architecture/example_image_5_3.png)

이 포트는 아웃고잉 포트이기 때문에 웹 어댑터는 인커밍 어댑터인 동시에 아웃고잉 어댑터가 된다. 한 어댑터가 동시에 두 가지 역할도 가능하다.

---

## 웹 어댑터의 책임
웹 어댑터는 일반적으로 다음과 같은 역할을 수행한다.
1. HTTP 요청을 자바 객체로 매핑
    - URL, 경로, HTTP 메서드, 콘텐츠 타입과 같이 특정 기준을 만족하는 HTTP 요청을 수신하고 HTTP 요청의 파라미터와 콘텐츠를 객체로 역직렬화한다.
2. 권한 검사
    - 인증과 권한 부여를 수행하고 실패할 경우 에러를 반환한다.
3. 입력 유효성 검증
    - 객체의 상태 유효성 검증을 수행한다. 유스케이스 입력 모델의 유효성 검증과는 다른 의미로 유스케이스의 맥락에서 유효성 검증이 아닌, 웹 어댑터의 입력 모델을 유스케이스의 입력 모델로 변환할 수 있다는 것을 검증한다.
4. 입력을 유스케이스의 입력 모델로 매핑
    - 입력 유효성 검증이 통과하면 해당하는 유스케이스를 찾아 매핑하고
5. 유스케이스 호출
    - 해당 유스케이스를 호출한다.
6. 유스케이스의 출력을 HTTP로 매핑
    - 유스케이스의 출력을 반환받고, HTTP 응답으로 직렬화해서
7. HTTP 응답을 반환
    - 호출자에게 전달한다.

 이 과정에서 한 군데서라도 문제가 생기면 예외를 던지고, 웹 어댑터는 에러를 호출자에게 보여줄 메시지로 변환하여 전달한다.

 이 책임들은 애플리케이션 계층에 영향을 주면 안된다. 바깥 계층에서 HTTP를 다루고 있다는 것을 애플리케이션 코어가 알게 되면 HTTP를 사용하지 않는 또 다른 인커밍 어댑터의 요청에 대해 동일한 도메인 로직을 수행할 수 있는 선택지를 잃게 된다. 좋은 아키텍처는 선택의 여지를 남겨둔다.
 > 애플리케이션 계층의 코드에서 HTTP에 대한 의존성을 지니게 되면 다른 인커밍 어댑터들에 대한 범용성이 떨어지게 된다는 이야기인 것 같다. 인터페이스인 포트를 활용하여 추상화하고 이를 통해 경계가 확실한 코드를 작성해야 한다고 이해함

 웹 어댑터와 애플리케이션 계층 간의 경계는 도메인과 애플리케이션 계층부터 개발하기 시작하면 자연스럽게 생긴다. 특정 인커밍 어댑터를 생각할 필요 없이 유스케이스를 먼저 구현하면 경계를 흐리게 만들지 않을 수 있다.

---

 ## 컨트롤러 나누기
 모든 요청에 응답할 수 있는 하나의 컨트롤러를 만들 필요는 없다. 웹 어댑터는 한 개 이상의 클래스로 구성해도 된다. 하지만 3장에서 이야기했듯이 클래스들이 같은 소속이라는 것을 표현하기 위해 같은 패키지 수준(hierarchy)에 놓아야 한다.

 컨트롤러는 너무 적은 것보다는 너무 많은 게 낫다. 각 컨트롤러가 가능한 좁고 다른 컨트롤러와 최대한 적게 공유하는 웹 어댑터를 구현해야 한다.

 BuckPal 애플리케이션의 `Account` 엔티티의 연산들을 예로 들어보자. 자주 사용되는 방식은 `AccountController`를 하나 만들어서 계좌와 관련된 모든 요청을 받는 것이다. Rest API를 제공하는 스프링 컨트롤러는 다음과 같을 것이다.
```java
package buckpal.adapter.web;

@RestController
@RequiredArgsConstructor
class AccountController {

    private final GetAccountBalanceQuery getAccountBalanceQuery;
    private final ListAccountsQuery listAccountsQuery;
    private final LoadAccountQuery loadAccountQuery;

    private final SendMoneyUseCase sendMoneyUseCase;
    private final CreateAccountUseCase createAccountUseCase;

    @GetMapping("/accounts")
    List<AccountResource> listAccounts() {
     ...
    }

    @GetMapping("/accounts/{accountId}")
    AccountResource getAccount(@PathVariable("accountId") Long accoutId) {
     ...
    }

    @GetMapping("/accounts/{id}/balance")
    long getAccountBalance(@PathVariable("accountId") Long accoutId) {
     ...
    }

    @PostMapping("/accounts")
    AccountResource createAccount(@RequestBody AccountResource accout) {
     ...
    }

    @PostMapping("/accounts/send/{sourceAccountId}/{targetAccountId}/{amount}")
    void sendMoney(
        @PathVariable("sourceAccountId") Long sourceAccountId,
        @PathVariable("targetAccountId") Long targetAccountId,
        @PathVariable("amount") Long amount) {
     ... 
    }
}
```

계좌 리소스와 관련된 모든 것이 하나의 클래스에 모여 있으면 괜찮아 보인다. 하지만 이 방식의 단점을 한번 살펴보자.

먼저, 클래스마다 코드는 적을수록 좋다. 많은 코드가 한 클래스에 있는 것 보다 적은 것이 코드를 파악하기 훨씬 수월하다. 또한 컨트롤러에 코드가 많으면 그에 해당하는 테스트 코드도 많아진다. 테스트 코드는 더 추상적이라서 프로덕션 코드에 비해 파악하기 어려운 경우가 많다. 따라서 특정 프로덕션 코드에 해당하는 테스트 코드를 찾기 쉽게 만들어야 하는데, 클래스가 작을수록 더 찾기 쉽다.

또한 모든 연산을 단일 컨트롤러에 넣는 것은 데이터 구조의 재활용을 촉진한다. 앞의 예제에서 많은 연산들이 `AccountResource` 모델 클래스를 공유한다. `AccountResource`가 모든 연산에서 필요한 데이터를 모두 담고 있어 아마 `id` 필드를 포함할 것이다. 하지만 `createAccount` 연산에서는 `id`가 필요없기 때문에 도움이 되기보다는 헷갈릴 수 있다. `Account`가 `User`와 일대다 관계를 맺고 있다면, 계좌를 생성하거나 업데이트할 때 `User` 객체도 필요한지, list 연산에 `User` 정보도 같이 반환해야 하는지 명확하지 않다.

그래서 저자는 가급적이면 별도의 패키지 안에 별도의 컨트롤러를 만드는 방식을 선호한다. 또한 메서드와 클래스명은 유스케이스를 최대한 반영해서 지어야 한다.
```java
package buckpal.adapter.in.web;

@RestController
@RequiredArgsConstructor
class SendMoneyController {

    private final SendMoneyUseCase sendMoneyUseCase;

    @PostMapping("/accounts/send/{sourceAccountId}/{targetAccountId}/{amount}")
    void sendMoney(
        @PathVariable("sourceAccountId") Long sourceAccountId,
        @PathVariable("targetAccountId") Long targetAccountId,
        @PathVariable("amount") Long amount) {

            SendMoneyCommand command = new SendMoneyCommand(
                new AccountId(sourceAccountId),
                new AccountId(targetAccountId),
                Money.of(amount));

            sendMoneyUseCase.sendMoney(command);
        }

}
```

또한 각 컨트롤러가 `CreateAccountResource`나 `UpdateAccountResource` 같은 컨트롤러 자체의 모델을 가지거나 앞의 예제 코드처럼 원시값을 받아도 된다.

이러한 전용 모델 클래스들은 컨트롤러 패키지에 대해 `private`로 선언할 수 있기 때문에 실수로 다른곳에서 재사용할 수 없다. 컨트롤러끼리는 모델을 공유할 수 있지만 다른 패키지에 있기 때문에 사용하기 전에 한번 더 생각하고 결국은 해당 컨트롤러에 맞는 모델을 새로 만들 확률이 높다.

컨트롤러명과 서비스명에 대해서도 잘 생각해봐야 한다. `CreateAccount`보다 `RegisterAccount`가 더 명확하다. 유스케이스에 완전히 적합한 네이밍을 찾아서 짓자.

---

## 유지보수 가능한 소프트웨어를 만드는 데 어떻게 도움이 될까?
애플리케이션의 웹 어댑터를 구현할 때는 HTTP 요청을 애플리케이션의 유스케이스에 대한 메서드 호출로 변환하고 결과를 다시 HTTP로 변환하고 어떤 도메인 로직도 수행하지 않는다는 점을 염두해야 한다.---
title: "[만들면서 배우는 클린 아키텍처] 05. 웹 어댑터 구현하기"
date: 2025-06-30 13:51:21 +0900
categories: [book, 만들면서 배우는 클린 아키텍처]
tags: [architecture, study]
image: /assets/img/study/clean_architecture/thumbnail.jpg
---

> 애플리케이션은 대부분 웹 인터페이스 같은 것을 제공한다. 웹 브라우저를 통해 상호작용할 수 있는 UI나 다른 시스템에서 우리 애플리케이션으로 호출하는 방식으로 상호작용하는 HTTP API가 여기에 해당한다. 우리가 목표로 하는 아키텍처에서 외부 세계와의 모든 커뮤니케이션은 어댑터를 통해 이뤄지기 때문에 웹 인터페이스를 제공하는 웹 어댑터 구현 방법을 살펴보자.

---

## 의존성 역전
![그림5.1](/assets/img/study/clean_architecture/example_image_5_1.png)

웹 어댑터는 '주도하는' 혹은 '인커밍' 어댑터다. 외부로부터 요청을 받아 애플리케이션 코어를 호출하고 무슨 일을 해야 할지 알려준다. 이때 제어 흐름은 웹 어댑터에 있는 컨트롤러에서 애플리케이션 계층에 있는 서비스로 흐른다.

애플리케이션 계층은 웹 어댑터가 통신할 수 있는 특정 포트를 제공한다. 서비스는 이 포트를 구현하고, 웹 어댑터는 이 포트를 호출할 수 있다.

의존성 역전 원칙이 적용될 것도 확인할 수 있다. 아래 그림과 같이 제어 흐름이 왼쪽에서 오른쪽으로 흐르기 때문에 웹 어댑터가 유스케이스를 직접 호출할 수 있다.

<p align="center">
  <img src="/assets/img/study/clean_architecture/example_image_5_2.png">
</p>

그럼 어댑터와 유스케이스 사이에 또 다른 간접 계층이 왜 필요한 걸까? 애플리케이션 코어가 외부 세계와 통신할 수 있는 곳에 대한 명세가 포트이기 때문이다. 적절한 곳에 포트가 위치하면 외부와 어떤 통신이 일어나고 있는지 정확히 알 수 있고, 이는 유지보수에 도움이 된다.

웹 소켓을 통해 실시간 데이터를 사용자의 브라우저로 보내는 경우는 어떨까? 애플리케이션 코어에서는 이러한 실시간 데이터를 어떻게 웹 데이터로 보내고, 웹 어댑터는 이 데이터를 어떻게 사용자의 브라우저로 전송하는 것일까?

이 경우에는 반드시 포트가 필요하다. 아래 그림과 같이 이 포트는 웹 어댑터에서 구현하고 애플리케이션 코어에서 호출해야 한다.

![그림5.3](/assets/img/study/clean_architecture/example_image_5_3.png)

이 포트는 아웃고잉 포트이기 때문에 웹 어댑터는 인커밍 어댑터인 동시에 아웃고잉 어댑터가 된다. 한 어댑터가 동시에 두 가지 역할도 가능하다.

---

## 웹 어댑터의 책임
웹 어댑터는 일반적으로 다음과 같은 역할을 수행한다.
1. HTTP 요청을 자바 객체로 매핑
    - URL, 경로, HTTP 메서드, 콘텐츠 타임과 같이 특정 기준을 만족하는 HTTP 요청을 수신하고 HTTP 요청의 파라미터와 콘텐츠를 객체로 역직렬화한다.
2. 권한 검사
    - 인증과 권한 부여를 수행하고 실패할 경우 에러를 반환한다.
3. 입력 유효성 검증
    - 객체의 상태 유효성 검증을 수행한다. 유스케이스 입력 모델의 유효성 검증과는 다른 의미로 유스케이스의 맥락에서 유효성 검증이 아닌, 웹 어댑터의 입력 모델을 유스케이스의 입력 모델로 변환할 수 있다는 것을 검증한다.
4. 입력을 유스케이스의 입력 모델로 매핑
    - 입력 유효성 검증이 통과하면 해당하는 유스케이스를 찾아 매핑하고
5. 유스케이스 호출
    - 해당 유스케이스를 호출한다.
6. 유스케이스의 출력을 HTTP로 매핑
    - 유스케이스의 출력을 반환반고, HTTP 응답으로 직렬화해서
7. HTTP 응답을 반환
    - 호출자에게 전달한다.

 이 과정에서 한 군데서라도 문제가 생기면 예외를 던지고, 웹 어댑터는 에러를 호출자에게 보여줄 메시지로 변환하여 전달한다.

 이 책임들은 애플리케이션 계층에 영향을 주면 안된다. 바깥 계층에서 HTTP를 다루고 있다는 것을 애플리케이션 코어가 알게 되면 HTTP를 사용하지 않는 또 다른 인커밍 어댑터의 요청에 대해 동일한 도메인 로직을 수행할 수 있는 선택지를 잃게 된다. 좋은 아키텍처는 선택의 여지를 남겨둔다.
 > 애플리케이션 계층의 코드에서 HTTP에 대한 의존성을 지니게 되면 다른 인커밍 어댑터들에 대한 범용성이 떨어지게 된다는 이야기인 것 같다. 인터페이스인 포트를 활용하여 추상화하고 이를 통해 경계가 확실한 코드를 작성해야 한다고 이해함

 웹 어댑터와 애플리케이션 계층 같의 경계는 도메인과 애플리케이션 계층부터 개발하기 시작하면 자연스럽게 생긴다. 특정 인커밍 어댑터를 생각할 필요 없이 유스케이스를 먼저 구현하면 경계를 흐리게 만들 유혹에 빠지지 않을 수 있다.

---

 ## 컨트롤러 나누기
 모든 요청에 응답할 수 있는 하나의 컨트롤러를 만들 필요는 없다. 웹 어댑터는 한 개 이상의 클래스로 구성해도 된다. 하지만 3장에서 이야기했듯이 클래스들이 같은 소속이라는 것을 표현하기 위해 같은 패키지 수준(hierarchy)에 놓아야 한다.

 컨트롤러는 너무 적은 것보다는 너무 많은 게 낫다. 각 컨트롤러가 가능한 좁고 다른 컨트롤러와 최대한 적게 공유하는 웹 어댑터를 구현해야 한다.

 BuckPal 애플리케이션의 `Account` 엔티티의 연산들을 예로 들어보자. 자주 사용되는 방식은 `AccountController`를 하나 만들어서 계좌와 관련된 모든 요청을 받는 것이다. Rest API를 제공하는 스프링 컨트롤러는 다음과 같을 것이다.
```java
package buckpal.adapter.web;

@RestController
@RequiredArgsConstructor
class AccountController {

    private final GetAccountBalanceQuery getAccountBalanceQuery;
    private final ListAccountsQuery listAccountsQuery;
    private final LoadAccountQuery loadAccountQuery;

    private final SendMoneyUseCase sendMoneyUseCase;
    private final CreateAccountUseCase createAccountUseCase;

    @GetMapping("/accounts")
    List<AccountResource> listAccounts() {
     ...
    }

    @GetMapping("/accounts/{accountId}")
    AccountResource getAccount(@PathVariable("accountId") Long accoutId) {
     ...
    }

    @GetMapping("/accounts/{id}/balance")
    long getAccountBalance(@PathVariable("accountId") Long accoutId) {
     ...
    }

    @PostMapping("/accounts")
    AccountResource createAccount(@RequestBody AccountResource accout) {
     ...
    }

    @PostMapping("/accounts/send/{sourceAccountId}/{targetAccountId}/{amount}")
    void sendMoney(
        @PathVariable("sourceAccountId") Long sourceAccountId,
        @PathVariable("targetAccountId") Long targetAccountId,
        @PathVariable("amount") Long amount) {
     ... 
    }
}
```

계좌 리소스와 관련된 모든 것이 하나의 클래스에 모여 있으면 괜찮아 보인다. 하지만 이 방식의 단점을 한번 살펴보자.

먼저, 클래스마다 코드는 적을수록 좋다. 많은 코드가 한 클래스에 있는 것 보다 적은 것이 코드를 파악하기 훨씬 수월하다. 또한 컨트롤러에 코드가 많으면 그에 해당하는 테스트 코드도 많아진다. 테스트 코드는 더 추상적이라서 프로덕션 코드에 비해 파악하기 어려운 경우가 많다. 따라서 특정 프로덕션 코드에 해당하는 테스트 코드를 찾기 쉽게 만들어야 하는데, 클래스가 작을수록 더 찾기 쉽다.

또한 모든 연산을 단일 컨트롤러에 넣는 것은 데이터 구조의 재활용을 촉진한다. 앞의 예제에서 많은 연산들이 `AccountResource` 모델 클래스를 공유한다. `AccountResource`가 모든 연산에서 필요한 데이터를 모두 담고 있어 아마 `id` 필드를 포함할 것이다. 하지만 `createAccount` 연산에서는 `id`가 필요없기 때문에 도움이 되기보다는 헷갈릴 수 있다. `Account`가 `User`와 일대다 관계를 맺고 있다면, 계좌를 생성하거나 업데이트할 때 `User` 객체도 필요한지, list 연산에 `User` 정보도 같이 반환해야 하는지 명확하지 않다.

그래서 저자는 가급적이면 별도의 패키지 안에 별도의 컨트롤러를 만드는 방식을 선호한다. 또한 메서드와 클래스명은 유스케이스를 최대한 반영해서 지어야 한다.
```java
package buckpal.adapter.in.web;

@RestController
@RequiredArgsConstructor
class SendMoneyController {

    private final SendMoneyUseCase sendMoneyUseCase;

    @PoseMapping("/accounts/send/{sourceAccountId}/{targetAccountId}/{amount}")
    void sendMoney(
        @PathVariable("sourceAccountId") Long sourceAccountId,
        @PathVariable("targetAccountId") Long targetAccountId,
        @PathVariable("amount") Long amount) {

            SendMoneyCommand command = new SendMoneyCommand(
                new AccountId(sourceAccountId),
                new AccountId(targetAccountId),
                Money.of(amount));

            sendMoneyUseCase.sendMoney(command);
        }

}
```

또한 각 컨트롤러가 `CreateAccountResource`나 `UpdateAccountResource` 같은 컨트롤러 자체의 모델을 가지거나 앞의 예제 코드처럼 원시값을 받아도 된다.

이러한 전용 모델 클래스들은 컨트롤러 패키지에 대해 `private`로 선언할 수 있기 때문에 실수로 다른곳에서 재사용할 수 없다. 컨트롤러끼리는 모델을 공유할 수 있지만 다른 패키지에 있기 때문에 사용하기 전에 한번 더 생각하고 결국은 해당 컨트롤러에 맞는 모델을 새로 만들 확률이 높다.

컨트롤러명과 서비스명에 대해서도 잘 생각해봐야 한다. `CreateAccount`보다 `RegisterAccount`가 더 명확하다. 유스케이스에 완전히 적합한 네이밍을 찾아서 짓자.

---

## 유지보수 가능한 소프트웨어를 만드는 데 어떻게 도움이 될까?
애플리케이션의 웹 어댑터를 구현할 때는 HTTP 요청을 애플리케이션의 유스케이스에 대한 메서드 호출로 변환하고 결과를 다시 HTTP로 변환하고 어떤 도메인 로직도 수행하지 않는다는 점을 염두해야 한다.

애플리케이션 계층은 HTTP에 대한 상세 정보를 노출시키지 않도록 HTTP와 관련된 작업을 해서는 안 된다. 이걸 잘 지킨다면 필요한 경우 웹 어댑터를 다른 어댑터로 쉽게 교체할 수 있다.

웹 컨트롤러를 나눌 때는 모델을 공유하지 않는 여러 작은 클래스들을 많이 만들어야 한다. 그래야 더 파악하기 쉽고, 테스트하기 쉽고, 동시 작업이 수월해진다. 이렇게 세분화된 컨트롤러는 처음에 만들때는 더 많은 수고가 필요하지만 유지보수가 훨씬 수월해진다.

애플리케이션 계층은 HTTP에 대한 상세 정보를 노출시키지 않도록 HTTP와 관련된 작업을 해서는 안 된다. 이걸 잘 지킨다면 필요한 경우 웹 어댑터를 다른 어댑터로 쉽게 교체할 수 있다.

웹 컨트롤러를 나눌 때는 모델을 공유하지 않는 여러 작은 클래스들을 많이 만들어야 한다. 그래야 더 파악하기 쉽고, 테스트하기 쉽고, 동시 작업이 수월해진다. 이렇게 세분화된 컨트롤러는 처음에 만들때는 더 많은 수고가 필요하지만 유지보수가 훨씬 수월해진다.