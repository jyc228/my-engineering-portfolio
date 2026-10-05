# Kinesis 이벤트 발행/구독 라이브러리

## 프로젝트 개요

- 기간
    - v1: 2021.06 - 2022.03 (첫 재직)
    - v2: 2026.06 - 2026.09 (재입사 후)
- 인원: 1명
- 핵심 기술: Kotlin, Kotlin Coroutine, Spring Boot, Kinesis (KPL, KCL)

## 한줄 요약

제가 쓰려고 만든 Kinesis 발행/구독 라이브러리가 퇴사 후 전사 공용 라이브러리로 채택되어 백엔드 전체가 쓰게 되었고,
5년 뒤 재입사해서 경직된 구조를 직접 다시 설계했습니다. v2는 상품 도메인 등에 적용되어 운영 중입니다.

## 배경

Kinesis 를 쓰려면 KPL, KCL 설정과 직렬화, 체크포인트 처리 등 프로젝트마다 상당한 양의 반복 코드가 필요했습니다.
당시 팀 방침상 Spring 통합 도구는 쓰지 않았기 때문에, 프로젝트마다 이 코드를 직접 구현하고 있었습니다.

구현이 프로젝트마다 조금씩 달라서 불안했고, 우선 제 프로젝트에서 쓰려고 라이브러리로 만들었습니다.

## v1: 처음 만든 구조 (2021)

발행과 구독을 각각 spring-boot-starter 로 만들었습니다.

**발행**: `EventPublisher` 빈이 자동 등록되고, yaml 에서 이벤트 타입마다 어느 스트림으로 보낼지 지정합니다.
Kinesis 외에 로그 시스템, 콘솔 출력 구현체도 같은 인터페이스로 제공했습니다.

```yaml
bunjang:
  event:
    publisher:
      kinesis:
        enabled: true
        events:
          [ event-type ]: stream-name
```

**구독**: `EventSubscriber` 인터페이스를 구현해 빈으로 등록하면 KCL 기반 구독이 자동으로 시작됩니다.
단일 스트림, 멀티 스트림 구독과 향상된 팬아웃(Enhanced Fan-Out) 설정을 지원했습니다.

```kotlin
interface EventSubscriber<T : Any> {
    val id: String
    val consumeEventType: KClass<T>
    fun subscribe(events: Sequence<T>)
}
```

그 외에 표준 이벤트 모델과 레거시 이벤트 호환, 파티션 키 지정, 재시도·벌크 전송 설정, 로컬스택 기반 테스트를 넣었습니다.

## 퇴사 이후: 전사 공용 라이브러리로

2022년 3월 퇴사 직전까지 구독 쪽 팬아웃 지원을 마지막으로 넣고 나왔습니다.

그 뒤 당시 팀장(현 CTO)이 이 라이브러리를 전사 공용 라이브러리로 채택했고, 이후 4년 동안 여러 개발자가 제 코드 위에서 유지보수를 이어갔습니다.
Spring Boot 3·4 대응, KCL 3.x 마이그레이션, 일부 스트림만 팬아웃하는 설정 등이 이 기간에 다른 분들이 추가한 기능입니다.

재입사했을 때는 백엔드 전체가 이 라이브러리를 쓰고 있었습니다.

## v2: 다시 설계한 이유

4년간 기능이 덧붙으면서 yaml 설정 구조와 코드가 경직되어 있었습니다.
새로운 요구가 생길 때마다 기존 구조를 비틀어서 맞춰야 하는 상태였습니다.

- 발행: 추상화 단위가 "이벤트 타입" 이라, 같은 스트림으로 가는 이벤트가 늘어날수록 설정이 길어지고, 스트림별로 KPL 설정을 다르게 줄 방법이 없었습니다.
- 구독: 인터페이스 하나에 `subscribe` 메서드 하나라, 여러 스트림을 구독하면 한 메서드 안에서 직접 분기해야 했습니다.

## v2: 발행

### 스트림 단위 추상화

Kinesis 에서 실제 단위는 스트림입니다. 이벤트 타입이 아니라 스트림을 기준으로 설정하도록 바꿨습니다.

운영 중인 스트림은 `StreamCatalog` 에 등록되어 있고, 각 항목이 환경별 실제 스트림 이름을 알고 있어서 서비스 설정에 스트림 이름을 적을 필요가 없습니다.
카탈로그에 없는 스트림은 `producer-raw` 로 이름을 직접 지정할 수 있도록 열어 두었습니다.

스트림 정의와 KPL 클라이언트 설정도 분리해서, 특정 스트림만 고처리량 전용 클라이언트를 쓰는 것을 yaml 만으로 할 수 있습니다.

```yaml
# v1
bunjang:
  event:
    publisher:
      kinesis:
        events:
          AD_PRODUCT_REGISTERED: bun-ad-product-event-${bunjang.kinesis-env}
          AD_PRODUCT_MODIFIED: bun-ad-product-event-${bunjang.kinesis-env}
          AD_PRODUCT_DELETED: bun-ad-product-event-${bunjang.kinesis-env}

# v2
bunjang:
  stream:
    producer-client:
      high-throughput-client:
        aggregation-max-count: 500
        record-max-buffered-time: 50
    producer:
      bun_ad_product_event:
        client: high-throughput-client # 이 스트림만 전용 클라이언트 사용
      bun_up_event:                    # 기본 클라이언트 + 카탈로그로 이름 해석
```

### 비동기 발행과 결과 핸들

`produce` 는 발행을 시작만 하고 바로 결과 핸들을 돌려줍니다.
조직에 WebFlux, 코루틴, 일반 동기 코드가 섞여 있어서, 하나의 결과에서 대기 방식을 골라 쓸 수 있게 했습니다.

```kotlin
interface StreamProducer<EVENT : Any> {
    fun produce(event: EVENT): ProduceResult<EVENT>
    fun produce(events: Collection<EVENT>): List<ProduceResult<EVENT>>
    // ...
}

sealed interface ProduceResult<out EVENT : Any> {
    suspend fun await(): List<EVENT>   // 코루틴에서 스레드를 붙잡지 않고 대기
    fun block(): List<EVENT>           // 호출 스레드에서 대기
    fun onComplete(callback: (events: List<EVENT>, error: Throwable?) -> Unit): ProduceResult<EVENT> // 콜백 통보
}
```

사용자가 헷갈리기 쉬운 부분은 공개 API 주석에 먼저 적었습니다.

- 여러 건을 발행하면 파티션 키로 묶은 뒤 청크 단위로 한 레코드에 담기 때문에, 결과 개수는 이벤트 수가 아니라 레코드 수입니다.
- 청크는 한 번의 발행으로 처리되어 부분 성공이 없습니다. 정상 반환은 그 청크 전체가 확정되었다는 뜻입니다.
- `produce` 가 직접 예외를 던지는 경우는 파티셔닝 실패뿐이고, 이때는 전송 전이라 한 건도 발행되지 않습니다. 나머지 실패는 모두 결과 핸들로 전달됩니다.
- 콜백은 발행 executor 스레드에서 실행되므로, 콜백에서 블로킹하면 같은 executor 의 다른 결과 처리와 재시도까지 밀립니다.

### 재시도해도 바뀌지 않는 이벤트 id

표준 이벤트 모델 `BunEvent` 의 `id` 는 발행 시점에 비어 있을 때만 ULID 로 채웁니다.
재시도마다 다시 인코딩하더라도 id 는 처음 정해진 값으로 고정되므로, 재시도로 같은 이벤트가 중복 적재되어도 소비하는 쪽이 id 로 걸러낼 수 있습니다.

`data` 영역은 마커 인터페이스 `BunEventData` 를 구현해야만 넣을 수 있게 해서, 이벤트 형식이 표준 모델 밖으로 벗어나지 않도록 했습니다.

## v2: 구독

### 어노테이션 기반 구독

인터페이스 구현 대신 Spring 의 `@RestController`, `@RequestMapping` 처럼 어노테이션으로 구독 함수를 선언하도록 바꿨습니다.
스트림마다 함수 하나로 처리하고, 이벤트 타입은 함수의 첫 번째 파라미터 타입에서 읽어 디코딩합니다.

```kotlin
// v1
@Service
class DefaultEventSubscriber(
    private val product: ProductEventSubscriber,
    private val report: ReportEventSubscriber
) : EventSubscriber<JsonNode> {
    override val consumeEventType: KClass<JsonNode> = JsonNode::class
    override val id: String = "multi-stream-subscriber"
    override fun subscribe(events: Sequence<JsonNode>) {
        val s = events.toList()
        product.subscribe(s)
        report.subscribe(s)
    }
}

// v2
@StreamConsumerContainer
class DefaultStreamConsumerContainer(
    private val product: ProductEventSubscriber,
    private val report: ReportEventSubscriber
) {
    @StreamConsumer("product-event")
    fun consumeProductEvent(events: List<JsonNode>) = product.subscribe(events)

    @StreamConsumer("report-event")
    suspend fun consumeReportEvent(events: List<JsonNode>) = report.subscribe(events) // 코루틴 지원
}
```

체크포인트를 직접 다뤄야 하는 경우를 위해 `StreamCheckPointer`, KCL 원본 체크포인터, KCL 원본 입력을 선택 파라미터로 받을 수 있게 했고,
테스트용 체크포인터 구현도 함께 제공합니다.

### 코루틴 지원: 측정 후 바꾼 구현

suspend 구독 함수를 지원하면서, 처음에는 KCL 의 실행자 스레드 풀을 코루틴 디스패처로 재사용하고 `launch` 후 `join` 으로 기다리는 방식을 시도했습니다.
같은 스레드에서 처리되어 비용이 적을 것으로 예상했지만, 실제로는 그렇게 동작하지 않았습니다.

- 디스패처로 보낸 코루틴 본문은 다른 스레드가 실행해야 하는데, 호출 스레드는 `join` 으로 대기 중이라 자기가 넣은 작업을 직접 꺼내지 못합니다.
- 결과적으로 배치 하나를 처리하는 데 같은 풀의 스레드가 2개(대기 1, 실행 1) 필요해집니다.
- 벤치마크로 확인해 보니, 풀 크기와 동시 처리 수가 같은 흔한 KCL 구성(풀 4, 동시 호출 4)에서는 데드락이 발생했습니다. 풀을 2배로 늘리면 동작했고, 이때 최대 활성 스레드 수가 정확히 2n 이었습니다.
- `runBlocking` 에 실행자 디스패처를 넘기는 방법도 처음부터 다른 스레드로 디스패치되어 같은 문제로 돌아갑니다.

그래서 디스패처 없이 `runBlocking` 을 호출해, 호출 스레드 자신의 이벤트 루프에서 실행되도록 했습니다.
코루틴 객체 할당 비용은 남지만 스레드 핸드오프와 데드락 위험에 비하면 무시할 수준입니다.

이 비용을 감수하고도 코루틴을 지원한 이유는, 사용자가 구독 함수 안에서 `launch` 로 비동기 처리를 띄우면 에러 전파와 체크포인트 시점이 어긋나기 쉬운데,
라이브러리가 suspend 함수를 직접 받아 주면 그런 잘못된 사용을 피할 수 있기 때문입니다.

### 운영 중인 구독자와의 호환

KCL 은 워커 식별자와 lease 테이블 이름으로 체크포인트를 이어갑니다. 이 값이 바뀌면 기존 체크포인트를 잃고 처음부터 다시 읽거나 데이터를 건너뛸 수 있습니다.
v2 에서는 워커 식별자를 v1 과 같은 형식으로 만들고, lease 테이블이 없으면 자동 생성하지 않고 기동을 실패시키는 옵션을 기본으로 두어,
설정 실수로 새 테이블이 만들어지면서 체크포인트가 끊기는 일을 막았습니다.

## 공개 API 를 한 파일에

발행은 `StreamProducer.kt`, 구독은 `StreamConsumers.kt` 한 파일에 공개 API 를 모으고, 구현은 다른 파일로 분리했습니다.

- 사용자는 파일 하나와 그 주석만 보면 사용법을 알 수 있습니다. README 는 시작 안내와 마이그레이션 가이드만 담당합니다.
- 리뷰할 때 이 파일이 바뀌었다면 사용자에게 영향이 있는 변경이라는 뜻이라, 하위호환이 깨지는 변경을 놓치기 어렵습니다.

v1 사용자를 위해서는 yaml 설정과 소스 코드 각각에 대해 변경 전후 예제를 담은 마이그레이션 가이드를 함께 배포했습니다.

## 결과

- v1: 개인 용도로 만든 라이브러리가 퇴사 후 전사 공용으로 채택되어, 4년간 다른 개발자들이 유지보수하며 백엔드 전체가 사용
- v2: 발행·구독 모두 재설계, 상품 도메인 등에 적용되어 운영 중

5년 전에 제가 만든 구조의 한계를 다시 제가 정리할 기회였습니다.
v1 은 처음부터 공용 라이브러리로 쓰일 거라고 생각하지 않고 만들었기 때문에, v2 에서는 처음부터 여러 팀이 오래 쓴다는 전제로
추상화 단위, 공개 API 범위, 호환성을 먼저 정하고 시작했습니다.
