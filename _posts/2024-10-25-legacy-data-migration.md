---
title: 상용 DB 마이그레이션 경험기(@Transactional 내부에서 예외 잡기?)
date: 2024-10-25 15:00:00 +0900
categories: [Spring, Jpa]
tags: [Spring] # TAG names should always be lowercase
toc: true
---

## 문제상황과 해결방법
&emsp;운영환경에서 사용되고 있던 레거시 테이블의 데이터를 마이그레이션 하면서 발생한 경험을 정리하고자 한다. <br/>
&emsp;운영환경에서 타겟 테이블의 character set 이 `euckr` 이고 application은 `utf-8`을 사용하고 있는 상황이었다. 타겟 테이블에 데이터를 업데이트 해야 하는 상황에서 `euc-kr` 과 `utf-8` 인코딩으로 문자가 인코딩/디코딩 되는 과정 중에 `Incorrect string value : \xEF\xBF\xBDc\xEB\xAc ...` 와 같은 예외가 발생하면서 데이터 마이그레이션이 완료되지 않았고,  로그를 찍어서 어떤 데이터가 문제인지 보려고 했지만 로그도 찍히지 않았다.(원인은 `@Transactional` 때문이었음) <br/>
&emsp;예외가 발생한 컬럼은 내가 업데이트하고자 하는 컬럼이 아닌 기존에 존재하는 값에 대한 컬럼이었으므로 `@DynamicUpdate` 를 통해 내가 업데이트하고자 하는 컬럼만 업데이트 함으로써 마이그레이션은 정상완료 하였다. 그러나 로그가 찍히지 않았던 이유에 대해 명확히 알지 못해 이번 기회에 그 내용에 대해 정리한다.

## 마이그레이션 코드 예외 발생시 로그가 찍히지 않았던 원인 -> @Transactional

```kotlin
@Transactional
fun configMigration(migrationType: MigrationType): Any? {
    when (migrationType) {
        MigrationType.COMPANY -> {
            ...

            companyIdToPassbookId
                .forEach { (companyId, passbookId, businessType) ->
                    val company = legacyCompanyRepository.findById(companyId).get()

                    // company field들에 대한 수정
                    ...

                    try {
                        legacyCompanyRepository.save(company)
                    } catch (e: Exception) {
                        logger.error("${company.companyId} ${company.businessRegNum} ${e.message} $e")
                        throw IllegalArgumentException("${company.companyId} ${company.businessRegNum} ${e.message}")
                    }
                }
        }
```

위의 코드와 같이 company 엔티티를 업데이트하고자 했고, save에서 예외가 발생한다 생각해서 try catch 구문을 넣었다. 그러나 예외가 로깅이 되지 않고 헤매다가, `@DynamicUpdate`를 이용하는 우회전략을 취했던 것이다. <br>
살펴보니 @Transactional 이 있으면 `configMigration()` 메서드 내부에서 `save()`를 호출할때 update 쿼리가 바로 DB로 전달되지 않는다(쓰기지연). 대신 영속성 컨텍스트에 해당 쿼리를 보관해 두고, 메서드가 종료되고 난후 `JpaTransactionManager` 에서 `doCommit()` 메서드가 호출될 때 영속성 컨텍스트에 있던 쿼리가 DB로 전달된다. 이 때 예외가 발생하기에 메서드 내부에서는 catch 를 통해 예외를 잡을 수 없었던 것이었다.

### @Transactional 의 작동 원리로 알아보는 예외가 잡히지 않은 이유

@Transactional 어노테이션이 메서드에 붙어 있으면 AoP 형태로 작동하며, `TransactionAspectSupport`의 `invokeWithinTransaction()` 에서 해당 부분을 살펴볼 수 있다.

```kotlin
// 1. Transaction 생성
TransactionInfo txInfo = createTransactionIfNecessary(ptm, txAttr, joinpointIdentification); 

Object retVal;
try {
    // 2. 실제 메서드를 호출하는 부분
    retVal = invocation.proceedWithInvocation(); 
}
catch (Throwable ex) {
    // 메서드 내부에서 예외 발생 시 처리하는 로직(이 경우는 아래에서 살펴볼 예정)
    // 위 상황에서는 query가 DB로 전달되지 않았기에 메서드 내부에서 예외가 발생할 수 없음
    completeTransactionAfterThrowing(txInfo, ex); 
    throw ex;
}
finally {
    // 3-1. Transaction rollback 
    cleanupTransactionInfo(txInfo); 
}

...
// 3-2. 메서드 정상 호출 시, Transaction Commit
commitTransactionAfterReturning(txInfo); 
```

`commitTransactionAfterReturning` 내부에서 호출 되는 `JpaTransactionManager` 의 `doCommit()` 부분을 살펴보면 아래와 같다.

```kotlin
try {
    EntityTransaction tx = txObject.getEntityManagerHolder().getEntityManager().getTransaction();
    tx.commit();
} catch (RollbackException var6) {
    RollbackException ex = var6;
    Throwable var5 = ex.getCause();
    if (var5 instanceof RuntimeException runtimeException) {
        DataAccessException dae = this.getJpaDialect().translateExceptionIfPossible(runtimeException);
        if (dae != null) {
            throw dae;
        }
    }

    throw new TransactionSystemException("Could not commit JPA transaction", ex);
} catch (RuntimeException var7) {
    RuntimeException ex = var7;
    throw DataAccessUtils.translateIfNecessary(ex, this.getJpaDialect());
}
```

위 코드를 살펴보면 `tx.commit()`이 호출될 때 update 쿼리가 db로 한번에 날아가게 되고, 그때 인코딩이 맞지 않는 오류로 인해 Data를 Update 칠 수 없어 예외가 발생하고 그 예외가 위 코드의 catch block에 잡혀서 configMigration() 메서드 내부에서는 예외를 잡을 수 없었던 것이다. (이미 해당 메서드는 종료되고 나왔으니까)

### @Transactional 내부에서도 예외를 잡을수 있는 경우가 있다?
기본적으로는 @Transactional 내부에서 save() 메서드 호출시 발생하는 예외는 내부에서 잡을 수 없지만(트랜잭션이 커밋되면서 DB 로 쿼리가 반영되니까), 예외 케이스가 존재한다. <br/>
Entity의 `@Id` 컬럼의 `GenerationType`이 `IDENTITY` 일때 save()를 호출하면 INSERT 쿼리가 DB로 바로 전달된다[^persist-merge-work]. 그 이유는
GenerationType이 IDENTITY 일 때는 save() 가 호출될 때 PK 값을 가져와야 하기 때문에(`AUTO_INCREMENT` 속성값) 쿼리가 DB로 바로 전달되게 된다(당연히 데이터는 Transaction이 commit 될 때 반영). <br/>
`spring.jpa.properties.hibernate.show_sql : true` 로 설정해두고 디버깅을 해보면 update 쿼리일 때는 `tx.commit()`이 호출될 때 sql 이 발생하지만, insert 쿼리일 때는 save()가 호출될 때 바로 sql이 프린트되는 것을 확인할 수 있다. <br />
그래서, save() 메서드 호출시 바로 예외가 발생하고, 메서드 내부에서 catch 로 해당 예외를 잡을 수 있게 되는 것이다. 만약 catch로 해당 예외를 잡지 않는다면, 위에서 보았던 `completeTransactionAfterThrowing(txInfo, ex);` 이부분의 catch block 으로 예외가 전파되게 된다.

> AUTO_INCREMENT에 대한 추가의문 <br/>
> 만약 TxA(트랜잭션A) 에서 save()를 호출하고 pk 값을 가져가고, TxA가 끝나기 전에 TxB에서 save()를 호출하여 pk 값을 가져간다면 두 값은 중복될까? <br/>
> No. AUTO_INCREMENT는 table 수준에서 결정되는 값이고, Transaction의 scope 밖에서 동작하기에 Transaction 과는 상관없다고 함 [^mysqldocs]
{: .prompt-tip }


[^persist-merge-work]:[How do persist and merge work in JPA](https://vladmihalcea.com/jpa-persist-and-merge/)
[^mysqldocs]:[AUTO_INCREMENT Handling in InnoDB](https://dev.mysql.com/doc/refman/8.4/en/innodb-auto-increment-handling.html)
