# Redis 기반 복권 당첨 처리 시스템 설계 및 구축

## 프로젝트 개요

- 기간: 2025.10 - 2025.12 (약 2개월)
- 인원: 백엔드 3명
- 핵심 기술: Kotlin, Spring Boot, Redis, MySQL
- 주요 역할: 구조 설계 및 코드 리뷰

## 한줄 요약

DAU 10%(8만 유저) 동시 접속을 가정한 고트래픽 설계 → DB 트랜잭션, 분산락 없이 중복 당첨 0% 원자성 확보

## 배경 & 과제

- 고 트래픽 환경에서 당첨 리워드의 중복 지급 및 초과 발급 방지
    - dau 10% - 8만 유저 시뮬레이션
    - 회차별 복권 96만개
- 실시간 당첨 현황 (1등, 2등, 3등 등등..) 제공

## 아키텍쳐

아래 플로우 차트가 보이지 않으시다면 깃허브에서 봐주세요.
[link](https://github.com/jyc228/my-engineering-portfolio/blob/main/bunjang/lottery.md#%EC%95%84%ED%82%A4%ED%85%8D%EC%B3%90)

```mermaid
flowchart
    USER <-- 광고 보고 복권 긁기\n실시간 당첨 현황 보기 --> API_SERVER
    BATCH_SERVER -- 생성 --> 회차
    BATCH_SERVER -- 생성 --> 고보상
    BATCH_SERVER -- 복권 담청 구조 생성 --> LIST
    LIST -- redis pop, 유저에게 복권 할당 --> API_SERVER
    HASH -- 실시간 데이터 조회 --> API_SERVER
    API_SERVER -- 고보상 당첨시 실시간 데이터 갱신 --> HASH
    API_SERVER -- 생성 --> 참여현황
    API_SERVER -- 고보상 당첨시 갱신 --> 고보상

    subgraph REDIS
        subgraph LIST
            N_회차([key: N회차, value: 꽝, 꽝, 꽝, 2등, 꽝, 1등, ...])
        end
        subgraph HASH
            실시간_데이터_버전
            실시간_데이터
        end
    end

    subgraph MYSQL
        회차
        고보상
        참여현황
    end
```

## 주요 기여

### Redis List를 활용한 당첨 시스템 설계

당첨 여부를 API 호출 시점에 DB에서 계산하는 대신, 배치 프로세스가 미리 생성한 '당첨 슬롯'을 Redis List에 적재하고 유저는 이를 꺼내가는(LPOP) 구조를 제안 및 설계함.
이를 통해 DB 트랜잭션 없이 원자성을 보장하며, 분산 락 없이도 중복 당첨 리스크를 근본적으로 제거함.

### 재현 가능한 당첨 슬롯 생성 배치 개발

lazy 자료 구조를 사용하여 96만개를 전부 메모리에 적재하지 않고 일정 단위로 처리할 수 있도록 구현.
회차별 고유 랜덤 시드를 저장 및 사용함으로써 언제든 당첨 구조를 재현할 수 있도록 구현.

```kotlin
fun generate(random: Random): Sequence<LotteryRewardSlot> {
    // 발급 가능한 등수별 남은 개수, rewards 는 발급 가능한 보상 개수로 정렬되어 있습니다.
    val rewardCount = rewards.associate { it.rewardId to it.maxCount }.toMutableMap()
    return generateSequence(1) { it + 1 }
        .map { order ->
            val interval = rewardByInterval.entries.find { order % it.key == 0 }
            if (interval == null) {
                LotteryRewardSlot(order, selectRewardId(random, rewardCount))
            } else {  // 고정된 슬롯 보상
                LotteryRewardSlot(order, interval.value.rewardId)
            }
        }
        .take(960000)
}

private fun selectRewardId(random: Random, rewardCount: MutableMap<Long, Int>): Long {
    val index = random.nextInt(1, rewardCount.values.sum() + 1)
    var offset = 0
    for ((id, remainRewardCount) in rewardCount) { // 발급 가능한 보상 개수가 낮은 순서부터 루프 진행
        offset += remainRewardCount
        if (index <= offset) {
            rewardCount[id] = remainRewardCount - 1
            if (rewardCount[id] == 0) {
                rewardCount -= id
            }
            return id
        }
    }
    error("슬롯 생성 실패. $rewardCount")
}
```

### 다중 계층 캐싱 시스템 구축

회차 정보 및 보상 스키마 등 참조가 빈번한 데이터를 위해 Local Cache와 Redis를 조합한 다중 계층 캐시 구조를 설계하여 DB I/O를 최소화함.
스케줄러 기반의 캐시 갱신 로직을 통해 실시간 데이터 정합성을 유지함.

### 실시간 당첨 현황 관리

당첨 이벤트 발생시 레디스를 통해 버전 기반 스냅샷을 생성하여, 클라이언트가 변경분만 수신하며 현재 당첨 현황을 실시간으로 동기화할 수 있도록 지원함.
실시간 당첨 데이터의 제공은 아래와 같은 이유로 소켓 대신 http N 초 풀링으로 구현했음.

1. 실시간이 아니어도 된다. (실시간일수록 좋음)
2. 스파이크성 트래픽이 예상되기 때문에 커넥션 유지하는 방식은 피하고 싶다.

### 장애 복구 설계

Redis 장애로 List 데이터가 유실될 경우를 대비한 복구 전략을 설계했습니다.

1. **메인테인 모드 도입**: 상태 변경 API를 전면 차단
2. **DB 기준 복구 배치**: DB의 참여현황을 기준으로 미사용 복권만 계산하여 Redis List에 재적재
3. **메인테인 모드 해제**: 서비스 재개

재현 가능한 랜덤 시드를 저장해둔 덕분에 복구 시에도 동일한 당첨 구조를 재현할 수 있습니다.

> 일정상 구현은 미완성이며, 복구 시나리오와 배치 코드는 설계 완료된 상태입니다.

## 설계자/리뷰어로서의 역할

- api 설계 가이드 : 광고 시청을 API에 직접 결합하려는 시도를 막고, "티켓을 획득하고 긁는다"는 도메인 본질이 드러나도록 설계를 유도했습니다.
  이를 통해 무료 티켓, 이벤트 티켓 등 티켓 획득 방식이 추가되어도 긁기 API는 변경 없이 재사용 가능한 확장성을 확보했습니다.
- 아키텍처 가이드: 실시간 서비스와 배치 프로세스의 역할을 명확히 분리하여 결합도를 낮추고 각 시스템의 안정성을 확보했습니다.
- 품질 관리: 팀 내 3명의 백엔드 개발자 중 설계 및 전체 코드 리뷰를 주도하며 기술적 가이드라인을 제시했습니다.

## 결과

- 서비스 런칭 후 회차당 2만명 참여, 월 매출 3천만원 규모로 안정적으로 운영 중
- 고트래픽 설계 기준(8만 유저) 대비 실제 트래픽에서 중복 당첨 0건
