---
layout: post
title: private 생성자나 열거 타입으로 싱글턴임을 보장하라
date: YYYY-MM-DD HH:MM:SS +09:00
categories:
  - Java
  - effective-java
tags:
  - singleton
---

# private 생성자나 열거 타입으로 싱글턴임을 보장하라

## singleton 패턴

인스턴스를 오직 하나만 생성할 수 있는 패턴

함수와 같은 stateless한 객체나 설계상 유일해야 하는 시스템 컴포넌트에 사용하기 적합하다.

### 장점

- 최초 한번의 new 연산자를 통해 객체를 생성하고 추후에 생성한 객체를 계속 이용함으로써 메모리, 객체 생성비용을 절약할 수 있다는 장점이 있다.

### 단점

- 클래스를 singleton으로 만들면 테스트하기가 어려워질 수 있다.
- singleton이 interface를 구현해서 만든 객체가 아니라면 mock으로 테스트할 수 없다.
- 자식 클래스를 만들 수 없고, 내부 상태를 변경한다면 동시성 문제를 고려해야 한다.

## singleton 패턴 구현 방식

### public static final 필드 방식

```java
public class Singleton {
    public static final Singleton INSTANCE = new Singleton();
    private Singleton() {}
}
```

#### 장점

- public static final 필드를 통해 해당 클래스가 싱글턴임이 API에 명백히 드러나고, 간결하다는 장점이 있다.

#### 단점

- Reflection API를 이용해 private 생성자를 생성할 수 있다는 문제점이 있다.
- 직렬화시 필드 transient, readResolve 메서드를 제공해야 한다.

Reflection API 문제점은 생성자에서 해당 객체가 이미 존재하면 예외를 던지는 방식으로 해결할 수 있다.

```java
private Singleton() {
	if (INSTANCE != null) {
		throw new RuntimeException("...");
	}
}
```

### static factory 방식

```java
public class Singleton {
    private static final Singleton INSTANCE = new Singleton();

    private Singleton() {
    }

    public static Singleton getInstance() {
        return INSTANCE;
    }
}
```

#### 장점

- API를 변경하지 않고도 singleton이 아니게 변경할 수 있다.
- static factory를 generic singleton factory로 만들 수 있다.
- static factory의 메서드 참조를 supplier로 사용할 수 있다.

#### 단점

- Reflection API를 이용해 private 생성자를 생성할 수 있다는 문제점이 있다.
- 직렬화시 필드 transient, readResolve 메서드를 제공해야 한다.

### 열거 타입 방식

```java
public enum Singleton {
    INSTANCE;
}
```

간결하고, 추가적인 작업 없이 직렬화가 가능하다.

Reflection API에 의한 공격도 완벽하게 막아준다.

대부분 상황에서는 원소가 하나뿐인 열거 타입이 싱글턴을 만드는 가장 좋은 방법이다.

하지만 싱글턴 클래스가 Enum 이외의 다른 클래스를 상속하는 경우는 사용할 수 없다.
