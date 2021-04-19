---
layout: post
title: 객체지향 쿼리 언어_JPQL(2)
categories: [JPA]
tags: [내부조인, 외부조인, 컬렉션조인, 세타조인, 페치조인]
---

이번 시간에는 JPQL에서 지원하는 조인의 종류와 특징에 대해서 살펴보자.

### 내부 조인
<hr>

내부 조인은 `INNER JOIN`을 사용한다.

~~~java
String teamName = "팀A";
String query = "SELECT m FROM Member m INNER JOIN m.team t " +
        " + WHERE t.name = :teamName";

List<Member> members = em.createQuery(query, Member.class)
    .setParameter("treamName", teamName)
    .getResultList();
~~~
<br>
위의 JPQL은 다음의 SQL로 나타낼 수 있다.

~~~sql
SELECT 
    M.ID AS ID,
    M.AGE AS AGE,
    M.TEAM_ID AS TEAM_ID,
    M.NAME AS NAME
FROM 
    MEMBER M INNER JOIN TEAM T ON M.TEAM_ID=T.ID
WHERE 
    T.NAME=?
~~~
<br>

**JPQL은 JOIN 명령어 다음에 조인할 객체의 연관 필드를 사용한다**. 위에서의 연관 필드는 `m.team`이다.
<br><br>

### 외부 조인
<hr>

~~~jpaql
SELECT m
FROM Member m LEFT [OUTER] JOIN m.team t
~~~
<br>
위의 JPQL은 다음의 SQL로 나타낼 수 있다.

~~~sql
SELECT
    M.ID AS ID,
    M.AGE AS AGE,
    M.TEAM_ID AS TEAM_ID,
    M.NAME AS NAME
FROM
    MEMBER M LEFT OUTER JOIN TEAM T ON M.TEAM_ID=T.ID
WHERE
    T.NAME=?
~~~
<br>

`OUTER` 키워드는 생략할 수 있다.
<br><br>

### 컬렉션 조인
<hr>

일대다 관계나 다대다 관계처럼 컬렉션을 사용하는 곳에 조인하는 것을 컬렉션 조인이라 한다.

~~~jpaql
SELECT t, m FROM Team t LEFT JOIN t.members m
~~~

<br>
팀과 팀이 보유한 회원목록을 컬렉션 값 연관 필드로 외부 조인했다.
<br><br>

### 세타 조인
<hr>
세타 조인은 내부 조인만 지원한다. 세타 조인을 사용하면 전혀 관계없는 엔티티도 조인할 수 있다.

~~~sql
--JPQL
select count(m) from Member m, Team t where m.username = t.name

--SQL
SELECT COUNT(M.ID)
FROM MEMBER M CROSS JOIN TEAM T 
WHERE M.USERNAME = T.NAME
~~~
_<center>전혀 관련없는 Member.username과 Team.name을 조인했다.</center>_
<br>

### 페치 조인
<hr>
페치<sub>fetch</sub>조인은 JPQL에서 성능 최적화를 위해 제공하는 기능으로, 연관된 엔티티나 컬렉션을 한 번에 같이 조회한다.

> 엔티티 페치 조인

~~~jpaql
select m from Member m join fetch m.team
~~~
_<center>회원과 팀을 함께 조회</center>_
<br>

페치 조인은 별칭을 사용할 수 없다. 위의 JPQL은 다음의 SQL로 나타낼 수 있다.

~~~SQL
SELECT M.*, T.* FROM MEMEBER M 
INNER JOIN TEAM T ON M.TEAM_ID=T.ID
~~~
<br>

JPQL에서 `select m`으로 회원 엔티티만 조회했는데 실행된 SQL에서는 `M.*, T.*`로 회원과 연관된 팀도 함께 조회된 것을 확인할 수 있다.

페치 조인을 사용해서 팀도 함께 조회했으므로 **연관된 팀 엔티티는 프록시가 아닌 실제 엔티티다**.
따라서 연관된 팀을 사용해도 **지연 로딩이 일어나지 않는다**.

그리고 프록시가 아닌 실제 엔티티이므로 회원 엔티티가 영속성 컨텍스트에서 분리되어 **준영속 상태가 되어도
연관된 팀을 조회할 수 있다**.
<br><br>

### 컬렉션 페치 조인
<hr>

~~~jpaql
select t
from Team t join fetch t.members
where t.name = '팀A'
~~~
<br>
다음은 실행된 SQL이다.

~~~sql
SELECT T.*, M.*
FROM TEAM T
INNER JOIN MEMBER M ON T.ID=M.TEAM_ID
WHERE T.NAME = '팀A'
~~~
<br>

<center>
<img src="{{ site.BASE_PATH }}/assets/images/2021/04/19/img1.png">
</center>

TEAM 테이블에서 '팀A' 는 하나지만 MEMBER 테이블과 조인하면서 결과가 증가해서 조인결과 위의 그림처럼 같은 주소 0x100 '팀A'가 2건 조회되었다.

> 페치 조인과 DISTINCT

JPQL의 `DISTINCT` 명령어는 SQL에 DISTINCT를 추가하는 것은 물론이고 애플리케이션에서 한 번 더 중복을 제거한다.

컬렉션 페치 조인 예제에서 팀A가 중복으로 조회된다.
하지만 SQL에서 DISTINCT를 사용해도 각 로우의 데이터가 다르므로 DISTINCT의 효과는 없다. 

~~~jpaql
select distinct t
from Team t join fetch t.members
where t.name = '팀A'
~~~
<br>

로우번호 | 팀 | 회원
---| --- | ---
1|팀A|회원1
2|팀A|회원2

하지만, 애플리케이션에서는 distinct는 **팀 엔티티의 중복을 제거해준다**. 
따라서 앞서 컬렉션 페치 조인에서 2번 조회되던 팀엔티티는 DISTINCT를 사용하면 1번 조회로 줄일 수 있다.
<br><br>

### 페치 조인과 일반 조인의 차이
<hr>

* 일반 조인을 사용하면 연관된 엔티티는 조회되지 않는다.
* 반면 페치 조인을 사용하면 연관된 엔티티도 함께 조회한다.

<br><br>

### 페치 조인의 특징과 한계
<hr>

* 페치 조인은 글로벌 로딩 전략보다 우선한다.
    * 따라서 글로벌 로딩 전략은 될 수 있으면 지연로딩을 사용하고 최적화가 필요하면 페치 조인을 적용하는 것이 효과적이다.
    
* 페치 조인을 사용하면 연관된 엔티티를 쿼리 시점에 조회하므로 지연로딩이 발생하지 않는다.
    * 준영속 상태에서도 객체 그래프를 탐색할 수 있다.
    
* 페치 조인 대상에는 별칭을 줄 수 없다.
    * SELECT, WHERE절, 서브 쿼리에 페치 조인 대상을 사용할 수 없다.
    
* 둘 이상의 컬렉션을 페치할 수 없다.
* 컬렉션을 페치 조인하면 페이징 API(setFirstResult, setMaxResults)를 사용할 수 없다.

<br><br>

### 참고
<hr>
* 자바 ORM 표준 JPA 프로그래밍(김영한) - 10장_객체지향 쿼리 언어
* [Java칩 프라푸치노](https://blog.naver.com/PostView.nhn?blogId=qjawnswkd&logNo=222078705093)
