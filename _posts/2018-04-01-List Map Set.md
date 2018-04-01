---
layout: post
title: List, Map, Set 차이
categories: [Data Structure]
tags: [List, Map, Set]
---

#Java Collection Framework

면접도 대비하고, 자바 기본도 다시 익힐겸 정리하자.

자바에서는 데이터를 저장하는 기본적인 자료구조를 **Java Collection Framework(JCF)** 로 사용자에게 편리하게 제공한다.

JCF는 크게 Collection 과 Map 두가지로 나눠지며 대략적인 구조는 아래와 같다.

----
* ## Collection
  * Set
    * 순서가 없는 데이터의 집합, 중복을 허용하지 않는다.
    * 구현 클래스 : HashSet, LinkedHashSet, TreeSet

  * List
    * 순서가 있으며, 중복을 허용한다.
    * 구현 클래스 : ArrayList, Vector, LinkedList

* ## Map
  * Key, Value 쌍으로 이뤄진 데이터 집합으로 순서는 없고, Key는 중복을 허용하지 않는다.
  * 구현 클래스 : HashMap, TreeMap, LinkedHashMap

----

## 정리

1. ArrayList 와 LinkedList 의 차이점
    * ArrayList는 배열을 기반으로 하기 때문에 배열은 한번 생성될 경우 그 길이를 변경할 수 없으므로 생성과 복사 과정이 복잡하다.



ArrayList | LinkedList
--------- | ----------
저장소의 용량을 늘리는 과정에서 많은 시간이 소요된다. | 저장소의 용량을 늘리는 과정이 간단한다.
데이터의 삭제에 필요한 연산과정이 매우 길다. | 데이터의 삭제가 매우 간단
데이터의 참조가 용이 | 데이터의 참조가 다소 불편




2. HashSet 은 hashCode 알고리즘을 적용한 것
    * HashSet 클래스는 해시 알고리즘을 적용하여 데이터를 저장하고 검색한다.
    * 매우 빠른 검색속도를 자랑함 -> 매우 빠른 데이터의 저장


3. TreeSet 은 데이터를 정렬된 상태로 저장하는 자료구조
    * TreeSet 은 생성 시 정렬기준을 함께 생성하여 그 결과를 바탕으로 데이터를 저장
    * TreeSet<String> tSet = new TreeSet<String>(new 비교연산())


4. HashMap 은 해시 알고리즘을 바탕으로 '매우 빠른 검색속도' 를 지원, TreeMap은 '트리' 자료구조를 기반으로 구현이 되어 있기 때문에 데이터는 정렬된 상태로 저장된다.



왜 컬렉션이 프레임 워크냐 라고 묻는다면...  *인터페이스 구조를 기반으로 클래스들이 정의되어 있기 때문* 이라고 한다.

참조 블로그
[공대인들이 직접쓰는 컴퓨터공부방](http://hackersstudy.tistory.com/26)
