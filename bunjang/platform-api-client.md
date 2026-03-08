# OpenAPI 기반 타입 안전 API 클라이언트 자동 생성

## 프로젝트 개요

- 상위 문서: [platform.md](platform.md)
- 핵심 기술: Kotlin, Gradle Plugin, OpenAPI

## 한줄 요약

`OpenAPI` 명세로부터 타입 안전 클라이언트를 자동 생성하는 `Gradle` 플러그인 직접 개발 → 코드-스펙 불일치 원천 차단, 외부 API `Enum` 불확실성을 타입 시스템으로 처리

## 배경 & 도전

수십 개의 서비스 간 통신을 위한 조직 공용 라이브러리가 있었습니다. 특정 API가 필요하면 개발자가 직접 추가하는 형태로 관리되고 있습니다.

저는 이 구조에 다음과 같은 문제가 있다고 정의했습니다.

- 책임 부재: 누가 만들었는지, 누가 유지보수해야 하는지 불명확
- 테스트 없음: 코드 변경 시 신뢰도 보장 불가
- 코드와 API 스펙 불일치: 수동 관리로 인한 구조적 문제
- 단일 라이브러리로 빌드되어 수많은 클래스가 프로젝트에 노출, 네임스페이스 오염

## 생성 결과물

플러그인이 `OpenAPI` 명세로부터 자동으로 생성하는 것들입니다.

- 타입 안전 API 호출 함수
- Builder DSL (import 없는 `Enum` 참조 포함)
- value class 기반 확장 가능한 `Enum`
- `EnumProperty`로 감싼 응답 `Enum` (알 수 없는 값 런타임 안전 처리)

```kotlin
// API 호출 예제
client.prepare {
    payMethod = it.PayMethod.card
    totalPrice = 10000
}

// 기존 방식도 지원
client.prepare(
    PrepareRequest(
        payMethod = PrepareRequest.PayMeothd.card,
        totalPrice = 10000
    )
)

// 응답 Enum이 알 수 없는 값이어도 런타임에 안 터집니다
response.status.onFailure { unknownValue ->
    logger.warn { "알 수 없는 status: $unknownValue" }
}
```

## 핵심 설계: 요청/응답 Enum 분리 처리

요청 Enum과 응답 Enum은 성격이 다르기 때문에 다르게 처리했습니다.

**요청 Enum**: 알 수 없는 값이 들어올 일이 없고 `exhaustive when`도 필요 없습니다.
`ordinal` 등 불필요한 것 없이 참조만 편하게 할 수 있도록 `@JvmInline value class`로 String을 감쌌습니다.
`ValueEnum`은 여기에 `entries`, `valueOf`를 제공하는 유틸입니다.

또한 빌더 패턴을 활용하면 import 없는 Enum 참조가 가능합니다. `companion object`를 Builder의 `it` 파라미터로 주입하는 방식입니다.

```kotlin
@JvmInline // ValueEnum 을 제외하곤 자동생성 코드입니다.
value class PayMethod(val name: String) {
    companion object : ValueEnum<PayMethod>() {
        val card: PayMethod = PayMethod("card")
        val vbank: PayMethod = PayMethod("vbank")
        val kakaopay: PayMethod = PayMethod("kakaopay")
    }
}

client.prepare {
    payMethod = it.PayMethod.card  // import 없이 타입 안전하게 참조
}
```

**응답 Enum**: 외부 API는 언제든 새로운 값을 추가할 수 있습니다.
`@JsonEnumDefaultValue` 같은 어노테이션으로 해결할 수도 있지만, 역직렬화 실패와 진짜 `UNKNOWN` 값을 구분할 수 없고 특정 직렬화 기술 의존성이 생성 코드에 침투하는 문제가 있습니다.
그래서 `EnumProperty`로 감싸서 역직렬화 실패를 타입으로 명시하고, 호출자가 반드시 처리하도록 강제했습니다.

```kotlin
sealed interface EnumProperty<E : Enum<out E>> {
    val value: String
    val valueOrNull: E?
    val valueOrThrow: E
    fun onFailure(action: (unknownEnumValue: String) -> Unit): EnumProperty<E>
}
```

## ApiPayload - 오버엔지니어링 인식 및 철회

빌더 패턴과 일반 DTO 전달 방식을 하나의 함수 시그니처로 통합하기 위해 `ApiPayload`를 설계했습니다.

```kotlin
fun interface ApiPayload<REQUEST, CONTEXT> {
    operator fun invoke(init: REQUEST.(CONTEXT) -> Unit)
}
```

그러나 `fun interface`와 리시버 람다 등 Kotlin 고급 기능에 대한 깊은 이해를 요구하기 때문에 동료 개발자들이 어려워할 수 있다고 판단했습니다.
실제 사용 사례에서 오버로딩은 3개를 넘지 않았기 때문에 **오버엔지니어링**이라고 결론 내리고, 더 단순하고 명시적인 두 개의 오버로딩 메서드로 되돌렸습니다.

## 백엔드 E2E 테스트 개선

이 플러그인의 궁극적인 목표는 통합테스트 작성 장벽을 낮추고 자동화된 E2E 회귀테스트를 구현하는 것입니다.

자동 생성된 API 클라이언트를 통합테스트에서 직접 사용하면, API 서버는 자신의 비즈니스 로직을 검증하는 동시에 API 클라이언트의 신뢰성까지 함께 보장할 수 있습니다.

```kotlin
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
abstract class IntegrationTest {
    val client by lazy { ApiClient("http://localhost:$port") }

    fun test(token: UserToken? = null, test: suspend Context.() -> Unit) = runBlocking {
        val token = token ?: userApi.signUpRandom()
        Context(token, client.withUser(token), coroutineContext).test()
    }
}

class AdRewardResourceTest : IntegrationTest() {
    @Test
    fun `광고 시청 및 보상 이력을 검색할 수 있다`() = test {
        api.callApi()
    }
}
```

향후 설정 변경만으로 통합테스트를 E2E 테스트로 전환할 수 있는 구조를 구현할 예정입니다.

> 플러그인 및 E2E 테스트 전환 기능 개발 중