---
title: "[만들면서 배우는 클린 아키텍처] 08. 경계 간 매핑하기"
date: 2025-07-05 01:26:22 +0900
categories: [book, 만들면서 배우는 클린 아키텍처]
tags: [architecture, study]
image: /assets/img/study/clean_architecture/thumbnail.jpg
---

> 두 계층 간에 매핑을 하지 않으면 양 계층에서 같은 모델을 사용해야 하는데 그러면 두 계층이 강하게 결합된다. 계층 간에 매핑을 하게 되면 보일러플레이트 코드를 너무 많이 만들게 되고, 계층에 걸쳐 같은 모델을 사용하기 때문에 계층 사이의 매핑이 과하게 느껴질 수 있다. 이러한 장단점이 바탕으로 매핑 전략을 적절하게 수립해야 한다.

---

## '매핑하지 않기' 전략
![그림8.1](/assets/img/study/clean_architecture/example_image_8_1.png)

매핑하지 않기(no mapping) 전략에서는 웹 컨트롤러가 `SendMoneyUseCase` 인터페이스를 호출해서 실행한다. 이 인터페이스는 `Account` 객체를 인자로 가지며, 이는 웹 계층과 애플리케이션 계층 모두 `Account` 클래스에 접근해야 한다는 것(두 계층이 같은 모델을 사용한다)을 의미한다.

반대쪽의 영속성 계층과 애플리케이션 계층도 같은 관계다. 모든 계층이 같은 모델을 사용하니 계층 간 매핑을 전혀 하지 않아도 된다.

웹 계층과 영속성 계층은 모델에 대해 특별한 요구사항이 있을 수 있다. 웹 계층에서 REST로 모델을 노출시켰다면 모델을 JSON으로 직렬화하기 위한 애너테이션을 모델 클래스의 특정 필드에 붙여야 할 수도 있다. 영속성 계층에 대해서도 마찬가지로 ORM 프레임워크를 사용한다면 데이터베이스 매핑을 위한 특정 애너테이션이 필요하다.

도메인과 애플리케이션 계층은 웹이나 영속성과 관련된 특수한 요구사항에 관심이 없음에도 `Account` 도메인 모델 클래스는 이러한 요구사항을 다뤄야 하는 것이다. 이러한 상황은 "컴포넌트를 변경하는 이유는 오직 하나뿐이어야 한다."는 단일 책임 원칙을 위반한다.

그럼 매핑하지 않기 전략은 언제 사용해야 할까? 간단한 CRUD 유스케이스를 생각해보자. 같은 필드를 가진 웹 모델을 도메인 모델로, 혹은 도메인 모델을 영속성 모델로 매핑할 필요가 있을까?

모든 계층이 정확히 같은 구조의, 정확히 같은 정보를 필요로 한다면 매핑하지 않기 전략이 가장 적합하다.

---

## '양방향' 매핑 전략
![그림8.2](/assets/img/study/clean_architecture/example_image_8_2.png)

각 계층이 전용 모델을 가진 매핑 전략을 양방향(two-way) 매핑 전략이라고 한다. 각 계층은 도메인 모델과는 완전히 다른 구조의 전용 모델을 가지고 있다.

웹 계층에서는 웹 모델을 인커밍 포트에서 필요한 도메인 모델로 매핑하고, 인커밍 포트에 의해 반환된 도메인 객체를 다시 웹 모델로 매핑한다.

영속성 계층은 아웃고잉 포트가 사용하는 도메인 모델과 영속성 모델 간의 매핑과 유사한 매핑을 담당한다.

각 계층이 전용 모델을 가지고 있어 변경하더라도 다른 계층에는 영향이 없다. 그래서 웹 모델은 데이터를 최적으로 표현할 수 있는 구조를 가질 수 있고, 도메인 모델은 유스케이스를 가장 잘 구현할 수 있는 구조를, 영속성 모델은 데이터베이스에 객체를 저장하기 위해 ORM에서 필요로 하는 구조를 가질 수 있다.

이 매핑 전략은 웹이나 영속성 관심사로 오염되지 않은 깨끗한 도메인 모델로 이어진다. JSON이나 ORM 매핑 애너테이션도 없어도 되므로 단일 책임 원칙을 만족하게 된다.

또 다른 장점은 개념적으로 매핑하지 않기 전략 다음으로 간단하고 책임이 명확하다. 바깥쪽 계층과 어댑터는 안쪽 계층의 모델로 매핑하고, 다시 반대 방향으로 매핑한다. 안쪽 계층은 해당 계층의 모델만 알면 되고 도메인 로직에 집중할 수 있다.

양방향 매핑의 단점은 너무 많은 보일러플레이트 코드가 생긴다는 것이다. 코드의 양을 줄이기 위해 매핑 프레임워크를 사용하더라도 두 모델 간의 매핑을 구현하는 데 꽤 시간이 든다. 매핑 프레임워크의 내부 동작 방식을 모르는 경우 디버깅도 어렵다.

도메인 모델이 계층 경계를 넘어서 통신하는 데 사용되고 있다는 것도 문제이다. 인커밍 포트와 아웃고잉 포트는 도메인 객체를 입력 파라미터와 반환값으로 사용한다. 도메인 모델은 도메인 모델의 필요에 의해서만 변경되는 것이 이상적이지만 바깥쪽 계층의 요구에 따른 변경에 취약해진다.

---

## '완전' 매핑 전략
![그림8.3](/assets/img/study/clean_architecture/example_image_8_3.png)
완전 매핑 전략은 각 연산마다 별도의 입출력 모델을 사용한다. 계층 경계를 넘어 통신할 때 도메인 모델을 사용하는 대신 `SendMoneyUseCase` 포트의 입력 모델로 동작하는 `SendMoneyCommand`처럼 각 작업에 특화된 모델을 사용한다. 이런 모델을 command, request와 같이 표현한다.

웹 계층은 입력을 애플리케이션 계층의 command 객체로 매핑할 책임을 가지고 있다. command 객체는 애플리케이션 계층의 인터페이스를 명확하게 만들어 준다. 각 유스케이스는 전용 필드와 유효성 검증 로직을 가진 전용 command를 가지며, 값을 비워둘 수 있는 필드를 허용할 경우에는 필요 없는 유효성 검증이 수행될 수도 있기 때문에 허용하면 안된다.

애플리케이션 계층은 command 객체를 유스케이스에 따라 도메인 모델을 변경하기 위해 필요한 무엇인가로 매핑할 책임을 가진다.

한 계층을 다른 여러 개의 command로 매핑하는 데는 하나의 웹 모델과 도메인 모델 간의 매핑보다 더 많은 코드가 필요하다. 하지만 이렇게 매핑하면 여러 유스케이스의 요구사항을 함께 다뤄야 하는 매핑에 비해 구현과 유지보수가 쉬워진다.

이 매핑 전략은 전역 패턴보다는 웹 계층(혹은 인커밍 어댑터 종류 중 아무거나)과 애플리케이션 계층 사이에서 상태 변경 유스케이스의 경계를 명확하게 할 때 가장 좋다. 애플리케이션 계층과 영속성 계층 사이에서는 매핑 오버헤드 때문에 사용하지 않는 것이 좋다.

`SendMoneyUseCase`가 업데이트된 잔고를 가진 채로 `Account` 객체를 그대로 반환하는 것처럼 연산의 입력 모델에 대해서만 이 매핑을 시도하고, 도메인 객체를 그대로 출력 모델로 사용하는 것도 좋다.

이처럼 매핑 전략은 여러 가지를 섞어서 쓸 수 있고 그래야만 한다. 어떤 매핑 전략도 모든 계층에 적용되는 전역 규칙일 필요가 없다.

---

## '단방향' 매핑 전략
![그림8.4](/assets/img/study/clean_architecture/example_image_8_4.png)
단방향(one-way) 매핑 전략은 모든 계층의 모델들이 같은 인터페이스를 구현한다. 이 인터페이스는 관련 있는 특성(attribute)에 대한 `getter` 메서드를 제공해서 도메인 모델의 상태를 캡슐화한다.

도메인 모델 자체에서 행동을 구현할 수 있고, 애플리케이션 계층 내의 서비스에서 이러한 행동에 접근할 수 있다. 도메인 객체가 인커밍/아웃고잉 포트에 필요한 상태 인터페이스를 구현하고 있기 때문에 매핑 없이 도메인 객체를 바깥 계층으로 전달할 수 있다.

바깥 계층에서는 상태 인터페이스를 사용할지, 전용 모델로 매핑할지를 결정한다. 행동을 변경하는 것이 상태 인터페이스에 노출되어 있지 않기 때문에 실수로 도메인 객체의 상태가 변경될 우려가 없다.

바깥 계층에서 애플리케이션 계층으로 전달하는 객체들도 이 상태 인터페이스를 구현하고 있다. 애플리케이션 계층에서는 이 객체를 실제 도메인 모델로 매핑해서 도메인 모델의 행동에 접근할 수 있게 된다. 이 매핑은 팩터리(factory)라는 DDD 개념과 잘 어울린다. DDD 용어인 팩터리는 특정 상태로부터 도메인 객체를 재구성할 책임을 가지고 있다.
> 단방향 매핑 전략으로 상태 인터페이스를 구현한 객체를 통해 데이터만 전달이 가능하고, 팩터리 메서드로 상태 기반 객체로부터 도메인 객체를 안전하게 생성할 수 있어서 잘 어울린다고 표현한 것 같다.

이 전략에서 매핑 책임은 명확하다. 한 계층이 다른 계층으로부터 객체를 받으면 해당 계층에서 이용할 수 있도록 다른 무언가로 매핑한다. 그러므로 각 계층은 한 방향으로만 매핑하고, 그래서 이름이 단방향 매핑 전략인 것이다.

이 전략은 계층 간 모델이 비슷할 때 효과적이다. 예를 들어, 읽기 전용 연산의 경우 상태 인터페이스가 필요한 모든 정보를 제공하기 때문에 웹 계층에서 전용 모델로 매핑할 필요가 없다.

---

## 언제 어떤 매핑 전략을 사용할 것인가?
각 매핑 전략이 장단점을 가지고 있기 때문에 상황과 목적에 맞게 사용해야 한다. 팀 내에서 합의할 수 있는 가이드라인을 정해둬야 한다. 왜 해당 전략을 최우선적으로 택해야 하는지 설명할 수 있어야 시간이 흐른 뒤에도 근거를 바탕으로 여전히 유효한 전략인지 평가가 가능하다. 그리고 시간이 흐르면서 최선의 전략이 바뀌기 때문에 가이드라인을 팀 차원에서 지속적으로 논의하고 수정해야 한다.

---

## 유지보수 가능한 소프트웨어를 만드는 데 어떻게 도움이 될까?
인커밍 포트와 아웃고잉 포트는 서로 다른 계층이 어떻게 통신해야 하는지를 정의한다. 여기에는 계층 사이에 매핑을 수행할지 여부와 어떤 매핑 전략을 선택할지도 포함된다.

각 유스케이스에 대해 좁은 포트를 사용하면 유스케이스마다 다른 매핑 전략을 사용할 수 있고, 다른 유스케이스에 영향을 미치지 않으면서 코드를 개선할 수 있기 때문에 특정 상황, 특정 시점에 최선의 전략을 선택할 수 있다.