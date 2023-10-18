---
layout: post
title: 생성자에 매개변수가 많다면 빌더를 고려하라
date: YYYY-MM-DD HH:MM:SS +09:00
categories:
  - Java
  - effective-java
tags:
  - Builder
---

# 생성자에 매개변수가 많다면 빌더를 고려하라

정적 팩토리 메서드(static factory method)와 생성자에는 선택적인 매개변수가 많을 때 적절하게 대응하기 어렵다는 제약조건이 있다.

한 객체를 생성하는데 생성자에 20개의 매개변수가 필요하다면?

생성자, 정적 팩토리 메서드는 불필요한 매개변수를 모두 입력 받아야 한다는 단점이 있다.

이를 해결하기 위해 점층적 생성자 패턴(telescoping constructor pattern), 자바 빈즈 패턴(java beans pattern), 빌터 패턴(builder pattern)의 대안이 있다.

## 점층적 생성자 패턴(telescoping constructor pattern)

필수 매개변수를 1개만 받는 생성자, 2개까지 받는 생성자 ... 의 형태로 갯수를 늘려가는 방식

### 장점

- 원하는 매개변수를 모두 포함한 생성자 중 가장 짧은 것을 사용할 수 있다.

### 단점

- 사용자가 설정하고 싶지 않은 매개변수도 값을 넣어주어야 한다.
- 매개변수 개수가 많아지면 코드를 작성하거나 읽기 어렵다. (조합이 너무 많아진다)

```java
public class Person {
    private final String name;
    private final int age;
    private final String department;
    private final boolean marriage;
    private final boolean hasChild;

    public Person(String name, int age) {
        this(name, age, null);
    }

    public Person(String name, int age, String department) {
        this(name, age, department, false);
    }

    public Person(String name, int age, String department, boolean marriage) {
        this(name, age, department, marriage, false);
    }

    public Person(String name, int age, String department, boolean marriage, boolean hasChild) {
        this.name = name;
        this.age = age;
        this.department = department;
        this.marriage = marriage;
        this.hasChild = hasChild;
    }

	public static void main(String[] args) {
		Person person = new Person("test", 20);
	}
}
```

## 자바 빈즈 패턴(java beans pattern)

매개변수가 없는 생성자로 객체를 만든 후, setter 메서드들을 호출해 원하는 값을 설정하는 방식

### 장점

- 선택 매개변수가 많을 떄 활용 가능하다. (원하는 매개변수만 선택해서 설정 가능)

### 단점

- 객체 하나를 만들려면 여러 메서드를 호출해야 한다.
- 객체가 완전히 생성되기 전에는 일관성(consistency)이 무너진 상태에 놓이게 된다.
- 클래스를 불변(immutable)으로 만들 수 없다. (non thread safe)

```java
public class PersonJavaBeans {
    private String name;
    private int age;
    private String department;
    private boolean marriage;
    private boolean hasChild;

    public PersonJavaBeans() {
    }

    public void setName(String name) {
        this.name = name;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public void setDepartment(String department) {
        this.department = department;
    }

    public void setMarriage(boolean marriage) {
        this.marriage = marriage;
    }

    public void setHasChild(boolean hasChild) {
        this.hasChild = hasChild;
    }

	public static void main(String[] args) {
        Person person = new Person();
        person.setName("test");
        person.setAge(20);
    }
}

```

## 빌더 패턴(builder pattern)

필요한 객체를 직접 만드는 대신, 필수 매개변수만으로 생성자를 호출해 빌더 객체를 얻는다.

빌더 객체가 제공하는 일종의 setter 메서드들을 활용해 선택적 매개변수의 값을 설정하고, build 메서드를 호출해 객체를 얻는다.

빌더 클래스는 정적 멤버 클래스(static member class)로 만드는 것이 관례이다.

점층적 생성자 패턴(telescoping constructor pattern)과 자바 빈즈 패턴(java beans pattern)의 장점을 모두 채택한 패턴이다.

### 장점

- 불변으로 객체를 만들 수 있다.
- 객체를 생성하기 쉽고, 가독성이 좋다.
- 계층적으로 설계된 클래스와 함께 사용하기 좋다.

### 단점

- 빌더 객체를 만들어야 하므로 성능에 민감한 상황에는 문제가 될 수 있다.
- 추가적인 코드가 필요하다. (매개변수가 많을때 사용하는 것을 추천한다.)

```java
public class Person {
    private final String name;
    private final int age;
    private final String department;
    private final boolean marriage;
    private final boolean hasChild;

    private Person(Builder builder) {
        this.name = builder.name;
        this.age = builder.age;
        this.department = builder.department;
        this.marriage = builder.marriage;
        this.hasChild = builder.hasChild;
    }

    public static class Builder {
        private final String name;
        private final int age;
        private String department;
        private boolean marriage;
        private boolean hasChild;

        public Builder(String name, int age) {
            this.name = name;
            this.age = age;
        }

        public Builder department(String department) {
            this.department = department;
            return this;
        }

        public Builder marriage(boolean marriage) {
            this.marriage = marriage;
            return this;
        }

        public Builder hasChild(boolean hasChild) {
            this.hasChild = hasChild;
            return this;
        }

        public Person build() {
            return new Person(this);
        }
    }

	public static void main(String[] args) {
        Person person = new Person.Builder("test", 20)
                                .department("dev")
                                .marriage(true)
                                .hasChild(false)
                                .build();
        System.out.println(person);
    }
}
```

## 용어 정리

**불변식(invariant)**

프로그램이 실행되는 동안, 혹은 정해진 기간 동안 반드시 만족해야 하는 조건.

변경을 허용할 수는 있으나, 주어진 조건 내에서만 허용한다.

예를 들어, 시작일은 항상 종료일보다 빨라야 한다는 조건
