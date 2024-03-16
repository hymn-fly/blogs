---
title: io.spring.dependency-management plugin에 대하여
date: 2024-03-16 11:50:00 +0900
categories: [Java, Gradle]
tags: [Gradle] # TAG names should always be lowercase
toc: true
---

## 서론
&ensp; 이번에 웹서버의 DB를 mssql에서 postgresql로 이관을 해야 할 필요가 있어서, flyway 또한도 같이 이관하기 위해 의존성을 바꾸는 작업을 진행하였다. `org.flywaydb:flyway-sqlserver` 를 단순히 postgresql 로만 바꾸면 되겠지 라는 생각에 금방 하겠지 싶었는데, [Flyway 문서](https://documentation.red-gate.com/flyway/flyway-cli-and-api/supported-databases/postgresql-database)에 나와있는 `org.flywaydb:flyway-databae-postgresql` 이라는 의존성을 사용하기 시작하며 문제가 발생했다. 그 와중에 `io.spring.dependency-management` gradle 플러그인에 대해 제대로 이해를 할 필요성이 있었고 그에 대해 정리하려 한다.  

## Flyway 를 mssql에서 postgres로 의존성 변경하며 생긴 문제
&ensp; 처음에, `org.flywaydb:flyway-sqlserver`를 `org.flywaydb:flyway-databae-postgresql`로 변경하고, postgres 드라이버(`org.postgresql:postgresql:42.6.0`)를 추가해주면 바로 되겠지 하고 application 실행을 했더니 아래 문제가 발생했다.
```
Caused by: org.springframework.beans.BeanInstantiationException: Failed to
instantiate [org.flywaydb.core.Flyway]: Factory method 'flyway' threw exception; 
nested exception is java.lang.AbstractMethodError: Receiver class org.flywaydb.
database.postgresql.PostgreSQLConfigurationExtension does not define or inherit
an implementation of the resolved method 'abstract void extractParametersFromConfiguration(java.util.Map)' 
of interface org.flywaydb.core.extensibility.ConfigurationExtension.
```

에러 메시지를 읽어보면 Flyway Core에 있는 `ConfigurationExtension` 인터페이스를 `PostgreSQLConfigurationExtension`이 잘못 구현하고 있다는 의미로 보인다. 그래서 flyway core의 버전을 flyway-database-postgresql의 버전(10.7.1)과 맞추면 문제가 해결될까 싶어, flyway core 의 버전을 업그레이드 하려고 했다.

그러나 여기서부터 문제가 발생했다. flyway-core 의 의존성을 build.gradle에 내가 원하는 버전으로 dependency를 설정해주었으나 `8.5.13`버전이 Intellij의 `External Libraries` 에서 제거되지 않았고 위 이슈가 계속 발생했다. Gradle에서 지원하는 기능인 [dependency constraint](https://docs.gradle.org/current/userguide/dependency_constraints.html#sec:adding-constraints-transitive-deps) 를 사용해도 `gradle dependencies` 태스크를 활용하여 의존성의 버전을 살펴보면 계속 버전이 아래와 같이 자동으로 변경되었다.
```
+--- org.flywaydb:flyway-database-postgresql:10.7.1
|    \--- org.flywaydb:flyway-core:10.7.1 -> 8.5.13

```

8.5.13 버전을 프로젝트에서 제거하려고 온갖 다양한 방법을 시도해보다가, `io.spring.dependency-management` 플러그인에 대해 알게 되었고 얘가 알아서 의존성의 버전을 변경해 버린다는 사실을 알게되었다.

## io.spring.dependency-management 플러그인이 하는 일
해당 플러그인의 [Github](https://github.com/spring-gradle-plugins/dependency-management-plugin)과 [Spring Docs](https://docs.spring.io/dependency-management-plugin/docs/current-SNAPSHOT/reference/html/#introduction)를 살펴보면 플러그인에 대해 이렇게 설명한다.

```
A Gradle plugin that provides Maven-like dependency management and 
exclusions Based on the configured dependency management. 
The plugin will control the versions of your project's direct and 
transitive dependencies and will honour any exclusions declared in the 
poms of your project's dependencies and any imported boms.
``` 

빌드 의존성 관리 도구인 Gradle에서 Maven 처럼 의존성을 관리하고 제거하는 기능을 제공하고, 프로젝트에서 직접적으로 혹은 간접적으로(transitive) 사용하는 의존성의 버전을 관리해준다고 한다. 그리고, [BOM](https://www.baeldung.com/spring-maven-bom) 을 import 하여 의존성 관리도 해준다 한다. 이러한 기능을 가진 plugin을 root 프로젝트의 build.gradle 파일에 선언해두었어서, flyway-core의 버전이 계속해서 `8.5.13`으로 자동으로 바뀌는 것이었다. 이 플러그인이 관리하는 모든 의존성과 자동으로 사용되는 버전에 대한 정보는 `gradle dependencyManagement` 태스크를 통해 확인할 수 있다.

```
...

org.flywaydb:flyway-core 8.5.13
org.flywaydb:flyway-firebird 8.5.13
org.flywaydb:flyway-mysql 8.5.13
org.flywaydb:flyway-sqlserver 8.5.13

...
```
### 해당 플러그인에서 특정 의존성의 버전을 지정하는 방법
[Spring Docs](https://docs.spring.io/dependency-management-plugin/docs/current-SNAPSHOT/reference/html/#introduction) 문서의 `4.1. Dependency Management DSL` 섹션에 보면 아래와 같은 방식으로 특정 의존성의 버전을 변경할 수 있다고 설명한다.

```
dependencyManagement {
    dependencies {
        dependency 'org.flywaydb:flyway-core:<specific-version>'
    }
}
```
그리고, `4.1.1. Dependency Sets` 에는 특정 Group 내의 모든 모듈에 대해 버전을 동시에 지정할 수 있는 기능도 설명해주고 있다. 

```
dependencyManagement {
     dependencies {
          dependencySet(group:'org.flywaydb', version: '<specific-version>') {
               entry 'flyway-core'
               entry 'flyway-sqlserver'
               ..
          }
     }
}
```

## 결론
&ensp;해당 플러그인에 대해서 알고 flyway-core의 버전을 바꾸었지만 그 문제가 아니였고, 결국 `org.flywaydb:flyway-databae-postgresql`를 제거하고 `org.flywaydb:flyway-core` 만 사용함으로써 postgresql DB에 대해 flyway 셋업은 완료하였다. <br />
이번 기회를 통해 Flyway 문서에 대한 신뢰도가 떨어졌고... 그러나 덕분에 이제까지 Spring을 사용하면서 잘은 몰랐던 Gradle이 어떻게 의존성을 관리하고 가져오는지에 대해 좀 더 이해할 수 있었다. 또한 gradle dependencies, dependencyManagement 태스크 등을 통해 의존성을 상세하게 확인할 수 있다는 걸 알게 되었다.

