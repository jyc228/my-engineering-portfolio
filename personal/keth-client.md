# keth-client: 사용자 친화적 고성능 Kotlin Ethereum SDK

[github](https://github.com/jyc228/keth-client)

## 배경 & 도전

기존의 JVM 계열 블록체인 라이브러리(Web3j 등)들은 실제 개발 현장에서 발생하는 비효율과 생산성 저하를 해결하지 못한다고 정의했습니다.

- 성능과 가독성의 트레이드오프: 성능을 위해 Batch Request(묶음 요청)를 쓰려면 코드 구조를 완전히 바꿔야 했고, 가독성을 위해 동기식 코드를 짜면 너무 많은 api 호출이 필요하며 성능이
  저하되었습니다.
- 타입 안전성의 부재: 스마트 컨트랙트 호출 시 ABI 스펙을 JSON 문자열로 다루거나, 생성된 Wrapper 클래스가 너무 무겁고 사용하기 불편했습니다.
- 모던 아키텍처 미지원: Coroutine 같은 Kotlin의 강력한 동시성 모델을 지원하지 않아, 비동기 처리가 복잡하고 리소스 효율이 낮았습니다.

라이브러리는 사용 시 모호함이 없어야 하고, 스펙이나 비즈니스를 이해했다면 직관적으로 쓸 수 있어야 합니다.
**복잡한 문서 없이 IDE 자동 완성만 따라가며 적당히 썼는데도 최고의 성능이 나오는 것**, 이것이 제가 생각하는 이상적인 라이브러리입니다.
이 프로젝트는 위의 모든 문제를 해결하여, DX 를 극적으로 끌어 올렸습니다.

## 과제

이 부분을 이해하기 위하여 아래 [JSON-RPC](#JSON-RPC-설명) 를 읽고 오시면 도움이 됩니다.

### batch DSL 설계 (성능 최적화의 추상화)

개발자가 성능 최적화를 위해 비즈니스 로직을 수정할 필요가 없도록, 실행 컨텍스트만 바꾸면 자동으로 배치가 적용되는 DSL을 설계했습니다.

설계: Kotlin DSL의 **수신 객체 지정 람다** 를 활용하여, `batch { ... }` 블록 내부의 호출을 가로채고 큐에 적재하는 구조를 구현했습니다.

구현: `ApiResult<T>`를 반환하여, 개별 요청은 즉시 실행되지 않고 배치 실행 시점에 한 번의 HTTP Call로 처리되도록 만들었습니다.

심화: 더 나아가 사용자가 `batch` 함수를 쓰지 않아도 일정 기간 동안 요청 수집후 단일 패킷으로 전송하는 기능을 구현하였습니다. 이는 Rate Limit을 회피하면서 처리량을 극대화합니다.
이 기능을 통해 어떤 코루틴에서 사용하던 간에 같은 client 인스턴스를 사용하는 경우, 요청 처리량을 최적화하며 개발 편의성과 네트워크 효율성을 동시에 잡았습니다.

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

KotlinPoet 라는 오픈소스가 존재하지만, 사용 방식이 마음에 들지 않았었습니다. 개인 프로젝트인 만큼, 생성기도 도전해보고 싶었기 때문에 직접 만들었습니다.
핵심 목표는 kotlin 언어 구조를 이해했을때 직관적으로 사용할 수 있는 구조입니다.
이 목표를 달성하기 위하여 kotlin 공식 문법 문서를 적극적으로 사용하여 문법 구조(AST)를 직접 모델링했으며 **독자적인 코드 생성 엔진(DSL)** 을 구현했습니다.

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

이렇게 탄생된 라이브러리는 기존 jvm ethereum 보다 훨씬 사용하기 편했으며,
이는 차후 intellij ethereum plugin 개발시, 개발 난이도 및 성능 최적화에 직접적인 영향을 주게 되었습니다.
그와 별개로, 개인적으로도 그래들 플러그인, 코드 생성기 등 평상시엔 접하기 힘든 다양한 경험을 해 볼 수 있었고,
이것은 [bun-platform.md](../bunjang/platform.md) 작업할때 직접적인 도움이 되었습니다.

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
