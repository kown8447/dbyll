---
layout: post
title: JPA 소개
excerpt: JPA 스터디를 진행하게 되었다. 오랜만에 하는 스터디라 설레기도 하고 굉장히 기대가 크다. 신규 프로젝트에서 JPA를 사용할 예정이기 때문에, 이번 스터디에서 정말 알차게 배울 수 있도록 노력해야겠다. 오늘은 역시나 처음 배우면 다 그렇듯 JPA란 무엇인가부터 알아보자.
categories: [JPA]
tags: [JPA, 패러다임 불일치]
---

### JPA란?
<hr>
객체 모델과 관계형 데이터베이스 모델은 지향하는 패러다임이 서로 다르다.
그렇기 때문에 개발자는 이 패러다임의 차이를 극복하기위해 많은 시간과 코드를 소비한다.
이러한 문제때문에 객체 모델링은 힘을 잃고 점점 데이터 중심의 모델로 변해간다.

자바 진영에서는 오랜 기간 이 문제에 대한 숙제를 안고 있었고, 패러다임의 불일치 문제를 해결하기 위해 많은 노력을 기울여왔다.
그리고 그 결과물이 바로 JPA다.
**JPA는 패러다임의 불일치 문제를 해결해주고 정교한 객체 모델링을 유지하게 도와준다.**
<br><br>

### SQL을 직접 다뤘을때의 문제점
<hr>
JPA에 대해서 자세히 알아보기 전에, JPA를 사용하지 않고 SQL을 직접 다뤘을때에 어떠한 문제점과 불편함이 있을지 먼저 알아보자.

> 반복

아래와 같이 회원 객체가 있다.

~~~java
public class Member {
    private String memberId;
    private String name;
}
~~~
<br>
이제 이 회원 객체의 CRUD를 관리해주는 DAO를 만들어보자

~~~java
public class MemberDAO {
    public Member find(String memeberId) {...}
    public void save(Member member) {...}
    public void update(Member member) {...}
    public void delete(Member member) {...}
}
~~~
<br>
데이터베이스는 객체 구조와는 다른 데이터 중심의 구조를 가지므로 객체를 데이터베이스에 직접 저장하거나 조회할 수는 없다.
따라서 개발자가 객체지향 애플리케이션과 데이터베이스 중간에서 SQL과 JDBC API를 사용해서 변환작업을 직접 해주어야 한다.

만약, 애플리케이션에서 사용하는 데이터베이스 테이블이 100개라면 위의 DAO와 비슷한 형태의 SQL을 무수히 작성해야 하고, 반복해야 한다.
<br><br>

> SQL 의존 개발

앞서 예시의 Member 객체에 연락처를 함께 저장해달라는 요구사항이 추가 되었다.
그렇다면 객체는 아래와 같이 수정될 것이다.

~~~java
public class Member {
    private String memberId;
    private String name;
    private String tel; //추가
}
~~~
<br>
이렇게 필드만 추가하면 끝일까? 자바 콜렉션에 담는다고 하면 아래와 같은 코드로 순식간에 처리할 수 있다.

~~~java
import java.util.ArrayList;
import java.util.List;

public class Test {
    public void test() {
        List<Member> memberList = new ArrayList<>();
        Member member = new Member();
        member.setMemberId("tset-id1");
        member.setName("홍길동");
        member.setTel("010-xxx-xxxx");
        
        memberList.add(member); //저장
        Member findMember = list.get(xx);   //조회
        member.setTel("011-xxx-xxxx");  //수정
    }
}
~~~
<br>
객체 단위로 컨트롤이 가능하기 때문에, 위의 코드처럼 수정할 부분이 많지가 않다.

하지만, SQL을 직접 다루게 될 경우에는 각각에 대응하는 DAO의 쿼리를 모두 변경해줘야한다.

~~~sql
INSERT INTO MEMBER(MEMBER_ID, NAME, TEL) VALUES(?,?,?); -- 삽입
SELECT MEMBER_ID, NAME, TEL FROM MEMBER WHERE MEMBER_ID = ? -- 조회
-- update, delete...
~~~
<br>
여간 귀찮은 작업이 아닐수가 없다. 또한, 여기서 회원 객체가 참조하는 팀 클래스가 추가된다고 해보자.

~~~java
public class Member {
    private String memberId;
    private String name;
    private String tel;
    private Team team;  //추가
}

public class Team {
    private String teamName;
}
~~~
<br>
이 상태에서 화면에 팀이름을 출력하려고 한다면, NPE가 날것이다.

~~~java
System.out.println("이름 : " + member.getName());
System.out.println("소속 팀 : " + member.getTeam().getTeamName()); //NPE가 나는 부분
~~~
<br>
당연하게도 기존 DAO에 존재하는 MemberDAO.find(memberId)에는 팀에 대한 정보가 없기 때문이다.
그런데, 다른 개발자분께서 MemberDAO에 'findWithTeam(memberId)'라는 메소드를 추가한 것을 확인했다.

~~~java
public class MemberDAO {
    public Member find(String memeberId) {...}
    public void save(Member member) {...}
    public void update(Member member) {...}
    public void delete(Member member) {...}
    public Member findWithTeam(String memberId) {...}   //추가된 메소드
}
~~~
<br>
그런데, 과연 저 메소드가 진짜로 팀 정보를 함께 가져오는 것일까?
DAO 안의 SQL을 확인하기 전까지 장담할 수가 없다.

데이터 접근 계층을 사용해서 SQL을 숨겨도 어쩔 수 없이 DAO를 열어서 어떤 SQL이 실행되는지 확인을 해야만한다.
이처럼 SQL에 모든 것을 의존하는 상황에서는 개발자들이 엔티티(ex : Member, Team)를 신뢰하고 사용할 수 없다.

SQL의 직접 다룰 때의 단점을 요약하자면 다음과 같다.
* 진정한 의미의 계층 분할이 어렵다.
* 엔티티를 신뢰할 수 없다.
* SQL에 의존적인 개발을 피하기 어렵다.
<br><br>
  
### 패러다임의 불일치
<hr>
앞서 JPA 소개에서, JPA는 객체와 관계형 데이터베이스의 패러다임 불일치 문제를 해결하기 위해 나타났다고 했다.
**객체와 관계형 데이터베이스는 지향하는 목적이 서로 다르므로 둘의 기능과 표현 방법도 다르기 때문에 패러다임의 불일치가 발생한다.**
그렇다면, 이 패러다임 불일치로 발생하는 문제에는 어떤것들이 있을까?
<br><br>

> 상속

다음과 같이 상속 구조를 가지고 있는 객체들이 있다.

~~~java
public abstract class Item {
    Long id;
    String name;
    int price;
}

class Album extends Item {
    String artist;
}

class Movie extends Item {
    String director;
    String actor;
}

class Book extends Item {
    String autor;
    String isbn;
}
~~~
<br>
객체는 이렇게 상속구조를 가질 수 있지만, 데이터베이스에는 상속이란 개념이 없다.
다만, 참조할 수 있는 외래키를 두고 슈퍼타입과 서브타입으로 나타낸다.
이 경우에 Album 객체를 저장하려면 객체를 분해해서 다음 두 SQL을 만들어야 한다.

~~~sql
INSERT INTO ITEM ...
INSERT INTO ALBUM ...
~~~
<br>
Movie 객체도 마찬가지다.

~~~sql
INSERT INTO ITEM ...
INSERT INTO MOVIE ...
~~~
<br>
JDBC API를 사용해서 이 코드를 완성하려면 부모 객체에서 부모 데이터만 꺼내서 ITEM용 INSERT SQL을 작성하고 자식 객체에서 자식 데이터만 꺼내서 ALBUM용 INSERT SQL을 작성해야 한다.
조회도 수정도 비용이 만만치가 않다.

만약 해당 객체들을 데이터베이스가 아닌 자바 컬렉션에 보관한다면 다음 같이 부모 자식이나 타입에 대한 고민 없이 해당 컬렉션을 그냥 사용하면 된다.

~~~java
list.add(album);
list.add(movie);

Album album = list.get(albumId);
~~~
<br>

> 연관관계

객체는 **참조를 사용해서 다른 객체와 연관관계를 가지고 참조에 접근해서 연관된 객체를 조회**한다.
반면에 테이블을 **외래 키를 사용해서 다른 테이블과 연관관계를 가지고 조인을 사용해서 연관된 테이블을 조회**한다.

~~~java
@Data
class Member {
    Team team;
}

class Team {
    ...
}

member.getTeam();   // member -> team 접근
~~~
<br>
반면 데이터베이스는 아래와 같이 외래키를 사용하여 테이블을 조회할 수 있다.

~~~sql
SELECT M.*, T.*
    FROM MEMBER M 
    JOIN TEAM T ON M.TEAM_ID = T.TEAM_ID
~~~
<br>
그렇다면, 객체를 테이블에 맞추어 모델링하면 문제가 해결되지 않을까?

~~~java
class Member {
    String id;          //MEMBER_ID 컬럼
    Long teamId;        //TEAM_ID 컬럼(FK)
    String username;    //USERNAME 컬럼
}

class Team {
    Long id;        //TEAM_ID 컬럼
    String name;    //NAME 컬럼  
}
~~~
<br>
얼핏보면 괜찮지만, 관계형 데이터베이스는 조인이라는 기능이 있으므로 외래 키의 값을 그대로 보관해도 되지만,
객체는 연관된 객체의 참조를 보관해야 아래와 같이 참조를 통해 객체를 찾을 수 있기 때문에 teamId 필드는 무용지물이다.

~~~java
Team team = member.getTeam();   //teamId는 어디에도 사용되지 않는다.
~~~
<br>
반대로 객체지향 모델링으로 참조할 클래스를 필드로 놔두면, 객체를 테이블에 저장하거나 조회하기가 어려워진다.
**객체는 참조가 필요하지만 외래 키가 필요없고, 테이블은 참조가 필요없고 외래 키만 있어야한다.**
결국 개발자가 중간에서 변환 역활을 해야 한다.
<br><br>

> 객체 그래프 탐색

<center>
<img src="{{ site.BASE_PATH }}/assets/images/2021/03/15/img1.png">
</center>

_<center>출처 : https://velog.io/@agugu95/%EC%9E%90%EB%B0%94-ORM-%ED%91%9C%EC%A4%80-JPA-%ED%94%84%EB%A1%9C%EA%B7%B8%EB%9E%98%EB%B0%8D-1%EC%9E%A5</center>_

위의 그림과 같이 객체의 연관관계가 설계되어있다고 하자. 객체는 마음껏 객체 그래프를 탐색할 수 있어야 한다.

~~~java
member.getOrder().getOrderItem()...   //자유로운 객체 그래프 탐색
~~~
<br>
예를 들어 MemberDAO에서 member 객체를 조회할 때 이런 SQL에 Team에 대한 데이터만 조인했다면
'member.getOrder();' 는 NPE가 날 것이다.

**SQL을 직접 다루면 처음 실행하는 SQL에 따라 객체 그래프를 어디까지 탐색할 수 있는지 정해진다.**
이것은 객체지향 개발자에겐 너무 큰 제약이다.
왜냐하면 비즈니스 로직에 따라 사용하는 객체 그래프가 다른데 언제 끊어질지 모를 개체 그래프를 함부로 탐색할 수는 없기 때문이다.
결국, 어디까지 객체 그래프 탐색이 가능한지 알아보려면 데이터 접근 계층인 DAO를 열어서 SQL을 직접 확인해야 한다.
<br><br>

### JPA의 패러다임 불일치 해결
<hr>
앞에서 언급한 패러다임 불일치 문제를 JPA가 해결해줄 수 있다.
상속과 연관관계 문제는 JPA는 개발자에게 객체사용을 제공하여, 뒷단에서 개발자가 조율해야할 문제들을 알아서 해결해준다.

~~~java
jpa.persist(alubm); //객체 형식으로 데이터 저장

String albumId = "id100";
Album album = jpa.find(Album.class, albumId);   //객체 정보만 넘겨주면 알아서 상속관계 조인도 가능

member.setTeam(team);   //회원과 팀 연관관계 설정
jpa.persist(member);    //회원과 연관관계 함께 저장
~~~
<br>

객체의 그래프 탐색 문제와 관련하여 JPA는 연관된 객체를 사용하는 시점에 적절한 SELECT SQL을 실행한다.
따라서 JPA를 사용하면 연관된 객체를 신뢰하고 마음껏 조회할 수 있다.
이 기능은 실제 객체를 사용하는 시점까지 데이터베이스 조회를 미룬다고 해서 **지연로딩**이라 한다.

~~~java
//처음 조회 시점에 SELECT MEMBER SQL
Meber member = jpa.find(Member.class, memberId);

Order order = member.getOrder();
order.getOrderDate();   //Order를 사용하는 시점에 SELECT ORDER SQL
~~~
<br>

### JPA는 왜 사용해야 하는가?
<hr>

> 생산성
>> JPA를 사용하면 반복적인 코드와 CRUD용 SQL을 개발자가 직접 작성하지 않아도 된다.
더 나아가서 JPA에는 DDL문을 자동으로 생성해주는 기능도 있다.
이런 기능들을 사용하면 데이터베이스 설계 중심의 패러다임을 객체 설계 중심으로 역전시킬 수 있다.

> 유지보수
>> 개발자가 작성해야 할 SQL과 JDBC API 코드를 JPA가 대신 처리해주므로 유지보수해야 하는 코드 수가 줄어든다.
또한, JPA가 패러다임의 불일치 문제를 해결해주므로 객체지향 언어가 가진 장점들을 활용해서 유연하고 유지보수하기 좋은 도메인 모델을 편리하게 설계할 수 있다.

> 패러다임 불일치 해결
>> JPA는 상속, 연관관계, 객체 그래프 탐색, 비교하기와 같은 패러다임의 불일치 문제를 해결해준다.

> 성능 
>> JDBC API는 동일한 쿼리라도 조회마다 데이터베이스와 통신하는 반면, JPA는 동일한 쿼리일 경우 한 번만 데이터베이스에 전달하고 두번째는 객체를 재사용하기 때문에
성능면에서 이점을 볼 수 있다.

> 데이터 접근 추상화와 벤더 독립성
>> JPA는 애플리케이션과 데이터베이스 사이에 추상화된 데이터 접근 계층을 제공해서 애플리케이션이 특정 데이터베이스 기술에 종속되지 않도록 한다.
만약 데이터베이스를 변경하면 JPA에게 다른 데이터베이스를 사용한다고 알려주기만 하면 된다.

> 표준
>> JPA는 자바 진영의 ORM 기술 표준이다.

<br><br>

### 참고
<hr>
* 자바 ORM 표준 JPA 프로그래밍(김영한) - 1장_JPA 소개
* [agugu95.log(dropKick)](https://velog.io/@agugu95)