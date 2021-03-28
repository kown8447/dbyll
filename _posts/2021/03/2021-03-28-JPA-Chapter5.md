---
layout: post
title: 연관관계 매핑 기초
excerpt: 객체 관계 매핑(ORM)에서 가장 어려운 부분이 바로 객체 연관관계와 테이블 연관관계를 매핑하는 일이다. JPA에서 객체의 참조와 테이블의 외래 키를 어떻게 매핑시키는지 알아보자.
categories: [JPA]
tags: [ManyToOne, OneToMany, mappedBy]
---

### 단방향 연관관계
<hr>

회원과 팀의 관계는 아래와 같다.

<center>
<img src="{{ site.BASE_PATH }}/assets/images/2021/03/28/img1.png">
</center>

객체 연관관계에서 회원 객체와 팀 객체는 **단방향 관계**이다.
하지만 테이블 연관관계에서 봤을때 `TEAM_ID` 외래 키를 통해서 양방향으로 JOIN 이 가능하기 때문에, 회원 테이블과 팀 테이블의 연관관계는 **양방향 관계**이다.

~~~sql
SELECT * FROM MEMBER M JOIN TEAM T ON M.TEAM_ID = T.TEAM_ID  -- Member -> Team

SELECT * FROM TEAM T JOIN ON T.TEAM_ID = M.TEAM_ID   -- Team -> Member
~~~
_<center>외래키 하나를 가지고서 양방향으로 테이블 조인이 가능하다.</center>_
<br>

정리하자면 참조를 사용하는 객체의 연관관계는 단방향이며, 외래 키를 사용하는 테이블의 연관관계는 양방향이다.
<br><br>

> 객체 그래프 탐색

~~~java
public static void main(String[] args) {
    Member member1 = new Member("member1", "회원1");
    Member member2 = new Member("member2", "회원2");
    Team team1 = new Team("team1", "팀1");
    
    member1.setTeam(team1);
    member2.setTeam(team1);
    
    Team findTeam = member1.getTeam();
}
~~~
<br>

이처럼 객체는 참조를 사용해서 연관관계를 탐색할 수 있는데 이것을 **객체 그래프 탐색**이라 한다.

> 객체 관계 매핑

<center>
<img src="{{ site.BASE_PATH }}/assets/images/2021/03/28/img2.png">
</center>

위의 테이블 연관관계를 아래의 코드를 통해 객체 연관관계로 매핑시킬 수 있다.

~~~java
@Entity
public class Member {
    
    @Id
    @Column
    private String id;
    
    private String username;
    
    //연관관계 매핑
    @ManyToOne
    @JoinColumn(name = "TEAM_ID")
    private Team team;
    
    //연관관계 설정
    public void setTeam(Team team) {
        this.team = team;
    }
    
    //Getter, Setter
}

@Entity
public class Team {
    
    @Id
    @Column(name = "TEAM_ID")
    private String id;
    
    private String name;
    
    //Getter, Setter
}
~~~
<br>

회원과 팀은 다대일(N:1) 관계이므로, 이 관계를 매핑시켜줄 어노테이션 `@ManyToOne` 을 사용하였다.
`@JoinColumn` 은 외래 키를 매핑할 때 사용한다. `name`에는 매핑할 외래 키 이름을 지정하며, 이 어노테이션을 생략할 수 있다.
<br><br>

### 연관관계 사용
<hr>

> 저장

~~~java
public class Test {
    public void testSave() {
        Team team1 = new Team("team1", "팀1");
        em.persist(team1);
        
        Member member1 = new Member("member1", "회원1");
        member1.setTeam(team1); //연관관계 설정 member1 -> team1
        em.persist(member1); 
        
        Member member2 = new Member("member2", "회원2");
        member2.setTeam(team1); //연관관계 설정 member1 -> team1
        em.persist(member2);
    }
}
~~~
<br>

회원과 팀 관계를 저장하는 코드이다. **JPA에서 엔티티를 저장할 때 연관된 모든 엔티티는 영속 상태여야 한다.**
위의 코드를 실행하면 아래와 같이 SQL이 실행된다.

~~~sql
INSERT INTO TEAM (TEAM_ID, NAME) VALUES ('team1', '팀1')
INSERT INTO MEMBER (MEMBER_ID, NAME, TEAM_ID) VALUES ('member1', '회원1', 'team1')
INSERT INTO MEMBER (MEMBER_ID, NAME, TEAM_ID) VALUES ('member2', '회원2', 'team1')
~~~
<br><br>

> 조회

~~~java
Member member = em.find(Member.class, "member1");
Team team = member.getTeam();   //객체 그래프 탐색
System.out.println("팀 이름 = " +  team.getname());
~~~
<br>

객체 그래프 탐색인 `member.getTeam()` 을 사용해서 member와 연관된 team 엔티티를 조회한다.
객체 그래프 탐색이외에도 `JPQL` 을 사용하여 직접 쿼리를 날릴 수도 있다.

~~~java
String sql = "select m from Member m join m.team t where t.name=:teamName";
~~~
<br><br>

> 수정

~~~java
Team team2 = new Team("team2", "팀2");
em.persist(team2);

Member member = em.find(Member.class, "member1");
member.setTeam(Team2);
~~~
<br>

단순히 불어온 엔티티의 값만 변경해두면 트랜잭션을 커밋할 때 플러시가 일어나면서 변경 감지 기능이 작동한다.
<br><br>

> 연관관계 제거

~~~java
Member member1 = em.find(Member.class, "member1");
member1.setTeam(null);  //연관관계 제거
~~~
<br><br>

> 연관된 엔티티 삭제

연관된 엔티티를 삭제하려면 기존에 있던 연관관계를 먼저 제거하고 삭제해야 한다.
그렇지 않으면 외래 키 제약조건으로 인해, 데이터베이스에서 오류가 발생한다.

~~~java
member1.setTeam(null);
member2.setTeam(null);
em.remove(team);
~~~
<br><br>

### 양방향 연관관계
<hr>

<center>
<img src="{{ site.BASE_PATH }}/assets/images/2021/03/28/img3.png">
</center>

회원은 하나의 팀을 가질 수 있고, 하나의 팀은 여러 회원을 가질 수 있는 양방향 구조이다.
이 구조를 객체를 이용한 양방향 연관관계 매핑으로 만들어보자.

> 양방향 연관관계 매핑

~~~java
@Entity
public class Member {

    @Id
    @Column
    private String id;

    private String username;

    //연관관계 매핑
    @ManyToOne
    @JoinColumn(name = "TEAM_ID")
    private Team team;

    //연관관계 설정
    public void setTeam(Team team) {
        this.team = team;
    }

    //Getter, Setter
}
~~~
<br>
회원 엔티티는 이미 단방향 관계가 맺어져 있기 때문에 수정할 부분이 없다.
<br>

~~~java
import java.util.ArrayList;

@Entity
public class Team {

    @Id
    @Column(name = "TEAM_ID")
    private String id;

    private String name;

    //추가되는 부분
    @OneToMany(mappedBy = "team")
    private List<Member> members = new ArrayList<Member>();

    //Getter, Setter
}
~~~
<br>
팀과 회원은 **일대다 관계**다. 따라서 일대다 관계를 매핑하기 위해 `@OneToMany` 어노테이션을 추가하였다.
`mappedBy` 속성은 양방향 매핑일 때 사용하는데 반대쪽 매핑의 필드 이름을 값으로 주면 된다.
반대쪽 매핑이 `Member.team` 이므로 `team` 을 값으로 주었다.
<br><br>

### 연관관계의 주인
<hr>

엄밀히 말하자면 객체에는 양방향 연관관계는 없다. 
**서로 다른 단방향 연관관계 2개를 애플리케이션 로직으로 잘 묶어서 양방향인 것처럼 보이게 할 뿐이다.**

* 회원 -> 팀(단방향)
* 팀 -> 회원(단방향)

엔티티를 양방향 연관과계로 설정하면 객체의 참조는 둘인데 외래 키는 하나다.
따라서 두 객체 연관관계 중 하나를 정해서 테이블의 외래 키를 관리해야 하는데 이것을 연관관계의 주인<sub>Onwer</sub>이라 한다.
<br><br>

> mappedBy

연관관계의 주인만이 데이터베이스 연관관계와 매핑되고 외래 키를 관리(등록, 수정, 삭제)할 수 있다.
반면에 주인이 아닌 쪽은 읽기만 할 수 있다.
그리고 어떤 연관관계를 주인으로 정할지는 `mappedBy` 속을을 사용하면 된다.
* 주인은 mappedBy속성을 사용하지 않는다.
* 주인이 아니면 mappedBy 속성을 사용해서 속성의 값으로 연관관계의 주인을 지정해야 한다.
<br><br>
  
> 연관과계의 주인은 외래 키가 있는 곳

<center>
<img src="{{ site.BASE_PATH }}/assets/images/2021/03/28/img4.png">
</center>

연관관계의 주인은 테이블에 외래 키가 있는 곳으로 정해야 한다.
여기서는 회원 테이블이 외래 키를 가지고 있으므로 Member.team이 주인이 된다. 
주인이 아닌 Team.members에는 `mappedBy="team"` 속성을 사용해서 주인이 아님을 설정한다.

데이터베이스 테이블의 다대일, 일대다 관계에서는 **항상 다 쪽이 외래 키를 가진다**.
다 쪽인 `@ManyToOne`은 항상 연관관계의 주인이 되므로 `mappedBy`를 설정할 수 없다.
**따라서 `@ManyToOne` 에는 `mappedBy` 속성이 없다.**
<br><br>

### 양방향 연관관계의 주의점
<hr>

> 연관관계의 주인만이 외래 키 값을 변경할 수 있다.

~~~java
public class Test {
    public void testSaveNonOwner() {
        Member member1 = new Member("member1", "회원1");
        em.persist(member1);
        
        Member member2 = new Member("member2", "회원2");
        em.persist(member2);
        
        Team team1 = new Team("team1", "팀1");
        team1.getMembers().add(member1);
        team1.getMembers().add(member2);
        
        em.persist(team1);
    }
}
~~~
<br>

위의 코드에서 `Team.members` 은 연관관계의 주인이 아니다. 연관관계의 주인은 `Member.team` 이다.
**연관관계의 주인만이 외래 키의 값을 변경할 수 있다.**
연관관계의 주인인 `Member.team` 에 아무 값도 입력하지 않았기 때문에, DB의 TEAM_ID 값은 null 로 저장된다.
<br><br>

> 양방향 모두 값을 입력하는 것이 안전한다.

연관관계의 주인에만 값을 저장하고 반대쪽에는 값을 저장하지 않으면, 순수한 객체 상태에서 문제가 발생할 수 있다.
JPA 입장에서는 연관관계의 주인이 아닌 객체에 값이 없어도 큰 문제가 되지 않지만, 
JPA가 아닌 순수 객체 관점에서 봤을 때 참조값에 값을 조회하려고 할 경우 문제가 될 수 있다.

~~~java
public class Test {
    public void test순수객체() {
        Team team1 = new Team("team1", "팀1");
        Member member1 = new Member("member1", "회원1");
        Member member2 = new Member("member2", "회원2");

        member1.setTeam(team1);
        member2.setTeam(team1);
        
        List<Member> members = team1.getMembers();
        System.out.println("memberes.size = " + members.size());
    }
}
~~~
_<center>결과는 0이 나오며, 우리가 기대한 결과가 아니다.</center>_

**따라서, 객체의 양방향 연관관계는 양쪽 모두 관계를 맺어주는 것이 좋다.**
<br><br>

> 연관관계 편의 메소드

양방향 연관관계는 결국 양쪽 다 신경써야 하기 때문에, 어느 한쪽에서 실수로 호출하는걸 깜빡하게 될 경우 양방향이 깨질 수 있다.
이러한 경우를 대비하여 **연관관계 편의 메소드**를 제공하여 양방향 연관관계를 안전하게 설정할 수 있다.

~~~java
public class Member {
    private Team team;
    
    public void setTeam(Team team) {
        this.team = team;
        team.getMembers().add(this);
    }
}
~~~
<br>
`setTeam()` 메소드 하나로 양방향 관계를 모두 설정하도록 변경했다.
하지만 이 코드에도 문제점이 있는데, 바로 기존에 유지되고 있던 관계를 다른 객체로 변경할 경우이다.

~~~java
member1.setTeam(teamA);
member1.setTeam(teamB);
Member findMember = teamA.getMember();
~~~
<br>
연관관계를 `teamB` 로 바꿨으나, `teamA`의 관계를 제거하지 않았기 때문에 `teamA` 의 `getMember()`를 호출하면 member가 반환된다.
**따라서 연관과계를 변경할 때는 기존 관계를 제거해주어야 한다.**

~~~java
public class Member {
    private Team team;
    
    public void setTeam(Team team) {
        //기존 팀과 관계를 제거
        if (this.team != null) {
            this.team.getMembers().remove(this);   
        }
        
        this.team = team;
        team.getMembers().add(this);
    }
}
~~~
_<center>연관관계 편의 메소드에 기존 관계를 제거하는 로직을 추가한다.</center>_
<br><br>

### 정리
<hr>

* **단방향 매핑만으로 테이블과 객체의 연관관계 매핑은 이미 완료되었다.**
* **단방향을 양방향으로 만들면 반대방향으로 객체 그래프 탐색 기능이 추가된다.**
* **양방향 연관관계를 매핑하려면 객체에서 양쪽 방향을 모두 관리해야 한다.**
* **연관관계의 주인은 외래 키의 위치와 관련해서 정해야지 비즈니스 중요도로 접근하면 안된다.**
* **양방향 매핑 시에는 무한 루프에 빠지지 않게 조심해야 한다.[(참고)](https://velog.io/@devsh/JPA-%EC%97%B0%EA%B4%80-%EA%B4%80%EA%B3%84-%EB%A7%A4%ED%95%91-OneToMany-ManyToOne-OneToOne-ManyToMany)**

<br><br>

### 참고
<hr>
* 자바 ORM 표준 JPA 프로그래밍(김영한) - 5장_연관관계 매핑 기초
* [akmj.log](https://velog.io/@devsh/JPA-%EC%97%B0%EA%B4%80-%EA%B4%80%EA%B3%84-%EB%A7%A4%ED%95%91-OneToMany-ManyToOne-OneToOne-ManyToMany)
* [conatuseus.log](https://velog.io/@conatuseus/%EC%97%B0%EA%B4%80%EA%B4%80%EA%B3%84-%EB%A7%A4%ED%95%91-%EA%B8%B0%EC%B4%88-2-%EC%96%91%EB%B0%A9%ED%96%A5-%EC%97%B0%EA%B4%80%EA%B4%80%EA%B3%84%EC%99%80-%EC%97%B0%EA%B4%80%EA%B4%80%EA%B3%84%EC%9D%98-%EC%A3%BC%EC%9D%B8)