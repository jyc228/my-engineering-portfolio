<!-- TOC -->
* [서비스간 통신 방식 개선: bun-spring-starter-api-client-lib (개발중)](#서비스간-통신-방식-개선-bun-spring-starter-api-client-lib-개발중)
  * [ApiPayload](#apipayload)
* [백엔드 E2E 테스트 개선](#백엔드-e2e-테스트-개선)
<!-- TOC -->

# 서비스간 통신 방식 개선: bun-spring-starter-api-client-lib (개발중)

***Problem***

수십개의 서비스간 통신을 위한 조직 공용 라이브러리가 있습니다. 어떤 api 가 필요로 하면 개발자가 api 를 직접 추가하는 형태로 관리 되고 있습니다.
저는 이 구조에 다음과 같은 문제가 있다고 정의했습니다.

- 책임 부재
- 테스트 없음
- 코드와 api 스펙이 동기화 되지 않음
- 단일 라이브러리로 빌드 되기 때문에 수많은 클래스가 프로젝트에 노출되며 네임스페이스 오염

***Solution***

OpenAPI 명세로부터 타입 안전한 API 클라이언트를 자동 생성하는 Gradle 플러그인을 직접 설계하고 개발하여, 서비스 간 통신 오류를 컴파일 타임에 차단하는 시스템을 구축했습니다.

이 라이브러리는 단순히 코드만 생성하는 것이 아니라, 사용하는 개발자의 경험을 극한으로 끌어올리도록 설계되었습니다.

- api 스펙에 http request body 가 있는 경우, 빌더 패턴 자동 생성
- 빌더 패턴 사용시, import 없는 enum 참조 제공
- 응답 데이터에 유효하지 않는 enum 구조 지원

```kotlin
// api call 예제입니다.
client.apicall {
    name = "새로운 상품"
    saleStatus = it.SaleStatus.SELLING
}

// 응답 데이터 구조중 enum 이 있는 경우, EnumProperty 로 wrapping 됩니다.
sealed interface EnumProperty<E : Enum<out E>> {
    val value: String
    val valueOrNull: E?
    val valueOrThrow: E

    fun onFailure(action: (unknownEnumValue: String) -> Unit): EnumProperty<E>
}
```

특히 import 없는 enum 을 구현하기 위해, enum 대신 `value class`를 사용하고 `companion object`와 스코프 함수를 조합하여,
개발자가 마치 원래 언어의 일부인 것처럼 자연스럽게 코드를 작성할 수 있는 새로운 DSL(도메인 특화 언어)을 창조했습니다.

저는 이 라이브러리를 통해 api 제공자는 api-client 도 함께 제공할 의무를 가지도록 강제하고,
통합테스트 작성을 유도하여 api client, api server 두 프로젝트를 동시에 검증하는걸 의도하고 있습니다.

## ApiPayload

***Problem***

함수의 과도한 오버로딩은 다음과 같은 문제가 있다고 정의했습니다.

- 단번에 어떤 함수를 써야 하는지 혼동을 줌
- 주석이 길 경우, 오버로딩 되는 함수 및 주석 파악이 쉽지 않음.

***Solution***

빌더 패턴과 일반 DTO 전달 방식을 하나의 함수 시그니쳐로 통합하기 위해, Kotlin의 `fun interface`와 확장 람다를 활용한 `ApiPayload` 인터페이스를 설계했습니다.

```kotlin
fun interface ApiPayload<REQUEST, CONTEXT> {
    operator fun invoke(init: REQUEST.(CONTEXT) -> Unit)
}
```

하지만 저는 이 설계가 `fun interface`와 리시버를 받는 람다 등 Kotlin의 고급 기능에 대한 깊은 이해를 요구하기 때문에, 어려워 하는 동료 개발자들이 많을 수 있다고 판단했습니다.
또한 저의 사용 사례에선 오버로딩은 3개 이상 넘지 않기 때문에 이건 **오버엔지니어링** 이다. 라는 판단을 하게 되었습니다.
따라서 기술적 순수함보다는 팀 전체의 명확성과 유지보수성을 우선하여, **더 단순하고 명시적인 두 개의 오버로딩된 메서드를 제공하는 현재의 방식을 최종적으로 선택했습니다.**

# 백엔드 E2E 테스트 개선

저는 위의 모든 플랫폼 작업을 **통합테스트 작성 장벽을 낮추고, 궁극적으로는 자동화된 E2E 회귀테스트를 구현**하기 위해 수행했습니다.
제가 이전에 구축한 통합테스트 프레임워크는 퇴사 후에도 지속적으로 활용되고 있었으며, 재입사 후 **조직장으로부터 "통합테스트 덕분에 개발과 검증이 훨씬 수월해졌다"는 직접적인 피드백**을 받을 수 있었습니다.
현재는 이 경험을 바탕으로 통합테스트 프레임워크를 한층 더 발전시켜 팀 전체에 확산하고 있습니다. 다음은 현재 사용 중인 테스트 코드의 일부입니다.

```kotlin
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
abstract class IntegrationTest {

    @Autowired
    lateinit var userApi: UserApi

    @LocalServerPort
    var port: Int = 0

    val client by lazy { ApiClient("http://localhost:$port") }

    fun test(
        token: UserToken? = null,
        test: suspend Context.() -> Unit
    ) = runBlocking {
        val token = token ?: userApi.signUpRandom()
        Context(token, client.withUser(token), coroutineContext).test()
    }

    fun UserApi.signUpRandom(
        name: String = "user-${Random.nextLong(1, Int.MAX_VALUE.toLong())}",
    ): UserToken = this.signUp(SignUpRequest(name = name))

    class Context(
        val user: UserToken,
        val api: ApiClient,
        override val coroutineContext: CoroutineContext,
    ) : CoroutineScope
}

class AdRewardResourceTest : IntegrationTest() {
    @Test
    fun `광고 시청 및 보상 이력을 검색할 수 있다`() = test {
        api.callApi()
    }
}
```

이 테스트 프레임워크를 통해 각 API 서버는 **자신의 비즈니스 로직을 검증하는 동시에 자동 생성된 API 클라이언트의 신뢰성까지 함께 보장**할 수 있습니다.
더 나아가, **설정만으로 테스트 더블 사용 여부를 결정**할 수 있도록 구현 예정입니다. 목표는 설정 변경만으로 통합테스트를 E2E 테스트로 전환하기 위함입니다.
**이러한 설정 기반 테스트 환경 전환이야말로 궁극의 E2E 테스트를 향한 핵심 기반**이라고 생각합니다.