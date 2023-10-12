---
layout: post
title: Equals와 HashCode
date: YYYY-MM-DD HH:MM:SS +09:00
categories:
- Java
- Basic
tags:
- EqualsAndHashCode
---

equals와 hashcode는 모든 Java 객체의 부모 객체인 Object 클래스에 정의되어 있다.

> Object에서 final이 아닌 메서드(eqauls, hashcode, toString, clone, finalize)는 모두 Override를 염두에 두고 설계되었다.
> Override할때 지켜야 하는 규약이 명확히 정해져 있다.
> - Effective Java Chapter 03.

## equals()

```java
public boolean equals(Object obj)
```

Identity가 아닌 논리적으로 두 객체가 동일한지 판단할때 사용한다.

이때 상위 클래스의 equals가 논리적 동치성을 비교하도록 재정의되지 않았다면 equals를 Override한다.

**reflexive (반사성)**

null이 아닌 참조 값 x에 대해, x.equals(x)는 true를 반환해야 한다.

**symmetric (대칭성)**

null이 아닌 참조값 x와 y에 대해, x.equals(y)는 y.equals(x)가 true를 반환할때만 true를 반환해야 한다.

**transitive (추이성)**

null이 아닌 참조값 x, y, z에 대해 x.equals(y)가 true이고 y.equals(z)가 true이면 x.equals(z)도 true를 반환해야 한다.

**consistent (일관성)**

null이 아닌 참조값 x, y에 대해 x.equals(y)를 여러번 수행해도 일관성있게 true / false를 반환해야 한다.

**null**

null이 아닌 참조값 x에 대해서, x.equals(null)은 false를 반환해야 한다.

equals를 재정의할때는 hashcode도 항상 재정의해야 한다.

equals 메서드는 꼭 필요한 경우가 아니라면 Override하지 않는것이 좋다.

## hashcode()

Object의 hash code 값을 반환한다.

HashMap과 같은 hash table을 사용할때 사용된다.

equals()만 재정의하고 hashcode()는 재정의하지 않을때 발생하는 문제점

- Collection 프레임워크에서 객체가 같은지 비교하는 로직에서 문제 발생
- hashCode() 같은가? → equals() 같은가? → 동등 객체
- **hashCode를 재정의하지 않으면 equals를 Overriding하형 논리적으로 같은 객체라고 생각했던 것들이 다른 객체로 판단되어 개발자의 의도대로 작동하지 않는다.**

Java의 StringLatin1 class의 hashCode 함수는 다음과 같이 구현되어 있다.

- StringLatin1 클래스는 Java 9부터 도입된 Compact String의 한 부분이다.

```java
public static int hashCode(byte[] value) {
	int h = 0;
	for (byte v : value) {
		h = 31 * h + (v & 0xff); // 31 : 홀수 + 소수
	}
	return h;
}
```

## 결론

- equals() 메서드는 꼭 필요한 경우에만 재정의하자
- equals() 메서드를 재정의할 때는 5가지 특성이 만족하는지 확인하여야 한다. (IDE의 지원을 받아도 된다)
- equals() 메서드를 재정의한다면 hashcode()도 꼭 재정의해야 한다.

## 참고

- https://tecoble.techcourse.co.kr/post/2020-07-29-equals-and-hashCode/
- https://docs.oracle.com/en/java/javase/11/docs/api/
- https://brunch.co.kr/@mystoryg/133
- https://inyl.github.io/programming/2017/09/27/compact_string.html
- Effective Java ch03
