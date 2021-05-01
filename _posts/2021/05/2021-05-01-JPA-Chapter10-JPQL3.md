---
layout: post
title: 객체지향 쿼리 언어_객체지향 쿼리 심화
categories: [JPA]
tags: [벌크연산, 플러시모드]
---

### 객체지향 쿼리 심화
<hr>

> 벌크 연산

엔티티를 수정하려면 영속성 컨텍스트의 변경 감지 기능이나 병합을 사용한다.
하지만 이 방법으로 수백개 이상의 엔티티를 하나씩 처리하기에는 시간이 너무 오래걸린다.
이럴 때 벌크 연산을 사용하면 효율적이다.

```java
string qlString = 
    "update Product p " +
    "set p.price = p.price * 1.1 " +
    "where p.stockAmount < :stockAmount";

int resultCount = em.createQuery(qlString)
                    .setParameter("stockAmount", 10)
                    .executeUpdate();
```
<br>

벌크 연산은 `executeUpdate()` 메소드를 사용한다. 삭제도 마찬가지이다.
또한 JPA 표준은 아니지만 하이버네이트는 INSERT 벌크 연산도 지원한다.
<br><br>

#### 벌크 연산의 주의점
**벌크 연산은 영속성 컨텍스트를 무시하고 데이터베이스에 직접 쿼리한다.**
다음은 벌크 연산을 사용할 때 발생할 수 있는 문제의 예시이다.

```java
Product productA = 
    em.createQuery("select p from Product p where p.name = :name", Product.class)
        .setParameter("name", "productA")
        .getSingleresult();

//출력 결과: 1000
System.out.println("productA 수정 전 = " + productA.getProduct());

//벌크 연산 수행으로 모든 상품 가격 10% 상승
em.createQuery("update Product p set p.price = p.price * 1.1")
    .executeUpdate();

//출력 결과: 1000
System.out.println("productA 수정 전 = " + productA.getProduct());
```
<br>

벌크 연산은 영속성 컨텍스트를 통하지 않고 데이터베이스에 직접 쿼리하기 때문에, 
영속성 컨텍스트에 있는 엔티티와 데이터베이스로 조회한 결과가 다를 수 있다.
문제의 해결방법은 아래와 같다.

* **em.refresh() 사용**
    * em.refresh()를 사용해서 데이터베이스에서 상품A를 다시 조회한다.
    * _em.refresh(productA)_
  
* **벌크 연산 먼저 실행**
  * 가장 실용적인 해결책은 벌크 연산을 먼저 실행하는 것이다. 이 방법은 JPA와 JDBC를 함께 사용할 때도 유용하다.
  
* **벌크 연산 수행 후 영속성 컨텍스트 초기화**
  * 영속성 컨텍스트를 초기화하면 이후 엔티티를 조회할 때 벌크 연산이 적용된 데이터베이스에서 엔티티를 조회한다.
  
  
<br><br>
> 영속성 컨텍스트와 JPQL

JPQL로 엔티티를 조회하면 영속성 컨텍스트에서 관리되지만 엔티티가 아니면 영속성 컨텍스트에서 관리되지 않는다.
(임베디드 타입, 값 타입은 영속성 컨텍스트에서 관리되지 않음)

또한 **JPQL은 조회한 엔티티가 이미 영속성 컨텍스트에 있으면, JPQL로 데이터베이스에서 조회한 결과를 버리고 대신에 영속성 컨텍스트에 있던 엔티티를 반환한다.**

<center>
<img src="{{ site.BASE_PATH }}/assets/images/2021/05/01/img1.png">
</center>

<center>
<img src="{{ site.BASE_PATH }}/assets/images/2021/05/01/img2.png">
</center>

위 그림을 설명하면 아래와 같다.

1. JPQL을 사용해서 조회를 요청한다.
2. JPQL은 SQL로 변환되어 데이터베이스를 조회한다.
3. 조회한 결과와 영속성 컨텍스트를 비교한다.
4. 식별자 값을 기준으로 member1은 이미 영속성 컨텍스트에 있으므로 버리고 기존에 있던 member1이 반환 대상이 된다.
5. 식별자 값을 기준으로 member2는 영속성 컨텍스트에 없으므로 영속성 컨텍스트에 추가한다.
6. 쿼리 결과인 member1, member2를 반환한다. 여기서 member1은 쿼리 결과가 아닌 영속성 컨텍스트에 있던 엔티티다.
<br><br>

#### find() vs JPQL

**JPQL은 항상 데이터베이스에 SQL을 실행해서 결과를 조회한다.**
이에 반해 `em.find()` 메소드는 영속성 컨텍스트에서 엔티티를 먼저 찾고 없으면 데이터베이스를 조회한다.
따라서 해당 엔티티가 영속성 컨텍스트에 있으면 메모리에서 바로 찾으므로 성능상 이점이 있다.(그래서 1차 캐시라 부른다.)
<br><br>

> JPQL과 플러시 모드

`플러시`는 영속성 컨텍스트의 변경 내역을 데이터베이스에 동기화 하는 것이다.
플러시 모드는 2가지가 있다.

* **FlushModeType.AUTO**
  * 기본값이며 트랜잭션 커밋 직전이나 쿼리 실행 직전에 자동으로 플러시를 호출한다.
  
* **FlushModeType.COMMIT**
  * 커밋 시에만 플러시를 호출하고 쿼리 실행 시에는 플러시를 호출하지 않는다.
  * 이 옵션은 성능 최적화를 위해 꼭 필요할 때만 사용해야 한다.
  
<br><br>

#### 쿼리와 플러시 모드
JPQL은 영속성 컨텍스트에 있는 데이터를 고려하지 않고 데이터베이스에서 데이터를 조회하기 때문에,
JPQL을 실행하기 전에 영속성 컨텍스트의 내용을 데이터베이스에 반영해햐 한다.

```java
//가격을 1000 -> 2000원으로 변경
product.setPrice(2000);

//가격이 2000원인 상품 조회
Product product2 = 
        em.createQuery("select p from Product p where p.price = 2000", Product.class)
        .getSingleResult();
```
<br>
`product.setPrice(2000)` 를 호출했지만 여전히 데이터베이스에는 1000원인 상태로 남아있다.
다음으로 JPQL을 호출했는데 이 때 플러시 모드가 `AUTO`이므로 쿼리 실행 직전에 영속성 컨텍스트가 플러시 된다.
따라서 수정한 상품을 조회할 수가 있다.

**만약 이런 상황에서 플러시 모드를 `COMMIT`으로 설정하면 쿼리시에는 플러시 하지 않으므로 방금 수정한 데이터를 조회할 수 없다.**
<br><br>

#### 플럿 모드와 최적화
그렇다면, 이 `COMMIT` 모드는 언제 사용하면 좋을까?
다음과 같이 **플러시가 너무 자주 일어나는 상황**에 이 모드를 사용하면 쿼리시 발생하는 플러시 횟수를 줄여서 성능을 최적화할 수 있다.

//비즈니스 로직<br>
등록()<br>
쿼리()  //플러시<br>
등록()<br>
쿼리()  //플러시<br>
등록()<br>
쿼리()  //플러시<br>
커밋()  //플러시<br>

* **FlushModeType.AUTO**: 쿼리와 커밋할 때 총 4번 플러시
* **FlushModeType.COMMIT**: 커밋 시에만 1번 플러시
<br>
  
JPA를 통하지 않고 JDBC로 쿼리를 직접 실행하면 JPA는 JDBC가 실행한 쿼리를 인식할 방법이 없다.
따라서 **별도의 JDBC 호출은 플러시 모드를 AUTO로 설정해도 플러시가 일어나지 않는다.**
이때는 JDBC로 쿼리를 실행하기 직전에 `em.flush()`를 호출해서 영속성 컨텍스트의 내용을 데이터베이스에 동기화 하는 것이 안전하다. 
<br><br>

### 정리
<hr>

* JPQL은 SQL을 추상화해서 특정 데이터베이스 기술에 의존하지 않는다.
* Criteria나 QueryDSL은 JPQL을 만들어주는 빌더 역할을 할 뿐이므로 핵심은 JPQL을 잘 알아야 한다.
* Criteria나 QueryDSL을 사용하면 동적으로 변하는 쿼리를 편리하게 작성할 수 있다.
* Criteria는 JPA가 공식 지원하는 기능이지만 직관적이지 않고 사용하기에 불편하다.
반면에 QueryDSL은 JPA가 공식 지원하는 기능은 아니지만 직관적이고 편리하다.
* JPA도 네이티브 SQL을 제공하므로 직접 SQL을 사용할 수 있다.
하지만 특정 데이터베이스에 종석적인 SQL을 사용하면 다른 데이터베이스로 변경하기 쉽지 않다.
  따라서 최대한 JPQL을 사용하고 그래도 방법이 없을 때 네이티브 SQL을 사용하자.
* JQPL은 대량에 데이터를 수정하거나 삭제하는 벌크 연산을 지원한다.

<br><br>

### 참고
<hr>
* 자바 ORM 표준 JPA 프로그래밍(김영한) - 10장_객체지향 쿼리 언어
