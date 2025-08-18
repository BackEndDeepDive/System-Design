<div align="center">

# Deep Dive  
Deep Dive into System Design

--- 
</div>

## Contents
- [Rate limiter](#rate-limiter)
- [Rate Limiter 구현 알고리즘](#rate-limiter-구현-알고리즘)
- [Rate Limiter의 적절한 위치](#rate-limiter의-적절한-위치)
- [SCG(Spring Cloud GateWay)에서의 RateLimiter](#scgspring-cloud-gateway에서의-ratelimiter) 
---
### Rate Limiter
Rate Limiter는 클라이언트 또는 서비스가 보내는 트래픽의 처리율(Rate)를 제어하기 위한 장치 입니다. HTTP 요청을 예로 들면 특정 기간 내에 전송되는 클라이언트의 요청 횟수를 제한합니다. API요청 횟수가 제한 장치에 정의된 임계치를 뛰어 넘으면 추가로 도달한 모든 호출은 처리를 중단하고 빠른 response 즉 429 Too Many Request를 응답합니다. 

> One of the imperative architectural concerns is to protect APIs and service endpoints from harmful effects, such as denial of service, cascading failure. or overuse of resources. Rate limiting is a technique to control the rate by which an API or a service is consumed. In a distributed system, no better option exists than to centralize configuring and managing the rate at which consumers can interact with APIs. Only those requests within a defined rate would make it to the API. Any more would raise an HTTP “Many requests” error.

Spring 공식문서에 따르면 서비스에 악영향을 미칠 수 있는 요청들을 사전에 방지하고 엔드포인트를 보호해야할 때 Rate Limiter을 사용한다고 알 수 있습니다. 

### Rate Limiter 구현 알고리즘 
Rate Limiter을 구현하는 알고리즘은 아래와 같이 크게 다섯가지로 나뉩니다.
1. 토큰 버킷 
2. 누출 버킷
3. 고정 윈도우 카운터
4. 이동 윈도우 로그
5. 이동 윈더 카운터

#### 토큰 버킷 
가장 많이 널리 사용되는 알고리즘입니다. 동작 원리를 간단하게 살펴보자면 처리할 수 있는 최대 용량을 가진 버킷 내부에 토큰이 지속적으로 채워지는 방식이기에 토큰이 꽉 찬 버킷에는 더이상 토큰이 추가되지 않습니다. 

![alt text](image.png)
#### 누출 버킷
누출 버킷 알고리즘은 토큰 버킷 알고리즘과 비슷하지만 요청 처리율이 고정되어 있는 점에서 다릅니다. 보통 FIFO 큐로 구현되어 있으며 요청이 도착하였을때 큐가 가득 차있는지 확인하고 빈자리가 있으면 큐에 요청을 추가합니다. 큐가 가득 차 있는 경우에는 새 요청을 버리고 지정된 시간마다 큐에서 요청을 처리하는 방식입니다.

![alt text](image-1.png)
#### 고정 윈도우 카운터
고정 윈도우 방식은 일정 시간 단위를 잘라서 그 구간 안에서 요청을 카운트하는 방식입니다. 다만 시간의 경계에 요청이 몰릴 경우 엣지 케이스가 발생할 가능성이 높습니다.
![alt text](image-2.png)
#### 이동 윈도우 로그
슬라이딩 윈도우 로그 방식은 요청이 들어올 때마다 로그를 저장하고 현재 시간에서 지정된 시간 안의 요청만 카운트하게 됩니다. 하지만 모든 로그를 저장해야하기 때문에 메모리와 성능에서의 장점은 가져갈 수 없습니다.
#### 이동 윈도우 카운터
이동 윈도우 카운터 방식은 위의 두 장점을 모두 결합하여 단점을 해소하는 방식입니다.

### Rate Limiter의 적절한 위치
Rate Limiter에 대해 학습하다보니 적절한 위치가 궁금해집니다. 일반적으로 생각해봤을때 Rate Limter가 위치할 수 있는 곳은 크게 세 가지 입니다.
1. CDN, NginX 등 서비스의 엣지
2. SCG(Spring Cloud GateWay)와 같은 API 게이트웨이
3. 비즈니스 로직이 구현되어있는 어플리케이션 레벨

1번과 같이 서비스의 엣지에 두는 경우는 Ddos, 봇, 스크래핑 등 폭주하는 트래픽을 1차적으로 차단하는곳에 사용합니다.

2번과 같은 게이트웨이에서 차단하는 경우는 인증된 주체, 라우트 별 정밀 레이트가 필요한 경우, 과금, 요금제, 엔드포인트 별 정책을 다르게 하는 경우에 사용합니다.

3번과 같은 어플리케이션에서 처리율을 제한하는 경우는 결제와 같은 비즈니스 로직의 재시도와 같이 도메인의 특수성을 정확히 반영해야하는 경우에 사용합니다. 

### SCG(Spring Cloud GateWay)에서의 RateLimiter
SCG에서 사용하는 RateLimiter을 확인해보겠습니다.
![alt text](image-4.png)

Spring Cloud Gateway(SCG)에서 RateLimiter는 크게 두 가지 구현체가 있습니다.
1. RedisRateLimiter (기본값, Default)
2. Bucket4jRateLimiter (옵션, 별도 선택)

![alt text](image-3.png)

AbstractRateLimiter 추상 클래스를 기반으로 다양한 RateLimiter 구현이 가능합니다. 
#### RedisRateLimiter (Default)
분산 환경 지원을 위해 Redis 기반으로 동작하고 토큰 버킷 알고리즘을 사용합니다. 여러 인스턴스의 게이트웨이가 떠있어도 같은 redis를 공유하기 때문에 강한 일관성이 보장됩니다. 

RedisRateLimiter는 다음과 같이 동작합니다.

```
요청이 들어올 때 Redis에서 현재 버킷 상태(토큰 수) 확인
replenishRate(초당 토큰 채워지는 속도)에 따라 토큰 갱신
요청당 requestedTokens만큼 차감
남은 토큰이 없으면 HTTP 429 (Too Many Requests) 반환
```
yaml에서는 아래와 같이 설정하여 사용할 수 있습니다.
```
spring:
  cloud:
    gateway:
      routes:
        - id: route1
          uri: http://localhost:8081
          predicates:
            - Path=/backend
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 500   # 초당 500개 요청
                redis-rate-limiter.burstCapacity: 1000 # 최대 1000개 버스트 허용
                redis-rate-limiter.requestedTokens: 1

```

```
    public Mono<RateLimiter.Response> isAllowed(String routeId, String id) {
        if (!this.initialized.get()) {
            throw new IllegalStateException("RedisRateLimiter is not initialized");
        } else {
            Config routeConfig = this.loadConfiguration(routeId);
            int replenishRate = routeConfig.getReplenishRate();
            int burstCapacity = routeConfig.getBurstCapacity();
            int requestedTokens = routeConfig.getRequestedTokens();

            try {
                List<String> keys = getKeys(id, routeId);
                List<String> scriptArgs = Arrays.asList("" + replenishRate, "" + burstCapacity, "", "" + requestedTokens);
                Flux<List<Long>> flux = this.redisTemplate.execute(this.script, keys, scriptArgs);
                return flux.onErrorResume((throwable) -> {
                    this.log.error("Error calling rate limiter lua", throwable);
                    return Flux.just(Arrays.asList(1L, -1L));
                }).reduce(new ArrayList(), (longs, l) -> {
                    longs.addAll(l);
                    return longs;
                }).map((results) -> {
                    boolean allowed = (Long)results.get(0) == 1L;
                    Long tokensLeft = (Long)results.get(1);
                    RateLimiter.Response response = new RateLimiter.Response(allowed, this.getHeaders(routeConfig, tokensLeft));
                    if (this.log.isDebugEnabled()) {
                        this.log.debug("response: " + response);
                    }

                    return response;
                });
            } catch (Exception var10) {
                Exception e = var10;
                this.log.error("Error determining if user allowed from redis", e);
                return Mono.just(new RateLimiter.Response(true, this.getHeaders(routeConfig, -1L)));
            }
        }
    }
```
`RedisRateLimiter`에서는 버킷 상태 확인, 갱신, 차감에 대한 로직이 lua script로 이루어져 있어 강한 일관성을 확보할 수 있습니다.

#### Bucket4jRateLimiter
반면 `Bucket4jRateLimiter`의 경우 JVM 내부에서 동작하므로 단일 인스턴스에서만 서비스가 동작할 때 사용하기 편리하다는 장점이 있습니다.

#### 커스터마이징
```
public interface RateLimiter<C> {
    Mono<Response> isAllowed(String routeId, String key);
}
```

특정 사용자 등급에 따라서 서로 다른 제한을 적용하거나 특정 엔드포인트별로 알고리즘을 다르게 적용시킬땐 위의 인터페이스를 직접 구현하여 사용할 수 있습니다. 

### Ref
https://spring.io/blog/2021/04/05/api-rate-limiting-with-spring-cloud-gateway
https://dev.gmarket.com/69