# keth-client: Kotlin Ethereum SDK

[github](https://github.com/jyc228/keth-client)

## 한줄 요약

요청 배치 처리 DSL과 ABI 기반 코드 생성기를 갖춘 Kotlin Ethereum SDK. 코드 생성기는 Kotlin 문법 구조를 직접 모델링해서 만들었습니다.

## 배경 & 도전

기존 JVM 계열 블록체인 라이브러리(Web3j 등)를 쓰면서 아래와 같은 불편함이 있었습니다.

- 성능과 가독성의 트레이드오프: 성능을 위해 Batch Request(묶음 요청)를 쓰려면 코드 구조를 완전히 바꿔야 했고, 가독성을 위해 동기식 코드를 짜면 너무 많은 api 호출이 필요하며 성능이
  저하되었습니다.
- 타입 안전성의 부재: 스마트 컨트랙트 호출 시 ABI 스펙을 JSON 문자열로 다루거나, 생성된 Wrapper 클래스가 너무 무겁고 사용하기 불편했습니다.
- Coroutine 미지원: Kotlin Coroutine을 지원하지 않아 비동기 처리가 번거로웠습니다.

제가 생각하는 좋은 라이브러리는 **문서를 많이 읽지 않아도 IDE 자동 완성을 따라가며 쓰면 되고, 그렇게 써도 성능이 잘 나오는 라이브러리**입니다.
이 프로젝트는 그 방향을 목표로 만들었습니다.

## 과제

이 부분을 이해하기 위하여 아래 [JSON-RPC](#JSON-RPC-설명) 를 읽고 오시면 도움이 됩니다.

### batch DSL 설계 (성능 최적화의 추상화)

개발자가 성능 최적화를 위해 비즈니스 로직을 수정할 필요가 없도록, 실행 컨텍스트만 바꾸면 자동으로 배치가 적용되는 DSL을 설계했습니다.

설계: Kotlin DSL의 **수신 객체 지정 람다** 를 활용하여, `batch { ... }` 블록 내부의 호출을 가로채고 큐에 적재하는 구조를 구현했습니다.

구현: `ApiResult<T>`를 반환하여, 개별 요청은 즉시 실행되지 않고 배치 실행 시점에 한 번의 HTTP Call로 처리되도록 만들었습니다.

추가로, `batch` 함수를 쓰지 않아도 일정 시간 동안 요청을 모았다가 한 번에 보내는 기능을 만들었습니다. Rate Limit에 덜 걸리고 처리량도 늘어납니다.
같은 client 인스턴스를 쓰면 서로 다른 코루틴에서 보낸 요청도 묶여서 전송됩니다.

* [EthereumClient github](https://github.com/jyc228/keth-client/blob/dev/src/main/kotlin/com/github/jyc228/keth/client/EthereumClient.kt)
* [EthereumClientFactory github](https://github.com/jyc228/keth-client/blob/dev/src/main/kotlin/com/github/jyc228/keth/client/EthereumClientFactory.kt)

```Kotlin
interface EthereumClient : EthereumApi {
    suspend fun <R> batch(init: suspend EthereumApi.() -> List<ApiResult<R>>): List<ApiResult<R>>
}

suspend fun example1() {
    val client = EthereumClient("https://... or wss://...")
    client.eth.getHeaders(1uL..10uL).awaitAllOrThrow() // 10 http calls
    client.batch { eth.getHeaders(1uL..10uL) }.awaitAllOrThrow() // 1 http call
    // Do not use it like this: client.batch { client.eth.getHeaders(1uL..10uL) }
}

suspend fun example2() {
    val client = EthereumClient("https://... or wss://...") { interval = 100.milliseconds }
    launch {
        client.eth.getHeaders(1uL..10uL).awaitAllOrThrow()
    }
    launch {
        client.eth.getHeaders(20uL..30uL).awaitAllOrThrow()
    }
    // 서로 다른 코루틴에서 사용해도 100 밀리세컨드 마다 요청을 전부 수집후 http 단일 요청으로 변환합니다. 
}
```

### 스마트 컨트랙트와 편하게 상호작용하기 위한 Code Generator

스마트 컨트랙트의 ABI(Application Binary Interface)를 분석하여, Type-Safe한 Kotlin 코드를 자동 생성하는 Gradle Plugin을 직접 개발했습니다.

플러그인을 사용하면 abi 를 코틀린 코드로 변환합니다. 변환시 사용되는 input(abi) 과 output(kotlin code) 를 확인하고 싶으시다면 아래 링크를 확인해주세요.

- [input](https://github.com/jyc228/keth-client/blob/dev/contract/generator/src/main/resources/abi/ERC20.abi)
- [output](https://github.com/jyc228/keth-client/blob/dev/src/main/kotlin/com/github/jyc228/keth/client/contract/library/ERC20.kt)

그리고 이렇게 변환된 코드는 다음과 같이 사용됩니다.

```kotlin
fun example3() {
    val client = EthereumClient("https://... or wss://...")
    val usdt: ContractAccessor<ERC20> = ERC20("0x....")

    client.contract[usdt].name().call {}.awaitOrThrow()

    client.batch(
        { contract[usdt].name().call {} },
        { contract[usdt].symbol().call {} }
    )
}
```

#### kotlin code generator

KotlinPoet이 있지만 Kotlin 문법 구조를 직접 모델링하고 싶었고, 코드 생성기 자체를 설계하는 경험을 쌓기 위해 직접 구현했습니다.
핵심 목표는 kotlin 언어 구조를 이해했을때 직관적으로 사용할 수 있는 구조입니다.
Kotlin 공식 문법 문서를 참고해서 문법 구조를 직접 모델링하고, 이를 기반으로 코드 생성 DSL을 구현했습니다.

아래는 테스트코드 예제입니다.

```kotlin
fun test1() {
    // 테스트라서 클래스를 직접 생성하지만 실제 사용시엔 문법에 맞게 쓰도록 되어 있습니다.
    // builder.function("test").override()...
    FunctionDeclaration("test1")
        .override()
        .suspend()
        .parameters {
            parameter("p1", TypeElement.kotlin.string, expression = "hello")
            parameter("p2", TypeElement.kotlin.int)
            parameter("p3", TypeElement.kotlin.int, true)
            parameter("p4", TypeElement.java.time.localDateTime, true, "null")
        }
        .`return`(TypeElement.kotlin.int)
        .toString() shouldBe "override suspend fun test1(p1: String = hello, p2: Int, p3: Int?, p4: LocalDateTime? = null): Int\n"
}

fun test2() {
    // 테스트라서 클래스를 직접 생성하지만 실제 사용시엔 문법에 맞게 쓰도록 되어 있습니다.
    // builder.type("hello").`class`()...
    TypeDeclaration("hello", TestKotlinFileMetadata())
        .`class`()
        .primaryConstructor { parameter("test1", TypeElement.kotlin.string) }
        .toString() shouldBe "class Hello(test1: String)\n"
}
```

- [code generator github](https://github.com/jyc228/keth-client/tree/dev/codegen/src/main/kotlin/com/github/jyc228/kotlin/codegen)
- [구현시 참고한 공식 코틀린 문법](https://kotlinlang.org/docs/reference/grammar.html)

## 결과

제가 쓰기에는 기존 JVM 라이브러리보다 편했고, 이후 IntelliJ Ethereum Plugin을 만들 때 그대로 활용했습니다.
그래들 플러그인, 코드 생성기 등을 만들어 본 경험은 이후 [공통 라이브러리](../bunjang/platform.md) 작업에도 도움이 되었습니다.

### 부록

#### JSON-RPC 설명

모든 요청을 단일 endpoint 에 http post 로 실행할 명령과, 파라메터를 보내서 처리하는 방식을 말합니다.
`method` 필드로 어떤 기능을 실행힐지 결정하며, `params` 필드로 실행 파라메터를 설정할 수 있습니다.  
호출시 body 는 이런 포멧이어야 합니다.

```json
{
  "jsonrpc": "2.0",
  "method": "eth_getBlockByNumber",
  "params": [
    "0x1",
    true
  ],
  "id": 1
}
```

단일 http call 로 여러 명령도 실행할 수 있습니다.

```
[
  {..., "id": 1},
  {..., "id": 2},
  {..., "id": 3}
]
```
