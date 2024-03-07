---
layout: post
title: AbstractAggregateRoot
date: YYYY-MM-DD HH:MM:SS +09:00
author: minjun
categories:
- JPA
- Event
tags:
- JPA
- Event
- AbstractAggregateRoot
---
# AbstractAggregateRoot

package: org.springframework.data.domain

aggregate root를 위한 편의 base class

```java
public class AbstractAggregateRoot<A extends AbstractAggregateRoot<A>> {
    @Transient
    private final transient List<Object> domainEvents = new ArrayList();

    public AbstractAggregateRoot() {
    }

    protected <T> T registerEvent(T event) {
        Assert.notNull(event, "Domain event must not be null");
        this.domainEvents.add(event);
        return event;
    }

    @AfterDomainEventPublication
    protected void clearDomainEvents() {
        this.domainEvents.clear();
    }

    @DomainEvents
    protected Collection<Object> domainEvents() {
        return Collections.unmodifiableList(this.domainEvents);
    }

    protected final A andEventsFrom(A aggregate) {
        Assert.notNull(aggregate, "Aggregate must not be null");
        this.domainEvents.addAll(aggregate.domainEvents());
        return this;
    }

    protected final A andEvent(Object event) {
        this.registerEvent(event);
        return this;
    }
}
```

`@AfterDomainEventPublication`

spring data에 의해 관리되는 aggregate의 method에 사용

aggregate의 event가 발행된 후에 호출할 method를 표시한다.

`@DomainEvents`

spring data repository에 의해 관리되는 aggregate root의 method에 사용

method에서 반환된 이벤트를 spring application event로 발행할 수 있다.

`registerEvent()`

spring data repository의 **save**나 **delete** method가 호출될때 발행되는 event 객체를 등록한다.


## AbstractAggregateRoot 사용시 주의점

AbstractAggregateRoot를 상속한 entity를 사용할때 spring data repository의 `save()`나 `delete()` 가 호출될떄 event가 발행된다.

JPA를 사용하면 transaction이 끝나는 시점에 entity의 변경을 감지하고 update 쿼리를 날리게 된다. (dirty checking)

이때는 어떻게 이벤트를 발행해야 할까?

예를 들어 회원의 전화번호가 바뀌었을때 domain event를 발행하고 싶다고 해보자.

```java
// Member.java
public void updatePhoneNumber(String phoneNumber) {
	this.phoneNumber = phoneNumber;
	registerEvent(new MemberPhoneNumberUpdatedEvent(this)); // 이벤트 등록
}
```

```java
// MemberService.java
@Transactional
public void update(Long id, String phoneNumber) {
	var member = memberRepository.findById(id)
					.orElseThrow(() -> new RuntimeException("not found member"));
	member.updatePhoneNumber(phoneNumber);
}
```

```java
// MemberEventListener.java
@TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT)
public void beforeCommit(MemberPhoneNumberUpdatedEvent event) {
	Member member = event.getMember();
	log.info("before commit member phoneNumber: {}", member.getPhoneNumber());
}
```

![log](/assets/img/posts/abstract-aggregate-root.png)

위의 로그를 보면 update query가 실행되고 나서 event listener가 정상적으로 작동되지 않는걸 볼 수 있다.

따라서 JPA의 dirty checking 기능을 사용하여 update 쿼리를 날리는 경우 domain event를 정상적으로 발행하고 싶다면 아래와 같이 save를 명시적으로 호출해주어야 한다.

```java
// MemberService.java
@Transactional
public void update(Long id, String phoneNumber) {
	var member = memberRepository.findById(id)
					.orElseThrow(() -> new RuntimeException("not found member"));
	member.updatePhoneNumber(phoneNumber);
	memberRepository.save(member); // save 호출
}
```
