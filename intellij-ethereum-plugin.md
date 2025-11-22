# IntelliJ Ethereum Plugin: 통합 블록체인 개발 환경 구축

## 배경 & 도전

이더리움 개발자는 코드를 작성하고(IDE), 실행하고(Terminal), 데이터를 확인하고(Etherscan), 디버깅하는(Remix/Log) 과정에서
끊임없이 **컨텍스트 스위칭(Context Switching)** 을 겪습니다.

저는 이 파편화된 도구 체인이 개발 생산성을 저해하는 가장 큰 병목이라 정의하고, **JetBrains IDE의 강력한 기능 안에서 모든 워크플로우를 완결하는 통합 환경**을 구축하고자 했습니다.

## 아키텍쳐

기존의 무거운 노드(`geth`)나 외부 라이브러리에 의존해서는 IDE에 걸맞은 가벼움과 반응성을 확보할 수 없었습니다. 따라서 Core부터 UI까지 전 계층을 직접 설계하고 구현하는 수직 통합 전략을 택했습니다.

- Layer 1 - Core Engine ([keth](keth.md)): 로컬 시뮬레이션을 위해 `evm`(가상머신)과 `StateDB`를 직접 구현하여, 외부 네트워크 없이도 트랜잭션을 실행하고 되돌릴 수 있는
  환경을 마련했습니다.
- Layer 2 - Data Pipeline ([keth-client](keth-client.md)): IDE의 UI 스레드를 차단하지 않기 위해, Coroutines와 Request Batching을 활용한
  비동기 데이터 파이프라인을 구축했습니다.
- Layer 3 - User Interface (`Plugin`): IntelliJ Platform의 `VirtualFileSystem(VFS)`을 확장하여, 블록체인 데이터를 마치 로컬 파일처럼 탐색하고 에디터
  기능을 활용할 수 있게 했습니다.

## 주요 / 예정된 기능

이 플러그인은 크게 2가지 개발 목표가 있습니다.

- `IntelliJ` 의 `Database` 와 최대한 비슷하게 만들어서 추가 학습을 최소한으로 높은 사용자 경험을 주는것을 목표로 삼았습니다.
- Major language 의 디버거 플러그인과 동일한 수준의 `Solidity(스마트 컨트랙트)` 디버거를 구현하는것을 목표로 삼았습니다.

### 1. 가상 파일 시스템(VFS) 기반 블록체인 탐색기

단순히 데이터를 테이블로 보여주는 것을 넘어, 이더리움의 블록, 트랜잭션 데이터를 IntelliJ의 에디터 탭에서 파일처럼 열람할 수 있도록 `VirtualFileSystem`을 구현했습니다.

이 기능을 구현할때, `keth-client` 에서 만든 `batch` 와, `interval` 속성을 통해 http call 을 최적화 할 수 있었습니다.
그 중 예로 `transaction` 상세를 조회하는 방법은 block에 저장된 모든 `transaction` 을 한번에 조회하거나,
`block` 과 `transaction` index 로 한개씩 조회하는 방법밖에 없습니다.
ui 에서 사용자 경험을 줄이기 위하여, 2단계 `batch` 호출로 `transaction` 페이징 처리를 했습니다.

```kotlin
// 하단 nextPage 에서 결정된 데이터로 transactions 를 조회하는 batch 요청을 보냅니다.
private fun launchUpdateUI(page: suspend () -> List<Pair<ULong, Int>>): Job = launchIO {
    val transactions = network.client.batch {
        page().map { (bn, txIndex) -> eth.getTransactionByBlockNumberAndIndex(bn, txIndex).map { it } }
    }.awaitAllOrThrow()

    invokeLater { model.initRows(transactions) }
}

// 하단 nextFlow 에서 pageSize 만큼 방출시킵니다. page size 만큼 어떤 블록의 몇번째 트랙잭션을 가져와아 하는지 알 수 있게 됩니다.
// [(10, 49), (10, 48), ..., (10, 0), (9, 29), ...]
suspend fun nextPage(): List<Pair<ULong, Int>> = nextFlow().take(pageSize).toList().updatePager()

private fun nextFlow(): Flow<Pair<ULong, Int>> = flow { // lazy stream 생성
    var blockNumberCursor = lastRow.blockNumber() ?: initBlockNumber()
    while (true) {
        val blockTxCounts = fetchBlockTxCount(blockNumberCursor - 19uL..blockNumberCursor).asReversed()
        blockTxCounts.forEach { (bn, txCount) -> // [(10, 50), (9, 30), (8, 16), ...]
            ((txCount - 1) downTo 0).forEach { txIndex -> emit(bn to txIndex) } // [(10, 49), (10, 48), (10, 47), ...]
        }
        blockNumberCursor = blockTxCounts.last().first - 1uL
    }
}

// block 에 실린 tx 개수를 http call 1번으로 조회합니다. 
private suspend fun fetchBlockTxCount(range: ULongRange) = client.batch {
    range.map { bn -> eth.getBlockTransactionCountByNumber(bn).map { bn to it.number.toInt() } }
}.awaitAllOrThrow()
```

### 2. Solidity 디버거 (예정)

스마트 컨트랙트의 실행 및 디버깅 환경을 혁신하기 위해, 로컬 시뮬레이션 아키텍처를 설계했습니다.
이를 위해 `keth` 프로젝트에서 `OffchainStateDatabase`(필요한 상태값만 메인넷에서 가져오는 가상 DB)와 확장 가능한 `EVMInterpreter를` 직접 구현했습니다.

핵심 엔진(Core)과 데이터 파이프라인의 기술적 난제들은 모두 해결하여 로컬 디버깅의 기술적 가능성(PoC)을 검증했습니다.
이후의 Swing UI 패널 구성 작업은 기술적 챌린지보다는 단순 반복 작업의 비중이 높아, 핵심 기술 검증을 마친 상태에서 프로젝트를 마무리했습니다.

## 최종 목표

이 플러그인을 설치하면, 아래와 같은 기능을 제공하고 싶었습니다.

- 프로젝트 내 스마트 컨트랙트 ABI 를 추출하여, `transaction` input 과, output 을 자동으로 디코딩 (지금도 미리 등록한 abi 는 가능)
- 내가 관심있는 계정 및 스마트 컨트랙트 의 변경사항 구독 (balance, nonce 는 실시간 변경 구독 가능)
- 스마트 컨트랙트 메모리 뷰어
- 스마트 컨트랙트 디버깅 (major language debugger 수준)
- 경량화된 내장 노드 제공

기술적으로 어려운 부분들은 개발 완료된 상태였기 때문에 충분히 가능성이 있다고 생각했고 모두 완성되었을때, 월 만원 정도의 구독을 생각했습니다.

## 프로젝트 의의 및 마무리

이 프로젝트는 **블록체인 Core(VM)부터 개발 도구(IDE Plugin)까지 풀 스택을 직접 설계하고 구현**해 본 경험이었습니다.

- 기술적 성취:
    - EVM과 StateDB를 직접 구현하여 로컬 시뮬레이션 환경을 구축했습니다.
    - IDE의 성능 제약 안에서 대용량 블록체인 데이터를 처리하기 위해 클라이언트 라이브러리 레벨부터 최적화를 수행했습니다.
- 마무리:
    - 핵심 엔진(Core)과 데이터 파이프라인(Client)의 기술적 난제들은 모두 해결하여 기술 검증(PoC)을 완료했습니다.
    - 이후의 UI 패널 구성 작업은 단순 반복적인 성격이 강하고, 현재는 블록체인 외의 분야로 커리어 방향을 전환하였기에 프로젝트를 현재 상태로 동결(Archive)하였습니다.

비록 상용화까지 가지는 않았지만, 이 프로젝트를 통해 복잡한 도메인 지식을 개발자 도구(Tooling)로 풀어내는 엔지니어링 역량을 확보할 수 있었습니다.
