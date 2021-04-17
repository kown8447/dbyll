---
layout: post
title: 값 타입
categories: [JPA]
tags: [embedded type, collection type]
---

JPA의 데이터 타입은 크게 엔티티 타입과 값 타입으로 나눌 수 있다.
그리고 값 타입은 다음의 3가지로 나눌 수 있다.

* 기본값 타입
    * 자바 기본 타입(예: int, double)
    * 래퍼 클래스(예: Integer)
    * String
    
* 임베디드 타입<sub>embedded type</sub>(복합 값 타입)
* 컬렉션 값 타입<sub>collection value type</sub>

각각의 값 타입에 대해서 알아보자.
<br><br>

### 기본 값 타입
<hr>

~~~java
@Entity
public class Member {
    @Id @GeneratedValue
    private Long id;
    
    private String name;
    private int age;
}
~~~
<br>

위 코드에서 `String`, `int`가 기본값 타입이다. 값 타입의 특징은 다음과 같다.

* 값 타입은 식별자 값이 없기 때문에 생명주기를 엔티티에 의존한다.
  * 엔티티 인스턴스를 제거하면 name, age 값도 제거된다.
* 값 타입은 공유하면 안된다.

<br><br>
  
### 임베디드 타입(복합 값 타입)
<hr>

새로운 값 타입을 직접 정희해서 사용할 수 있는데, JPA에서는 이것을 임베디드 타입<sub>embedded type</sub>이라 한다.

~~~java
@Entity
public class Member {
    @Id @GeneratedValue
    private Long id;
    
    private String name;

    //근무 기간
    @Temporal(TemporalType.DATE) 
    Date startDate;
    
    @Temporal(TemporalType.DATE)    
    Date endDate;
    
    //집 주소 표현
    private String city;
    private String street;
    private String zipcode;
}
~~~
<br>

같은 성격을 지닌 필드들을 위의 코드처럼 그대로 나열하는 것은 객체지향적이지 않고 응집력을 떨어뜨린다.
따라서, 같은 타입의 데이터들은 임베디드 타입을 사용하여 묶어보자.

~~~java
@Entity
public class Member {
    @Id @GeneratedValue
    private Long id;
    
    private String name;

    @Embedded Period workPeriod;    //근무 기간
    @Embedded Address homeAddress;  //집 주소
}

@Embeddable
public class Period {
    @Temporal(TemporalType.DATE)
    Date startDate;
  
    @Temporal(TemporalType.DATE)
    Date endDate;
    
    public boolean isWork(Date date) {
        // .. 값 타입을 위한 메소드를 정의할 수 있다.
    }
}

@Embeddable
public class Address {
    @Column(name = "city")  //매핑할 컬럼 정의 가능
    private String city;
    private String street;
    private String zipcode;
}
~~~
<br>

임베디드 타입을 사용하여 Member 엔티티가 더욱 의미 있고 응집력 있게 변하였다.
또한, Period.isWork() 처럼 해당 값 타입만 사용하는 의미 있는 메소드도 만들 수 있다.
임베디드 타입의 특징은 다음과 같다.

* `@Embeddedable` : 값 타입을 정희하는 곳에 표시
* `@Embedded` : 값 타입을 사용하는 곳에 표시
* 위의 두 개의 어노테이션 중 하나는 생략할 수 있다.
* 임베디드 타입은 기본 생성자가 필수다.

임베디드 타입은 엔티티의 값일 뿐으로, 값이 속한 엔티티의 테이블에 매핑한다.(별도의 임베디드 타입을 위한 테이블은 없다)
임베디드 타입 덕분에 객체와 테이블을 아주 세밀하게 매핑하는 것이 가능하다.
**잘 설계한 ORM 애플리케이션은 매핑한 테이블의 수보다 클래스의 수가 더 많다.**
<br><br>

> @AttributeOverride: 속성 재정의

임베디드 타입에 정의한 매핑정보를 재정의하려면 엔티티에 `@AttributeOverride`를 사용하면 된다.

~~~java
@Entity
public class Member {
    @Id @GeneratedValue
    private Long id;
    
    private String name;

    @Embedded Address homeAddress;
    @Embedded Address companyAddress;
}
~~~
<br>
Member엔티티에 회사주소를 추가했는데, 문제는 테이블에 매핑하는 컬럼명이 중복되는 것이다.
이 때, `@AttributeOverride` 어노테이션을 사용해서 매핑정보를 재정의하는 것이다.

~~~java
@Entity
public class Member {
    @Id @GeneratedValue
    private Long id;
    
    private String name;

    @Embedded Address homeAddress;
    
    @Embedded
    @AttributeOverrides({
            @AttributeOverride(name = "city", column=@Column(name = "COMPANY_CITY")),
            @AttributeOverride(name = "street", column=@Column(name = "COMPANY_STREET")),
            @AttributeOverride(name = "zipcoe", column=@Column(name = "COMPANY_ZIPCODE")),
    })
    Address companyAddress;
}
~~~
<br>

생성된 테이블 명세는 다음과 같다.

~~~sql
CREATE TABLE MEMBER(
    COMPANY_CITY varchar(255),
    COMPANY_STREET varchar(255),
    COMPANY_ZIPCODE varchar(255),
    city varchar(255),
    street varchar(255),
    zipcode varchar(255)
    ...
)
~~~
<br>

`@AttributeOverrides`는 엔티티에 설정해야 한다.
임베디드 타입이 임베디드 타입을 가지고 있어도 엔티티에 설정해야 한다.
<br><br>

> 임베디드 타입과 null

**임베디드 타입이 null이면 매핑한 컬럼 값은 모두 null이 된다.**

~~~java
member.setAddress(null);    //null 입력
em.persist(member);
~~~
<br>

회원 테이블의 주소와 관련된 CITY, STREET, ZIPCODE 컬럼 값은 모두 null이 된다.
<br><br>

### 값 타입과 불변 객체
<hr>

값 타입이 공유될 때 발생하는 문제에 대해서 알아보자.

<center>
<img src="{{ site.BASE_PATH }}/assets/images/2021/04/17/img1.png">
</center>

위의 상황을 코드로 나타내면 다음과 같다.

~~~java
member1.setHomeAddresss(new Address("OldCity"));
Address address = member1.getHomeAddress();

address.setCity("newCity"); //회원1의 address 값을 공유해서 사용
member2.setHomeAddress(address);
~~~
<br>

회원2의 주소만 바꾸길 의도했으나, 회원1의 주소도 변경되어 버리는 상황을 초래한다.
**영속성 컨텍스트는 회원1과 회원2 둘 다 city 속성이 변경된 것으로 판단해서 회원1, 회원2 각각 UPDATE SQL을 실행한다.**
이러한 부작용을 막는 가장 간단한 방법은 값을 복사해서 사용하는 것이다.

~~~java
member1.setHomeAddresss(new Address("OldCity"));
Address address = member1.getHomeAddress();

//회원 1의 address 값을 복사해서 새로운 newAddress 값을 생성
Address newAddress = address.clone();

newAddress.setCity("newCity");
member2.setHomeAddress(newAddress);
~~~
<br>

자바는 기본 타입은 항상 값을 복사해서 사용하기 때문에 문제가 없지만, 위의 Address 같은 임베디드 타입은 객체타입이라는 것에 문제가 있다.

~~~java
int a = 10;
int b = a;  //기본 타입은 항상 값을 복사한다.
b = 4;

System.out.println("a = " + a + ", b = " + b);  //a = 10, b = 4

Address a = new Address("Old");
Address b = a;  //객체 타입은 항상 참조 값을 전달한다.
b.setCity("New");

System.out.println("a city = " + a.getCity() + ", b city = " + b.getCity());  // 둘다 new
~~~
<br>
객체 타입도 값을 복사해서 사용하면 문제는 없지만, 복사하지 않고 원본의 참조 값을 직접 넘기는 것은 막을 방법이 없다.

~~~java
Address a = new Address("Old");
Address b = a.clone();  //복사해서 넘기지만
b = a;  //의도적으로 참조를 넘길 경우, 자바에서는 이 객체가 참조값인지 복사된 객체인지 알 수가 없다.
b.setCity("New");
~~~
<br>

결국 이러한 문제를 확실하게 막을 수 있는 방법은 `불변 객체`로 만드는 것이다.
<br><br>

> 불변 객체

**객체를 불변하게 만들면 값을 수정할 수 없으므로 부작용을 원천 차단할 수 있다.**
불변 객체의 값은 조회할 수 있지만 수정할 수 없다.

~~~java
@Embeddable
public class Address {
    private String city;
    
    protected Address() {}  //JPA에서 기본 생성자는 필수다.
  
    //생성자로 초기 값을 설정한다.
    public Address(String city) {this.city = city;}
  
    //Getter는 노출한다.
    public String getCity() {
        return this.city;
    }
    
    //Setter는 만들지 않는다.
}

Address address = member1.getHomeAddress();

//회원 1의 주소값을 조회해서 새로운 주소값을 생성
Address newAddress = new Address(address.getCity());
member2.setHomeAddress(newAddress);
~~~
<br><br>

### 값 타입의 비교
<hr>

자바가 제공하는 객체 비교는 2가지다.

* **동일성<sub>Identity</sub> 비교** : 인스턴스 참조 값을 비교, == 사용
* **동등성<sub>Equivalence</sub> 비교** : 인스턴스의 값을 비교, equals() 사용

~~~java
Address a = new Address("서울시", "종로구", "1번지");
Address b = new Address("서울시", "종로구", "1번지");
~~~
<br>

`a == b`는 false이다. 둘은 서로 다른 인스턴스이기 때문이다.
하지만, **값 타입은 비록 인스턴스가 달라도 그 안에 값이 같으면 같은 것으로 봐야 한다.**
따라서 값 타입을 비교할 때는 `a.equals(b)`를 사용해서 **동등성 비교**를 해야 한다.

값 타입의 `equals()`를 재정의하면 `hashCode()`도 재정의 하는것이 안전한다.
그렇지 않으면 해시를 사용하는 컬렉션(HashSet, HashMap)이 정상 동작하지 않는다.
<br><br>

### 값 타입 컬렉션
<hr>

값 타입을 하나 이상 저장하려면 컬렉션에 보관하고 `@ElementCollection`, `@CollectionTable` 어노테이션을 사용하면 된다.

~~~java
@Entity
public class Member {
    
    @Id @GeneratedValue
    private Long id;
    
    @Embedded
    private Address homeAddress;
    
    @ElementCollection
    @CollectionTable(name = "FAVORITE_FOODS", joinColumns = @JoinColumn(name = "MEMBER_ID"))
    @Column(name = "FOOD_NAME")
    private Set<String> favoriteFoods = new HashSet<String>();
    
    @ElementCollection
    @CollectionTable(name = "ADDRESS", joinColumns = @JoinColumn(name = "MEMBER_ID"))
    private List<Address> addressHistory = new ArrayList<Address>();
    
}

@Embeddable
public class Address {
    
    private String city;
    private String street;
    private String zipcode;
}
~~~
<br>

`favoriteFoods`는 기본값 타입인 String을 컬렉션으로 가진다.
관계형 데이터베이스의 테이블은 컬럼안에 컬렉션을 포함할 수 없다.
따라서, **별도의 테이블을 추가하고 `@CollectionTable`를 사용해서 추가한 테이블을 매핑해야 한다.**

`addressHistory`는 임베디드 타입인 Address를 컬렉션으로 가진다.
이것도 마찬가지로 별도의 테이블을 사용해야 한다.
<br><br>

> 값 타입 컬렉션 사용

~~~java
Member member = new Member();

//임베디드 값 타입
member.setHomeAddress(new Address("통영", "몽돌해수욕장", "660-123"));

//기본값 타입 컬렉션
member.getFavoriteFoods().add("짬뽕");
member.getFavoriteFoods().add("짜장");
member.getFavoriteFoods().add("탕수육");

//임베디드 값 타입 컬렉션
member.getAddressHistory().add(new Address("서울","강남","123-123"));
member.getAddressHistory().add(new Address("서울","강북","000-000"));

em.persist(member);
~~~
<br>
위의 코드로 호출되는 실제 쿼리는 다음과 같다.

~~~sql
INSERT INTO MEMBER (ID, CITY, STREET, ZIPCODE) VALUES (1, '통영', '몽돌해수욕장', '660-123')
INSERT INTO FAVORITE_FOODS (MEMBER_ID, FOOD_NAME) VALUES (1, '짬뽕')
INSERT INTO FAVORITE_FOODS (MEMBER_ID, FOOD_NAME) VALUES (1, '짜장')
INSERT INTO FAVORITE_FOODS (MEMBER_ID, FOOD_NAME) VALUES (1, '탕수육')
INSERT INTO ADDRESS (MEMBER_ID, CITY, STREET, ZIPCODE) VALUES 
(1, '서울', '강남', '123-123')
INSERT INTO ADDRESS (MEMBER_ID, CITY, STREET, ZIPCODE) VALUES 
(1, '서울', '강북', '000-000')
~~~
<br>

**값 타입 컬렉션은 영속성 전이(Cascade) + 고아 객체 제거(ORPHAN REMOVE) 기능을 필수로 가진다고 볼 수 있다.**

값 타입 컬렉션도 조회할 때 페치 전략을 선택할 수 있는데 `LAZY`가 기본이다.(`@ElementCollection(fetch = FetchType.LAZY)`)

값 타입 컬렉션이 수정되면 어떻게 동작하는지 알아보자.

~~~java
Member member = em.find(Member.class, 1L);

//1. 임베디드 값 타입 수정
member.setHomeAddress(new Address("새로운 도시", "신도시1", "123456"));

//2. 기본값 타입 컬렉션 수정
Set<String> favoriteFoods = member.getFavoriteFoods();
favoriteFoods.remove("탕수육");
favoriteFoods.add("치킨");

//3. 임베디드 값 타입 컬렉션
List<Address> addressHistory = member.getAddressHistory();
addressHistory.remove(new Address("서울","기존 주소","123-123"));
addressHistory.add(new Address("새로운도시","새로운 주소","123-456"));
~~~
<br>

1. 임베디드 값 타입 수정: homeAddress 임베디드 값 타입은 MEMBER 테이블과 매핑했으므로 MEMBER 테이블만 UPDATE한다.
사실 Member 엔티티를 수정하는 것과 같다.
2. 기본값 타입 컬렉션 수정: 탕수육을 치킨으로 변경하려면 탕수육을 제거하고 치킨을 추가해야 한다.
자바의 String 타입은 수정할 수 없다.
3. 임베디드 값 컬렉션 수정: 값 타입은 불변해야 한다. 따라서 컬렉션에서 기존 주소를 삭제하고 새로운 주소를 등록했다.
참고로 값 타입은 equals, hashCode를 꼭 구현해야 한다.
   
<br><br>

> 값 타입 컬렉션의 제약사항

* 값 타입 컬렉션에 보관된 값 타입들은 별도의 테이블에 보관되므로, 여기에 보관된 값 타입의 값이 변경되면 데이터베이스에 있는 원본 데이터를 찾기 어렵다.
* JPA는 값 타입 컬렉션에 변경 사항이 발생하면, 값 타입 컬렉션이 매핑된 테이블의 연관된 모든 데이터를 삭제하고, 현재 값 타입 컬렉션 객체에 있는 모든 값을 데이터베이스에 다시 저장한다.
* 값 타입 컬렉션을 매핑하는 테이블은 모든 컬럼을 묶어서 기본 키를 구성해야 한다.
  * 따라서 데이터베이스 기본 키 제약 조건으로 인해 컬럼에 null을 입력할 수 없고, 같은 값을 중복해서 저장할 수 없다.

위의 제약사항 때문에, 값 타입 컬렉션이 매핑된 테이블에 데이터가 많다면 값 타입 컬렉션 대신에 `일대다 관계` 관계를 고려해야 한다.
여기에 추가로 영속성 전이(Cascade) + 고아 객체 제거(ORPHAN REMOVE) 기능을 적용하면 값 타입 컬렉션처럼 사용할 수 있다.

~~~java
@Entity
public class Member {
    
    @Id @GeneratedValue
    private Long id;
    
    @Embedded
    private Address homeAddress;
    
    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
    @JoinCoumn(name = "MEMBER_ID")
    private List<AddressEntity> addressHistory = new ArrayList<AddressEntity>();
    
}

@Entity
public class AddressEntity {
    
    @Id @GeneratedValue
    private Long id;
    
    @Embedded Address address;
}

@Embeddable
public class Address {

  private String city;
  private String street;
  private String zipcode;
}
~~~
<br><br>

### 정리
<hr>

#### 엔티티 타입의 특징
* 식별자(@Id)가 있다
  * 엔티티 타입은 식별자가 있고 식별자로 구별할 수 있다.
  
* 생명 주기가 있다.
  * 생성하고, 영속화하고, 소멸하는 생명 주기가 있다.
  * em.persist(entity)로 영속화한다.
  * em.remove(entity)로 제거한다.
  
* 공유할 수 있다.
  * 참조 값을 공유할 수 있다. 이것을 공유 참조라 한다.
  * 예를 들어 회원 엔티티가 있다면 다른 엔티티에서 얼마든지 회원 엔티티를 참조할 수 있다.
  
#### 값 타입의 특징
* 식별자가 없다.
* 생명 주기를 엔티티에 의존한다.
  * 스스로 생명주기를 가지지 않고 엔티티에 의존한다. 의존하는 엔티티를 제거하면 같이 제거된다.
  
* 공유하지 않는 것이 안전한다.
  * 엔티티 타입과는 다르게 공유하지 않는 것이 안전하다. 대신에 값을 복사해서 사용해야 한다.
  * 오직 하나의 주인만이 관리해야 한다.
  * 불변<sub>Immutable</sub>객체로 만드는 것이 안전하다.
  
값 타입은 정말 값 타입이라 판단될 때만 사용해야 한다.
특히 엔티티와 값 타입을 혼동해서 엔티티를 값 타입으로 만들면 안 된다.
**식별자가 필요하고 지속해서 값을 추적하고 구분하고 변경해야 한다면 그것은 값 타입이 아닌 엔티티다.**
<br><br>

### 참고
<hr>
* 자바 ORM 표준 JPA 프로그래밍(김영한) - 9장_값 타입
* [require('develop')](https://m.blog.naver.com/adamdoha/222142595355)

