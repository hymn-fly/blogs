---
title: Spring JpaRepository와 @Repository의 Proxy 생성방식 비교
date: 2023-06-10 15:30:00 +0900
categories: [Spring, Jpa]
tags: [Spring] # TAG names should always be lowercase
toc: true
---

## 서론
 Redis와 Connection을 담당하는 DAO 클래스를 다루면서 생긴 의문점에서 시작되었다. 해당 Repository 클래스도 Proxy로 동작하는 것을 발견(invoke 메서드를 통하여 실제 메소드가 호출됨)하였고 이게 왜 Proxy로 동작하는지 궁금증이 생겨서 파고 들어보았다(@Repository를 붙였었음). JpaRepository를 확장(`extends`)하는 인터페이스는 어떻게 Proxy로 래핑되는지, 그리고 @Repository를 붙인 클래스는 어떻게 Proxy로 래핑되는지 알아보자.

## JpaRepository 를 확장하는 Interface
### 마커 Repository interface

`JpaRepository` 인터페이스를 살펴보면 위로 확장하는 인터페이스 들이 꽤 많다. 제일 상위로 올라가 보면 `Repository`라는 마커 인터페이스를 만나게 된다.
![Repository Interface](/assets/images/repository_interface.png)_Repository Interface_

JavaDocs를 읽어보면 Central repository marker interface라고 되어 있고 주 목적은 타입 정보를 유지하고 해당 interface를 확장하는 interface를 class path scanning 동안 찾아내어 Spring Bean으로 만들기 위함이라고 적혀 있다. 

### JpaRepository

우리가 Jpa를 활용하기 위해 extend 하는 대표적인 인터페이스이다. 이 interface를 살펴보면 `@NoRepositoryBean`이라는 어노테이션이 붙어 있고, 이 어노테이션이 하는 역할은 이름에서도 알 수 있듯이, 리포지토리 빈이 아니므로 Spring Bean으로 만들지 마 라고 명시해주는 목적이다.

제일 상위 인터페이스인 `Repository` 아래로 많은 interface 들이 존재하는데(`CrudRepository`, `JpaRepository` ..) 이러한 인터페이스들을 다 살펴보면 `@NoRepositoryBean` 어노테이션이 다 붙어 있는 것을 확인할 수 있다. 그래서 얘네들은 빈으로 만들어지지 않는다. ( 그렇다면 우리가 `JpaRepository`를 extend 하고 `@NoRepositoryBean`을 붙이면 당연히 빈으로 등록되지 않겠군 )

### Proxy로 래핑하는 과정

위의 과정을 통해서 왜 `JpaRepository`를 extends하는 interface가 Bean으로 만들어지는 지는 알 수 있다. 그렇담 어떻게 Proxy로 래핑이 되는 것일까?

차례대로 살펴보자. <br>
1. `AbstractAutowireCapableBeanFactory` 에서 initializeBean() 을 호출 <br>
![initialize-bean](/assets/images/initialize_bean.png)
- 위에서 Bean으로 Create를 할 테니 Bean으로 초기화 해야함
- initializeBean()의 JavaDoc을 보면 factory callback을 통해 bean을 초기화 한다고 적혀있고, 위의 initializeBean 메서드의 호출 argument를 보면 bean의 타입이 `JpaRepositoryFactoryBean`이고, beanName은 `JpaRepository`를 구현한 인터페이스의 이름임. 즉, FactoryBean을 통해서 우리의 인터페이스 타입을 갖는 빈을 찍어내겠다는 의미.
- `JpaRepositoryFactoryBean`는 `InitializingBean` interface를 구현하고 있어서, afterPropertiesSet 메서드를 정의할 수 있고 이 안에서 빈이 생성된 후 프록시로 교체하는 부분이 수행됨(bean 생성 후 호출되는 메서드 )

2. `JpaRepositoryFactoryBean`의 afterPropertiesSet() 메서드에서 super(`RepositoryFactoryBeanSupport` 의 afterPropertiesSet())를 호출하고 거기서 `JpaRepositoryFactory`를 통해 getRepository() 호출
![afterproperties-set](/assets/images/afterproperties_set.png)

3. `RepositoryFactoryBeanSupport`의 getRepository() 안에서 target 클래스 할당, proxy 생성을 진행함
![create_repository](/assets/images/create_repository.png)_getRepository() 메서드 일부 발췌_
- repository 인터페이스를 타입으로 갖고 target을 SimpleJpaRepository로 하는 Proxy를 JDK Dynamic Proxy를 통해서 생성함
- target : `SimpleJpaRepository`
- ProxyFactory에 repositoryInterface를 할당 후 Proxy 생성

위와 같은 과정을 통해 `JpaRepository` 를 확장하는 인터페이스의 Proxy를 Bean으로 등록하는 과정이 일어나게 되는 것임

## @Repository가 붙은 클래스
그렇다면 이 클래스는 어떻게 Proxy로 만들어지는가? 잘 몰랐어서, 처음에는 @Repository를 붙이면 Bean으로만 등록되는 줄 알았다. 근데 살펴보니 CGLib을 통해서 Proxy로 만들어진다. CGLib은 JDK Dynamic Proxy와 달리 Bytecode를 직접 생성하기에 따로 프록시를 생성하는 코드를 찾지는 못했다. (아마 없는게 맞을 듯함)

![cglib-repository](/assets/images/repository_annotation.png)_@Repository를 붙인 RoomInstanceRepository의 CGLIB instance_

- `@Component`를 붙이면? 그냥 Bean으로만 등록
- `@Service`를 붙이면? 그냥 Bean으로만 등록 ! ! (여기도 Proxy로 등록될 줄 알았는데 그렇지 않았다. 그냥 Bean으로 등록된다)
  - 그렇지만 `@Transactional`을 붙이면? CGLib으로 Proxy만들어서 Bean으로 등록한다. 즉, AoP가 필요할 때만 PointCut처럼 작동하여 Proxy로 해당 Bean을 대체하는 식으로 작동하는 것 같다

## 결론
- `JpaRepository`를 extend하는 interface는 Jdk Dynamic Proxy가 type은 우리가 구현한 interface 이고 target은 `SimpleJpaRepository` 인 Proxy 클래스를 생성해줌
- `@Repository`가 붙은 클래스는 CGLib에 의해 Proxy가 생성이 된다.
  - `@Component`와 `@Service`는 그냥 Bean으로만 등록함
  - `@Transactional`과 같이 aop가 필요한 어노테이션이 붙으면 CGLib이 Proxy로 생성해줌
