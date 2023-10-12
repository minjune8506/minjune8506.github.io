---
layout: post
title: Serialization / Deserialization
date: YYYY-MM-DD HH:MM:SS +09:00
categories:
- Java
- Basic
tags:
- Serialization
---
## Serialization / Deserialization

Serialization은 객체의 상태를 byte stream으로 변환하는 방법이다.

Deserialization은 byte stream을 메모리의 Java Object로 재생성하기 위해 사용되는 방법이다.

![](https://media.geeksforgeeks.org/wp-content/cdn-uploads/gq/2016/01/serialize-deserialize-java.png)
_출처: https://media.geeksforgeeks.org/wp-content/cdn-uploads/gq/2016/01/serialize-deserialize-java.png_

byte stream은 platform independent하게 만들어진다.

따라서, platform에서 serialized된 객체는 다른 platform에서 deserialized될 수 있다.

**객체의 상태를 저장하거나 / 네트워크, 파일로 전송하려고 할때 사용할 수 있다.**

### java.io.Serializable

Java 객체를 serializable하게 만드려면 Serializable interface를 구현해야 한다.

Serializable 인터페이스는 Marker Interface로 인터페이스 안에 property나 method가 존재하지 않는다.

Marker Interface는 말 그대로 `표시(mark)`하는 역할만 하게 된다.

대표적인 marker interface에는 Serializable, Cloneable이 있다.

**Marker Interface는 어떻게 사용될까?**

객체를 파일로 쓰거나 네트워크를 통해 전송할때 **ObjectOutputStream**을 사용한다.

이때 writeObject 메서드의 내부에서 쓰이는 로직이다.

![writeObject](/assets/img/posts/writeobject.png)
_WriteObject_

if 문을 보다가 보면 `obj instanceof Serializable` 부분이 있다.

여기서 객체가 Serializable을 구현했는지 검사한다.

구현하지 않았다면 `NotSerializableException` 예외를 발생시키게 된다.

**Serialization 특징**

- 부모 클래스가 Serializable interface를 구현했다면 자식 클래스는 Serializable interface를 구현하지 않아도 된다.
    
    하지만 이 반대는 성립하지 않는다.
    
- `non-static` 멤버들만 Serialization을 통해 저장된다.
- `static / transient` 멤버들은 Serialization을 통해 저장되지 않는다.
- 객체가 deserialized될때 객체의 **생성자는 호출되지 않는다.**
- 연관 객체들도 반드시 Serializable interface를 구현해야 한다.
    
    ex) class A 내부에 class B의 객체를 가지고 있는 경우
    

### SerialVersionUID

Serialization 런타임은 Serializable class에 version number를 매핑시킨다.

이때 이 버전 번호를 `SerialVersionUID`라고 한다.

Deserialization을 수행할 때, SerialVersionUID를 이용해 sender와 receiver가 서로 호환가능한 객체를 주고받고 있는지 검사한다.

이때 UID가 다르면 `InvalidClassException` 예외가 발생한다.

```java
class Example {
	접근제어자 static final long serialVersionUID = 숫자L;
}
```

명시적으로 SerialVersionUID를 선언하지 않으면 serialization runtime이 계산한 값을 부여한다.

**그러나 serialVersionUID 값을 명시적으로 선언하는 것이 바람직하다.**

serialization runtime이 계산을 통해 값을 부여하게 되면 클래스의 내부 변경사항에 민감하게 작동한다.

클래스의 내부 멤버들이 변할때 UID가 달라질 수 있고, 이때 `InvalidClassException` 예외가 발생할 수 있기 때문이다.

### Example

```java
package study;

import java.io.Serial;
import java.io.Serializable;

public class Test implements Serializable {
    @Serial
    private static final long serialVersionUID = 123456L;
    private static final String staticFinalVal = "static final value";
    public static String staticVal = "static value";
    private final transient String finalTransientVal = "transient final value";
    private transient String transientVal = "transient value";
    private String id;

    @Override
    public String toString() {
        return "Student{" +
                "transient value='" + transientVal + '\'' +
                ", final transient value='" + finalTransientVal + '\'' +
                ", static final value='" + staticFinalVal + '\'' +
                ", static value='" + staticVal + '\'' +
                '}';
    }
}
```

Test Class 선언

```java
package study;

import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;

public class TestMain {
    public static void main(String[] args) {
        Test student = new Test("42");

        try (ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream("student.obj"));
             ObjectInputStream in = new ObjectInputStream(new FileInputStream("student.obj"))) {
            out.writeObject(student);
            System.out.println("=====after serialized=====");
            System.out.println(student);

            Test.staticVal = "Another Static Value";

            Test deserialized = (Test) in.readObject();
            System.out.println("=====after deserialized=====");
            System.out.println(deserialized);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }

    }
}
```

Test Main

Output

```
=====after serialized=====
Student{transient value='transient value', final transient value='transient final value', static final value='static final value', static value='static value', id='42'}
=====after deserialized=====
Student{transient value='null', final transient value='transient final value', static final value='static final value', static value='Another Static Value', id='42'}
```

- transient 값은 직렬화되지 않기 때문에 null로 초기화된다. (객체는 null, 원시값은 0)
- final 키워드가 붙으면 컴파일러가 값으로 치환을 시키기 때문에 transient 키워드가 소용이 없어진다.
- static 멤버들은 직렬화되지 않지만, 역직렬화될때 객체가 가지고 있는 static value로 초기화된다.
- static value 값을 바꾸고 역직렬화했을때 Another Static Value를 가지고 있으므로 위의 내용이 성립
- 다른 기본 값 (id)는 정상적으로 직렬화/역직렬화 수행

### 참고

[Serialization and Deserialization in Java with Example - GeeksforGeeks](https://www.geeksforgeeks.org/serialization-in-java/)
