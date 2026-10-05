# OpenAPI 기반 타입 안전 API 클라이언트 자동 생성

## 프로젝트 개요

- 상위 문서: [platform.md](platform.md)
- 기간
    - 1차: 2025.06 - 2025.08 (생성기, Enum 처리, 페이지 모델)
    - 2차: 2026.09 - 2026.10 (Gradle 플러그인 강화, 스펙 lock, Spring 자동 구성)
- 인원: 1명 (업무 외 시간에 자발적으로 진행)
- 핵심 기술: Kotlin, Gradle Plugin, OpenAPI, Spring Boot

## 한줄 요약

`OpenAPI` 명세로 API 클라이언트 코드를 생성하는 `Gradle` 플러그인 개발.
사용하는 쪽이 필요한 API만 선언하면 그 프로젝트 빌드 안에서 클라이언트를 생성하고, 상대 서비스가 그 API 스펙을 바꾸면 빌드에서 바로 드러나도록 했습니다.

## 배경 & 도전

서비스 간 통신은 조직 공용 라이브러리 `client-kotlin` 으로 하고 있습니다.
첫 재직 때 당시 팀장(현 CTO)이 단일 저장소에서 관리하는 클라이언트 라이브러리를 요청했고, Spring 자동 구성을 포함한 내부 구조는 제가 설계하고 구현했습니다. 재입사 후에는 한 차례 구조를 개편하기도 했습니다. ([mas-client-governance.md](mas-client-governance.md))

당시 제가 만든 구조는 사람이 손으로 관리하는 방식이었습니다. 어떤 서비스의 API 가 필요하면 개발자가 클라이언트 클래스를 직접 작성하고,
자동 구성 클래스에 설정 빈과 클라이언트 빈 한 쌍을 추가합니다. 50여 개 서비스가 이렇게 추가되면서 자동 구성 파일 하나에 같은 모양의 빈 정의가 100개 가까이 쌓였습니다.

```kotlin
// 서비스마다 반복되는 손으로 쓴 자동 구성 (50여 개 서비스 중 하나)
@Bean
@ConditionalOnProperty(prefix = "bunjang.client.pms", name = ["enabled"], havingValue = "true", matchIfMissing = false)
@ConfigurationProperties(prefix = "bunjang.client.pms")
fun pmsConfig(env: Environment) = Config("pms", env)

@Bean
@ConditionalOnBean(name = ["pmsConfig"])
@ConditionalOnMissingBean(PmsClient::class)
fun restPMSClient(@Valid pmsConfig: Config, @Qualifier(MAPPER_BEAN_NAME) mapper: ObjectMapper): PmsClient {
    logger.info("Initialized PMSClient. $pmsConfig")
    return RestPmsClient(WebClientFactory.createClient(pmsConfig, mapper))
}
```

이 방식에는 다음과 같은 문제가 있었습니다.

- 책임 부재: 누가 만들었는지, 누가 유지보수해야 하는지 불명확
- 테스트 없음: 코드 변경 시 신뢰도 보장 불가
- 코드와 API 스펙 불일치: 상대 서비스가 API 를 바꿔도 손으로 쓴 클라이언트는 그대로라, 호출이 실패하고 나서야 알게 됨
- 네임스페이스 오염: 서비스 하나의 API 만 써도 50여 개 서비스의 클라이언트 클래스가 전부 프로젝트에 들어옴

인증·인가 처리는 그대로 `client-kotlin` 에 두고, 이 플러그인은 서비스 클라이언트 부분을 대체하는 것을 목표로 합니다.

## 플러그인 사용 방식

사용하는 프로젝트는 어떤 서비스의 어떤 API를 쓰는지만 선언합니다.

```kotlin
plugins {
    id("com.bunjang.plugin.api-client-generator") version "..."
}

bunApi {
    // 자기 서비스의 스펙. 생성된 클라이언트는 테스트에서만 사용합니다.
    projectOpenapiFilePath = file("$projectDir/openapi.json")

    // 호출하는 서비스와, 그중 실제로 쓰는 API 만 선언합니다.
    dependency("bun-pms-api") {
        get("/api/1/user_product_descriptions")
    }
    dependency("bun-ams-api") {
        put("/api/ams/v2/sets/actions/term-extension")
    }
}
```

선언한 서비스마다 플러그인이 다음 태스크를 등록합니다.

1. **스펙 받기**: 서비스 이름 규칙으로 staging 의 OpenAPI 주소를 찾아 받습니다. 스펙 파일을 저장소에 두고 쓸 수도 있습니다.
2. **스펙 필터링**: 선언한 API 만 남기고 나머지를 걸러냅니다.
3. **코드 생성**: 걸러낸 스펙으로 클라이언트 코드와 Spring 자동 구성 클래스를 생성하고, `main` 소스셋에 붙입니다.

생성된 코드는 빌드 디렉터리에만 존재하고 저장소에는 커밋하지 않습니다.

그 외에 플러그인이 대신 처리하는 것들입니다.

- **Spring 자동 구성 등록**: 생성된 자동 구성 클래스(아래 참고)를 Spring Boot 가 찾을 수 있도록 `AutoConfiguration.imports` 파일을 만들어 리소스로 등록합니다. 클래스 이름은 생성 규칙에서 그대로 유도되므로 생성 결과를 다시 읽지 않습니다.
- **core 라이브러리 버전 고정**: 생성 코드는 core 라이브러리의 타입을 참조합니다. 버전이 어긋나면 사용자가 쓰지도 않은 생성 파일에서 컴파일 에러가 나기 때문에, 플러그인과 같은 버전의 core 를 자동으로 의존성에 넣습니다.
- **자기 스펙 클라이언트는 테스트 소스셋에만**: 자기 API 를 호출할 일은 없으므로, 자기 스펙으로 만든 클라이언트는 통합 테스트용으로만 붙입니다.

### 생성되는 Spring 자동 구성

서비스마다 자동 구성 클래스가 생성되어, 의존성을 선언하기만 하면 클라이언트가 빈으로 올라옵니다.

```kotlin
// 자동 생성 코드
@AutoConfiguration(after = [BunApiClientAutoConfiguration::class])
public class AmsApiAutoConfiguration {
    @Bean
    @ConfigurationProperties(prefix = "bunjang.api.client.ams")
    public fun amsApiClientProperties(): ApiClientProperties = ApiClientProperties()

    @Bean
    @ConditionalOnMissingBean(AmsApi::class)
    public fun amsApi(
        factory: BunApiClientFactory,
        @Qualifier("amsApiClientProperties") properties: ApiClientProperties
    ): AmsApi = AmsApiSpringWebClient(factory.webClient("ams", properties))
}
```

- 설정은 서비스 이름을 접두사로 씁니다(`bunjang.api.client.<서비스>.*`). 타임아웃, 커넥션 풀, 워밍업 같은 HTTP 클라이언트 설정을 서비스마다 따로 줄 수 있습니다.
- `@ConditionalOnMissingBean` 이라 특별한 구성이 필요하면 사용자가 직접 빈을 등록해 덮어쓸 수 있습니다.
- **주소는 빌드 시점이 아니라 실행 시점에 정합니다.** 생성된 클라이언트는 서비스 이름만 알고 있고, 실제 주소는 라이브러리의 환경별 기본 주소표에서 활성 프로파일(`prod` / `staging`)로 찾습니다. 주소를 생성 코드에 심지 않으므로, 서비스 주소가 바뀌어도 클라이언트를 다시 생성할 필요 없이 라이브러리 버전만 올리면 됩니다. 주소표에 없는 서비스나 로컬 테스트는 `url` 설정으로 직접 지정합니다.

이 구조로 배경의 문제들이 이렇게 정리됩니다.

| 문제 | 해결 |
|---|---|
| 책임 부재 | 어떤 서비스의 어떤 API 를 쓰는지가 사용하는 쪽 빌드 파일에 명시됨 |
| 네임스페이스 오염 | 공용 아티팩트 없이, 선언한 API 의 클라이언트만 생성 |
| 코드와 스펙 불일치 | 스펙에서 생성하고, 스펙이 바뀌면 아래의 lock 으로 감지 |
| 테스트 없음 | 자기 스펙 클라이언트로 통합 테스트 작성 ([아래](#백엔드-e2e-테스트-개선)) |

## 스펙 변경 감지: lock 파일

코드를 스펙에서 생성해도, 상대 서비스가 스펙을 바꾸면 다음 빌드에서 조용히 다른 코드가 생성됩니다.
특히 스펙은 staging 에서 받아오기 때문에, 아직 prod 에 배포되지 않은 변경이 그대로 클라이언트에 들어올 수 있습니다.

그래서 패키지 매니저의 lock 파일과 같은 방식을 도입했습니다.
소비자는 자기가 확인하고 고정한 시점의 계약으로만 빌드하고, staging 스펙이 바뀌면 그 변경을 받아들일지 사람이 diff 를 보고 결정합니다.

```json
{"bun-ams-api":"309fa82e...c7d3c96","bun-pms-api":"953fe236...a4a508fd6"}
```

- 서비스마다 **필터링한 스펙**의 해시를 lock 파일에 저장하고 저장소에 커밋합니다.
- 필터링한 스펙을 기준으로 하므로, 상대 서비스가 **내가 쓰는 API 를 바꿨을 때만** lock 이 깨집니다. 쓰지 않는 API 변경에는 반응하지 않습니다.
- lock 이 깨지면 이전 스냅샷과 비교해 무엇이 바뀌었는지 출력합니다.
- 변경을 확인한 뒤 갱신 태스크로 lock 을 새로 씁니다.

검증 태스크는 두 가지 시점을 정해 두었습니다.

```kotlin
// 버전과 무관하게 항상 검증한다. 상대가 스펙을 바꾼 날 바로 드러나야 한다.
target.tasks.matching { it.name == "check" }.configureEach {
    dependsOn(lockVerifyTask)
}

// 바뀐 계약으로 코드를 생성하고 컴파일까지 한 뒤에 실패하면 늦다. 생성보다 먼저 검증한다.
// mustRunAfter 라 둘 다 실행될 때만 순서를 강제하고, 생성 태스크를 단독으로 돌릴 때는 끼어들지 않는다.
target.tasks.withType<GenerateApiClientTask>().configureEach {
    mustRunAfter(lockVerifyTask)
}
```

결과적으로 사용하는 쪽 기준의 가벼운 계약 테스트가 됩니다.
상대 서비스가 내가 쓰는 API 를 바꾼 날, 내 CI 에서 무엇이 바뀌었는지와 함께 바로 실패합니다.

## 생성 결과물

플러그인이 `OpenAPI` 명세로부터 자동으로 생성하는 것들입니다.

- 타입 안전 API 호출 함수 (Ktor, Spring WebClient)
- Spring 자동 구성 클래스
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
        payMethod = PrepareRequest.PayMethod.card,
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

이 플러그인을 만든 또 다른 목적은 통합테스트 작성을 쉽게 하고, 나아가 E2E 회귀테스트를 자동화하는 것입니다.

자기 스펙으로 생성한 클라이언트를 통합테스트에서 직접 사용하면, API 서버의 비즈니스 로직과 생성된 API 클라이언트를 같이 검증할 수 있습니다.

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

> 플러그인은 2차 작업까지 완료했고, 첫 적용을 앞두고 있습니다. E2E 테스트 전환 기능은 개발 중입니다.
