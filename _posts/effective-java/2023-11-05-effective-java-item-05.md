---
layout: post
title: 자원을 직접 명시하지 말고 의존 객체 주입을 사용하라
date: YYYY-MM-DD HH:MM:SS +09:00
author: minjun
categories:
- java
- effective-java
tags:
- dependency injectin
---

# 자원을 직접 명시하지 말고 의존 객체 주입을 사용하라

클래스가 하나 이상의 자원에 의존하는 경우가 있다.

맞춤법 검사기는 사전에 의존한다고 가정해보자.

유틸리티 클래스, 싱글턴 클래스로 구현할 수 있다.

```java
// 정적 유틸리티를 잘못 설계한 예 - 유연하지 않고 테스트가 어렵다.
public class SpellChecker {
    private static final KoreanDictionary koreanDictionary = new KoreanDictionary();

    private SpellChecker() {
    }
}
```

```java
// 싱글턴을 잘못 사용한 예 - 유연하지 않고 테스트가 어렵다.
public class SpellChecker {
    public static SpellChecker INSTANCE = new SpellChecker();
    private final KoreanDictionary koreanDictionary = new KoreanDictionary();

    private SpellChecker() {
    }
}
```

두 방식 모두 **하나의 사전에 의존**한다는 문제점이 있다.

영어 사전이나, 아랍어 사전이 필요한 경우는 어떻게 대처할 수 있을까?

SpellChecker에서 dictionary를 final이 아니게 만든뒤, 다른 사전으로 교체하는 방법도 있지만, 오류가 발생하기 쉽고 multi-thread 환경에서 사용할 수 없다.

**사용하는 자원에 따라 동작이 달라지는 클래스에는 정적 유틸리티 클래스나 싱글턴 방식이 적합하지 않다.**

인스턴스를 생성할 떄 생성자에 필요한 자원을 넘겨주는 방식을 사용할 수 있다.

이를 의존 객체 주입이라고 한다.

## 의존 객체 주입

```java
public interface Dictionary {
}

public class SpellChecker {
    private final Dictionary dictionary;

    public SpellChecker(Dictionary dictionary) {
        this.dictionary = dictionary;
    }

    public static boolean isValid(String word) {
        return true;
    }

    public static List<String> suggestions(String type) {
        return List.of();
    }
}
```

- 불변을 보장하여 여러 클라이언트가 의존 객체를 안심하고 공유할 수 있다.
- 생성자, 정적 팩터리, 빌더 모두에 똑같이 응용할 수 있다.
- 생성자에 자원 팩터리를 넘겨주는 방식으로 응용 가능하다.

의존 객체 주입은 유연성과 테스트 용이성을 개선해주지만, 의존성이 많은 큰 프로젝트는 코드를 이해하기 어렵게 만든다.


> 클래스가 내부적으로 하나 이상의 자원에 의존하고, 그 자원이 클래스의 동작에 영향을 준다면 싱글턴과 정적 유틸리티 클래스는 사용하지 않는 것이 좋다. 이 자원들을 클래스가 직접 만들게 해서도 안된다. 대신 필요한 자원을 생성자에 넘겨주자.
의존 객체 주입은 클래스의 유연성, 재사용성, 테스트 용이성에 효과적이다. - effective java item 05
