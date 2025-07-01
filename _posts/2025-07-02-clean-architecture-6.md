---
title: "[만들면서 배우는 클린 아키텍처] 06. 영속성 어댑터 구현하기"
date: 2025-07-02 03:56:41 +0900
categories: [book, 만들면서 배우는 클린 아키텍처]
tags: [architecture, study]
image: /assets/img/study/clean_architecture/thumbnail.jpg
---

> 계층형 아키텍처는 영속성 계층에 의존하게 되어 '데이터베이스 주도 설계'가 된다고 이야기했다. 
이러한 의존성을 역전시키기 위해 영속성 계층을 애플리케이션 계층의 플러그인으로 만드는 방법을 살펴보자.

## 의존성 역전
![그림6.1](/assets/img/study/clean_architecture/example_image_6_1.png)

애플리케이션 서비스에서는 영속성 기능을 사용하기 위해 포트 인터페이스를 호출한다. 이 포트는 실제로 영속성 작업을 수행하고 데이터베이스와 통신할 책임을 가진 영속성 어댑터 클래스에 의해 구현된다.

헥사고날 아키텍처에서 영속성 어댑터는 '주도되는' 혹은 '아웃고잉' 어댑터이다. 애플리케이션에 의해 호출될 뿐, 애플리케이션을 호출하지는 않는다.

포트는 애플리케이션 서비스와 영속성 코드 사이의 간접적인 계층이다. 영속성 문제에 신경쓰지 않고 도메인 코드를 개발하기 위해, 즉 영속성 계층에 대한 코드 의존성을 없애기 위해 존재한다.

---

## 영속성 어댑터의 책임
영속성 어댑터는 다음과 같은 일들을 한다.
1. 입력을 받는다.
    - 포트 인터페이스를 통해 입력을 받는다. 입력 모델은 인터페이스가 지정한 도메인 엔티티나 특정 데이터베이스 연산 전용 객체이다.
2. 입력을 데이터베이스 포맷으로 매핑한다.
    - 입력 모델을 데이터베이스를 쿼리하거나 변경하는 데 사용할 수 있는 포맷으로 매핑한다. 
    - 자바 프로젝트에서는 일반적으로 JPA를 사용하기 때문에 입력 모델을 데이터베이스 테이블 구조를 반영한 JPA 엔티티 객체로 매핑한다.
      > 영속성 어댑터의 입력 모델이 영속성 어댑터 내부에 있는 것이 아니라 애플리케이션 코어에 있기 때문에 영속성 어댑터 내부를 변경하는 것이 코어에 영향을 미치지 않는다.
3. 입력을 데이터베이스로 보낸다.
    - 데이터베이스에 쿼리를 보내고 쿼리 결과를 받아온다.
4. 데이터베이스 출력을 애플리케이션 포맷으로 제공한다.
    - 데이터베이스 응답을 포트에 정의된 출력 모델로 매핑해서 반환한다.
      > 출력 모델은 영속성 어댑터가 아닌, 애플리케이션 코어에 위치한다.
5. 출력을 반환한다.

---

## 포트 인터페이스 나누기
데이터베이스 연산을 정의하고 있는 포트 인터페이스를 어떻게 나눠야 할까?

![그림6.2](/assets/img/study/clean_architecture/example_image_6_2.png)

이 그림처럼 특정 엔티티가 필요로 하는 모든 데이터베이스 연산을 하나의 리포지토리 인터페이스에 넣어 두는 게 일반적인 방법이다.

이 방식을 사용하면 데이터베이스 연산에 의존하는 각 서비스는 인터페이스에서 단 하나의 메서드만 사용하더라도 하나의 '넓은 포트 인터페이스'에 의존성을 갖게 되어 불필요한 의존성이 생긴다.

맥락 안에서 필요하지 않은 메서드에 생긴 의존성은 코드를 이해하고 테스트하기 어렵게 만든다. `ResisterAccountService`의 단위 테스트를 작성한다고 생각해보자. 서비스가 `AccountRepository`의 어떤 메서드를 호출하는지 찾고 일부 메서드를 모킹해야 한다. 그러면 다음에 이 테스트에서 작업하는 사람은 인터페이스 전체가 모킹됐다고 생각하여 에러가 발생해 코드를 다시 확인해야 하는 상황이 생길 수 있다.

인터페이스 분리 원칙(Interface Segregation Principle, ISP)은 이 문제의 답을 제시한다. 이 원칙은 클라이언트가 오로지 자신이 필요로 하는 메서드만 알게 되면 되도록 넓은 인터페이스를 특화된 인터페이스로 분리해야 한다고 설명한다.

이 원칙을 예제에 적용하면 다음과 같이 변경된다.

![그림6.3](/assets/img/study/clean_architecture/example_image_6_3.png)

이제 각 서비스는 필요한 메서드에만 의존한다. 또한 포트의 이름이 역할을 명확하게 표현한다. 테스트에서는 포트당 하나의 메서드만 존재할 것이기 때문에 모킹이 필요없어진다.

물론 모든 상황에 '포트 하나 당 하나의 메서드'를 적용하기는 힘들다. 응집성이 높고 함께 사용될 때가 많기 때문에 하나의 인터페이스에 묶고 싶은 인터페이스 연산들이 있을 수 있다.

## 영속성 어댑터 나누기
영속성 연산이 필요한 도메인 클래스(또는 DDD에서의 애그리거트) 하나당 하나의 영속성 어댑터를 구현하는 방식도 있다.

![그림 6.4](/assets/img/study/clean_architecture/example_image_6_4.png)

이렇게 하면 영속성 어댑터들은 각 영속성 기능을 이용하는 도메인 경계를 따라 자동으로 나눠진다. 영속성 어댑터를 훨씬 더 많은 클래스로 나눌 수도 있다. JPA 같은 OR 매퍼를 이용한 영속성 포트를 구현하면서 성능 개선을 위해 일반적인 SQL을 이용하는 다른 종류의 포트를 함께 구현할 수 있다.

'애그리거트당 하나의 영속성 어댑터' 접근 방식은 나중에 여러 개의 바운디드 컨텍스트(bounded context)의 영속성 요구사항을 분리하기 위한 좋은 기반이 된다. 

![그림 6.5](/assets/img/study/clean_architecture/example_image_6_5.png)

각 바운디드 컨텍스트는 영속성 어댑터를 하나 이상씩 가지고 있다. `account` 컨텍스트의 서비스가 `billing` 컨텍스트의 영속성 어댑터에 접근하지 않는다. 어떤 컨텍스트가 다른 컨텍스트에 있는 무언가를 필요로 한다면 전용 인커밍 포트를 통해 접근해야 한다.

---

## 스프링 데이터 JPA 예제
앞에 그림에서 본 `AccountPersistenceAdapter`를 구현한 코드를 살펴보자. 이 어댑터는 데이터베이스로부터 계좌를 가져오거나 저장할 수 있어야 한다.
```java
package buckpal.domain;

@AllArgsConstructor(access = AccessLevel.PRIVATE)
public class Account {

    @Getter private final AccountId id;
    @Getter private final ActivityWindow activityWindow;
    private final Money baselineBalance;

    public static Account withoutId(
            Money baselineBalance,
            ActivityWindow activityWindow) {
        return new Account(null, baselineBalance, activityWindow);
    }

    public static Account withId(
            AccountId accountId,
            Money baselineBalance,
            ActivityWindow activityWindow) {
        return new Account(accountId, baselineBalance, activityWindow);
    }

    public Money calculateBalance() {
        // ...
    }

    public boolean withdraw(Money money, AccountId targetAccountId) {
        // ...
    }

    private boolean mayWithdraw(Money money) {
        // ...
    }

}
```

`Account` 클래스는 `getter`와 `setter`만 가진 간단한 데이터 클래스가 아니며 최대한 불변성을 유지하려 한다는 것을 기억하자. 유효한 상태의 `Account` 엔티티만 생성할 수 있는 팩터리 메서드를 제공하고 출금 전에 계좌의 잔고를 확인하는 등의 유효성 검증을 모든 상태 변경 메서드에서 수행하기 때문에 유효하지 않은 도메인 모델을 생성할 수 없다.

데이터베이스와의 통신에 스프링 데이터 JPA를 사용할 것이므로 계좌의 데이터베이스 상태를 표현하는 `@Entity` 애너테이션이 추가된 클래스도 필요하다.

```java
package buckpal.adapter.persistence;

@Entity
@Table(name = "account")
@Data
@AllArgsConstructor
@NoArgsConstructor
class AccountJpaEntity {

    @Id
    @GeneratedValue
    private Long id;

}
```

다음은 activity 테이블을 표현하기 위한 코드다.

```java
package buckpal.adapter.persistence;

@Entity
@Table(name = "activity")
@Data
@AllArgsConstructor
@NoArgsConstructor
class ActivityJpaEntity {

    @Id
    @GeneratedValue
    private Long id;

    @Column private LocalDateTime timestamp;
    @Column private Long ownerAccountId;
    @Column private Long sourceAccountId;
    @Column private Long targetAccountId;
    @Column private Long amount;
}
```

다음으로 기본적인 CRUD 기능과 데이터베이스에서 `activity`들을 로드하기 위한 커스텀 쿼리를 제공하는 리포지토리 인터페이스를 생성하기 위해 스프링 데이터를 사용한다.

```java
interface AccountRepository extends JpaRepository<AccountJpaEntity, Long> {
}
```

다음은 `ActivityRepository` 코드다.
```java
interface ActivityRepository extends JpaRepository<ActivityJpaEntity, Long> 
{
    @Query("select a from ActivityJpaEntity a " +
            "where a.ownerAccountId = :ownerAccountId " +
            "and a.timestamp >= :since")
    List<ActivityJpaEntity> findByOwnerSince(
            @Param("ownerAccountId") Long ownerAccountId,
            @Param("since") LocalDateTime since);

    @Query("select sum(a.amount) from ActivityJpaEntity a " +
            "where a.targetAccountId = :accountId " +
            "and a.ownerAccountId = :accountId " +
            "and a.timestamp < :until")
    Long getDepositBalanceUntil(
            @Param("accountId") Long accountId,
            @Param("until") LocalDateTime until);

    @Query("select sum(a.amount) from ActivityJpaEntity a " +
            "where a.sourceAccountId = :accountId " +
            "and a.ownerAccountId = :accountId " +
            "and a.timestamp < :until")
    Long getWithdrawalBalanceUntil(
            @Param("accountId") Long accountId,
            @Param("until") LocalDateTime until);

}
```

스프링 부트는 이 리포지토리를 자동으로 찾고, 스프링 데이터는 실제로 데이터베이스와 통신하는 리포지토리 인터페이스 구현체를 제공한다.

이 JPA 엔티티와 리포지토리를 바탕으로 영속성 기능을 제공하는 영속성 어댑터를 구현해보자.

```java
@RequiredArgsConstructor
@Component
class AccountPersistenceAdapter implements
        LoadAccountPort,
        UpdateAccountStatePort {

    private final AccountRepository accountRepository;
    private final ActivityRepository activityRepository;
    private final AccountMapper accountMapper;

    @Override
    public Account loadAccount(
            AccountId accountId,
            LocalDateTime baselineDate) {

        AccountJpaEntity account =
                accountRepository.findById(accountId.getValue())
                    .orElseThrow(EntityNotFoundException::new);

        List<ActivityJpaEntity> activities =
            activityRepository.findByOwnerSince(
                accountId.getValue(),
                    baselineDate);

        Long withdrawalBalance = orZero(activityRepository
            .getWithdrawalBalanceUntil(
                accountId.getValue(),
                baselineDate));

        Long depositBalance = orZero(activityRepository
            .getDepositBalanceUntil(
                accountId.getValue(),
                baselineDate));

        return accountMapper.mapToDomainEntity(
            account,
            activities,
            withdrawalBalance,
            depositBalance);

    }

    private Long orZero(Long value){
        return value == null ? 0L : value;
    }


    @Override
    public void updateActivities(Account account) {
        for (Activity activity : account.getActivityWindow().getActivities()) {
            if (activity.getId() == null) {
                activityRepository.save(accountMapper.mapToJpaEntity(activity));
            }
        }
    }

}
```

영속성 어댑터는 애플리케이션에 필요한 `LoadAccountPort`와 `UpdateAccountStatePort`라는 2개의 포트를 구현했다.

데이터베이스로부터 계좌를 가져오기 위해 `AccountRepository`로 계좌를 불러온 다음, `ActivityRepository`로 해당 계좌의 특정 시간 범위 동안의 `activity`를 가져온다.

유효한 `Account` 도메인 엔티티를 생성하기 위해서는 이 `activityWindow`의 시작 직전의 계좌 잔고가 필요하다. 그래야 데이터베이스로부터 모든 출금과 입금 정보를 가져와 합할 수 있다. 마지막으로 이 모든 데이터를 `Account` 도메인 엔티티에 매핑하고 반환한다. 

계좌 상태를 업데이트하기 위해서는 `Account` 엔티티의 모든 `Activity`의 ID가 있는지 확인해야 한다. ID가 없다면 새로운 `Activity`이므로 `ActivityRepository`를 이용해 저장해야 한다.

이러한 시나리오에서는 `Account`와 `Activity` 도메인 모델, `AccountJpaEntity`와 `ActivityJpaEntity` 데이터베이스 모델 간에 양방향 매핑이 존재한다. 그냥 JPA 애너테이션을 `Account`와 `Activity`에 옮기고 그대로 데이터베이스 엔티티로 저장하지 않는 이유가 뭘까?

JPA 엔티티는 기본 생성자가 필요하고, 영속성 계층에서는 성능 측면에서 `@ManyToOne` 관계를 설정하는 것이 적절할 수 있지만 도메인 모델에서는 반대가 되기를 원한다.

그러므로 영속성 측면과의 타협 없이 도메인 모델을 생성하고 싶다면 도메인 모델과 영속성 모델을 매핑하는 것이 좋다.

## 데이터베이스 트랜잭션은 어떻게 해야 할까?
트랜잭션 경계는 어디에 위치시켜야 할까?

트랜잭션은 하나의 특정한 유스케이스에 대해서 일어나는 모든 쓰기 작업에 걸쳐 있어야 한다. 그래야 실패 시 롤백이 가능하다.

영속성 어댑터는 어떤 데이터베이스 연산이 같은 유스케이스에 포함되는지 모르기 때문에 언제 트랜잭션을 열고 닫을지 결정할 수 없다. 이 책임은 영속성 어댑터 호출을 하는 서비스에 위임해야 한다.

자바와 스프링에서 가장 쉬운 방법은 `@Transactional` 애너테이션을 애플리케이션 서비스 클래스에 붙여서 스프링이 모든 `public` 메서드를 트랜잭션으로 감싸게 하는 것이다.
```java
package buckpal.application.service;

@Transactional
public class SendMoneyService implements SendMoneyUseCase {
    ...
}
```

만약 서비스가 `@Transactional` 애너테이션으로 오염되지 않고 깔끔하게 유지되길 원한다면 AspectJ 같은 도구를 이용해 관점 지향 프로그래밍(aspect-oriented programming)으로 트랜잭션 경계를 코드에 위빙(weaving)할 수 있다.

## 유지보수 가능한 소프트웨어를 만드는 데 어떻게 도움이 될까?
도메인 코드에 플러그인처럼 동작하는 영속성 어댑터를 만들면 도메인 코드가 영속성과 관련된 것들로부터 분리되어 도메인 모델을 만드는 것이 편해진다. 또한 좁은 포트 인터페이스를 사용하면 포트마다 다른 방식으로 구현할 수 있는 유연함이 생기고 다른 영속성 기술을 사용하거나 영속성 계층 전체를 교체할 수도 있다.