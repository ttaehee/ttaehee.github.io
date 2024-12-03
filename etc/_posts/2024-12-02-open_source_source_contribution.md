---
title: Resilience4j 오픈소스 기여 과정
excerpt: 리팩토링 pr 머지
---

<br/>

- 저번에 [resilience4j 오픈소스 기여 시도 실패 과정](https://ttaehee.github.io/etc/open_source/)을 적었는데 이번에는 pr merge 까지 성공하였다
- 에러 해결은 아니고 (시작은 분명 버그해결이었으나,,) 리팩토링 pr 인데, 첫 기여 기념 + 에러 해결을 위해 테스트 했던 과정도 있어서 같이 적어두려고 한다

<br/>

## 이슈

지난번에 기여 실패한게 아쉬워서 이번에도 Resilience4j로 도전하였다    
이번에는 내가 직접 겪은 버그는 아니고, 이미 등록되어 있는  [\[executeSuspendFunction() ignores retryOnResults predicate #2098\]](https://github.com/resilience4j/resilience4j/issues/2098) 이슈를 선택했다   

![스크린샷 2024-12-02 오전 12 23 53](https://github.com/user-attachments/assets/801a5d57-e47c-47b6-869b-7a9da0c716d6)

<br/>

### 선택한 이유   

- 이슈 내용을 관련 코드까지 넣어서 상세하게 설명해주기도 했고,   
- maintainer 가 직접 구현과정에서 놓친부분이라고 피드백을 남겼기 때문에 요건 확실히 문제가 있는 부분이겠다! 생각하고 도전해보았다 (ㅎㅎ 아니었지만)   

![스크린샷 2024-12-02 오전 12 06 09](https://github.com/user-attachments/assets/ac6ae526-50d4-4772-bacc-60bcb3b5006b)

<br/>

## 이슈  파악    

- Retry.kt

```java
suspend fun <T> Retry.executeSuspendFunction(block: suspend () -> T): T {
    val retryContext = asyncContext<T>()
    while (true) {
        try {
            val result = block()
            val delayMs = retryContext.onResult(result)
            if (delayMs >= 0) {
                delay(delayMs)
                continue
            }
            retryContext.onComplete()
            return result
        } catch (e: Exception) {
            val delayMs = retryContext.onError(e)

            if (delayMs >= 0) {
                delay(delayMs)
                continue
            }
            throw e
        }
    }
}
```

<br/>

- CoroutineRetryTest.kt

```java
//81번째줄
@Test
fun `should execute function with retry of result`() {
    runBlocking {
        val helloWorldService = CoroutineHelloWorldService()
        val retry = Retry.of("testName") {
            RetryConfig {
                waitDuration(Duration.ofMillis(10))
                retryOnResult { helloWorldService.invocationCounter < 2 }
            }
        }
        val metrics = retry.metrics

        //When
        val result = retry.executeSuspendFunction {
            helloWorldService.returnHelloWorld()
        }

        //Then
        Assertions.assertThat(result).isEqualTo("Hello world")
        Assertions.assertThat(metrics.numberOfSuccessfulCallsWithoutRetryAttempt).isZero()
        Assertions.assertThat(metrics.numberOfSuccessfulCallsWithRetryAttempt).isEqualTo(1)
        Assertions.assertThat(metrics.numberOfFailedCallsWithoutRetryAttempt).isZero()
        Assertions.assertThat(metrics.numberOfFailedCallsWithRetryAttempt).isZero()
        // Then the helloWorldService should be invoked twice
        Assertions.assertThat(helloWorldService.invocationCounter).isEqualTo(2)
    }
}
```

이슈에서 \[the predicate is never checked and the specified number of retries always occurs\] 라고 말하였고 나는 specified number를 maxAttempts로 이해하였다   
다시 말해 retryOnResult 와 상관없이 maxAttempts 까지 시도한다고 이슈를 제기한것으로 파악하였다      

<br/>

- 내가 이해한 이슈 정리
  - retryOnResult 조건이 무시됨
  - retryOnResult 조건이 있음에도 무시되고 maxAttempts 까지 시도한다는 말 같음 

<br/><br/>

## 테스트 

- 관련 테스트코드 :: CoroutineRetryTest.kt 에 maxAttempts 조건 추가해서 테스트해보기

```java
//80번째줄
@Test
fun `should execute function with retry of result`() {
    runBlocking {
        val helloWorldService = CoroutineHelloWorldService()
        val retry = Retry.of("testName") {
            RetryConfig {
                waitDuration(Duration.ofMillis(10))
                maxAttempts(6)  // 기존에는 없던 조건으로, 이번 테스트를 위해 추가
                retryOnResult { helloWorldService.invocationCounter < 2 }
            }
        }
        val metrics = retry.metrics
        //When
        val result = retry.executeSuspendFunction {
            helloWorldService.returnHelloWorld()
        }
        //Then
        Assertions.assertThat(result).isEqualTo("Hello world")
        Assertions.assertThat(metrics.numberOfSuccessfulCallsWithoutRetryAttempt).isZero()
        Assertions.assertThat(metrics.numberOfSuccessfulCallsWithRetryAttempt).isEqualTo(1)
        Assertions.assertThat(metrics.numberOfFailedCallsWithoutRetryAttempt).isZero()
        Assertions.assertThat(metrics.numberOfFailedCallsWithRetryAttempt).isZero()
        // Then the helloWorldService should be invoked twice
        Assertions.assertThat(helloWorldService.invocationCounter).isEqualTo(2)
    }
}
```

```java
class CoroutineHelloWorldService {
    var invocationCounter = 0
        private set

    private val sync = Channel<Unit>(Channel.UNLIMITED)

    suspend fun returnHelloWorld(): String {
        delay(0) // so tests are fast, but compiler agrees suspend modifier is required
        invocationCounter++
        return "Hello world"
    }

    suspend fun throwException() {
        delay(0) // so tests are fast, but compiler agrees suspend modifier is required
        invocationCounter++
        error("test exception")
    }

    suspend fun cancel() {
        invocationCounter++
        coroutineContext.cancel(CancellationException("test cancel"))
        yield() //so CancellationException is thrown
    }

    /**
     * Suspend until a matching [proceed] call.
     */
    suspend fun wait() {
        invocationCounter++
        sync.receive()
    }

    /**
     * Allow a call into [wait] to proceed.
     */
    fun proceed() = sync.trySend(Unit)

}
```

기존 테스트 코드에는 maxAttempts 조건이 없어서 이번이슈 테스트를 위해 추가하여 테스트하였다   
위의 이슈가 제기한바대로라면 테스트가 실패하고 `Assertions.assertThat(helloWorldService.invocationCounter).isEqualTo(6)` 의 테스트가 성공해야한다   

<br/>

그러나 테스트 진행 시, maxAttempts 6까지 시도하지 않고 helloWorldService.invocationCounter가 2보다 작을 때까지만 시도하였다       
즉, 정상 동작으로 파악하였다     
디버깅을 해보아도 문제가 없어보였고 당황스러웠다   

<br/>

이슈 내용도 상세하고, maintainer 까지 실수를 인정했는데 이슈가 아닐거라는 사실을 받아들이기가 찝찝했다 그렇지만 재현은 안되고    
그래서 추가 질문을 남겨두었다 (지금 캡쳐하니까 contributor 라벨이 생겼네 오호)       

![스크린샷 2024-12-02 오전 12 10 50](https://github.com/user-attachments/assets/379928f6-fa34-4eef-b903-403ba8b26b25)

<br/><br/>

## 기여한 부분
해결할 에러는 없지만, 테스트 과정에서 발견한 개선되면 좋겠을 부분에 대해 리팩토링을 진행해보았다    
resilience4j 에서는 여러 retry를 제공하는데 Rety.kt와 FlowRetry.kt 에서의 조건 비교가 상이했다   

- Retry.kt

```java
val delayMs = retryContext.onResult(result)
if (delayMs < 1) {
    retryContext.onComplete()
    return result
```

<br/>

- FlowRetry.kt

```java
 val delayMs = retryContext.onResult(it) 
 if (delayMs >= 0) { 
     delay(delayMs) 
     throw RetryDueToResultException() 
```

동일하게 retryContext.onResult(result) 를 사용하는데 Retry는 `delayMs < 1` 로, FlowRetry은 `delayMs >= 0` 로 비교하고있다   
`retryContext.onResult(result)` 에서 문제가 있을 시, 무조건 -1을 반환하고 있어서 현재는 동작에 상관이 없지만 코드 통일성 및 추후 retryContext.onResult(result) 의 코드 변경을 대비하여 리팩토링하였다   

<br/><br/>

### 리팩토링한 코드 :: Retry.kt

- 기존 전체 코드

```java
try {
    val result = block()
    val delayMs = retryContext.onResult(result)
    if (delayMs < 1) {
        retryContext.onComplete()
        return result
    } else {
        delay(delayMs)
        }
    } catch (e: Exception) {
        val delayMs = retryContext.onError(e)
        if (delayMs < 1) {
            throw e
        } else {
            delay(delayMs)
        }
    }
```

<br/>

- 수정한 전체 코드
  
```java
try {
    val result = block()
    val delayMs = retryContext.onResult(result)
    if (delayMs >= 0) {
        delay(delayMs)
        continue
    }
    retryContext.onComplete()
    return result
} catch (e: Exception) {
    val delayMs = retryContext.onError(e)

    if (delayMs >= 0) {
        delay(delayMs)
        continue
    }
    throw e
}
```

- 비교 조건을 0으로 통일
- if-else 문 구조 개선으로 가독성 개선

<br/>

### PR merge

결과는 1주일도 안되어서 merge 되었다 신기해    

- [merge pr](https://github.com/resilience4j/resilience4j/pull/2245)
  
오픈소스 기여 자체도 신기했지만, 버그 해결이 아닌 리팩토링으로도 기여된다는 점이 새로웠다    
사실 사내코드나 프로젝트로 따지면 되게 당연한 일인데 말이다   

<br/><br/>

## 느낀점

- 오픈소스도 사람이 만드는거다      
- 오픈소스 기여는 버그 해결뿐 아니라 사내에서 코드 및 성능 개선하듯이 얼마든지 할 수 있다
  
예전에는 오픈소스에서 제공하는 기능만을 사용하고 믿는 입장이었지만, 나의 기여를 통해 프로젝트가 발전할 수 있다는 가능성을 자각하게되었다     

<br/>

또한 이 과정에서 나 자신도 성장하는 느낌이 들어 좋았다   
다음에는 더 복잡한 디버깅도 겪어보고, 여러 코드리뷰를 받으며 다양한 개발자의 사고방식을 배우고 싶다   
이를 통해 더 시야가 넓고 깊은 개발을 하고싶다      
+) 나도 이제 contributor 니까 코드리뷰나 피드백도 도전해보아야지     

<br/>

처음 시작할땐 막연한 두려움이 있었는데 막상 디버깅하면서 코드를 따라가다보니 다 똑같다    
평소에 하던것처럼 단지 코드의 흐름을 차근차근 따라가면 된다   
그 과정에서 구조 파악하는것도 재미있었고, 여러 사람이 만든 코드이다 보니 코드 구조나 컨벤션도 일관되지 않은점이 보였는데 그것마저 재미있는 포인트였다    
내가 이번에 기여한 부분처럼 말이다   

<br/>
  
이번에 새롭게 무한재귀 관련 이슈 파악을 시작했는데 얘는 정말 디버깅조차 쉽지않다       
쉽지않은만큼 요 문제를 해결하는 과정에서 여러가지 배우고 겪을것 같다           
다음엔 요 해결 과정으로 기록할 수 있길 바람 ㅎ.ㅎ             

<br/>
