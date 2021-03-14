---
layout: post
title: Spring Cloud Config 설정(2)
excerpt: 지난 포스팅에서 각 서비스에 흩어진 속성들을 Cloud Config Server 한 곳으로 모아 관리하는 방법에 대해서 알아봤다. 하지만 이는 속성을 중앙집중식으로 바꿨을 뿐, 각자의 서비스에서 속성을 관리하던 이전 상태와 비교해서 큰 이점을 가질 수가 없다. 왜냐하면, 속성이 변경되게 될 경우 결국 변경될 속성을 갱신할 모든 서비스를 배포해야 되는 상황은 마찬가지이기 때문이다. 이번 포스팅에서는 속성 변경에 대한 갱신과 spring cloud bus를 사용하여 갱신행위를 최소화하는 방법에 대해서 알아보자.
categories: [Spring]
tags: [Spring Cloud Config, Spring Boot Actuator, Spring Cloud Bus, RabbitMQ]
---

### Spring Boot Actuator
<hr>
속성을 ConfigServer에 모으긴 했지만, 여전히 속성이 변경될 경우에는 각 서비스를 재배포해야 하는 불편함이 있다.
이 불편함을 해소하기 위해 Spring boot Actuator를 사용해보자.

Actuator는 간단히 말하자면 **실행 중인 스프링 애플리케이션의 내부 정보를 REST 엔드포인트와 JMX MBeans로 노출시키는 스프링 부트의 확장 모듈**이다.
자세한 내용은 추후의 포스팅에서 알아보도록 하겠다.

여러가지 Endpoint 중에서 일단 refresh endpoint 를 사용하여 실시간으로 속성을 갱신할 예정이다.

Actuator 사용을 위해 Config Server 와 Config Client 에 아래와 같이 '**spring-boot-starter-actuator**' 의존성을 추가한다.

~~~groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'org.springframework.cloud:spring-cloud-starter-config'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
~~~
<br>
Actuator는 기본적으로 /health와 /info 엔드포인트만 제공한다. 나머지는 비활성화 처리가 되어있다.
앞으로 사용할 엔드포인트가 'refresh' 이기 때문에 Config Client쪽 application.yml에 refresh 엔드포인트를 노출시켜주자.

~~~yaml
management:
  endpoints:
    web:
      exposure:
        include: refresh
~~~
<br>
위와 같이 노출시킬 엔드포인트까지 지정했으면, 이제 Config Client 쪽에서 갱신 대상이 되는 Bean에 **@RefreshScope** 을 추가해주자.
앞선 포스팅에서 Config Server로 부터 받아온 정보를 확인하는 Controller를 작성했는데 그 Controller로 예시를 들겠다.

~~~java
package com.configclient;

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
<br>
Config Server에서 변동이 생길 경우, refresh endpoint를 호출하면 '@RefreshScope'이 사용된 Bean을 대상으로 갱신된 속성정보를 세팅하여 Bean을 리로드시킨다.

~~~java
/*
 * Copyright 2012-2020 the original author or authors.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      https://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

package org.springframework.cloud.endpoint;

import java.util.Collection;
import java.util.Set;

import org.springframework.boot.actuate.endpoint.annotation.Endpoint;
import org.springframework.boot.actuate.endpoint.annotation.WriteOperation;
import org.springframework.cloud.context.refresh.ContextRefresher;

/**
 * @author Dave Syer
 * @author Venil Noronha
 */
@Endpoint(id = "refresh")
public class RefreshEndpoint {

	private ContextRefresher contextRefresher;

	public RefreshEndpoint(ContextRefresher contextRefresher) {
		this.contextRefresher = contextRefresher;
	}

	@WriteOperation
	public Collection<String> refresh() {
		Set<String> keys = this.contextRefresher.refresh();
		return keys;
	}

}
~~~
<br>
여담이지만 다른 엔드포인트와는 달리 위 코드에서 보듯이 refresh 엔드포인트는 actuator에 종속되어 있지 않다.
Spring Cloud context쪽에 관리가 되는데, actuator에서 별도로 추가한 형식인것 같다.
여튼, 이렇게하면 refresh 할 준비는 다 끝났다.

> 갱신 전 속성과 호출 결과

~~~yaml
server:
  name: project-local
  message: local 서버입니다.
~~~
<br>

~~~shell
$ curl -X GET "http://localhost:8080/test"
local 서버입니다.
~~~
<br>
이제 GIT 저장소의 속성을 아래와 같이 변경하고, refresh url을 호출해보자.

~~~yaml
server:
  name: project-local
  message: local 서버입니다. 오늘 첫번째 수정입니다.
~~~
<br>

~~~shell
$ curl -X POST http://localhost:8080/actuator/refresh
["config.client.version","server.message"]

$ curl -X GET "http://localhost:8080/test"
local 서버입니다. 오늘 첫번째 수정입니다.
~~~
<br>
refresh url은 '**/actuator/refresh**' 이다. 
이 URL이 spring boot나 cloud 버전에 따라 상이할 수도 있기 때문에, URL을 잘 모르겠다면 '/actuator'를 호출해서 사용가능한 엔드포인트를 확인해보자.
그리고 **'/actuator/refresh' URL은 POST메소드만 지원**한다. GET으로 호출하지 말자.

자 이제, actuator를 통해서 변경된 속성을 실시간으로 갱신할 수 있게 되었다! 하지만 아직도 불편한 점은 존재한다.
만약, 변경되어야할 클라이언트가 10개라면 10개 클라이언트에 모두 refresh URL을 호출해야한다.
클라이언트가 많으면 많을 수록 비효율적이다.
하지만 내가 불편하다고 생각되는 로직은 다른 프로그래머들도 당연히 불편하다고 생각하고, 이미 똑똑한 분들께서 해결책을 제시해주셨다.
<br><br>

### Spring Cloud Bus
<hr>
그렇다면, 늘어난 클라이언트 개수만큼 refresh endpoint를 호출해야하는 문제는 어떻게 개선할까?
물론 스크립트를 작성해서 한방에 돌리는 방법도 있을 것이다. 하지만, MSA 특성상 서비스는 늘어날 수도 있기 때문에 서비스가 추가될 때마다 스크립트에 추가해줘야하고
심지어 추가 작업을 깜빡하기라도 한다면 자칫 대형사고로 이어질 수 있을 것이다.

이럴 때 사용할 수 있는 방법이 바로 '**Spring Cloud Bus**' 이다.

<center>
<img src="{{ site.BASE_PATH }}/assets/images/2021/03/14/img1.png">
</center>
_<center> 출처 : 오늘도 MadPlay! - (https://madplay.github.io/post/spring-cloud-bus-example) </center>_
<br>
RabbitMQ 메시지 브로커를 사용해서 Config Client들을 subscriber로 등록하고, 이벤트가 발생하면 브로드캐스팅으로 전파하는 방식이다.

이렇게 하면, 단 하나의 Config Client에서 refresh를 해도 나머지 메시지브로커에 구독된 클라이언트들에게 이벤트가 발생하게 되어 설정값을 갱신할 수가 있다.

spring cloud bus는 메시지브로커로 RabbitMQ, Kafka, Redis 이 3가지를 지원하는데 RabbitMQ를 제외하고는 제대로 정리된 문서도 없고, 심지어 Redis는 이제 지원을 하지 않는것으로 보인다.
그래서 예제는 RabbitMQ로 진행하겠다.

먼저, local에 RabbitMQ를 설치하자. 편의를 위해서 Docker를 사용하겠다.

~~~shell
$ docker run -d --name rabbitmq \
    -p 5672:5672 -p 8087:15672 \
    -e RABBITMQ_DEFAULT_USER=kown8447 \
    -e RABBITMQ_DEFAULT_PASS=1234 \
    rabbitmq:management
~~~
<br>

이벤트는 Config Client들간에서만 일어날 것이기 때문에, client쪽에 아래와 같이 **'spring-cloud-starter-bus-amqp'** 의존성을 추가해준다.

~~~groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'org.springframework.cloud:spring-cloud-starter-config'
    implementation 'org.springframework.cloud:spring-cloud-starter-bus-amqp'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
~~~
<br>
그리고 나서 application.yml에 rabbitMQ 관련 접속 정보와 **bus-refresh 엔드포인트를 추가**하자.

~~~yaml
server:
  port: 8080

spring:
  application:
    name: demo
  profiles:
    active: local
  config.import: configserver:http://localhost:8888

  rabbitmq: # RabbitMQ 관련 설정
    host: localhost
    port: 5672
    username: kown8447
    password: 1234

management:
  endpoints:
    web:
      exposure:
        include: refresh, bus-refresh
~~~
<br>

이제 준비는 끝났다. 서버와 클라이언트를 실행한다음, RabbitMQ 웹 관리페이지에 접속해보자.

<center>
<img src="{{ site.BASE_PATH }}/assets/images/2021/03/14/img3.png">
</center>
_<center>springCloudBus Exchange가 추가된걸 확인할 수 있다.</center>_
<br><br>

<center>
<img src="{{ site.BASE_PATH }}/assets/images/2021/03/14/img2.png">
</center>
_<center>3개의 클라이언트가 커넥션을 가지고 있다.</center>_

현재 예제에서는 같은 모듈을 각각 다른 포트(8080,8081,8082)로 사용하여 3개의 모듈로 bus에 연결했다.
이제 각각의 속성값을 확인해보자.

~~~shell
$ curl -X GET "http://localhost:8080/test"
local 서버입니다. 오늘 첫번째 수정입니다.

$ curl -X GET "http://localhost:8081/test"
local 서버입니다. 오늘 첫번째 수정입니다.

$ curl -X GET "http://localhost:8082/test"
local 서버입니다. 오늘 첫번째 수정입니다.
~~~
<br>

3개의 클라이언트가 같은 값을 표시하고 있다. 이제 GIT 저장소의 속성값을 변경해보자.

~~~yaml
server:
  name: project-local
  message: local 서버입니다. 오늘 두번째 수정입니다.
~~~
<br>
이제 실제로 Bus를 통해서 한번에 갱신이 이뤄지는지 확인해보자.
bus를 통한 refresh url은 **'/actuator/busrefresh'** 이다. 이 URL 또한 버전마다 다를 수 있으니, 사용전에 '/actuator'로 확인해보길 바란다.

~~~shell
$ curl -X POST http://localhost:8080/actuator/busrefresh

$ curl -X GET "http://localhost:8080/test"
local 서버입니다. 오늘 두번째 수정입니다.

$ curl -X GET "http://localhost:8081/test"
local 서버입니다. 오늘 두번째 수정입니다.

$ curl -X GET "http://localhost:8082/test"
local 서버입니다. 오늘 두번째 수정입니다.
~~~
<br>
당연하게도, 한번의 호출로 3개의 클라이언트가 모두 잘 갱신되는 것을 확인할 수 있다.

그런데 이정도 사양으로도 잘 사용할 수 있을거 같긴하지만, bus-refresh 엔드포인트를 수동으로 호출해야 함에는 변함이 없다.
이전 상황과 마찬가지로 작업자가 속성 파일을 수정한 이후에 bus-refresh 엔드포인트를 호출하지 않으면 소용이 없다.
속성 파일을 수정함과 동시에 자동으로 엔드포인트를 호출할 수 있으면 참 편하지 않을까?
<br><br>

### Spring Cloud Config Monitor
<hr>
위의 궁금증을 해소할 수 있는 것이 바로 **'Spring Cloud Config Monitor'** 이다.
Spring Cloud Cconfig monitor는 **webhook 이벤트를 받아서 최신 속성 정보를 가져오고 이를 Spring Cloud Bus에 전달**한다.

바로 설정에 들어가보자. Config Server도 이제 메시지 브로커를 사용하기 때문에, 'spring-cloud-starter-bus-amqp' 의존성을 추가하고 당연히
'/monitor' 엔드포인트를 사용하기 위해서 **'spring-cloud-config-monitor'** 의존성도 함께 추가해주자.

~~~groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'org.springframework.cloud:spring-cloud-config-server'
    implementation 'org.springframework.cloud:spring-cloud-starter-bus-amqp'
    implementation 'org.springframework.cloud:spring-cloud-config-monitor'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
~~~
<br>
서버쪽에서도 버스를 사용하기 위해 application.yml에 **spring.cloud.bus.enabled** 속성을 추가해주자.
이 속성은 기본적으로 false이고, **false일 경우 Git Push에 의한 이벤트를 처리할 수가 없다.**

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
          uri: https://github.com/kown8447/springCloudCofigDemo
    bus:
      enabled: true

  rabbitmq: # RabbitMQ 관련 설정
    host: localhost
    port: 5672
    username: kown8447
    password: 1234
~~~
<br>
어플리케이션단의 설정을 끝났다. 이제 남은 일은 Github 저장소에 Webhook 이벤트를 추가하는 것인데, 나는 local 환경에서 테스트할거기 때문에 이는 생략하고 webhook 이벤트를 수동으로 호출할 것이다.

~~~shell
$ curl -X GET "http://localhost:8080/test"
local 서버입니다. 오늘 두번째 수정입니다.

$ curl -X GET "http://localhost:8081/test"
local 서버입니다. 오늘 두번째 수정입니다.

$ curl -X GET "http://localhost:8082/test"
local 서버입니다. 오늘 두번째 수정입니다.
~~~
<br>
webhook 호출하기전 각 클라이언트들의 속성값을 출력해본 결과이다.

다음으로 GIT 저장소의 설정값을 변경해보자.

~~~yaml
server:
  name: project-local
  message: local 서버입니다. 오늘 세번째 수정입니다.
~~~
<br>
설정값 변경 이후, 이제 webhook url을 호출하고 다시한번 속성값을 출력해보자.

~~~shell
$ curl -X POST "http://localhost:8888/monitor" \
then cmdand> -H "Content-Type: application/json" \
then cmdand> -H "X-Event-Key: repo:push" \
then cmdand> -H "X-Hook-UUID: webhook-uuid" \
then cmdand> -d '{"push": {"changes": []} }'
["*"]

$ curl -X GET "http://localhost:8080/test"
local 서버입니다. 오늘 세번째 수정입니다.

$ curl -X GET "http://localhost:8081/test"
local 서버입니다. 오늘 세번째 수정입니다.

$ curl -X GET "http://localhost:8082/test"
local 서버입니다. 오늘 세번째 수정입니다.
~~~
<br>

모든 클라이언트에서의 값이 변경되어 출력되는 것을 확인할 수 있다.
이제 저장소의 변경만으로도 각 클라이언트의 속성 변경까지 알아서 자동화가 된 것이다.
처음 포스팅때의 상황에 비해서 아주 심플하고 스마트한 상황으로 바뀐 것이다.

그런데, 갑자기 문득 이런 생각이 들었다.
만약 '/monitor' 엔드포인트가 actuator에서 관리되지 않고 Config Server에 cloud-config-monitor가 관리하는 것이고,
앞서 언급한것 처럼 refresh 엔드포인트도 actuator에서 관리하지 않는다면, actuator는 필요 없지 않을까?

그래서 실험해봤다. 서버와 클라이언트에 의존성으로 있는 'spring-boot-starter-actuator' 을 제거하고, 
클라이언트에 있는 management.endpoints.web.exposure.include 설정도 삭제해버렸다.

그러고 나서 동일하게 저장소 속성을 변경하고 monitor를 호출하니 정상적으로 다 갱신이 되었다.

~~~java
/*
 * Copyright 2015-2019 the original author or authors.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      https://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

package org.springframework.cloud.config.monitor;

import java.util.Collections;
import java.util.HashMap;
import java.util.LinkedHashSet;
import java.util.List;
import java.util.Map;
import java.util.Set;

import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;

import org.springframework.cloud.bus.event.RefreshRemoteApplicationEvent;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.context.ApplicationEventPublisherAware;
import org.springframework.http.HttpHeaders;
import org.springframework.http.MediaType;
import org.springframework.util.StringUtils;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestHeader;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

/**
 * HTTP endpoint for webhooks coming from repository providers.
 *
 * @author Dave Syer
 *
 */
@RestController
@RequestMapping(path = "${spring.cloud.config.monitor.endpoint.path:}/monitor")
public class PropertyPathEndpoint implements ApplicationEventPublisherAware {

	private static Log log = LogFactory.getLog(PropertyPathEndpoint.class);

	private final PropertyPathNotificationExtractor extractor;

	private ApplicationEventPublisher applicationEventPublisher;

	private String busId;

	public PropertyPathEndpoint(PropertyPathNotificationExtractor extractor, String busId) {
		this.extractor = extractor;
		this.busId = busId;
	}

	/* for testing */ String getBusId() {
		return this.busId;
	}

	@Override
	public void setApplicationEventPublisher(ApplicationEventPublisher applicationEventPublisher) {
		this.applicationEventPublisher = applicationEventPublisher;
	}

	@RequestMapping(method = RequestMethod.POST)
	public Set<String> notifyByPath(@RequestHeader HttpHeaders headers, @RequestBody Map<String, Object> request) {
		PropertyPathNotification notification = this.extractor.extract(headers, request);
		if (notification != null) {

			Set<String> services = new LinkedHashSet<>();

			for (String path : notification.getPaths()) {
				services.addAll(guessServiceName(path));
			}
			if (this.applicationEventPublisher != null) {
				for (String service : services) {
					log.info("Refresh for: " + service);
					this.applicationEventPublisher
							.publishEvent(new RefreshRemoteApplicationEvent(this, this.busId, service));
				}
				return services;
			}

		}
		return Collections.emptySet();
	}

	@RequestMapping(method = RequestMethod.POST, consumes = MediaType.APPLICATION_FORM_URLENCODED_VALUE)
	public Set<String> notifyByForm(@RequestHeader HttpHeaders headers, @RequestParam("path") List<String> request) {
		Map<String, Object> map = new HashMap<>();
		String key = "path";
		map.put(key, request);
		return notifyByPath(headers, map);
	}

	private Set<String> guessServiceName(String path) {
		Set<String> services = new LinkedHashSet<>();
		if (path != null) {
			String stem = StringUtils.stripFilenameExtension(StringUtils.getFilename(StringUtils.cleanPath(path)));
			// TODO: correlate with service registry
			int index = stem.indexOf("-");
			while (index >= 0) {
				String name = stem.substring(0, index);
				String profile = stem.substring(index + 1);
				if ("application".equals(name)) {
					services.add("*:" + profile);
				}
				else if (!name.startsWith("application")) {
					services.add(name + ":" + profile);
				}
				index = stem.indexOf("-", index + 1);
			}
			String name = stem;
			if ("application".equals(name)) {
				services.add("*");
			}
			else if (!name.startsWith("application")) {
				services.add(name);
			}
		}
		return services;
	}

}
~~~
<br>
역시 '/monitor' 엔드포인트는 cloud-config-monitor 에서 관리하고 있었다. 설정 갱신도 '/refresh' 엔드포인트를 직접호출하지 않는것으로 보아 별도의 리스너가 동작하는 것으로 보인다.

최종적으로는 'spring-cloud-config-server', 'spring-cloud-starter-config', 'spring-cloud-starter-bus-amqp', 'spring-cloud-config-monitor' 이 의존성들로 동작하게 구성하였다.
<br><br>

### 정리
<hr>
cloud-config를 통해서 설정을 중앙집중식으로 변경하고, actuator를 통해서 endpoint를 제공하여 실시간 속성 갱신을 하였다.
또한, cloud-bus를 통해서 갱신 행위를 한번만 할 수 있도록 하였고 마지막으로 cloud-config-monitor를 사용해서 webhook 이벤트 발생으로 인한 저장소 속성 변경 시 자동으로 엔드포인트 호출하게 하여 갱신할 수 있도록 해봤다.

하지만 아직도 해결해야 할 문제들은 남아있는것 같다. 
속성에서 노출되는 민감한 값 및 endpoint의 보안문제, config-server가 down되었을 때의 가용성은 어떻게 가져갈지 등을 추가적으로 고려해봐야할 것 같다.
<br><br>

### 참고
* Spring in Action 제 5판 - Chapter16. 스프링 부트 액추에이터 사용하기
* [Circlee7](https://circlee7.medium.com/spring-cloud-config-%EC%A0%95%EB%A6%AC-1-%EB%B2%88%EC%99%B8-da81585400fa)
* [오늘도 MadPlay!](https://madplay.github.io/post/spring-cloud-bus-example)