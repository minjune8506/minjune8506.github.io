---
layout: post
title: 생성자 대신 정적 팩터리 메서드를 고려하라
date: YYYY-MM-DD HH:MM:SS +09:00
author: minjun
categories:
- java
- effective-java
tags:
- static factory method
---
# 생성자 대신 정적 팩터리 메서드를 고려하라

클라이언트가 클래스의 인스턴스를 얻는 정통적인 수단은 public 생성자이다.

클래스는 생성자와 별도로 `정적 팩터리 메서드(static factory method)`를 제공할 수 있다.

정적 팩터리 메서드는 클래스의 인스턴스를 반환하는 단순한 정적 메서드(static method)이다.

Long Boxed Class의 valueOf 연산자도 static factory method의 하나이다.

```java
public static Long valueOf(long l) {
	final int offset = 128;
	if (l >= -128 && l <= 127) { // will cache
		return LongCache.cache[(int)l + offset];
	}
	return new Long(l);
}
```

Long 클래스의 팩터리 메서드를 보면 -128 ~ 127 사이의 값들은 Cache에서 찾아서 반환하는 것을 볼 수 있고, 호출할 때마다 인스턴스를 새로 생성하지 않는다는 정적 팩터리 메서드의 장점도 알아볼 수 있다.

## 장점

- 이름을 가질 수 있다.
- 호출될 때마다 인스턴스를 새로 생성하지 않아도 된다.
- 반환 타입의 하위 타입 객체를 반환할 수 있는 능력이 있다.
- 입력 매개변수에 따라 매번 다른 클래스의 객체를 반환할 수 있다.
- 정적 팩터리 메서드를 작성하는 시점에는 반환할 객체의 클래스가 존재하지 않아도 된다.

## 단점

- 상속을 하려면 public이나 protected 생성자가 필요하다. 즉, 정적 팩터리 메서드만 제공하면 하위 클래스를 만들 수 없다.
- 정적 팩터리 메서드는 프로그래머가 찾기 어렵다.

## 정적 팩터리 메서드 네이밍 방식

```
from : 매개변수를 하나 받아서 해당 타입의 인스턴스를 반환하는 형변환 메서드

of : 여러 매개변수를 받아 적합한 타입의 인스턴스를 반환하는 집계 메서드

valueOf : from과 of의 더 자세한 버전

instance, getInstance : 매개변수로 명시한 인스턴스를 반환하지만, 같은 인스턴스임을 보장하지 않는다.

create, newInstance : instance, getInstace와 같지만 매번 새로운 인스턴스를 생성해 반환함을 보장한다.

getType : getInstance와 같으나, 생성할 클래스가 아닌 다른 클래스에 팩터리 메서드를 정의할 때 사용한다. "Type"은 팩터리 메서드가 반환할 객체의 타입이다.

newType : newInstance와 같으나, 생성할 클래스가 아닌 다른 클래스에 팩터리 메서드를 정의할 때 사용한다. "Type"은 팩터리 메서드가 반환할 객체의 타입이다.

type : getType과 newType의 간결한 버전
```

## 정리

정적 팩터리 메서드와 public 생성자는 장단점을 이해하고 사용하는 것이 좋다.

대부분의 경우 정적 팩터리 메서드의 장점이 도움이 되는 경우가 많으니 고려해 보는 것이 좋다.
