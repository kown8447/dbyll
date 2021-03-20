---
layout: post
title: 엔티티 매핑
excerpt: 이번 장에서는 객체와 테이블 매핑, 기본 키 매핑, 필드와 컬럼 매핑에 대해 알아보자.
categories: [JPA]
tags: [기본키 매핑, IDENTITY 전략, SEQUENCE 전략, TABLE 전략, 필드 매핑, 컬럼 매핑]
---

### @Entity
<hr>

| 속성 | <center>기능</center> | 기본값|
|---|---|---|
|`name` | JPA에서 사용할 엔티티 이름을 지정한다.<br> 보통 기본값인 클래스 이름을 사용한다. <br> 만약 다른 패키지에 이름이 같은 엔티티 클래스가 있다면 이름을 지정해서 충돌하지 않도록 해야 한다. | 설정하지 않으면 클래스 이름을 그대로 사용한다.|

<br>
* **주의사항**
    * 기본 생성자는 필수다(파라미터가 없는 public 또는 protected 생성자).
    * final 클래스, enum, interface, inner 클래스에는 사용할 수 없다.
    * 저장할 필드에 final을 사용하면 안된다.
    
<br><br>

### @Table
<hr>

| <center>속성</center> | <center>기능</center> | <center>기본값</center>|
|:---:|:---:|---|
|`name` | 매핑할 테이블 이름 | 엔티티 이름을 사용한다. |
|`catalog`|catalog 기능이 있는 데이터베이스에서 catalog를 매핑한다.| |
|`schema`|schema 기능이 있는 테이터베이스에서 schema를 매핑한다.||
|`uniqueConstraints`|DDL 생성 시에 유니크 제약조건을 만든다.<br> 2개 이상의 복합 유니크 제약조건도 만들 수 있다.<br>참고로 이 기능은 스키마 자동 생성 기능을 사용해서 DDL을 만들때만 사용된다.||

<br><br>

### 데이터베이스 스키마 자동 생성
<hr>

~~~xml
<property name="hibernate.hdm2ddl.auto" value="create"/>
~~~
<br>
위 속성을 추가하면 **애플리케이션 실행 시점에 데이터베이스 테이블을 자동으로 생성**한다.

다음은 hibernate.hdm2ddl.auto 속성이다.
<br>

| 옵션 | 설명 |
|---|---|
|`create`| 기존 테이블을 삭제하고 새로 생성한다. DROP + CREATE|
|`create-drop`| create 속성에 추가로 애플리케이션을 종료할 때 생성한 DDL을 제거한다.|
|`update`| 데이터베이스 테이블과 엔티티 매핑정보를 비교해서 변경 사항만 수정한다.|
|`validate`| 데이터베이스 테이블과 엔티티 매핑정보를 비교해서 차이가 있으면 경고를 남기고 애플리케이션을 실행하지 않는다. 이 설정은 DDL을 수정하지 않는다.|
|`none`| 자동 생성 기능을 사용하지 않으려면 hiberante.hdm2ddl.auto 속성 자체를 삭제하거나 유효하지 않는 옵션 값을 주면 된다.(참고로 none은 유효하지 않은 옵션 값이다.)|

<br><br>

### 기본 키 매핑
<hr>
기본 키를 애플리케이션에서 직접 할당하는 대신에 데이터베이스가 생성해주는 값을 사용하려면 어떻게 매핑해야 할까?

데이터베이스마다 기본 키를 생성하는 방식이 서로 다르므로 이 문제를 해결하기는 쉽지 않다. JPA는 이런 문제들을 어떻게 해결하는지 알아보자.

* JPA의 기본 키 생성 전략
    * 직접 할당: 기본 키를 애플리케이션에서 직접 할당한다.
    * 자동 생성: 대리 키 사용 방식
        * IDENTITY: 기본 키 생성을 데이터베이스에 위임한다.
        * SEQUENCE: 데이터베이스 시퀀스를 사용해서 기본 키를 할당한다.
        * TABLE: 키 생성 테이블을 사용한다.
    
기본 키를 직접 할당하려면 @Id만 사용하면 되고, 자동 생성 전략을 사용하려면 @Id에 @GeneratedValue를 추가하고 원하는 키 생성 전략을 선택하면 된다.
<br><br>

> 기본 키 직접 할당 전략

기본 키 직접 할당 전략은 em.persist()로 엔티티를 저장하기 전에 애플리케이션에서 기본 키를 직접 할당하는 방법이다.

~~~java
@Id
@Column(name = "id")
private String id;
~~~
<br>
@Id 적용 기능 자바 타입은 다음과 같다.
    
* 자바 기본형
* 자바 래퍼<sub>Wrapper</sub>형
* `String`, `java.util.Date`, `java.sql.Date`, `java.math.BigDecimal`, `java.math.BigInteger`
<br><br>
  
> IDENTITY 전략

~~~java
@GeneratedValue(strategy = GenerationType.IDENTITY)
~~~

IDENTITY는 기본 키 생성을 데이터베이스에 위임하는 전략이다.
**IDENTITY 전략은 데이터베이스에 값을 저장하고 나서야 기본 키 값을 구할 수 있을 때 사용한다.**
이 전략을 사용하면 JPA는 기본 키 값을 얻어오기 위해 데이터베이스를 추가로 조회한다.

엔티티가 영속 상태가 되려면 식별자가 반드시 필요하다.
그런데 IDENTITY 식별자 생성 전략은 엔티티를 데이터베이스에 저장해야 식별자를 구할 수 있으므로 em.persist()를 호출하는 즉시 INSERT SQL이 데이터베이스에 전달된다.
따라서 **이 전략은 트랜잭션을 지원하는 쓰기 지연이 동작하지 않는다.**
<br><br>

> SEQUENCE 전략

~~~java
@Entity
@SequenceGenerator(
        name = "BOARD_SEQ_GENERATOR",
        sequenceName = "BOARD_SEQ",
        initalValue = 1, allocationSize = 1)
public class Board {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY,
                        gernator = "BOARD_SEQ_GENERATOR")
    private Long id;
}
~~~
<br>
데이터베이스 시퀀스는 유일한 값을 순서대로 생성하는 특별한 데이터베이스 오브젝트다. 

시퀀스 사용 코드는 `IDENTITY` 전략과 같지만 내부 동작 방식은 다르다.
`SEQUENCE` 전략은 `em.persist()`를 호출할 때 먼저 데이터베이스 시퀀스를 사용해서 식별자를 조회한다.
그리고 조회한 식별자를 엔티티에 할당한 후에 엔티티를 영속성 컨텍스트에 저장한다.
이후 트랜잭션을 커밋해서 플러시가 일어나면 엔티티를 데이터베이스에 저장한다.
<br><br>

> @SequenceGenerator

|속성|기능|기본값|
|---|---|---|
|`name`| 식별자 생성기 이름|필수|
|`sequenceName`|데이터베이스에 등록되어 있는 시퀀스 이름|hiberate_sequence|
|`initialValue`|DDL 생성 시에만 사용됨. 시퀀스 DDL을 생성할 때 처음 시작하는 수를 지정한다.|1|
|`allocationSize`|시퀀스 한 번 호출에 증가하는 수(성능 최적화에 사용됨)|50|
|`catalog, schema`|데이터베이스 catalog, schema 이름||

<br>

JPA는 시퀀스에 접근하는 횟수를 줄이기 위해 `@SequenceGenerator.allocationSize`를 사용한다.
여기에 설정한 값만큼 한 번에 시퀀스 값을 증가시키고 나서 그만큼 메모리에 시퀀스 값을 할당한다.
이 최적화 방법은 시퀀스 값을 선점하므로 여러 JVM이 동시에 동작해도 기본 키 값이 충돌하지 않는 장점이 있다.
반면에 데이터베이스에 직접 접근해서 데이터를 등록할 때 시퀀스 값이 한번에 많이 증가한다는 점을 염두해두어야 한다.
참고로 `hibernate.id.new_generator_mappings` 속성을 `true`로 설정해야 지금까지 설명한 최적화 방법이 적용된다.
<br><br>

> TABLE 전략

`TABLE` 전략은 키 생성 전용 테이블을 하나 만들고 여기에 이름과 값으로 사용할 컬럼을 만들어 데이터베이스 시퀀스를 흉내내는 전략이다.

~~~sql
create table MY_SEQUENCES {
    sequence_name varchar(255) not null,
    next_val bigint,
    primary key (sequence_name)
}
~~~
<br>

`sequence_name` 컬럼을 시퀀스 이름으로 사용하고 `next_val` 컬럼을 시퀀스 값으로 사용한다.

~~~java
@Entity
@SequenceGenerator(
        name = "BOARD_SEQ_GENERATOR",
        table = "MY_SEQUENCES",
        pkColumnValue = "BOARD_SEQ", allocationSize = 1)
public class Board {
    
    @Id
    @GeneratedValue(strategy = GenerationType.TABLE,
                        gernator = "BOARD_SEQ_GENERATOR")
    private Long id;
}
~~~
<br>

`TABLE` 전략은 시퀀스 대신에 테이블을 사용한다는 것만 제외하면 SEQUENCE 전략과 내부 동작방식이 같다.
`TABLE` 전략은 값을 조회하면서 SELECT 쿼리를 사용하고 다음 값으로 증가시키기 위해 UPDATE 쿼리를 사용한다.
이 전략은 `SEQUENCE` 전략과 비교해서 데이터베이스와 한 번 더 통신하는 단점이 있다.
`TABLE` 전략을 최적화하려면 `@TableGenerator.allocationSize` 를 사용하면 된다.
이 값을 사용해서 최적화하는 방법은 `SEQUENCE` 전략과 같다.
<br><br>

> @TableGenerator

<br>

|속성|기능|기본값|
|---|---|---|
|`name`| 식별자 생성기 이름|필수|
|`table`|키생성 테이블명|hibernate_sequences|
|`pkColumnName`|시퀀스 컬럼명|sequence_name|
|`valueColumnName`|시퀀스 값 컬럼명|next_val|
|`pkColumnValue`|키로 사용할 값 이름|엔티티 이름|
|`initialValue`|초기 값. 마지막으로 생성된 값이 기준이다.|0|
|`allocationSize`|시퀀스 한 번 호출에 증가하는 수(성능 최적화에 사용됨)|50|
|`catalog, schema`|데이터베이스 catalog, schema 이름||
|`uniqueConstraints(DDL)`|유니크 제약 조건을 지정할 수 있다.||

<br><br>

> AUTO 전략

<br>
`GenerateionType.AUTO`는 선택한 데이터베이스 방언에 따라 IDENTITY, SEQUENCE, TABLE 전략 중 하나를 자동으로 선택한다.
`AUTO` 전략의 장점은 **데이터베이스를 변경해도 코드를 수정할 필요가 없다는 것**이다.
<br><br>

> 기타 참고

비지니스 환경은 언젠가 변하기 때문에, 자연 키보다는 대리키를 권장한다.
<br><br>

### 필드와 컬럼 매핑: 레퍼런스
<hr>

분류|매핑 어노테이션|설명|속성|기능|기본값
---|---|---|---|---|---
필드와 컬럼 매핑|`@Column`|컬럼을 매핑한다.|`name`|필드와 매핑할 테이블의 컬럼 이름|객체의 필드 이름
|||`nullable(DDL)`|null 값의 허용 여부를 설정한다. false로 설정하면 DDL 생성 시에 not null 제약조건이 붙는다.|true
|||`unique(DDL)`|@Table의 uniqueConstraints와 같지만 한 컬럼에 간한히 유니크 제약조건을 걸 때 사용한다.|
|||`columnDefinition(DDL)`|데이터베이스 컬럼 정보를 직접 줄 수 있다.|필드의 자바 타입과 방언 정보를 사용해서 적절한 컬럼 타입을 생성한다.
|||`length(DDL)`|문자 길이 제약조건, String 타입에만 사용한다.|255
|||`precision, scale(DDL)`|BigDecimal, BigInteger 타입에서 사용한다. precision은 소수점 포함 전체 자릿수, scale은 소수의 자릿수다.(double, float 타입에는 적용되지 않는다.)|precision=19, scale=2
 |`@Enumerated`|자바의 enum 타입을 매핑한다.|`value`|`EnumType.ORDINAL`: enum 순서를 데이터베이스에 저장<br>`EnumType.STRING`: enum 이름을 데이터베이스에 저장|EnumType.ORDINAL 
 |`@Temporal`|날짜 타입을 매핑한다.|`value`|`TemporalType.DATE`: 날짜, 데이터베이스 date타입 <br> `TemporalType.TIME`: 시간, 데이터베이스 time타입<br>`TemporalType.TIMESTAMP`: 날짜와 시간, 데이터베이스 timestamp타입|TemporalType은 필수로 지정해야 한다. 
 |`@Lob`|BLOB, CLOB 타입을 매핑한다.
 |`@Transient`|특정 필드를 데이터베이스에 매핑하지 않는다.
기타|`@Access`|JPA가 엔티티에 접근하는 방식을 지정한다.||`AceesType.FIELD`: 필드에 직접 접근한다.<br>`AccessType.PROPERTY`: 접근자<sub>getter</sub>를 사용한다.

<br>
<br>

### 참고
<hr>
* 자바 ORM 표준 JPA 프로그래밍(김영한) - 4장_엔티티 매핑