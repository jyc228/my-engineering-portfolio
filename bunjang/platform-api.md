<!-- TOC -->
* [페이징 모델 재정의](#페이징-모델-재정의)
* [조직 표준 응답 데이터 모델 구조화](#조직-표준-응답-데이터-모델-구조화)
* [인증 리졸버를 client-kotlin 에서 분리](#인증-리졸버를-client-kotlin-에서-분리)
* [예외 처리 구조 재정리](#예외-처리-구조-재정리)
* [자동 구성 분리와 환경별 기본값](#자동-구성-분리와-환경별-기본값)
<!-- TOC -->

# 페이징 모델 재정의

***Problem***

기존에 쓰던 Spring Pageable은 데이터 계층의 모델이 API 계층까지 노출되고, 정렬이 필요 없는 API에도 sort 파라미터가 노출되는 문제가 있었습니다.

***Solution***

cursor 페이징 지원, sort 파라미터 분리, 타입 안전한 sort 파라미터를 지원하는 `ApiPageParameter`를 만들었습니다.

```Kotlin
sealed interface ApiPageParameter {
    sealed interface OffsetBase : ApiPageParameter { /* ... */ }
    sealed interface CursorBase : ApiPageParameter { /* ... */ }
}

sealed interface ApiOffsetParameter : ApiPageParameter.OffsetBase
sealed interface ApiOffsetWithSortParameter : ApiPageParameter.OffsetBase {
    val sort: Iterable<String>
}
sealed interface ApiOffsetWithTypeSortParameter<E : Enum<E>> : ApiOffsetWithSortParameter
```

정렬이 필요 없는 메서드에 정렬 파라미터가 넘어가는 실수는 컴파일 단계에서 잡힙니다.
`ApiOffsetWithTypeSortParameter<E: Enum<E>>`는 Enum 타입 파라미터를 이용해서 Spring 파라미터 리졸버에서 sort 값을 검증하고,
springdoc 문서에 허용 값을 표시하며, 다른 객체로 변환할 때 sort key를 Enum으로 넘겨줍니다.

# 조직 표준 응답 데이터 모델 구조화

***Problem***

사내 문서에 표준 응답 스펙은 정의되어 있었지만 이를 위한 라이브러리는 없었습니다. 그래서 프로젝트마다 응답 모델을 만드는 방법이 제각각이었고,
가장 자주 보는 코드인데도 프로젝트를 옮길 때마다 다시 파악해야 했습니다.

***Solution***

표준 응답 모델과 이를 만드는 확장 함수를 제공합니다. 리시버 타입에 따라 알맞은 응답 구조로 변환됩니다.

```kotlin
/**
 * 표준 api 응답 모델. 이 클래스가 가장 기본적인 형태이며 추가 필드에 따라 하단과 같은 모델이 있습니다.
 *
 * - [ApiOffsetResult] : offset 페이징.
 * - [ApiSliceResult] : offset 페이징.
 * - [ApiCursorResult] : 단방향 커서 페이징.
 * - [ApiBiCursorResult] : 양방향 커서 페이징.
 * - [ApiErrorResult] : 에러 응답 (일반적으론 직접 생성하지 않으며 [HttpException] 발생시 [com.bunjang.api.spring.web.BunApiWebFluxExceptionHandler] 에서 해당 객체를 생성합니다.)
 *
 * [Any.toResult] 로 간단하게 자료구조에 맞는 인스턴스로 변환할 수 있습니다. 대부분 `ApiResult<T>` 형태로 변환이 되며 특정케이스에 한해서 위에 언급한 클래스로 변환 될 수 있습니다.
 */
data class ApiResult<R>(val data: R) {
    companion object
}

fun <E, R> org.springframework.data.domain.Page<E>.toResult(
    transformer: (E) -> R
): ApiOffsetResult<R> = ApiOffsetResult(
    data = content.map(transformer),
    page = number,
    size = size, // 요청한 페이지 사이즈, content 의 사이즈가 아닙니다! getNumberOfElements 로 데이터 설정중이라면 점검 필요
    totalPages = totalPages,
    totalElements = totalElements,
)

fun <K, V, R> Map<K, V>.toResult(transformer: (Map.Entry<K, V>) -> R): ApiResult<List<R>> = ApiResult(map(transformer))
fun <R> R.toResult(): ApiResult<R> = ApiResult(this)

fun ApiErrorResult(
    errorCode: String,
    reason: String,
    init: (ApiErrorResult.Builder.() -> Unit)? = null,
): ApiErrorResult = HashMapApiErrorResult(errorCode, reason).apply { init?.invoke(this) }

interface ApiErrorResult {
    val errorCode: String
    val reason: String

    operator fun get(key: String): Any?

    interface Builder {
        var errorCode: String
        var reason: String
        operator fun get(key: String): Any?
        operator fun set(key: String, value: Any?)
    }

    companion object
}

```

모든 함수 이름은 `toResult`로 통일했습니다. 프로젝트마다 응답 모델을 따로 만들 필요가 없어졌고, 페이지 정보를 잘못 채우는 것 같은 실수도 줄일 수 있었습니다.

# 인증 리졸버를 client-kotlin 에서 분리

***Problem***

API 요청의 인증·인가(토큰으로 사용자 조회, 권한 확인)는 서비스 간 통신 라이브러리인 `client-kotlin` 에 함께 들어 있었습니다.
인증만 쓰고 싶어도 50여 개 서비스 클라이언트를 함께 가져와야 했고, api-lib 은 `client-kotlin` 의 클래스가 런타임에 있는지 리플렉션으로 확인해 가며 연동하고 있었습니다.

***Solution***

인증을 `AuthUserResolver` 인터페이스로 추상화해 api-lib 으로 옮겼습니다. 서비스 간 통신 부분은 [API 클라이언트 생성 플러그인](platform-api-client.md)이 대체하는 중이라, 두 작업이 끝나면 `client-kotlin` 의 책임이 각자 맞는 라이브러리로 나뉩니다.

```kotlin
interface AuthUserResolver {
    /** 지원하는 토큰 헤더. 순서가 헤더 우선순위 */
    val supportedKeys: List<String>

    suspend fun resolve(provider: AuthProvider, permissions: Set<String>, requestData: Set<RequestUserData.Type>): AuthUser
}
```

- 토큰 종류별(사용자, 어드민, 서비스 간 JWT) 구현을 하나로 묶는 `CompositeAuthUserResolver` 를 두고, 헤더 우선순위는 등록 순서로 정합니다. 기동 시 어떤 인증이 몇 순위로 활성화됐는지 로그로 남깁니다.
- 테스트용 `InMemoryAuthUserResolver` 를 함께 제공합니다. 사용자를 등록하고 권한을 허용/거절하는 것만으로, 인증 서버 없이 통합 테스트를 작성할 수 있습니다.
- 인증 실패(401)와 인증 서버 장애(503)를 다른 예외로 나눴습니다. `AuthUser?` 처럼 nullable 로 받으면 인증 실패는 `null` 이 되지만, 인증 서버 장애는 그대로 전파됩니다. 장애를 비로그인 사용자로 조용히 처리하지 않기 위해서입니다.
- 서비스 간 토큰 인증은 디폴트로 비활성화했습니다. 다른 서비스를 서비스 토큰으로 호출하는 프로젝트라도, 자기 API 가 서비스 토큰을 받을 필요는 없는 경우가 있기 때문입니다.

### 마이그레이션 누락은 기동 단계에서 막는다

인증 어노테이션도 `com.bunjang.client.auth` 에서 `com.bunjang.api.auth` 로 옮겼습니다.
예전 어노테이션이 남아 있으면 컴파일은 되지만 더 이상 처리되지 않아, 권한 검사가 조용히 빠지는 위험이 있었습니다.

그래서 `client-kotlin` 인증을 쓰는 서비스에서 예전 어노테이션이 클래스, 메서드(상속·인터페이스 포함), 파라미터 어디에든 남아 있으면 **기동을 실패**시키고,
에러 메시지에 어느 핸들러인지와 함께 프로젝트 전체를 한 번에 바꾸는 명령을 안내합니다. 안내하는 명령은 테스트에서 실제로 실행해 검증했습니다.

```
com.bunjang.client.auth 의 어노테이션은 지원하지 않습니다. com.bunjang.api.auth 의 어노테이션으로 변경해주세요. {핸들러}
프로젝트 루트에서 아래 명령으로 일괄 변경할 수 있습니다. (어노테이션 이름, 속성 동일)
grep -rlE '...' --include='*.kt' --include='*.java' . | xargs perl -pi -e 's/.../com.bunjang.api.auth.$1/g'
```

문서로 안내하고 사람이 챙기기를 기대하는 대신, 놓치면 시스템이 멈추고 고치는 방법까지 알려주도록 했습니다.
