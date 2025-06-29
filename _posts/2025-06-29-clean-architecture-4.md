---
title: "[만들면서 배우는 클린 아키텍처] 04. 유스케이스 구현하기"
date: 2025-06-29 21:04:21 +0900
categories: [book, 만들면서 배우는 클린 아키텍처]
tags: [architecture, study]
image: /assets/img/study/clean_architecture/thumbnail.jpg
---

> 현재 구현하고 있는 아키텍처에서는 애플리케이션, 웹, 영속성 계층이 아주 느슨하게 결합되어 있기 때문에 필요한 대로 도메인 코드를 자유롭게 모델링할 수 있다. 이번 장에서는 헥사고날 아키텍처 스타일에서 유스케이스를 구현하기 위해 이 책에서 제시하는 방법을 설명한다.

---

## 도메인 모델 구현하기
한 계좌에서 다른 계좌로 송금하는 유스케이스를 구현해보자. 이를 객체지향적인 방식으로 모델링하는 한 가지 방법은 입금과 출금을 할 수 있는 `Account` 엔티티를 만들고 출금 계좌에서 돈을 출금해서 입금 계좌로 돈을 입금하는 것이다.
```java
package buckpal.domain;

public class Account {
	
	private final AccountId id;
	private final Money baselineBalance;
	private final ActivityWindow activityWindow;

	//생성자와 getter 생략

	public Money calculateBalance() {
		return Money.add(
			this.baselineBalance,
			this.activityWindow.calculateBalance(this.id));
	}

	public boolean withdraw(Money money, AccountId targetAccountId) {

		if (!mayWithdraw(money)) {
			return false;
		}

		Activity withdrawal = new Activity(
			this.id,
			this.id,
			targetAccountId,
			LocalDateTime.now(),
			money);
		this.activityWindow.addActivity(withdrawal);
		return true;
	}

	private boolean mayWithdraw(Money money) {
		return Money.add(
				this.calculateBalance(),
				money.negate())
			.isPositive();
	}

	public boolean deposit(Money money, AccountId sourceAccountId) {
		Activity deposit = new Activity(
			this.id,
			sourceAccountId,
			this.id,
			LocalDateTime.now(),
			money);
		this.activityWindow.addActivity(deposit);
		return true;
	}
}
```
`Account` 엔티티는 실제 계좌의 현재 스냅샷을 제공한다. 계좌에 대한 모든 입금과 출금은 `Activity` 엔티티에 저장된다. 한 계좌에 대한 모든 `Activity`들을 항상 메모리에 함께 올리는 것은 비효율적이기 때문에 `Account` 엔티티는 `ActivityWindow` 값 객체(value object)에서 지난 며칠 혹은 몇 주간의 범위에 해당하는 활동만을 가진다.

계좌의 현재 잔고를 계산하기 위해서 `Account` 엔티티는 `ActivityWindow`의 첫 번째 활동 바로 전의 잔고를 표현하는 `baselineBalance` 속성을 가지고 있다. 현재 총 잔고는 `baselineBalance`에 `ActivityWindow`의 모든 `Activity`들의 잔고를 합한 값이 된다.

계좌에서 일어나는 입금과 출금은 각각 `withdraw()`와  `deposit()` 메서드에서처럼 새로운 `Activity`를 `ActivityWindow`에 추가하기만 한다. 출금하기 전에는 잔고를 초과하는 금액을 출금할 수 없도록 하는 비즈니스 규칙을 검사한다.

---

## 유스케이스 둘러보기
먼저, 유스케이스가 실제로 무슨 일을 하는지 살펴보자. 일반적으로 유스케이스는 다음과 같은 단계를 따른다.

1. 입력을 받는다.
    - 인커밍 어댑터로부터 입력을 받는다. 이 단계에서 '입력 유효성 검증'을 수행할 수도 있지만, 유스케이스는 도메인 로직에만 집중할 수 있도록 '입력 유효성 검증'은 다른 곳에서 처리한다.
2. 비즈니스 규칙을 검증한다.
    - 도메인 엔티티와 함께 비즈니스 규칙(business rule)에 대해서 검증한다.
3. 모델 상태를 조작한다.
    - 이를 충족하면 입력을 기반으로 모델의 상태를 변경한다. 일반적으로 도메인 객체의 상태를 바꾸고 영속성 어댑터를 통해 구현된 포트로 이 상태를 전달해서 저장될 수 있도록 하거나 또 다른 아웃고잉 어댑터를 호출한다.
4. 출력을 반환한다.
    - 아웃고잉 어댑터에서 온 출력값을 유스케이스를 호출한 어댑터로 반환할 출력 객체로 변환한다.

이 단계들을 바탕으로 '송금하기' 유스케이스를 구현해보자. 1장에서 이야기했던 넓은 서비스 문제를 해결하기 위해서 모든 유스케이스를 한 서비스 클래스에 넣지 않고 각 유스케이스별로 분리된 각각의 서비스로 만들어보자.

```java
package buckpal.application.service;

@RequiredArgsConstructor
@Transactional
public class SendMoneyService implements SendMoneyUseCase {

	private final LoadAccountPort loadAccountPort;
	private final AccountLock accountLock;
	private final UpdateAccountStatePort updateAccountStatePort;

	@Override
	public boolean sendMoney(SendMoneyCommand command) {
		// TODO: 비즈니스 규칙 검증
		// TODO: 모델 상태 조작
		// TODO: 출력 값 반환
	}
}
```
서비스는 인커밍 포트 인터페이스인 `SendMoneyUseCase`를 구현하고, 계좌를 불러오기 위해 아웃고잉 포트 인터페이스인 `LoadAccountPort`를 호출한다. 그리고 데이터베이스의 계좌 상태를 업데이트하기 위해 `UpdateAccountStatePort`를 호출한다.

![그림4.1](/assets/img/study/clean_architecture/example_image_4_1.png)

---

## 입력 유효성 검증
입력 유효성 검증은 유스케이스의 책임이 아니긴 하지만 애플리케이션 계층의 책임에 해당한다.

입력 모델(input model)에서 이 문제를 다루도록 해보자. '송금하기' 유스케이스에서 입력 모델은 예제 코드에서 본 `SendMoneyCommand` 클래스다. 생성자 내에서 입력 유효성을 검증하도록 해보자.

```java
package buckpal.application.port.in;

@Getter
public class SendMoneyCommand {

	private final AccountId sourceAccountId;
	private final AccountId targetAccountId;
	private final Money money;

	public SendMoneyCommand(
			AccountId sourceAccountId;
			AccountId targetAccountId;
			Money money) {
		this.sourceAccountId = sourceAccountId;
		this.targetAccountId = targetAccountId;
		this.money = money;

		requireNonNull(sourceAccountId);
		requireNonNull(targetAccountId);
		requireNonNull(money);
		requireGreaterThan(money, 0);
	}
}
```
송금을 위해서는 출금 계좌와 입금 계좌의 ID, 송금할 금액이 필요하다. 모든 파라미터가 null이 아니어야 하고 송금액은 0보다 커야 한다. 이러한 조건 중 하나라도 위배되면 객체를 생성할 때 예외를 던져서 객체 생성을 막으면 된다.

`SendMoneyCommand`의 필드에 `final`을 지정해 불변 필드로 만들었다. 따라서 일단 생성에 성공하고 나면 상태를 유효하고, 이후에 잘못된 상태로 변경할 수 없다는 사실을 보장할 수 있다.

`SendMoneyCommand`는 유스케이스 API의 일부이기 때문에 인커밍 포트 패키지에 위치한다. 그러므로 유효성 검증이 애플리케이션의 코어(헥사고날 아키텍처의 육각형 내부)에 남아있지만 유스케이스 코드를 오염시키진 않는다.

자바에는 `Bean Validation API`가 이러한 작업을 위한 사실상 표준 라이브러리다. 이 API를 이용하면 필요한 유효성 규칙들을 필드의 애너테이션으로 표현할 수 있다.

```java
package buckpal.application.port.in;

@Getter
public class SendMoneyCommand extends SelfValidating<SendMoneyCommand> {

	@NotNull
	private final Account.AccountId sourceAccountId;
	@NotNull
	private final Account.AccountId targetAccountId;
	@NotNull
	private final Money;

	public SendMoneyCommand(
            Account.AccountId sourceAccountId,
            Account.AccountId targetAccountId,
            Money money){
		this.sourceAccountId = sourceAccountId;
		this.targetAccountId = targetAccountId;
		this.money = money;
		requireGreaterThan(money, 0);
		this.validateSelf();
	}
}
```
`SelfValidating` 추상 클래스는 `validateSelf()` 메서드를 제공하며, 생성자 코드의 마지막 문장에서 이 메서드를 호출하고 있다. 이 메서드가 필드에 지정된 Bean Validation 애너테이션(@NotNull 같은)을 검증하고, 유효성 검증 규칙을 위반한 경우 예외를 던진다. Bean Validation이 특정 유효성 검증 규칙을 표현하기에 충분하지 않다면 송금액이 0보다 큰지 검사했던 것처럼 직접 구현할 수도 있다.

```java
package com.shared;

public abstract class SelfValidating<T> {

	private final Validator validator;

	public SelfValidating(){
		ValidatorFactory factory = Validation.buildDefaultValidatorFactory();
		validator = factory.getValidator();
	}

	protected void validateSelf() {
		Set<ConstraintViolation<T>> violations = validator.validate((T) this);
		if (!violations.isEmpty()) {
			throw new ConstraintViolationException(violations);
		}
	}
}
```

입력 모델에 있는 유효성 검증 코드를 통해 오류 방지 계층(anti corruption layer)을 만들었다. 여기서 말하는 계층은 하위 계층을 호출하는 계층형 아키텍처에서의 계층이 아니라 잘못된 입력을 호출자에세 돌려주는 유스케이스 보호막을 의미한다.

---

## 생성자의 힘
앞에서 살펴본 입력 모델인 `SendMoneyCommand`는 생성자에 많은 책임을 지우고 있다. 클래스가 불변이기 때문에 생성자의 인자 리스트에는 클래스의 각 속성에 해당하는 파라미터들이 포함되어 있다. 파라미터 유효성 검증까지 하고 있기 때문에 유효하지 않은 상태의 객체를 만드는 것은 불가능하다.

예제 코드의 생성자에는 3개의 파라미터만 있다. 파라미터가 더 많다면 빌더(Builder) 패턴을 활용하여 더 편하게 만들 수 있다. 긴 파라미터 리스트를 받아야 하는 생성자를 `private`로 만들고 빌더의 `builder()` 메서드 내부에 생성자 호출을 숨길 수 있다. 그러면 파라미터가 많은 생성자를 호출하는 대신 다음과 같이 객체를 만들 수 있을 것이다.
```java
new SendMoneyCommandBuilder()
    .sourceAccountId(new AccountId(41L))
    .targetAccountId(new AccountId(42L))
    // ... 다른 여러 필드를 초기화
    .build();
```

유효성 검증 로직은 생성자에 그대로 둬서 빌더가 유효하지 않은 상태의 객체를 생성하지 못하도록 막을 수 있다.

그러면 `SendMoneyCommandBuidler`에 필드를 새로 추가해야 하는 상황을 생각해보자. 만약 생성자와 빌더에 새로운 필드를 추가하고 나서 빌더를 호출하는 코드에 새로운 필드를 추가하지 않는다면, 컴파일러는 이처럼 유효하지 않은 상태의 불변 객체를 만들려는 시도에 대해서는 경고해주지 못한다. 물론 런타임에 유효성 검증 로직이 동작해서 누락된 파라미터에 대한 에러가 발생한다. 하지만 빌더 뒤에 숨기는 대신 생성자를 직접 사용했더라면 새로운 필드를 추가하거나 필드를 삭제할 때마다 컴파일 에러를 따라 나머지 코드에 변경사항을 적용할 수 있었을 것이다.

---

## 유스케이스마다 다른 입력 모델
각 유스케이스 전용 입력 모델은 유스케이스를 훨씬 명확하게 만들고 다른 유스케이스와의 결합도 제거해서 불필요한 부수효과가 발생하지 않게 한다. 물론 들어오는 데이터를 각 유스케이스에 해당하는 입력 모델에 매핑해야 하기 때문에 비용이 발생하지 않는 것은 아니다.

---

## 비즈니스 규칙 검증하기
입력 유효성 검증은 유스케이스 로직의 일부가 아니지만, 비즈니스 규칙 검증은 분명히 유스케이스 로직에서 수행해야 한다.

비즈니스 규칙을 검증하는 것은 도메인 모델의 현재 상태에 접근해야 하는 반면, 입력 유효성 검증은 그럴 필요가 없다. 입력 유효성 검증은 `@NotNull` 애너테이션을 붙인 것처럼 선언적으로 구현이 가능하지만 비즈니스 규칙 검증은 조금 더 맥락이 필요하다.

입력 유효성 검증은 구문상의(syntactical) 유효성을 검증하는 것이라면 비즈니스 규칙은 유스케이스의 맥락 속에서 의미적인(semantical) 유효성을 검증하는 일이라고 할 수 있다.

예를 들어 "출금 계좌는 초과 출금되어서는 안 된다"라는 규칙을 보자. 이 규칙은 출금 계좌와 입금 계좌가 존재하는지 확인하기 위해 모델의 현재 상태에 접근해야 하기 때문에 비즈니스 규칙이다.

반면 "송금되는 금액은 0보다 커야 한다"라는 규칙은 모델에 접근하지 않고도 검증될 수 있으므로 입력 유효성 검증이다.

이러한 구분법은 논쟁의 여지가 있긴 하지만(송금액은 매우 중요하기 때문에 비즈니스 규칙으로 다뤄야 한다 등) 특정 유효성 검증 로직을 코드 상의 어느 위치에 둘지 결정하고 나중에 그것이 어디에 있는지 파악하는 데 도움이 된다.

그러면 비즈니스 규칙 검증은 어떻게 구현하는 것이 좋을까? 가장 좋은 방법은 앞에서 "출금 계좌는 초과 인출되어서는 안 된다" 규칙에서처럼 비즈니스 규칙을 도메인 엔티티 안에 넣는 것이다.
```java
package buckpal.domain;

public class Account {

    // ...

    public boolean withdraw(Money money, AccountId targetAccountId) {
        if (!mayWithdraw(money)) {
            return false;
        }
        // ...
    }
}
```
이렇게 하면 이 규칙을 지켜야 하는 비즈니스 로직 바로 옆에 규칙이 위치하기 때문에 위치를 정하고 추론하기 쉽다.

만약 도메인 엔티티에서 비즈니스 규칙을 검증하기가 여의치 않다면 유스케이스 코드에서 도메인 엔티티를 사용하기 전에 해도 된다.

```java
package buckpal.application.service;

@RequiredArgsConstructor
@Transactional
public class SendMoneyService implements SendMoneyUseCase {

	// ...
    
    @Override
    public boolean sendMoney(SendMoneyCommand command){
    	requireAccountExists(command.getSourceAccountId());
        requireAccountExists(command.getTargetAccountId());
        ...
    }
}
```
유효성을 검증하는 코드를 호출하고, 유효성 검증이 실패할 경우 전용 예외를 던진다. 사용자와 통신하는 어댑터는 이 예외를 에러 메시지로 사용자에게 보여주거나 적절한 방법으로 처리한다.

앞의 예제에서 살펴본 유효성 검증은 단순히 출금 계좌와 입금 계좌가 데이터베이스에 있는지 확인하는 것이었다. 더 복잡한 비즈니스 규칙의 경우에는 먼저 데이터베이스에서 도메인 모델을 로드해서 상태를 검증해야 할 수도 있다. 도메인 모델을 로드해야 한다면 앞에서 도메인 엔티티 내에서 비즈니스 규칙을 구현해야 한다.

---

## 풍부한(rich) 도메인 모델 vs 빈약한(anemic) 도메인 모델
풍부한 도메인 모델에서는 애플리케이션 코어에 있는 엔티티에서 가능한 많은 도메인 로직이 구현된다. 엔티티들은 상태를 변경하는 메서드를 제공하고, 비즈니스 규칙에 맞는 유효한 변경만을 허용한다.

유스케이스는 도메인 모델의 진입점으로 동작한다. 또한 사용자의 의도만을 표현하면서 이 의도를 실제 작업을 수행하는 체계화된 도메인 엔티티 메서드 호출로 변환한다. 많은 비즈니스 규칙이 유스케이스 구현체 대신 엔티티에 위치하게 된다.

빈약한 도메인 모델에서는 엔티티 자체가 굉장히 얇다. 일반적으로 엔티티는 상태를 표현하는 필드와 이 값을 바꾸기 위한 getter, setter 메서드만을 포함하고 어떤 도메인 로직도 가지고 있지 않다.

이 말은 곧 도메인 로직이 유스케이스 클래스에 구현돼 있다는 것이다. 비즈니스 규칙을 검증하고, 엔티티의 상태를 바꾸고, 데이터베이스 저장을 담당하는 아웃고잉 포트에 엔티티를 전달할 책임 역시 유스케이스 클래스에 있다.

---

## 유스케이스마다 다른 출력 모델
유스케이스가 할 일을 다하고 나면 호출자에게 무엇을 반환해야 할까?

입력과 비슷하게 출력도 가능하면 각 유스케이스에 맞게 구체적일수록 좋다. 출력은 호출자에게 꼭 필요한 데이터만을 가져야 한다.

호출자가 정말로 이 값을 필요로 할까? 만약 그렇다면 다른 호출자도 사용할 수 있도록 해당 데이터에 접근할 전용 유스케이스를 만들어야 하지 않을까? 이러한 질문을 유스케이스를 최대한 구체적으로 만들기 위해 계속해서 해야 하며, 필요한 값만을 반환해야 한다.

유스케이스들 간에 같은 출력 모델을 공유하게 되면 유스케이스들도 강하게 결합된다. 한 유스케이스에서 출력 모델에 새로운 필드가 필요해지면 이 값과 관련이 없는 다른 유스케이스들에서도 이 필드를 처리해야 한다. 공유 모델은 장기적으롸 봤을 때 점점 커지게 된다. 단일 책임 원칙을 적용하고 모델을 분리해서 유지하는 것이 유스케이스의 결합을 제거하는 데 도움이 된다. 같은 이유로 도메인 엔티티를 출력 모델로 사용하는 것도 피해야 한다.

---

## 읽기 전용 유스케이스는 어떨까?
모델의 상태를 변경하는 것이 아닌, 읽기 전용 유스케이스는 어떻게 구현하는게 좋을까?

애플리케이션 코어의 관점에서 단순 읽기 작업은 유스케이스라기 보다는 간단한 데이터 쿼리다. 때문에 프로젝트 맥락에서 유스케이스로 간주되지 않는다면 실제 유스케이스와 구분하기 위해 쿼리로 구현할 수 있다.

이 책의 아키텍처 스타일에서 이를 구현하는 한 가지 방법은 쿼리를 위한 인커밍 전용 포트를 만들고 이를 '쿼리 서비스(query service)'에 구현하는 것이다.
```java
package buckpal.application.service

@RequiredArgsConstructor
class GetAccountBalanceService implements GetAccountBalanceQuery {
    
    private final LoadAccountPort loadAccountPort;

    @Override
    public Money getAccountBalance(AccountId accountId) {
        return loadAccountPort.loadAccount(accountId, LocalDateTime.now())
            .calculateBalance();
    }
}
```
쿼리 서비스는 유스케이스 서비스와 동일한 방식으로 동작한다. `GetAccountBalanceQuery`라는 인커밍 포트를 구현하고, 데이터베이스로부터 실제로 데이터를 로드하기 위해 `LoadAccountPort`라는 아웃고잉 포트를 호출한다.

이처럼 읽기 전용 쿼리는 쓰기가 가능한 유스케이스(또는 커맨드)와 코드 상에서 명확하게 구분된다. 이런 방식은 CQS(Command-Query Separation)나 CQRS(Command-Query Responsibility) 같은 개념과 잘 맞는다.

---

## 유지보수 가능한 소프트웨어를 만드는 데 어떻게 도움이 될까?
도메인 로직을 원하는 대로 구현하고, 입출력 모델을 독립적으로 모델링한다면 원치 않는 부수효과를 피할 수 있다.

물론 유스케이스 간에 모델을 공유하는 것보다는 더 많은 작업이 필요하다. 각 유스케이스마다 별도의 모델을 만들어야 하고, 이 모델을 엔티티와 매핑해야 한다.

그러나 유스케이스별로 모델을 만들면 유스케이스를 명확하게 이해할 수 있고, 장기적으로 유지보수가 쉬워진다. 또한 여러 명의 개발자가 다른 사람이 작업중인 유스케이스를 건드리지 않고 여러 개의 유스케이스를 동시에 작업할 수 있다.

꼼꼼한 입력 유효성 검증, 유스케이스별 입출력 모델은 지속 가능한 코드를 만드는 데 도움이 된다.

---

## 4장 느낀점?
프로젝트를 진행하면서 입력값에 대한 유효성 검증 코드를 어디에 배치할지에 대해서 고민했던 기억이 나는데 이번 장에서 두 가지 방법을 알게 되어 좋았고, 앞으로 개발할 때 적용하면 고민을 덜 수 있을 것 같다 :)