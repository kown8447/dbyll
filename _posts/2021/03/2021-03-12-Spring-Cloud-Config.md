---
layout: post
title: Spring Cloud Config 설정(1)
excerpt: 회사에서 드디어 MSA 프로젝트를 시작한다. 이 플젝에서 내가 맡은 역활은 Configuration Server 를 설계하고 구축하는 것이다. 프로젝트 전체가 Spring으로 진행될 거라 Spinrg Cloud Config 를 사용하면 좋겠다고 생각했고, 그래서 여러가지 리서치한 결과를 기록하고자 한다.
categories: [Spring]
tags: [Spring Cloud Config]
---

### Config Server란?
<hr>

애플리케이션의 구성 속성들은 각 애플리케이션에 맞게 구성될 수 있으며, 이때는 배포되는 애플리케이션 패키지에 있는 application.properites나 application.yml 파일에 구성 속성을 지정한다.

그러나 마이크로서비스에서 애플리케이션을 구축할 때 이러한 방식은 여러 마이크로서비스에 걸쳐 동일한 구성 속성이 적용되므로 문제가 될 수 있다.

**Config Server는 애플리케이션의 모든 마이크로서비스에 대해 중장 집중식의 구성을 제공한다.**
따라서 Config Server를 사용하면 애플리케이션의 모든 구성을 한 곳에서 관리할 수 있다.
<br><br>

### Config Server 가 필요한 이유
<hr>
일반적인 애플리케이션의 경우, 자바 시스템 속성이나 운영체제의 환경 변수에 구성 속성을 설정하는 경우는 해당 속성의 변경으로 인해 애플리케이션이 다시 시작되어야 한다.
속성만 변경하기 위해 애플리케이션을 재배포하거나 재시작한다는 것은 매우 불편하며, 최악의 경우 애플리케이션에 결함이 생길 수도 있다.
게다가 다수의 배포 인스턴스에서 속성을 관리해야 하는 마이크로서비스 기반 애플리케이션의 경우는 실행 중인 애플리케이션의 모든 서비스 인스턴스에 동일한 변경을 적용하는 것이 불합리하다.

또한, 데이터베이스 비밀번호와 같은 일부 속성들은 보안에 민감한 값을 갖는다.
이런 속성 값은 각 애플리케이션의 속성에 지정될 때 암호화될 수 있지만, 사용 전에 해당 속성 값을 복호화하는 기능이 애플리케이션에 포함되어야 한다.
그렇지만 어떤 구성 속석들은 애플리케이션 개발자조차도 접근할 수 없도록 해야 하므로 이런 속성들을 운영체제의 환경 변수에 설정하는 것은 바람직하지 않다.
<br><br>

> 중앙 집중식으로 구성 관리할 경우의 장점

* 구성이 더 이상 애플리케이션 코드에 패키징되어 배포되지 않는다.
따라서 애플리케이션을 다시 빌드하거나 배포하지 않고 구성을 변경하거나 원래 값으로 환원할 수 있다. 또한, 애플리케이션을 다시 시작하지 않아도 실행 중에 구성을 변경할 수 있다.
  
* 공통적인 구성을 공유하는 마이크로서비스가 자신의 속성 설정으로 유지,관리하지 않고도 동일한 속성들을 공유할 수 있다.
그리고 속성 변경이 필요하면 한 곳에서 한 번만 변경해도 모든 마이크로서비스에 적용할 수 있다.
  
* 보안에 민감한 구성 속성은 애플리케이션 코드와는 별도로 암호화하고 유지,관리할 수 있다.
그리고 복호화된 속성 값을 언제든지 애플리케이션에서 사용할 수 있으므로 복호화를 하는 코드가 애플리케이션에 없어도 된다.
  
<br><br>

### Config Server 구현
<hr>

일단 Config Server에 필요한 의존성을 아래와 같이 추가해주자
~~~groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'org.springframework.cloud:spring-cloud-config-server'
    implementation 'org.springframework.cloud:spring-cloud-starter-bus-amqp'
    implementation 'org.springframework.cloud:spring-cloud-config-monitor'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
~~~

_<center>actuator 와 bus-amqp, config-monitor 는 실시간 속성 갱신을 위함인데 이는 추후에 설명하도록 하겠다.</center>_

구성 서버의 스타터 의존성이 지정되었으므로 이제 구성 서버를 활성화시키자. 
애플리케이션이 시작되는 부트스트랩 클래스인 ConfigServerApplication에 **@EnableConfigServer** 애노테이션을 추가하자.
이름이 암시하듯, 이 애노테이션은 애플리케이션이 실행될 때 구성 서버를 활성화하여 자동-구성한다.

~~~java
package com.configserver.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.config.server.EnableConfigServer;

@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }

}
~~~
<br>

다음으로는 Config Server가 처리할 구성 속성들이 있는 곳(repository)를 알려주어야 함으로, application.yml 또는 application.properties 에 속성을 입력한다.
(여기서는 Git을 사용하겠다.)
~~~yaml
server:
  port: 8888
  profiles:
    active: local

spring:
  cloud:
    config:
      server:
        git:
          default-label: main
          uri: {깃 저장소}
~~~
<br>

스프링 부트 웹 애플리케이션인 Config Server는 기본적으로 8080 포트를 리스닝한다. 
하지만, Config Client 가 Config Server로부터 데이터를 가져올 때 사용하는 기본 포트 번호가 8888이기 때문에 port 를 변경해준다.

이제 기본적인 서버 설정은 마쳤고, 구성 속성 repository(여기에서는 git)에 구성 파일을 업로드 한다.

<center>
<img src="{{ site.BASE_PATH }}/assets/images/2021/03/12/img1.png">
</center>

_<center>local 과 dev 환경을 나누었다</center>_

속성 파일 안에는 아래와 같이 내용을 작성해 두었다.

~~~yaml 
#demo-local.yml
server:
  name: project-local
  message: local 서버입니다.

#demo-dev.yml
server:
  name: project-local
  message: dev 서버입니다.
~~~
<br>

이제 아래의 url 을 호출해보자

~~~
curl -X GET "http://localhost:8888/demo/local"
~~~
<br>

위의 URL에서 'demo' 는 ConfigServer에 요청하는 애플리케이션 이름이다. 
추후에 다시 말하겠지만, 이 속성은 ConfigClient의 application.yml에서 spring.application.name 으로 지정할 수 있다.
그리고 뒤의 'local'은 요청하는 애플리케이션에 활성화된 스프링 프로파일의 이름이다.
Config Client의 프로파일을 'dev' 로 활성화시키면 config server 에는 'http://localhost:8888/demo/dev' 로 요청할 것이다.
사실 위의 URL 에서 추가적으로 label 과 branch를 지정해 줄 수 있는데, 이것을 지정하지 않으면 'master' 브랜치가 기본이 된다.
하지만, 나는 위에서 ConfigServer의 application.yml 에서 'spring.cloud.config.server.git.default-label' 의 속성 값을 'main' 으로 주었기 때문에,
repository의 main 브랜치를 기본적으로 탐색할 것이다.

(search-paths 속성을 지정하여 Git 하위 경로로 구성 속성을 저장할 수도 있지만, 여기서는 자세히 다루지 않겠다.)

URL 호출결과는 아래와 같다.

~~~json
{
  "name": "demo",
  "profiles": [
    "local"
  ],
  "label": null,
  "version": "5077e66dcfeb5e2c967c03eb756bda62505bca3d",
  "state": null,
  "propertySources": [
    {
      "name": "https://github.com/kown8447/springCloudCofigDemo/demo-local.yml",
      "source": {
        "server.name": "project-local",
        "server.message": "local 서버입니다."
      }
    }
  ]
}
~~~
<br><br>

### Config Client 구현
<hr>
클라이언트 구현은 서버에 비해서 훨씬 간단하다.
먼저 아래와 같이 의존성을 추가해주자.

~~~ groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'org.springframework.cloud:spring-cloud-starter-config'
    implementation 'org.springframework.cloud:spring-cloud-starter-bus-amqp'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
~~~
<br>

다음으로 application.yml에 아래의 속성을 추가하자.
~~~yaml
spring:
  application:
    name: demo
  profiles:
    active: local
  config:
    uri: http://localhost:8888
~~~
<br>
앞서 언급한것처럼, 기본적으로 ConfigClient는 localhost의 8888 포트를 구성서버 실행중인것으로 간주한다.
하지만 실제 환경에서는 도메인이나 포트가 변경될 수 있으니 위의 설정처럼 spring.cloud.config.uri 속성을 설정하여 구성 서버의 위치를 알려주자.
또한, 구성 서버에 애플리케이션과 프로파일을 알려주어 URL을 맵핑할 수 있도록 spring.application.name과 spring.profiles.active를 지정하면 된다.

(아, 참고로 많은 blog에서 config client 설정을 bootstrap.yml 로 하고 있는데, 나도 처음에 이걸로 설정했다가 속성을 받아오지 못해서 한참동안 삽질을 했다.
알고 보니, spring cloud config Ilford 버전부터는 bootstrap.yml 을 지원하지 않는다고 하니 이점 참고하자. [Client 의 bootstrap.yml](https://hongdori2.tistory.com/112))

이제 Config Server에서 제대로 속성을 가져왔는지 확인할 수 있는 Test용 Controller를 만들어보자.

~~~java
package com.configclient.demo;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RefreshScope
public class ConfigTestController {
    @Value("${server.message}")
    private String str;

    @GetMapping("/test")
    public String test() {
        return str;
    }
}
~~~
_<center>@RefreshScope에 대해서는 다음 포스팅에서 알아보자</center>_
<br>

이제 준비는 다 끝났다. Config 클라이언트를 실행하고 아래의 테스트용 URL 을 호출해 보자.

~~~
❯ curl -X GET "http://localhost:8080/test"
local 서버입니다.
~~~

_<center>git에 올려진 속성 값을 Config Server를 통해 받아와서 출력</center>_
<br><br>

### 정리
<hr>
이제 각 애플리케이션의 속성을 Config Server에 올려 중앙 집중식으로 관리할 수 있게 되었다.
하지만 이 상태로는 속성이 변경될 경우, 애플리케이션을 재배포해야하기 때문에 아직까지는 큰 이점을 볼 수가 없다.
다음 시간에는 repository 에서 변경된 속성을 실시간으로 클라이언트에 갱신하는 방법을 알아보고, 
추가로 여러대의 클라이언트에 한번의 행위로 갱신을 전파하는 방법에 대해서도 알아보도록 하겠다.
<br><br>

### 참고
* Spring in Action 제 5판 - Chapter14. 클라우드 구성 관리
* [내 마음대로 적는 공간](https://hongdori2.tistory.com/)