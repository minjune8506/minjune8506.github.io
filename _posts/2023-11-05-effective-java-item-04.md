---
layout: post
title: 인스턴스화를 막으려거든 private 생성자를 사용하라
date: YYYY-MM-DD HH:MM:SS +09:00
categories:
- Java
- effective-java
tags:
- private constructor
---

# 인스턴스화를 막으려거든 private 생성자를 사용하라

static method와 static field만을 담은 클래스를 만들 때가 있다.

이러한 유틸리티 클래스들은 인스턴스로 만들어 사용하려고 설계 한 것이 아니다.

하지만, 생성자를 명시하지 않으면 컴파일러가 자동으로 기본 생성자를 생성한다.

추상 클래스를 만드는 것으로는 인스턴스화를 막을 수 없다.

하위 클래스를 만들어 인스턴스화 할 수 있기 떄문이다. 또한, 추상 클래스는 사용자가 상속해서 사용하라는 의미로 오해할 수 있다.

private 생성자를 추가하면 간단하게 인스턴스화를 막을 수 있다.

```java
public class Util {
	private Util() {
		throw new AssertionError();
	}
}
```

private 생성자이므로 클래스 밖에서는 접근할 수 없고, 상속받을 수도 없다.

생성자 내부에서 AssertionError 예외를 발생시키도록 해놓으면 클래스 내부에서의 호출도 막을 수 있다.
