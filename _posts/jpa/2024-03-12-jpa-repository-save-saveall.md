---
layout: post
title: jpa save와 saveall 성능차이
date: YYYY-MM-DD HH:MM:SS +09:00
categories:
- JPA
tags:
- JPA
---

# JPA의 save와 saveAll의 성능 차이

entity를 영속화시키기 위해 save()와 saveAll()을 사용할 수 있다.

```java
public interface CrudRepository<T, ID> extends Repository<T, ID> {
    <S extends T> S save(S entity);

    <S extends T> Iterable<S> saveAll(Iterable<S> entities);
}
```

save는 단일 entity를 영속화하기 위한 용도로, saveAll은 여러 entity들을 한번에 영속화하기 위한 용도로 쓰인다.

그리고 당연히 한번에 여러개의 객체를 저장하는게 빠를 것이다.

테스트 코드를 작성해서 실행시간을 비교해보자.

```java
// running time :214
@Test
void save() {
	List<Member> members = createMembers(100);
	long start = System.currentTimeMillis();
	for (Member member : members) {
		memberRepository.save(member);
	}
	long end = System.currentTimeMillis();
	System.out.println("running time :" + (end - start));
}

// running time :102
@Test
void saveAll() {
	List<Member> members = createMembers(100);
	long start = System.currentTimeMillis();
	memberRepository.saveAll(members);
	long end = System.currentTimeMillis();
	System.out.println("running time :" + (end - start));
}
```
당연히 saveAll이 더 적은 실행시간을 가지는 것을 확인할 수 있다. 하지만, 똑같이 100번의 insert문이 나가는 것은 동일하였다.

그럼 왜 2배정도의 시간이 차이가 날까?

## 성능 차이

이유를 알아보기 위해 transaction 로그를 활성화하는 설정을 추가하고 로그를 확인해보자.

```yml
logging:
  level:
    org.springframework.transaction: DEBUG
```
### save

```log
Creating new transaction with name [com.example.demo.repository.SaveTest.save]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT 
Opened new EntityManager [SessionImpl(527783934<open>)] for JPA transaction
Exposing JPA transaction as JDBC [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@71d2261e]
Found thread-bound EntityManager [SessionImpl(527783934<open>)] for JPA transaction
 Participating in existing transaction
Hibernate: 
    insert 
    into
        member
        (name,phone_number,id) 
    values
        (?,?,default)
Found thread-bound EntityManager [SessionImpl(527783934<open>)] for JPA transaction
Participating in existing transaction
Hibernate: 
    insert 
    into
        member
        (name,phone_number,id) 
    values
        (?,?,default)
Found thread-bound EntityManager [SessionImpl(527783934<open>)] for JPA transaction
Participating in existing transaction
이후 반복
Initiating transaction rollback
Rolling back JPA transaction on EntityManager [SessionImpl(527783934<open>)]
Closing JPA EntityManager [SessionImpl(527783934<open>)] after transaction
```

### saveAll

```
Creating new transaction with name [com.example.demo.repository.SaveTest.saveAll]: PROPAGATION_REQUIRED ISOLATION_DEFAULT
Opened new EntityManager [SessionImpl(375399219<open>)] for JPA transaction
Exposing JPA transaction as JDBC [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@6d6d81c]
Found thread-bound EntityManager [SessionImpl(375399219<open>)] for JPA transaction
Participating in existing transaction
Hibernate: 
    insert 
    into
        member
        (name,phone_number,id) 
    values
        (?,?,default)
이후 반복
Initiating transaction rollback
Rolling back JPA transaction on EntityManager [SessionImpl(375399219<open>)]
Closing JPA EntityManager [SessionImpl(375399219<open>)] after transaction
```

### 로그 비교

두 로그의 차이를 보면 save 메서드는 insert문이 실행될때마다, saveAll의 경우 처음 한번만 존재하는 transaction에 참가하는 작업을 거친다.

참고로 이미 존재하는 transaction에 참가하는 이유는 PROPAGATION_REQUIRED 모드가 transactional의 default propagation이기 때문이다.

> propagation required : 이미 존재하는 트랜잭션이 있으면 참여하고, 없으면 트랜잭션을 새로 시작한다.

## 정리

```java
@Transactional
public <S extends T> S save(S entity) {
	Assert.notNull(entity, "Entity must not be null");
	if (this.entityInformation.isNew(entity)) {
		this.entityManager.persist(entity);
		return entity;
	} else {
		return this.entityManager.merge(entity);
	}
}

@Transactional
public <S extends T> List<S> saveAll(Iterable<S> entities) {
	Assert.notNull(entities, "Entities must not be null");
	List<S> result = new ArrayList();
	Iterator var3 = entities.iterator();

	while(var3.hasNext()) {
		S entity = var3.next();
		result.add(this.save(entity));
	}

	return result;
}
```

기본적으로 `@DataJpaTest`를 이용한 테스트를 진행하면 트랜잭션 안에서 테스트가 진행되고 테스트가 종료되면 rollback 한다.

따라서 saveAll의 경우 맨 처음에만 기존 트랜잭션에 참가하면되고, 이후에는 트랜잭션에 참가하지 않아도 된다.

참고로 같은 클래스 내부에 있는 메서드를 호출할때는 AOP가 동작하지 않는다. (proxy)

save의 경우에는 한번 엔티티를 저장할때마다 트랜잭션에 참가해야 된다.

따라서 save와 saveAll의 성능 차이가 발생한 것이다.

-끝!-
