# Resilience4j-CircuitBreaker 실험

## 설정

아래와 같이 management 설정을 위와 같이 추가하면 `/actuator/health` 호출 결과에 circuit breaker 상태가 포함돼서 표시된다.

`instances` 바로 하위에 있는 `instanceA`은 코드에 적용할 서킷 브레이커 인스턴스 이름을 가리키며 `@CircuitBreaker(name = "instanceA")`와 같은 형식으로 사용된다.

```yml
resilience4j.circuitbreaker:
  configs:
    default:
      slidingWindowType: COUNT_BASED
      slidingWindowSize: 10
      permittedNumberOfCallsInHalfOpenState: 5
      waitDurationInOpenState: 60000
      failureRateThreshold: 50
      registerHealthIndicator: true
  instances:
    instanceA:
      baseConfig: default

management:
  endpoint:
    health:
      show-details: always
  health:
    circuitbreakers:
      enabled: true
```

## 결론 먼저

>1. 최초 CLOSED 상태에서 최초 요청이 들어오면 `bufferedCalls`가 1 증가하고, 요청이 실패하면 `failedCalls`도 1 증가한다.
>2. `bufferedCalls` 값이 `slidingWindowSize`에 도달하면 `failedCalls / bufferedCalls * 100%`를 통해 `failureRate`가 계산된다.  
>3. 계산 결과 `failureRateThreshold` 미만이면 CLOSED 상태가 유지되고,  
>3.1 이후 요청이 들어오면 최초에 들어왔던 요청이 버퍼에서 빠져나가면서 `bufferedCalls`가 1 감소하지만 새로 들어온 요청에 의해 1 증가해서 결국 10으로 그대로 유지된다.  
>3.2 빠져나간 요청이 성공한 요청이었다면 `failedCalls` 값도 그대로 유지되고, 빠져나간 요청이 실패한 요청이었다면 `failedCalls` 값도 1 감소한다. 그리고 새로 들어온 요청이 실패라면 `failedCalls` 값이 1 증가하고, 이 값을 기준으로 `failureRate` 값이 새로 계산된다.
>4. 계산 결과 `failureRateThreshold` 이상이면 서킷 브레이커 상태가 OPEN 으로 변경되고,  
>4.1 OPEN 상태에서 `waitDurationInOpenState` 동안 들어오는 요청은 성공/실패와 무관하게 `bufferedCalls`도 `failedCalls`도 증가시키지 않고, `io.github.resilience4j.circuitbreaker.CallNotPermittedException`가 발생하면서 `notPermittedCalls` 값만 1씩 증가한다.  
>4.2 `waitDurationInOpenState`이 지나서 요청이 들어오면 HALF_OPEN 으로 상태가 변경되고 10이었던 `bufferedCalls` 값이 1로 변경된다. 요청이 성공하면 `failedCalls`은 0, 실패하면 1로 변경된다.
>5. HALF_OPEN 상태에서 `bufferedCalls` 값이 `permittedNumberOfCallsInHalfOpenState`에 도달하면 `failureRate`가 계산되는데 이 값이,  
>5.1 `failureRateThreshold` 값 미만이면 상태가 CLOSED 로 변경되고 `bufferedCalls`, `failedCalls` 모두 0으로 초기화된다. 이후 과정은 1번 과정과 동일하다.  
>5.2 `failureRateThreshold` 값 이상이면 상태가 OPEN 으로 변경되고 `bufferedCalls`, `failedCalls` 값은 그대로 남는다. 이후 과정은 2.2번 과정과 동일하다.  


## 실험 내용

- COUNT_BASED 타입 기준으로 `slidingWindowSize: 10`이므로 최소 10개의 요청이 있어야 `failureRate`가 계산된다. 
  - 9개 요청까지는 실패 요청 수인 `failedCalls`값과 무관하게 `failureRate` 값은 `-1.0%` 인채로 유지된다.
- 10개의 요청이 들어와서 `failureRate` 값이 계산되면 `failureRateThreshold` 값과 비교해서 서킷 상태 변경이 발생할 수 있다.
  - `failureRateThreshold: 50`이므로 10개 중 5개 요청이 에러이면 즉 50%에 도달하면 `CIRCUIT_OPEN`으로 서킷 상태가 변경된다.
  - **[공식 문서](https://resilience4j.readme.io/docs/circuitbreaker)에는 `above` 즉 50%를 초과해야 OPEN 상태로 변경되는 것으로 나와있지만, 실제 테스트 결과 50% 초과가 아니라 50% 일 때도 OPEN 상태로 변경된다.**
- OPEN 됐을 때 주요 상태값은 다음과 같다.
  - state: OPEN
  - failureRate: 50%
  - bufferedCalls: 10
  - failedCalls: 5
  - notPermittedCalls: 0
- **OPEN 상태가 된 후 아무런 요청이 없으면 계속 OPEN 상태와 위 주요 상태값이 그대로 남아 있는다.**
- OPEN 상태에서 1번째 요청이 에러를 발생시키면 다음과 같이 주요 상태 값이 변경된다.
  - state: HALF_OPEN
  - failureRate: -1.0%
  - bufferedCalls: 1
  - failedCalls: 1
  - notPermittedCalls: 0
- HALF_OPEN 상태에서 2번째 요청이 에러를 발생시키면 다음과 같이 주요 상태 값이 변경된다.
  - state: HALF_OPEN
  - failureRate: -1.0%
  - bufferedCalls: 2
  - failedCalls: 2
  - notPermittedCalls: 0
- HALF_OPEN 상태에서 3번째 요청이 에러를 발생시키면 다음과 같이 주요 상태 값이 변경된다.
  - state: HALF_OPEN
  - failureRate: -1.0%
  - bufferedCalls: 3
  - failedCalls: 3
  - notPermittedCalls: 0
- HALF_OPEN 상태에서 4번째 요청이 에러를 발생시키면 다음과 같이 주요 상태 값이 변경된다.
  - state: HALF_OPEN
  - failureRate: -1.0%
  - bufferedCalls: 4
  - failedCalls: 4
  - notPermittedCalls: 0
- **`permittedNumberOfCallsInHalfOpenState: 5`로 설정돼있으므로, HALF_OPEN 상태에서 5번째 요청이 에러를 발생시키면 `failureRate` 값이 계산되고 다음과 같이 주요 상태 값이 변경된다.**
  - state: OPEN
  - failureRate: 100.0%
  - bufferedCalls: 5
  - failedCalls: 5
  - notPermittedCalls: 0
- **OPEN 상태에서 `waitDurationInOpenState` 시간에 도달하기 전에 들어오는 요청은 성공/실패 무관하게 `notPermittedCalls`만 1씩 계속 증가시킨다.**
  - state: OPEN
  - failureRate: 100.0%
  - bufferedCalls: 5
  - failedCalls: 5
  - notPermittedCalls: 1
- **OPEN 상태에서 `waitDurationInOpenState` 시간이 지난 후에 1번째 요청이 들어오면 성공/실패 무관하게 HALF_OPEN 으로 상태가 바뀌고 정상 처리되면 다음과 같이 주요 상태 값이 변경된다.**
  - state: HALF_OPEN
  - failureRate: -1.0%
  - bufferedCalls: 1
  - failedCalls: 0
  - notPermittedCalls: 0
- HALF_OPEN 상태에서 2번째 요청이 정상 처리되면 다음과 같이 주요 상태 값이 변경된다.
  - state: HALF_OPEN
  - failureRate: -1.0%
  - bufferedCalls: 2
  - failedCalls: 0
  - notPermittedCalls: 0
- HALF_OPEN 상태에서 3번째 요청이 정상 처리되면 다음과 같이 주요 상태 값이 변경된다.
  - state: HALF_OPEN
  - failureRate: -1.0%
  - bufferedCalls: 3
  - failedCalls: 0
  - notPermittedCalls: 0
- HALF_OPEN 상태에서 4번째 요청이 에러를 발생시키면 다음과 같이 주요 상태 값이 변경된다.
  - state: HALF_OPEN
  - failureRate: -1.0%
  - bufferedCalls: 4
  - failedCalls: 1
  - notPermittedCalls: 0
- **`permittedNumberOfCallsInHalfOpenState: 5`로 설정돼있으므로, HALF_OPEN 상태에서 5번째 요청이 에러를 발생시키면 `failureRate` 값이 계산되고(2/5 == 40%) 50% 미만이므로 CLOSED 상태가 되며 다음과 같이 주요 상태 값이 초기화된다.**
  - state: CLOSED
  - failureRate: -1.0%
  - bufferedCalls: 0
  - failedCalls: 0
  - notPermittedCalls: 0