---
layout: post
title: 객체지향 쿼리 언어_JPQL(1)
categories: [JPA]
tags: [JPQL]
---

객체지향 쿼리 언어는 `JPQL`, `Criteria Query`, `QueryDSL` 등이 있는데, 오늘부터 이 세가지 객체지향 쿼리 언어에 대해서 알아보고자 한다.
먼저 알아볼 객체지향 쿼리 언어는 `JPQL`이다.
기능적인 설명이 많아서 포스팅 내용이 길어져서 앞으로 여러차례 나눠올릴 예정이다.

### JPQL
<hr>

JPQL<sub>Java Persistence Query Language</sub>은 엔티티 객체를 조회하는 객체지향 쿼리다.
JPQL을 다음과 같은 특징을 가진다.

* JPQL은 객체지향 쿼리 언어다. 따라서 테이블을 대상으로 쿼리하는 것이 아니라 엔티티 객체를 대상으로 쿼리한다.
* JPQL은 SQL을 추상화해서 특정 데이터베이스 SQL에 의존하지 않는다.
* JPQL은 결국 SQL로 변환된다.

앞으로의 예제는 다음의 ERD 모델을 바탕으로 설명하겠다.

<center>
<img src="{{ site.BASE_PATH }}/assets/images/2021/04/18/img1.png">
</center>

<br><br>

#### 기본 문법과 쿼리 API
<hr>

> SELECT 문

~~~jpaql
SELECT m FROM Members AS m where m.username = 'Hello'
~~~

* 대소문자 구분
    * 엔티티와 속성은 대소문자를 구분한다. 반면에 SELECT, FROM 같은 JPQL 키워드는 대소문자를 구분하지 않는다.
    
* 엔티티 이름
    * `Member`는 클래스명이 아니라 엔티티 명이다. 엔티티명을 지정하지 않으면 클래스명을 기본값으로 사용한다.
    
* 별칭은 필수
    * JPQL은 별칭을 필수로 사용해야 한다.
    
<br><br>

> TypeQuery, Query

반환할 타입을 명확하게 지정할 수 있으면 `TypeQuery`, 반환 타입을 명확하게 지정할 수 없으면 `Query` 객체를 사용한다.

~~~java
TypeQuery<Member> query = em.createQuery("SELECT m FROM Member m", Member.class);

List<Member> resultList = query.getResultList();
for(Member member : resultList) {
    System.out.println("member = " + member);    
}
~~~

_<center>Member.class 라는 명확한 타입이 있다.</center>_
<br>

~~~java
Query query = em.createQuery("SELECT m.username, m.age FROM Member m");
List resultList = query.getResultList();
        
for(Object o : resultList) {
    Object[] result = (Object[]) o;
    System.out.println("username = " + result[0]);    
    System.out.println("age = " + result[1]);    
}
~~~

_<center>명확한 타입이 없으므로, Query로 받고 그 결과는 Object 또는 Object[]로 반환된다.</center>_
<br><br>

> 결과 조회

* query.getResultList(): 결과를 예제로 반환한다. 결과가 없으면 빈 컬렉션을 반환한다.
* query.getSingleResult(): 결과가 정확히 하나일 때 사용한다.
    * 결과가 없으면 javax.persistence.NoResultException 예외가 발생한다.
    
<br><br>

#### 파라미터 바인딩
<hr>

> 이름 기준 파라미터

이름 기준 파라미터는 앞에 `:`을 사용한다.
~~~java
String usernameParam = "User1";

List<Member> members = em.createQuery("SELECT m FROM Member m where m.username = :username", Member.class)
                            .setParameter("username", usernameParam)
                            .getResultList();
~~~
<br><br>

> 위치 기준 파라미터

`?` 다음에 위치 값을 준다. 위치 값은 1부터 시작한다.
~~~java
String usernameParam = "User1";

List<Member> members = em.createQuery("SELECT m FROM Member m where m.username = ?1", Member.class)
                            .setParameter(1, usernameParam)
                            .getResultList();
~~~
<br>

위치 기준 파라미터 방식보다는 이름 기준 파라미터 바인딩 방식을 사용하는 것이 더 명확하다.

파라미터 바인딩 방식은 파라미터의 값이 달라도 같은 쿼리로 인식해서 JPA는 JPQL을 SQL로 파싱한 결과를 재사용할 수 있다.
그리고 데이터베이스도 내부에서 실행한 SQL을 파싱해서 사용하는데 같은 쿼리는 파싱한 결과를 재사용할 수 있다.
결과적으로 애플리케이션과 데이터베이스 모두 해당 쿼리의 파싱 결과를 재사용할 수 있어서 전체 성능이 향상된다.

<br><br>

#### 프로젝션
<hr>

SELECT 절에 조회할 대상을 지정하는 것을 프로젝션<sub>projection</sub>이라 한다.

> 엔티티 프로젝션

~~~jpaql
SELECT m FROM Member m;         -- 회원
SELECT m.team FROM Member m;    -- 팀
~~~
<br>

**이렇게 조회한 엔티티는 영속성 컨텍스트에서 관리된다.**
<br><br>


> 임베디드 타입 프로젝션

~~~java
String query = "SELECT o.address FROM Order o";
List<Address> address = em.createQuery(query, Address.class)
                        .getResultList();
~~~
<br>

임베디드 타입은 엔티티 타입이 아닌 값 타입이므로, **이렇게 조회한 임베디드 타입은 영속성 컨텍스트에서 관리되지 않는다.**
<br><br>

> 스칼라 타입 프로젝션

숫자, 문자, 날짜와 같은 기본 데이터 타입들을 스칼라 타입이라 한다.

~~~java
List<String> usernames = em.createQuery("SELECT m.username FROM Member m", String.class)
                            .getResultList();
~~~
<br><br>

> 여러 값 조회

프로젝션에 여러 값을 선택하면 `TypeQuery`를 사용할 수 없고 대신에 `Query`를 사용해야 한다.

~~~java
List<Object[]> resultList = em.createQuery("SELECT m.username, m.age FROM Member m")
                            .getResultList();

for (Object[] row : resultList) {
    String username = (String) row[0];
    Integer age = (Integer) row[1];
}
~~~
<br>

~~~java
List<Object[]> resultList = em.createQuery("SELECT o.member, o.product, o.orderAmount FROM Order o")
                            .getResultList();

for (Object[] row : resultList) {
    Member member = (Member) row[0];
    Product product = (Product) row[1];
    int orderAmount = (Integer) row[2];
}
~~~

_<center>엔티티 타입도 여러 값을 조회할 수 있으며, 이렇게 조회한 엔티티도 영속성 컨텍스트에서 관리한다.</center>_
<br><br>

> New 명령어

실제 애플리케이션 개발시에는 `Object[]`를 직접 사용하지 않고, `DTO`와 같은 의미 있는 객체로 변환해서 사용한다.

~~~java
public class UserDTO {
    private String username;
    private int age;
    
    public UserDTO(String username, int age) {
        this.username = username;
        this.age = age;
    }
}

TypeQuery<UserDTO> query = em.createQuery("SELECT new jpabook.jpql.UserDTO(m.username, m.age) FROM Member m", UserDTO.class);
List<UserDTO> resultList = query.getResultList();
~~~
<br>

SELECT 다음에 NEW 명령어를 사용하면 반환받을 클래스를 지정할 수 있는데 이 클래스의 생성자에 JPQL 조회 결과를 넘겨줄 수 있다.
NEW 명령어 사용시 주의점은 다음과 같다.

1. 패키지 명을 포함한 전체 클래스 명을 입력해야 한다.
2. 순서와 타입이 일치하는 생성자가 필요하다.

<br><br>

#### 페이징 API
<hr>

데이터베이스마다 페이징을 처리하는 SQL 문법이 다르다.
JPA는 페이징을 다음 두 API로 추상화했다.

* setFirstResult(int startPosition): 조회 시작 위치(0부터 시작한다)
* setMaxResults(int maxResult): 조회할 데이터 수

~~~java
TypeQuery<Member> query = 
    em.createQuery("SELECT m FROM Member m ORDER BY m.username DESC", Member.class);

query.setFirstResult(10);   // 시작이 10이므로 11번째부터 시작
query.setMaxmResults(20);   // 20건의 데이터 조회
query.getResultList();      // 11 ~ 30번 데이터 조회
~~~
<br>

페이징 SQL을 더 최적화하고 싶다면 JPA가 제공하는 페이징 API가 아닌 네이티브 SQL을 직접 사용해야 한다.
<br><br>

#### 집합과 정렬
<hr>

> 집합 함수

함수 | 설명
---|---
`COUNT` | 결과 수를 구한다. 반환 타입: Long
`MAX, MIN` | 최대, 최소 값을 구한다. 문자, 숫자, 날짜 등에 사용한다.
`AVG` | 평균값을 구한다. 숫자타입만 사용할 수 있다. 반환 타입:Double
`SUM` | 합을 구한다. 숫자타입만 사용할 수 있다. 반환 타입: 정수합 Long, 소수합: Double, BigInteger합: BigInteger, BigDecimal합: BigDecimal

<br><br>

> 참고 사항

1. NULL 값은 무시하므로 통계에 잡히지 않는다.
2. 만약 값이 없는데 SUM, AVG, MAX, MIN 함수를 사용하면 NULL 값이 된다. 단, COUNT는 0이 된다.
3. DISTINCT를 집합 함수 안에 사용해서 중복된 값을 제거하고 나서 집합을 구할 수 있다.(예: select COUNT(DISTINCT m.age) from Member m)
4. DISTINCT를 COUNT에서 사용할 때 임베디드 타입은 지원하지 않는다.

<br><br>

> GROUP BY, HAVING

~~~jpaql
select t.name, COUNT(m.age), SUM(m.age), AVG(m.age), MAX(m.age), MIN(m.age)
from Member m LEFT JOIN m.team t
GROUP BY t.name
HAVING AVG(m.age) >= 10
~~~

_<center>팀이름으로 그룹핑한 데이터 중에서 평균 나이가 10살 이상인 그룹 조회</center>_

문법은 다음과 같다.

~~~
group by_절 ::= GROUP BY {단일값 경로 | 별칭} +
having_절 ::= HAVING 조건식
~~~
<br>

처리될 결과가 아주 많다면 통계 결과만 저장하는 테이블을 별도로 만들어 두고 사용자가 적은 새벽에 통계 쿼리를 실행해서 그 결과를 보관하는 것이 좋다.
<br><br>

> 정졀(ORDER BY)

* ASC: 오름차순(기본값)
* DESC: 내림차순

~~~jpaql
select m from Member m order by m.age DESC, m.username ASC
~~~

_<center>나이를 기준으로 내림차순으로 정렬하고 나이가 같으면 이름 기준으로 오름차순 정렬</center>_

<br><br>

### 참고
<hr>
* 자바 ORM 표준 JPA 프로그래밍(김영한) - 10장_객체지향 쿼리 언어