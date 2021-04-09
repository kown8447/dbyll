---
layout: post
title: 프록시와 연관관계 관리
categories: [JPA]
tags: [proxy, eager, lazy, 영속성 전이, 고아객체]
---

### 프록시
<hr>

연관관계가 있는 객체 사이에서, 한쪽 객체만 조회하여 사용하려는데 연관관계에 있는 객체까지 데이터베이스에서 조회가 되는 것은 효율적이지 않다.
JPA는 이런 문제를 해결하려고 엔티티가 실제 사용될 때까지 데이터베이스 조회를 지연하는 방법을 제공하는데 이것을 `지연 로딩`이라 한다.

이 지연 로딩 기능을 사용하려면 실제 엔티티 객체 대신에 데이터베이스 조회를 지연할 수 있는 가짜 객체가 필요한데 이것을 `프록시 객체`라 한다.
<br><br>

> 프록시 특징

<center>
<img src="{{ site.BASE_PATH }}/assets/images/2021/04/08/img1.png">
</center>
_<center>member.getName()처럼 실제 사용될 때 데이터베이스 조회하여 프록시 객체를 초기화한다.</center>_

1. **프록시 객체는 처음 사용할 때 한 번만 초기화 된다.**
   * 여기서의 초기화는 프록시 객체가 영속성 컨텍스트에 실제 엔티티 생성을 요청하는 것을 의미한다.
2. **프록시 객체가 초기화한다고 프록시 객체가 실제 엔티티로 바뀌는 것은 아니다.**
   * 프록시 객체를 통해서 실제 엔티티에 접근할 수 있다.
3. **프록시 객체는 원본 엔티티를 상속받은 객체이므로 타입 체크 시에 주의해서 사용해야 한다.**
4. **영속성 컨텍스트에 찾는 엔티티가 이미 있으면 데이터베이스를 조회할 필요가 없으므로, 프록시가 아닌 실제 엔티티를 반환한다.**
5. **초기화는 영속성 컨텍스트의 도움을 받아야 가능하다.**
   * 준영속 상태에서 프록시를 초기화하면 예외가 발생한다.
   
<br><br>

> 프록시와 식별자

~~~java
Team team = em.getReference(Team.class, "team1")    //식별자 보관
team.getId();   //초기화되지 않음
~~~
<br>

프록시 객체는 식별자 값을 가지고 있으므로 식별자 값을 조회하는 team.getId()를 호출해도 초기화하지 않는다.
단 엔티티 접근 방식을 프로퍼티(@Access(AccessType.PROPERTY))로 설정한 경우에만 초기화하지 않는다.
<br><br>

> 프록시 확인

`PersistenceUitUtil.isLoaded(Object entity)` 또는 `Object(Entity).class()` 를 호출하여 프록시 인스턴스인지 확인할 수 있다.
프록시 인스턴스일 경우 클래스명 뒤에 `..javaassist..` 같은 부수적인 name이 덧붙여진다.
<br><br>

### 즉시 로딩과 지연 로딩
<hr>

* 즉시 로딩 : 엔티티를 조회할 때 연관된 엔티티도 함께 조회한다.
* 지연 로딩 : 연관된 엔티티를 실제 사용할 때 조회한다.

<br><br>

> 즉시 로딩

~~~java
@Entity
public class Member {
    @ManyToOne(fetch = FetchType.EAGER)
    @JoinColumn(name = "TEAM_ID")
    private Team team;
}
~~~

_<center>'fetch = FetchType.EAGER' 적용</center>_
<br>

위의 상황에서 `em.find(Member.class, "memeber1")`로 회원을 조회하는 순간 팀도 함께 조회한다.
대부분의 JPA 구현체에는 **즉시 로딩을 최적화하기 위해 가능하면 조인 쿼리를 사용한다.**

~~~sql
SELECT
    M.MEMBER_ID AS MEMBER_ID,
    M.TEAM_ID as TEAM_ID
    M.USERNAME as USERNAME,
    T.TEAM_ID as TEAM_ID,
    T.NAME as NAME
FROM
    MEMBER M LEFT OUTER JOIN TEAM T
        ON M.TEAM_ID = T.TEAM_ID
WHERE
    M.MEMBER_ID = 'member1'
~~~
<br>

위의 쿼리문에서 외부조인(outer join)을 한것을 확인할 수 있는데, 
내부조인(inner join)을 할 경우 팀에 소속되지 않은 회원일 경우 팀은 물론이고 회원데이터도 조회할 수 없는 상황이 나온다.
그래서 JPA는 기본적으로 외부 조인을 사용한다.
하지만, 성능적과 최적화 측면에서는 내부조인이 더 유리하다.
조인 방식을 결정하는 방법은 아래와 같다.

* `@JoinColumn` 에 nullable 설정
   * @JoinColumn(nullable = true) : 외부 조인 사용
   * @JoinColumn(nullable = false) : 내부 조인 사용
   
* `optional` 설정
   * @ManyToOne(optional = true) : 외부 조인 사용
   * @ManyToOne(optional = false) : 내부 조인 사용
   
<br><br>

> 지연 로딩

~~~java
@Entity
public class Member {
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "TEAM_ID")
    private Team team;
}
~~~

_<center>'fetch = FetchType.LAZY' 적용</center>_
<br>

`em.find(Member.class, "member1")`을 조회해도 회워만 조회하고 팀은 조회하지 않는다.
`Team team = member.getTeam()`을 호출해도 반환되는 건 프록시 객체이다.
실제로 team을 사용하는 부분인 `team.getName()`을 호출해야 프록시가 초기화되므로, 이를 지연 로딩이라 한다.

~~~sql
SELECT * FROM MEMBER
WHERE MEMBER_ID = 'member1'
~~~
_<center>'em.find(Member.class, "member1")' : MEMBER 테이블만 조회한다.</center>_
<br>

~~~sql
SELECT * FROM TEAM
WHERE TEAM_ID = 'team1'
~~~
_<center>'team.getName()' : 실제로 사용하는 경우가 되어야 TEAM 테이블을 조회한다.</center>_
<br>

참고로, 조회 대상이 영속성 컨텍스트에 이미 있으면 프록시 객체를 사용하지 않고 실제 객체를 반환한다.
<br><br>

### 지연 로딩 활용
<hr>

> 프록시와 컬렉션 래퍼

하이버네이트는 엔티티를 영속 상태로 만들 때 엔티티에 컬렉션이 있으면 컬렉션을 추적하고 관리할 목적으로 원본 컬렉션을 하이버네이트가 제공하는 내장 컬렉션으로 변경하는데 이것을 `컬랙션 래퍼`라 한다.
엔티티를 지연 로딩하면 프록시 객체를 사용해서 지연 로딩을 수행하지만 컬렉션은 컬렉션 래퍼가 지연 로딩을 처리해준다.
<br><br>

> 기본 페치 전략

* @ManyToOne, @OneToOne : 즉시 로딩
* @ManyToMany, @OneToMany : 지연 로딩

JPA의 기본 페치<sub>fetch</sub> 전략은 연관된 엔티티가 하나면 즉시 로딩을, 컬렉션이면 지연로딩을 사용한다.
<br><br>

> 컬렉션 FetchType.EAGER 사용 시 주의점

1. 컬렉션을 하나 이상 즉시 로딩하는 것은 권장하지 않는다.
    * 일대다 조인은 결과 데이터가 다 쪽에 있는 수만큼 증가하게 된다.
    
2. 컬렉션 즉시 로딩은 항상 외부 조인<sub>OUTER JOIN</sub>을 사용한다.
    * JPA에서는 일대다 관계를 즉시 로딩할 때 항상 외부 조인을 사용한다.
    * 내부조인을 사용할 경우, 다(N)쪽의 관계가 하나도 없으면 일(1)쪽의 데이터도 조회되지 않는 문제가 생기기 때문
    
<br>

#### `FetchType.EAGER` 설정과 조인 전략
* @ManyToOne, @OneToOne
    * optional = false : 내부 조인
    * optional = true : 외부 조인
    
* @OneToMany, @ManyToMany
    * optional = false : 외부 조인
    * optional = true : 외부 조인
    
<br><br>

### 영속성 전이: CASCADE
<hr>
JPA는 `CASCADE` 옵션으로 영속성 전이를 제공한다.
**JPA에서 엔티티를 저장할 때 연관된 모든 엔티티는 영속 상태여야 한다.**
이럴 때 영속성 전이를 사용하면 부모만 영속 상태로 만들면 연관관계의 객체까지 한 번에 영속 상태로 만들 수 있다.
<br><br>

> 영속성 전이 활용

~~~java
@Entity
public class Parent {
    @OneToMany(mappedBy = "parent", cascade = CascadeType.PERSIST)
    private List<Child> children = new ArrayList<Child>();
}
~~~
_<center>'cascade = CascadeType.PERSIST' 옵션 지정</center>_
<br>

~~~java
private static void saveWithCascade(Entitymanager em) {
    
    Child child1 = new Child();
    Child child2 = new Child();
    
    Parent parent = new Parent();
    child1.setParent(parent);
    child2.setParent(parent);
    parent.getChildren().add(child1);
    parent.getChildren().add(child2);
    
    //부모 저장, 연관된 자식들 저장
    em.persist(parent);
}
~~~
<br>

위의 코드에서 볼 수 있다시피, 자식객체의 영속화 과정이 없다.
부모만 영속화하면 `CascadeType.PERSIST` 로 설정한 자식 엔티티까지 함께 영속화해서 저장한다.
<br><br>

> CASCADE 종류

~~~java
public enum CascadeType {
    ALL,
    PERSIST,
    MERGE,
    REMOVE,
    REFRESH,
    DETACH
}
~~~
<br>

`CascadeType.PERSIST`, `CascadeType.REMOVE` 는 em.persist(), em.remove() 를 실행할 때 바로 전이가 발생하지 않고
플러시를 호출할 때 전이가 발생한다.
<br><br>

> 고아 객체

JPA는 부모 엔티티와 연관관계가 끊어진 자식 엔티티를 자동으로 삭제하는 기능을 제공하는데 이것을 `고아 객체(ORPHAN) 제거`라 한다.

~~~java
@Entity
public class Parent {
    @Id @GenerateValue
    private Long id;
    
    @OneToMany(mappedBy = "parent", orphanRemoval = true)
    private List<Child> children = new ArrayList<>();
}
~~~
<br>

`orphanRemoval` 을 true 로 설장했다. 이제 컬렉션에서 제거한 엔티티는 자동으로 삭제된다.

~~~java
Parent parent1 = em.find(Parent.class, id);
parent1.getChildrent().remove(0);   //자식 엔티티를 컬렉션에서 제거
~~~
_<center>영속성 컨텍스트 플러시가 일어날 때 DELETE 쿼리가 실행된다.</center>_
<br>

**고아 객체 제거는 참조가 제거된 엔티티는 다른곳에서 참조하지 않는 고아 객체로 보고 삭제하는 기능**이다.
따라서 이 기능은 참조하는 곳이 하나일 때만 사용해야한다.
**이런 이유로 `orphanRemoval`은 `@OneToOne`, `@OneToMany` 에만 사용할 수 있다.**
<br><br>

### 정리
<hr>

* JPA 구현체들은 객체 그래프를 마음껏 탐색할 수 있도록 지원하는데 이때 프록시 기술을 사용한다.
* 객체를 조회할 때 연관된 객체를 즉시 로딩하는 방법을 즉시 로딩이라 하고, 연관된 객체를 지연해서 로딩하는 방법을 지연 로딩이라 한다.
* 객체를 저장하거나 삭제할 때 연관된 객체도 함께 저장하거나 삭제할 수 있는데 이것을 영속성 전이라 한다.
* 부모 엔티티와 연관관계가 끊어진 자식 엔티티를 자동으로 삭제하려면 고아 객체 제거 기능을 사용하면 된다.

<br><br>

### 참고
<hr>
* 자바 ORM 표준 JPA 프로그래밍(김영한) - 8장_프록시와 연관관계 관리
* [ystone](https://velog.io/@ljinsk3/JPA-%ED%94%84%EB%A1%9D%EC%8B%9C%EC%99%80-%EC%97%B0%EA%B4%80%EA%B4%80%EA%B3%84-%EC%A0%95%EB%A6%AC)

