> 실패와 지연 처리하기

# 15.1 서킷 브레이커 이해하기

서킷 브레이커 패턴 (circuit breaker pattern) 

- 우리가 작성한 코드가 실행에 실패하는 경우 안전하게 처리되도록 한다.
- 마이크로 서비스에서 더 빛을 발하는 패턴
- **한 마이크로 서비스의 실패가 다른 마이크로서비스의 연쇄적인 실패로 확산이 되는 걸 방지해야 하기 때문에.**
- 문제가 발생할 시 차단기를 내려 전류를 막는 실제 회로 (circuit)과 유사한 형태
- 어떤 이유로 메서드의 실행이 실패하면 (실행횟수나 시간 등 미리 설정해둔 한계값을 초과) 서킷 브레이커가 개방이 되고(차단기를 내리고) 더 이상 실패한 메소드의 호출이 더이상 이뤄지지 않는다.
- 이때 풀백을 제공해 자체적으로 실패를 처리한다.

![https://drek4537l1klr.cloudfront.net/walls7/Figures/15fig01.jpg](https://drek4537l1klr.cloudfront.net/walls7/Figures/15fig01.jpg)

- closed가 서킷상태의 시작. (서킷브레이커로 보호되고 있는 메서드)
- 실행에 성공하면 Success라 닫힘 상태를 유지
- 실패할 시 지금 실행되고 있는 실패한 메서드 대신 풀백 메서드가 호출된다. 상태는 open으로 바뀐다
- 만약 시간이 지나는 등의 조건으로 인해 서킷이 half-open 상태가 되면 실패했던 메서드를 다시 호출해본다.
- 성공하면 closed, 실패하면 또 open 상태가 된다.

**서킷 브레이커를 더 강력한 형태의 try-catch 라고 생각하면 좋다.** 닫힘 상태는(잘 실행이 됨) try와 유사하고, 실패 후 호출되는 풀백 메서드는 catch 와 유사한다. try-catch가 한 번의 실패로 실행이 된다면 서킷 브레이커는 설정한 한계값을 넘어서 실패했을 경우 풀백 메서드를 호출한다. 

서킷 브레이커는 메서드 단위로 적용이 된다. 따라서 하나의 맘이크로 서비스에 많은 서키시 브레이커를 둘 수 있다. 때문에 코드의 어디에 서킷 브레이커를 선언할지 결정할 때는 실패의 대상이 되는 메서드(실패할 위험이 있는)를 식별하는 게 중요하다. 후보로 뽑을 수 있는 메서드는 

1. REST를 호출하는 메서드 (원격 서비스로 인해 실패)
2. 데이터베이스 쿼리를 수행하는 메서드 (데이터베이스가 무반응 상태가 될 수 있거나 스키마 변경으로 인해 실패)
3. 느리게 실행될 가능성이 있는 메서드 (필시 실패하는 메서드는 아지지만 너무 오랫동안 실행되면 비정상적 상태를 고려)

1과 2는 서킷 브레이커로 실패 처리를 할 수 있지만 마지막 유형은 메서드의 실패보다는 지연(latency)가 문제다. 그러나 이 경우에도 서킷 브레이커의 장점을 살릴 수 있다. **지연은 마이크로서비스 관점에서도 매우 중요한데, 지나치게 느린 메서드가 상위 서비스에 연쇄적인 지연을 유발하면 더 큰 문제가 되기 때문이다.** 

책에서 사용할 서킷 브레이커 패턴은 NetFlix의 Hystrix로, 자바로 구현이 되었고 대상 메서드가 실패할 때 폴백 메서드를 호출하는 동시에 대상 메서드가 얼마나 실패를 하는지 추적하고, 실패율이 한계값을 초과하면 풀백 메서드를 호출한다. 서킷브레이커를 선언할 때는 @HystrixCommand 를 선언하고 폴백 메서드만 제공해주면 된다.

[Netflix, Inc.](https://github.com/netflix)

# 15.2 서킷 브레이커 선언하기

1. pom.xl에 dependency 추가 

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
```

2. 스프링 클라우드 버전 지정.  

```xml
<properties>
  ...
  <spring-cloud.version>Hoxton.SR3</spring-cloud.version>
</properties>

...

<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-dependencies</artifactId>
      <version>${spring-cloud.version}</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

3. application에 Hystrix 활성화 해주기 

```java
@SpringBootApplication
@EnableHystrix
public class IngredientServiceApplication {
    ...
}
```

4. @HystrixCommand 를 실제 메서드에 사용해보기 

```java
@HystrixCommand(fallbackMethod="getDefaultIngredients")
public Iterable<Ingredient> getAllIngredients() {
  ParameterizedTypeReference<List<Ingredient>> stringList =
      new ParameterizedTypeReference<List<Ingredient>>() {};
  return rest.exchange(
      "http://ingredient-service/ingredients", HttpMethod.GET,
      HttpEntity.EMPTY, stringList).getBody();
}
```

- RestTemplate을 사용해서 식자재 서비스로부터 Ingredient 객체들이 저장된 컬렉션을 가져오는 메서드 getAllIngredients에 @HystrixCommand를 적용했다.
- 실제로 통신을 하는 exchange() 가 호출 실패의 잠재적인 원인이다.
- 어떠한 이유로 호출이 실패한다면 RestClientException (unchecked, run-time exception)이 발생한다.
- RestTemplate이 try-catch를 하지 않았기 때문에 여기서 직접 처리를 해줘야 하는데 현재는 하지 않은 상태로, 이대로 실패한다면 호출 스택의 상위 호출자들에게 계속 에러가 전달이 된다.
- 계속 전달되는 Unchecked예외는 골칫거리이고, 특히 조밀하게 연결이 되어있는 마이크로서비스에서 더 심하다.
- 마이크로서비스에서 생긴 에러는 다른 곳에 전파하지 않고 해당 마이크로서비스안에 남겨야 한다(마이크로서비스의 Vegas Rule)
- 이때 getAllIngredients에 서킷브레이커를 선언해주면 베가스 룰을 지킬 수 있다.
- HystrixCommand 어노테이션을 선언 후, fallback 메서드를 getDefaultIngredients로 지정

5. 실패할 때 실행이 될 폴백 메서드 만들기 

```java
private Iterable<Ingredient> getDefaultIngredients() {
  List<Ingredient> ingredients = new ArrayList<>();
  ingredients.add(new Ingredient(
        "FLTO", "Flour Tortilla", Ingredient.Type.WRAP));
  ingredients.add(new Ingredient(
        "GRBF", "Ground Beef", Ingredient.Type.PROTEIN));
  ingredients.add(new Ingredient(
        "CHED", "Shredded Cheddar", Ingredient.Type.CHEESE));
  return ingredients;
}
```

- unchecked 예외가 발생해서 getAllIngredients를 벗어나면 서킷브레이커가 그 예외를 자바아서 getDefaultIngredients를 호출한다.
- 폴백메서드는 원래의 의도대로 실패한 메서드가 실행이 불가능할 때 대비하는 의도로 만드는 것이 제일 좋다. (stand-alone 형태, dummy 데이터라도 채워서 줘야한다)
- 폴백 메서드는 원래의 메서드와 시그니처가 같아야한다. 파라메터 종류, 갯수, 리턴값이 같아야 한다.
- dummy 값을 채워주는 이 경우는 폴백 메서드가 실패할 일이 없지만 다르게 구현을 한다면 이 메서드 또한 잠재적인 실패가 될 수 있다.
- 이때 이 폴백 메서드에 HytrixCommand를 달고 폴백 메서드를 또 지정을 할 수 있다. 서킷 브레이커는 연쇄적으로 사용이 가능하다.
- **단, 폴백 스택의 제일 밑에는 실행에 실패하지 않아 서킷 브레이커가 필요없는 확실한 메서드가 이어야 한다.**

# 15.2.1 지연 시간 줄이기

서킷 브레이커는 메서드의 실행이 끝나고 복귀하는 시간이 너무 오래 걸릴 경우 타임아웃을 사용해 지연시간을 줄일 수 있다. HystrixCommand를 지정하면 디폴트로 메서드는 1초후에 타임아웃이 되고 폴백메서드가 호출된다. 이유를 불문하고 타임아웃이 되면 무조건 풀백메서드를 호출한다고 보면 된다. 

1초의 타임아웃은 대부분의 경우에 맞는 합리적인 값이지만 Hystrix는 commandProperties를 통해 속성을 지정하여 타임아웃을 변경할 수 있다. commandProperties는 설정될 속성의 이름과 값을 지정하는 하나 이상의 @HystrixProperty 어노테이션을 저장한 배열이다. 

(어노테이션을 이용해서 다른 어노테이션의 속성을 설정한다는 성격이 다소 특이하다) 

```java
@HystrixCommand(
    fallbackMethod="getDefaultIngredients",
    commandProperties={
        @HystrixProperty(
            name="execution.isolation.thread.timeoutInMilliseconds",
            value="500")
    })
public Iterable<Ingredient> getAllIngredients() {
  ...
}
```

```java
@HystrixCommand(
    fallbackMethod="getDefaultIngredients",
    commandProperties={
        @HystrixProperty(
            name="execution.timeout.enabled",
            value="false")
    })
public Iterable<Ingredient> getAllIngredients() {
  ...
}
```

- execution.isolation.tread.timeoutMilliseconds : 타임아웃 속성 설정, 밀리세컨드로 되어있어 타임아웃을 0.5 초로 줄이고자 한다면 500으로 설정하면 된다.
- execution.timeout.enabled: 타임아웃 설정이 필요없다면 여기를 false로 해주면 된다.
- 이처럼 타임아웃을 사용하지 않으면 메서드가 지연되어 10분 30분이 지나도 머물러있기 때문에 연쇄 지연 효과가 발생하므로 주의해야한다.

# 15.2.2 서킷 브레이커 한계값 관리하기

서킷브레이커로 감싸진 메서드는 기본으로 10초 동안에 20번 이상 호출되고 이 중 50%이상이 실패하면 열림 상태가 된다. **이후의 모든 호출은 폴백 메서드에 의해 처리되다**가 5초후에 half-open 상태가 되어 원래 메서드 호출을 시도한다. 

```java
@HystrixCommand(
    fallbackMethod="getDefaultIngredients",
    commandProperties={
        @HystrixProperty(
            name="circuitBreaker.requestVolumeThreshold",
            value="30"),
        @HystrixProperty(
            name="circuitBreaker.errorThresholdPercentage",
            value="25"),
        @HystrixProperty(
            name="metrics.rollingStats.timeInMilliseconds",
            value="20000")
				@HystrixProperty(
            name="circuitBreaker.sleepWindowInMilliseconds",
            value="60000")
    })
public List<Ingredient> getAllIngredients() {
  // ...
}
```

- circuitBreaker.requestVolumeThreshold : 지정된 시간 내에 메서드가 호출되어야하는 횟수
- circuitBreaker.errorThresholdPercentage : 지정된 시간 내에 실패한 메서드 호출 비율 (%)
- metrics.rollingStats.timeInMilliseconds : 요청 횟수와 에러 비율이 고려되는 시간
- circuitBreaker.sleepWinfowInMilliseconds : half-open 상태로 진입하여 실패한 메서드가 다시 시도되기 전에 열림 상태의 서킷이 유지되는 시간

> metrics.rollingState.timeMilliseconds에 지정된 시간 이내에 circuitBreaker.requestVolumeThreshold와 circuitBreaker.errorThresholdPercentage 가 초가된다면 circuitBreaker.sleepWinfowInMilliseconds 에 지정된 시간동안 열림 상태에서 머무른다.

위의 코드와 같이 설정을 한다면 20초 이내에 메서드가 30번 이상 초풀이 되어 이중 25%가 실패일경우 서킷은 열림 상태가 되고 1분간 폴백 메서드가 요청을 대신 받아서 처리한다. 

메서드 실패와 지연을 처리하는 것 외에 어플리케이션에 있는 각 서킷 브레이커의 메트릭도 스트림으로 발행을 한다. 앞으로는 Hystrix가 발행한 스트림으로 어플리테이션의 상태를 모니터링하는 방법을 알아보겠다. 

# 15.3 실패 모니터링하기

서킷 브레이커로 보호되는 메서드가 매번 호출될 때마다 해당 호출에 관한 여러 데이터가 수집되어 Hystrix 스트림으로 발행된다. 앞에서 말한 것처럼 이 스트림은 실행중인 어플리케이션의 상태를 실시간으로 모니터링하는데 사용할 수 있다. Hystrix 스트림 안에는 아래의 사항들이 포함된다. 

- 메서드가 몇 번 호출되는지
- 성공적으로 몇 번 호출되는지
- 풀백 메서드가 몇 번 호출되는지
- 메서드가 몇 번 타임아웃되는지

이와 같은 Hystrix 스트림을 활성화하기 위한 방법

1. pom.xml에 dependency 추가 

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

 2. Hystrix 앤드포인트 경로 노출시키기 

```yaml
management:
  endpoints:
    web:
      exposure:
        include: hystrix.stream
```

2번을 마이크로서비스에 사용이 되는 각 어플리케이션의 application.yml 파일에 추가시켜주면 구성 서버의 모든 클라이언트 서비스가 이 속성을 사용할 수 있다. 

이제 Hystrix 스트림이 노출되어 어떤 REST 클라이언트를 사용해도 스트림을 사용할 수 있지만 그냥 Hystrix 스트림의 각 항목은 갖은 JSON 데이터로 구성이 되어있기 때문에 클라이언트 측의 많은 작업을 원하지 않는다면 Hystrix 대시보드의 사용을 고려해보자.

![https://howtodoinjava.com/wp-content/uploads/2017/07/HystrixStream.jpg](https://howtodoinjava.com/wp-content/uploads/2017/07/HystrixStream.jpg)

# 15.3.1 Hystrix 대시보드 개요

(새로운 스프링부트 프로젝트로 진행해야 한다)

1. pom.xml dependency 추가 

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
</dependency>
```

 2. application에서 활성화를 위해 @EnableHysritxDashboard  추가

```java
@SpringBootApplication
@EnableHystrixDashboard
public class HystrixDashboardApplication {
  public static void main(String[] args) {
    SpringApplication.run(HystrixDashboardApplication.class, args);
  }
}
```

 3. 포트 충돌 예방을 위한 서버 포트 지정 (개발시 로컬에서 여러가지 서버를 돌리고 있을 경우) 

```yaml
server:
  port: 7979
```

4. [localhost](http://localhost:7979/hystrix):7979[/hystrix](http://localhost:7979/hystrix) 에 접속

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/554fc09d-e717-4902-96cc-7b625911ea68/Untitled.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/554fc09d-e717-4902-96cc-7b625911ea68/Untitled.png)

- Hystrix 를 구현해 스트림을 받아오는 서비스 어플리케이션의 스트림을 보기 위해서는 고슴도치 밑에 있는 창에 해당 어플리케이션 포트와 strem url을 써주면 된다. [localhost:8080/hystrix.stream](http://localhost:8080/hystrix.stream) (아까hystrix의 앤드포인트를 hystrix.stream으로 설정)
- Delay: 폴링 간격을 나타내며 기본값은 2초. 결국 hystrix.stream 앤드포인트로부터 2초에 한번씩 Hystrix 결과를 받아온다
- Title: 모니터 페이지 제목

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/d1604529-5cae-4943-ac4c-c8bf5990b67a/Untitled.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/d1604529-5cae-4943-ac4c-c8bf5990b67a/Untitled.png)

- getAllIngredients에 대한 모니터링화면 (유일한 서킷 브레이커)
- 만약 Loding이라는 단어만 보인다면 서킷 브레이커로 보호된 메서드의 호출이 아직 없기 때문이다.
- 대시보드에 값을 표현하고 싶으면 해당 메서드를 호출해야 한다.

![https://drek4537l1klr.cloudfront.net/walls7/Figures/15fig04_alt.jpg](https://drek4537l1klr.cloudfront.net/walls7/Figures/15fig04_alt.jpg)

(더 자세한 대시보드)

- 왼쪽 위 모서리의 그래프: 지정된 메서드의 지난 2분 동안의 트래픽을 나타내며, 메서드가 얼마나 바쁘게 실행되었는지 볼 수 있다.
- 색깔이 입혀진 원: 원의 크기는 현재의 트래픽 수량이며 원의 색상은 해당 서킷 브레이커의 상태를 나타낸다. (초록: 좋음, 노랑: 가끔 실패, 빨강: 실패)
- 오른쪽 위 첫째 열:  (아래쪽으로 읽기) 첫번째: 현재 성공한 호출 횟수, 두번째: short circuited 요청 횟수, 마지막:  잘못된 요청의 횟수
- 두번째 열: 타임 아웃된 요청 횟수/스레드 풀이 거부한 횟수/실패한 요청 횟수
- 세번째 열: 지난 10초간의 에러 비융
- Host와 Cluster의 초당 요청 수를 나타내는 숫자 두개, 그 아래로는 해당 서킷 브레이커의 상태.
- 제일 아래: 지연시간의 중간값과 평균치, 백분위 수 를 보여주고 있다.

# 15.3.2 Hystrix 스레드 풀 이해하기

너무 수행시간이 오래 걸리는 메서드 → 다른 서비스에 Http 요청을 하고 있는데 해당 서비스의 응답이 느림→ 이런 경우 Hystrix는 응답을 기다리면서 관련 스레드를 블로킹한다. 

> 이런 메서드가 호출자와 같은 스레드의 컨텍스트에서 실행 중이라면 호출자는 오래 실행되는 메서드로부터 벗어날 기회가 없다. 게다가 블로킹된 스레드가 제한된 수의 스레드 중 하나인데 문제가 계속 생긴다면 사용 가능한 모든 스레드가 포화 상태가 되어 응답을 기다리게 된다.

이런 상황을 방지하기 위해 Hystrix는 각 의존성 모듈의 스레드 풀을 할당한다. (하나 이상의 Hystrix 명령 메서드를 갖는 각 스프링 빈을 위해) 

→ Hystrix 명령 메서드 중 하나가 호출될 때 이 메서드는 Hystrix가 관리하는 스레드 풀의 스레드(호출 스레드와는 별도) 에서 실행이 된다. 

→ 따라서 이 메서드가 너무 오래 걸린다면 호출 스레드는 해당 호출을 포기하고 벗어나며, 잠재적인 스레드 포화를 Hystrix가 관리하는 스레드 풀에 버릴 수 있다. 

![https://drek4537l1klr.cloudfront.net/walls7/Figures/15fig05_alt.jpg](https://drek4537l1klr.cloudfront.net/walls7/Figures/15fig05_alt.jpg)

(쓰레드 풀에 중점을 둔 대시보드, 서킷 브레이커 모니터와 흡사한 형태) 

- 상태를 나타내는 원이 있는 것은 동일하나 지난 몇 분 동안의 스레드 풀 활동을 나타내는 선 그래프는 없다.
- 스레드 풀의 이름: ingredientServiceImpl
- 왼쪽 아래 모서리 : 활성 스레드의 현재 개수/현재 큐에 있는 스레드 개수(기본적으로 큐가 비활성화 되어있어서 값은 항상 0)/스레드 풀에 있는 스레드의 개수
- 오른쪽 아래 모서리: 샘플링 시간동안 최대활성 스레드 카운트/Hystrix 실행을 처리하기 위해 스레드 풀의 스레드가 호출된 횟수/스레드 풀 슈의 크기(기본 비활성)

# 15.4 다수의 Hystrix 스트림 종합하기

원래 Hystrix 대시보드는 한 번에 하나의 Hystrix  스트림을 모니터링 할 수 있지만 Netflix의 Turbine 라이브러리를 사용하면 발행하는 스트림을 하나로 종합해준다.  

(새로운 스트링부트 프로젝트에 진행해야 한다. 

1. pom.xml에 dependency 추가 

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-turbine</artifactId>
</dependency>
```

 2. application에 활성화를 위해 @EnableTurbine 추가 

```java
@SpringBootApplication
@EnableTurbine
public class TurbineServerApplication {
  public static void main(String[] args) {
    SpringApplication.run(TurbineServerApplication.class, args);
  }
}
```

 3. 서비스 포트 충돌을 막기 위해 서버 포트 설정

```yaml
server:
  port: 8989
```

이제 마이크로서비스로부터 Hystrix 스트림이 생성되면 서킷 브레이커 메트릭들이 Turbine에 의해 하나의 Hystrix 스트림으로 종합이 된다. Trubine은 유레카의 클라이언트로 작동하므로 Hystrix  스트림을 종합할 서비스들을 유레카에 찾는다. 그런데 모든 서비스의 스트림을 종합하지는 않기 때문에 따로 설정을 해주어야 한다. 

4. turbine.app-config 속성을 사용해 Hystrix 스트림 종합 설정하기 

```yaml
turbine:
  app-config: ingredient-service,taco-service,order-service,user-service
  cluster-name-expression: "'default'"
```

- 유레카에서 찾을 서비스 이름들을 설정한다.
- 이름이 default인 클러스터에 있는 모든 종합될 스트림을 Turbine이 수집한다(cluster0name-expression)
- 만약 이름을 설정하지 않으면 지정된 어플리케이션들로부터 종합될 어떤 스트림 데이터도 Turbine 스트림에 포함이 안된다.

5. [localhost:8989/turbine.stream](http://localhost:8989/turbine.streamfmf) 를 대시보드에 입력 

![https://drek4537l1klr.cloudfront.net/walls7/Figures/15fig06_alt.jpg](https://drek4537l1klr.cloudfront.net/walls7/Figures/15fig06_alt.jpg)

# 요약

- 서킷 브레이커 패턴은 유연한 실패처리를 할 수 있다.
- Hystrix는 메서드가 실행에 실패하거나 너무 느릴 때 폴백 처리를 활성화하는 서킷 브레이커 패턴을 구현한다
- Hystrix가 제공하는 각 서킷 브레이커는 어플리케이션의 상태를 모니터링할 목적으로 Hystrix 스트림 매트릭스를 발행한다.
- Hystrix 스트림은 Hystrix 대시보드가 사용할 수 있다. Hystrix 대시보드는 서킷 브레이커 매트릭스를 보여주는 웹 어플리케이션이다.
- Turbine을 사용하면 어플리케이션들에서 발행이 되는 스트림을 하나로 병합할 수 있다.

# 참고

1. Hystrix 패턴에 대한 더 자세한 설명 

[(Spring Cloud) Hystrix](https://supawer0728.github.io/2018/03/11/Spring-Cloud-Hystrix/)

 2. Hystrix 를 사용하게 된 우아한 형제들의 실제 개발 사례

[Hystrix! API Gateway를 도와줘! - 우아한형제들 기술 블로그](https://woowabros.github.io/experience/2017/08/21/hystrix-tunning.html)

3. 간단하게 만들 수 있는 Hystrix 구현 예제

[Hystrix Circuit Breaker Pattern - Spring Cloud - HowToDoInJava](https://howtodoinjava.com/spring-cloud/spring-hystrix-circuit-breaker-tutorial/)