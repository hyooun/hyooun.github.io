---
title: "[만들면서 배우는 클린 아키텍처] 07. 아키텍처 요소 테스트하기"
date: 2025-07-04 17:07:13 +0900
categories: [book, 만들면서 배우는 클린 아키텍처]
tags: [architecture, study]
image: /assets/img/study/clean_architecture/thumbnail.jpg
---

> 헥사고날 아키텍처에서의 테스트 전략과 각 요소들을 테스트할 수 있는 테스트 유형에 대해서 알아보자.

## 테스트 피라미드

![그림7.1](/assets/img/study/clean_architecture/example_image_7_1.png)

테스트의 기본 전제는 만드는 비용이 적고, 유지보수하기 쉽고, 빨리 실행되고, 안정적인 작은 크기의 테스트들에 대해 높은 커버리지를 유지해야 한다.

여러 개의 단위와 단위를 넘는 경계를 결합하는 테스트는 만드는 비용이 더 비싸지고, 실행이 더 느려지며 깨지기 더 쉬워진다. 테스트 피라미드는 테스트가 비싸질수록 테스트의 커버리지 목표는 낮게 잡아야 한다는 것을 나타낸다. 그렇지 않으면 기능을 만드는 것보다 테스트를 만드는 데에 더 많은 시간을 쓰게 된다.

맥락에 따라 테스트 피라미드에 포함되는 계층은 달라질 수 있다. 프로젝트마다 다른 의미를 가질 수 있다는 것이고 다음 정의는 이번 장에서 사용하는 의미다.

단위 테스트는 피라미드의 토대에 해당한다. 일반적으로 하나의 클래스를 인스턴스화하고 해당 클래스의 인터페이스를 통해 기능들을 테스트한다. 만약 테스트하는 클래스가 다른 클래스에 의존한다면 의존되는 클래스들은 인스턴스화하지 않고 테스트하는 동안 mock으로 대체한다.

다음 계층인 통합 테스트는 연결된 여러 유닛을 인스턴스화하고 시작점이 되는 클래스의 인터페이스로 데이터를 보낸 후 유닛들의 네트워크가 기대한 대로 잘 동작하는지 검증한다. 두 계층 간의 경계를 걸쳐 테스트할 수 있기 때문에 객체 네트워크가 완전하지 않거나 특정 시점에는 mock을 대상으로 수행해야 한다.

시스템 테스트 위에는 애플리케이션의 UI를 포함하는 엔드투엔드(end-to-end) 테스트 계층이 있을 수 있지만 이 책은 백엔드 아키텍처가 핵심이므로 다루지 않는다.

---

## 단위 테스트로 도메인 엔티티 테스트하기
헥사고날 아키텍처의 중심인 도메인 엔티티를 살펴보자. `Account` 엔티티의 상태는 과거 특정 시점의 `baselineBalance`와 그 이후의 `activity`로 구성돼 있다. 다음은 `withdraw()` 메서드가 기대한 대로 동작하는지 검증하는 코드이다.

```java
class AccountTest {

	@Test
	void withdrawalSucceeds() {
		AccountId accountId = new AccountId(1L);
		Account account = defaultAccount()
			.withAccountId(accountId)
			.withBaselineBalance(Money.of(555L))
			.withActivityWindow(new ActivityWindow(
				defaultActivity()
					.withTargetAccount(accountId)
					.withMoney(Money.of(999L)).build(),
				defaultActivity()
					.withTargetAccount(accountId)
					.withMoney(Money.of(1L)).build()))
			.build();
		
		boolean success = account.withdraw(Money.of(555L), new AccountId(99L));

		assertThat(success).isTrue();
		assertThat(account.getActivityWindow().getActivities()).hasSize(3);
		assertThat(account.calculateBalance()).isEqualTo(Money.of(1000L));
	}
}
```

이 코드는 특정 상태의 `Account`를 인스턴스화하고 `withdraw()` 메서드를 호출해서 출금이 성공했는지 검증하고 `Account` 객체의 상태에 대해 기대하는 부수효과들이 잘 이루어졌는지 확인하는 단위 테스트이다.

만들고 이해하기 쉽고 아주 빠르게 실행된다. 이런 식의 단위 테스트가 도메인 엔티티에 녹아 있는 비즈니스 규칙을 검증하기에 가장 적절한 방법이다. 도메인 엔티티의 행동은 다른 클래스에 거의 의존하지 않기 때문에 다른 종류의 테스트는 필요하지 않다.

---

## 단위 테스트로 유스케이스 테스트하기
다음으로 테스트할 아키텍처 요소는 유스케이스다. `SendMoneyService`의 테스트를 살펴보자. `SendMoney` 유스케이스는 출금 계좌의 잔고가 다른 트랜잭션에 의해 변경되지 않도록 락을 건다. 출금 계좌에서 돈이 출금되고 나면 똑같이 입금 계좌에 락을 걸고 돈을 입금시킨 뒤 락을 모두 해제한다.

다음 코드는 트랜잭션이 성공했을 때 기대한 대로 동작하는지 검증한다.

```java
class SendMoneyServiceTest {

	// 필드 선언은 생략
	
	@Test
	void transactionSucceeds() {

		Account sourceAccount = givenSourceAccount();
		Account targetAccount = givenTargetAccount();

		givenWithdrawalWillSucceed(sourceAccount);  
		givenDepositWillSucceed(targetAccount);

		given(account.withdraw(any(Money.class), any(AccountId.class)))  
	       .willReturn(true);
		given(account.deposit(any(Money.class), any(AccountId.class)))  
	       .willReturn(true);
	
		Money money = Money.of(500L);
		
		SendMoneyCommand command = new SendMoneyCommand(
			sourceAccount.getId(),
			targetAccount.getId(),
			money);
	
		boolean success = sendMoneyService.sendMoney(command);
		assertThat(success).isTrue();
		
		AccountId sourceAccountId = sourceAccount.getId();
		AccountId targetAccountId = targetAccount.getId();
		
		then(accountLock).should().lockAccount(eq(sourceAccountId));
		then(sourceAccount).should().withdraw(eq(money), eq(targetAccountId));
		then(accountLock).should().releaseAccount(eq(sourceAccountId));

		then(accountLock).should().lockAccount(eq(targetAccountId));
		then(targetAccount).should().deposit(eq(money), eq(sourceAccountId));
		then(accountLock).should().releaseAccount(eq(targetAccountId));

		thenAccountsHaveBeenUpdated(sourceAccountId, targetAccountId);
	}

	// 헬퍼 메서드는 생략
}
```
테스트의 가독성을 높이기 위해 행동-주도 개발(behavior driven development)에서 일반적으로 사용하는 방식대로 `given/when/then`으로 나눠져 있다.

`given`에서는 출금 및 입금 `Account`의 인스턴스를 각각 생성하고 적절한 상태로 만들어서 `given...()`으로 시작하는 메서드에 인자로 넣었다. `SendMoneyCommand` 인스턴스도 만들어서 유스케이스의 입력으로 사용했다. `when`에서는 유스케이스를 실행하기 위해 `sendMoney()` 메서드를 호출했다. `then`에서는 트랜잭션이 성공적이었는지 확인하고, 출금 및 입금 `Account`, 그리고 계좌에 락을 걸고 해제하는 책임을 가진 `AccountLock`에 대해 특정 메서드가 호출됐는지 검증한다.

코드에는 없지만 테스트에는 Mockito 라이브러리를 이용해 `given...()` 메서드의 mock 객체를 생성한다. Mockito는 mock 객체에 대해 특정 메서드가 호출됐는지 검증할 수 있는 `then()` 메서드도 제공한다.

테스트 중인 유스케이스 서비스는 stateless하기 때문에 `then`에서 특정 상태를 검증할 수 없다. 대신 테스트는 서비스가 mocking된 의존 대상의 특정 메서드와 상호작용했는지 여부를 검증한다. 이는 테스트가 코드의 행동 변경뿐만 아니라 코드의 구조 변경에도 취약해진다는 의미다. 자연스럽게 코드가 리팩터링되면 테스트도 변경해야 할 것이다.

때문에 테스트에서 어떤 상호작용을 검증하고 싶은지 신중하게 생각해야 한다. 앞의 예제처럼 모든 동작을 검증하는 것이 아닌, 중요한 핵심만 골라 집중적으로 테스트하는 것이 좋다. 모든 동작을 검증하려고 하면 클래스가 조금이라도 바뀔 때마다 테스트를 변경해야 한다. 이는 테스트의 가치를 떨어뜨리는 행위이다.

이 테스트는 단위 테스트이긴 하지만 의존성의 상호작용을 검증하고 있기 때문에 통합 테스트에 가깝다. 그렇지만 mock으로 작업하고 있어 실제 의존성을 관리해야 하는 것은 아니므로 완전한 통합 테스트에 비해서는 만들고 유지보수하기가 쉽다.

---

## 통합 테스트로 웹 어댑터 테스트하기
웹 어댑터는 JSON 문자열 등의 형태로 HTTP를 통해 입력을 받고, 입력에 대한 유효성 검증을 하고, 유스케이스에서 사용할 수 있는 포맷으로 매핑하고 전달한다. 그리고 다시 유스케이스의 결과를 JSON으로 매핑하고 HTTP 응답을 클라이언트에 반환한다.

모든 단계들이 기대한 대로 동작하는지 검증해보자.
```java
@WebMvcTest(controllers = SendMoneyController.class)
class SendMoneyControllerTest {
	
	@Autowired
	private MockMvc mockMvc;

	@MockBean
	private SendMoneyUseCase sendMoneyUseCase;
	
	@Test
	void testSendMoney() throws Exception {

		mockMvc.perform(
			post("/accounts/send/{sourceAccountId}/{targetAccountId}/{amount}",
			41L, 42L, 500)
		        .header("Content-Type", "application/json"))
		        .andExpect(status().isOk());
		
		then(sendMoneyUseCase).should()
			.sendMoney(eq(new SendMoneyCommand(
				new AccountId(41L),
				new AccountId(42L),
				Money.of(500L))));
	}
}
```
위 코드는 스프링 부트에서 웹 컨트롤러를 테스트하는 표준적인 통합 테스트 방법이다. `testSendMoney()` 메서드에서는 입력 객체를 만들고 mock HTTP 요청을 웹 컨트롤러에 보낸다. 요청 바디는 JSON 문자열의 형태로 입력 객체를 포함한다.

`isOk()` 메서드로 HTTP 응답의 status가 200임을 검증하고, mocking한 유스케이스가 잘 호출됐는지 검증한다.

이 테스트에서 하나의 웹 컨트롤러 클래스만 테스트한 것처럼 보이지만, 사실 보이지 않는 곳에서 더 많은 일들이 일어나고 있다. `@WebMvcTest` 애너테이션은 스프링이 특정 요청 경로, 자바와 JSON 간의 매핑, HTTP 입력 검증 등에 필요한 전체 객체 네트워크를 인스턴스화하도록 만든다. 그리고 테스트에서는 웹 컨트롤러가 이 네트워크의 일부로서 잘 동작하는지 검증한다.

웹 컨트롤러가 스프링 프레임워크에 강하게 묶여 있기 때문에 통합된 상태로 테스트하는 것이 합리적이다. 웹 컨트롤러를 단위 테스트로 테스트하면 모든 매핑, 유효성 검증, HTTP 항목에 대한 커버리지가 낮아지고 프레임워크를 구성하는 요소들이 프로덕션 환경에서 정상적으로 작동할지 확인할 수 없다.

---

## 통합 테스트로 영속성 어댑터 테스트하기
비슷한 이유로 영속성 어댑터에 단위 테스트보다 통합 테스트를 적용하는 것이 합리적이다.
단순히 어댑터의 로직이 아닌, 데이터베이스 매핑을 검증하고 싶기 때문이다.

```java
@DataJpaTest
@Import({AccountPersistenceAdapter.class, AccountMapper.class})
class AccountPersistenceAdapterTest {

	@Autowired
	private AccountPersistenceAdapter adapterUnderTest;

	@Autowired
	private ActivityRepository activityRepository;

	@Test
	@Sql("AccountPersistenceAdapterTest.sql")
	void loadsAccount() {
		Account account = adapter.loadAccount(
			new AccountId(1L),
			LocalDateTime.of(2018, 8, 10, 0, 0));
			
		assertThat(account.getActivityWindow().getActivities()).hasSize(2);
		assertThat(account.calculateBalance()).isEqualTo(Money.of(500));
	}

	@Test
	void updatesActivities() {
		Account account = defaultAccount()
			.withBaselineBalance(Money.of(555L))
			.withActivityWindow(new ActivityWindow(
				defaultActivity()
			.withId(null)
			.withMoney(Money.of(1L)).build()))
			.build();
	
		adapter.updateActivities(account);
		
		assertThat(activityRepository.count()).isEqualTo(1);

		ActivityJpaEntity savedActivity = activityRepository.findAll().get(0);
		assertThat(savedActivity.getAmount()).isEqualTo(1L);
	}
}
```
`@DataJpaTest` 애너테이션으로 스프링 데이터 리포지토리들을 포함해서 데이터베이스 접근에 필요한 객체 네트워크를 인스턴스화해야 한다고 스프링에 알려준다. `@Import` 애너테이션을 추가해서 특정 객체가 이 네트워크에 추가됐다는 것을 명확하게 표현할 수 있다. 이 객체들은 테스트 상에서 어댑터가 도메인 객체를 데이터베이스 객체로 매핑하는 등의 작업에 필요하다.

`loadAccount()` 메서드에 대한 테스트에서는 SQL 스크립트를 이용해 데이터베이스를 특정 상태로 만들고, 어댑터 API를 이용해 계좌를 가져온 후 SQL 스크립트에서 설정한 상태값을 가지고 있는지 검증한다.

`updateActivities()` 메서드에 대한 테스트는 반대로 동작한다. 새로운 계좌 `activity`를 가진 `Account` 객체를 만들어서 저장하기 위해 어댑터로 전달한다. 그리고 `ActivityRepository`의 API를 이용해 이 `activity`가 데이터베이스에 잘 저장됐는지 확인한다.

이 테스트에서는 데이터베이스를 mocking 하지 않아서 실제 데이터베이스에 접근한다. 데이터베이스를 mocking 했더라도 높은 커버리지를 보여주고 스프링에서는 기본적으로 in-memory 데이터베이스를 테스트에서 사용하기 때문에 아무것도 설정할 필요 없이 곧바로 테스트할 수 있어 실용적이지만, 실제 데이터베이스 환경에서는 문제가 생길 가능성이 존재한다.

이러한 이유로 영속성 어댑터 테스트는 실제 데이터베이스를 대상으로 진행해야 한다. Testcontainers 같은 라이브러리는 필요한 데이터베이스를 도커 컨테이너에 띄울 수 있기 때문에 아주 유용하다.

---

## 시스템 테스트로 주요 경로 테스트하기
피라미드 최상단에 있는 시스템 테스트는 전체 애플리케이션을 띄우고 API를 통해 요청을 보내고, 모든 계층이 조화롭게 잘 동작하는지 검증한다.

'송금하기' 유스케이스의 시스템 테스트에서는 애플리케이션에 HTTP 요청을 보내고 계좌의 잔고를 확인하는 것을 포함해서 응답을 검증한다.

```java
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)  
class SendMoneySystemTest {
  
    @Autowired
    private TestRestTemplate restTemplate;  
  
    @Test
    @Sql("SendMoneySystemTest.sql")  
    void sendMoney() {  
  
        Money initialSourceBalance = sourceAccount().calculateBalance();  
        Money initialTargetBalance = targetAccount().calculateBalance();  
  
        ResponseEntity response = whenSendMoney(  
            sourceAccountId(),  
            targetAccountId(),  
            transferredAmount());  
  
        then(response.getStatusCode())  
            .isEqualTo(HttpStatus.OK);  
  
        then(sourceAccount().calculateBalance())  
            .isEqualTo(initialSourceBalance.minus(transferredAmount()));  
  
        then(targetAccount().calculateBalance())  
            .isEqualTo(initialTargetBalance.plus(transferredAmount()));  
  
    }  

    private ResponseEntity whenSendMoney(  
            AccountId sourceAccountId,  
            AccountId targetAccountId,  
            Money amount) { 
            
        HttpHeaders headers = new HttpHeaders();  
        headers.add("Content-Type", "application/json");  
        HttpEntity<Void> request = new HttpEntity<>(null, headers);
  
        return restTemplate.exchange(  
                "/accounts/send/{sourceAccountId}/{targetAccountId}/{amount}",  
                HttpMethod.POST,  
                request,  
                Object.class,  
                sourceAccountId.getValue(),  
                targetAccountId.getValue(),  
                amount.getAmount());
    }
    
    // 일부 헬퍼 메서드는 생략
}
```
`@SpringBootTest` 애너테이션은 스프링이 애플리케이션을 구성하는 모든 객체 네트워크를 랜덤 포트로 띄우게 한다.

`test` 메서드에서는 요청을 생성해서 애플리케이션에 보내고 응답 상태와 계좌의 새로운 잔고를 검증한다.

웹 어댑터처럼 MockMvc를 이용해 요청을 보내는 것이 아니라 `TestRestTemplate`을 이용해서 요청을 보낸다. 프로덕션 환경에 더 가깝게 만들기 위해서 실제로 HTTP 통신을 하는 것이다.

실제 출력 어댑터도 이용한다. 예제의 출력 어댑터는 애플리케이션과 데이터베이스를 연결하는 영속성 어댑터 뿐이다. 다른 시스템과 통신하는 경우에는 다른 출력 어댑터들이 있을 수 있고, 시스템 테스트라고 하더라도 언제나 서드파티 시스템을 실행해서 테스트할 수 있는 것은 아니기 때문에 mocking을 해야 할 때도 있다. 헥사고날 아키텍처는 이러한 경우 몇 개의 출력 포트 인터페이스만 mocking하면 되기 때문에 편리하다.

테스트 가독성을 높이기 위해 지저분한 로직들을 헬퍼 메서드 안으로 감췄다. 이 헬퍼 메서드들은 여러 가지 상태를 검증할 때 사용할 수 있는 도메인 특화 언어(domain-specific language, DSL)를 형성한다.

이러한 도메인 특화 언어는 시스템 테스트에서 더 큰 의미를 갖는다. 시스템 테스트는 다른 테스트보다 실제 상황에 가깝기 때문에 사용자 관점에서 애플리케이션을 검증할 수 있다. JGiven 같은 행동 주도 개발을 위한 라이브러리를 통해 테스트 코드를 plain text로 알기 쉽게 만들 수 있다.

시스템 테스트는 계층 간 매핑 버그와 같이 단위 테스트와 통합 테스트가 발견하는 버그와는 다른 종류의 버그를 발견해서 수정할 수 있게 한다. 또한 여러 유스케이스를 결합한 시나리오를 검증할 수 있다.

---

## 얼마만큼의 테스트가 충분할까?
라인 커버리지(line coverage)는 테스트 성공을 측정하는 데 있어서는 잘못된 지표다. 코드의 중요한 부분이 전혀 커버되지 않을 수 있기 때문에 100%가 아니라면 무의미하다. 심지어 100%라고 하더라도 버그가 잘 잡혔는지 확신할 수 없다.

이 책의 저자는 얼마나 마음 편하게 소프트웨어를 배포할 수 있느냐를 테스트의 성공 기준으로 삼는다. 또한 배포 후 발생하는 버그들을 수정하며 테스트를 추가하다 보면 견고한 테스트가 만들어진다.

이 책의 헥사고날 아키텍처에서는 다음과 같은 테스트 전략을 사용한다.
- 도메인 엔티티를 구현할 때는 단위 테스트로 커버하자.
- 유스케이스를 구현할 때는 단위 테스트로 커버하자.
- 어댑터를 구현할 때는 통합 테스트로 커버하자.
- 사용자가 취할 수 있는 중요 애플리케이션 경로는 시스템 테스트로 커버하자.

만약 테스트가 기능 개발 후가 아닌 개발 중에 이뤄지는 것이 좋다. 하지만 새로운 필드를 추가할 때마다 테스트를 고치는 데 시간을 오래 써야 한다면 테스트 코드가 구조적 변경에 너무 취약한 것이므로 개선이 필요하다. 리팩터링할 때마다 테스트 코드도 변경해야 한다면 테스트 코드로서의 가치를 잃게 된다.

---

## 유지보수 가능한 소프트웨어를 만드는 데 어떻게 도움이 될까?
헥사고날 아키텍처는 도메인 로직과 아웃고잉 어댑터를 깔끔하게 분리한다. 덕분에 핵심 도메인 로직은 단위 테스트로, 어댑터는 통합 테스트로 처리하는 명확한 테스트 전략 수립이 가능하다.

입출력 포트는 테스트에서 아주 뚜렷한 mocking 지점이 된다. 각 포트에 대해서 mocking할지, 실제 구현을 이용할지 선택할 수 있다. 포트 인터페이스가 더 적은 메서드를 제공할수록 mocking하는 것이 더 쉬워진다.

mocking하는 것이 너무 버거워지거나 코드의 특정 부분을 커버하기 위해 어떤 종류의 테스트를 해야 할지 헷갈린다면 개선이 필요하다.