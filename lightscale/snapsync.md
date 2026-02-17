# p2p 네트워크간 데이터 동기화 기술 구현

## 프로젝트 개요

- 기간: 2024.4 - 2024.10 (약 6개월)
- 인원: 2명
- 핵심 기술: golang
- 주요 역할: 설계, 개발, 지식 전파

`geth` 에는 `스냅싱크` 라는 기능이 있습니다. 신규 노드가 네트워크에 참여할 때 전체 이력을 검증하는 대신,
최신 상태(State)의 스냅샷만 동기화하여 노드 준비 시간을 획기적으로 단축하는 핵심 기능입니다.
2023년 이더리움 메인넷 기준 이 기능을 사용여부에 따라 준비 완료까지 1달 이상의 차이가 발생합니다.

당시 조직의 `geth`는 이더리움의 핵심이라고 할 수 있는 low level 자료구조를(이하 `trie`) 를 다른 구현체를 사용했습니다. 이로 인한 여파중 하나로, `스냅싱크`가 동작하지 않고 있었습니다.
이는 블록체인의 이념상 중요한 문제였고, 저는 이 기능을 다시 활성화 하는 작업을 할당받았습니다.

## 주요 해결 과제

### trie 이해 및 재구현

`스냅싱크` 의 핵심중 하나는, `상태를 구축하기 위하여 트랜잭션을 실행하지 않는다.` 라는 것입니다.
이를 위해서 `trie` 의 `leaf node` 를 직접 순회하면서, 데이터를 받고, 검증하는 방식을 사용하고 있습니다.

`geth` 의 low level 로 내려갈수록 인터페이스보단 구조체에 직접적으로 의존하는 코드가 많아지는데, `스냅싱크`의 하단부분은 `mpt`와 강결합 되어 있는 상태였습니다.

#### 핵심 구현 내용

- **`Iterator Key` 개념 도입** : `mpt` 는 키를 기반으로 트리 구조가 결정됩니다. 스냅싱크는 `mpt` 의 노드를 전위순회 하고, `범위 조회`도 사용합니다.
  그러나 `zkrie` 로 구축된 노드들은 정렬된 순서가 `mpt` 와 동일하지 않았습니다. 이는 즉, `범위 조회`를 할 수 없음을 나타냅니다.
  그렇기 때문에 `zktrie` 의 key 를 `스냅싱크` 프로토콜에 호환되게 하기 위하여, 정렬 가능한 키로 변환 해야 했습니다.
  코어 소스의 변경을 최소화 하기 위하여, `Iterator Key`라는 개념을 도입하였습니다.

```golang
package trie

import (
	zkt "github.com/kroma-network/zktrie/types"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/trie/zk"
)

// File for defining an iterator key and managing key <-> iterator key conversion functions.
//
// NodeIterator.LeafKey must satisfy the following characteristics
// ```
// preordertraversal(tree).filter(x -> x is leaf).map(x -> x.leafKey) == sort(leafKeys)
// ```
// Since Trie satisfies this condition, functions that utilize it do not require any additional code (e.g. snapsync).
// However, zktrie does not, so unless we introduce a new key concept, we need to modify the core source code quite a bit.
// Therefore, we introduced the concept of iteratorKey, which satisfies the above condition, and implemented functions to convert key and iteratorKey.

func HashToIteratorKey(hash common.Hash, isZk bool) common.Hash {
	if isZk {
		return HashToZkIteratorKey(hash)
	}
	return hash
}
... 이하 생략
```

- **$2^{512}$ `trie` 키 공간의 분할 및 부분 검증 로직 재구현** : 이더리움의 상태 데이터(계정 및 컨트랙트)는 거대한 키 공간을 가집니다.
  이를 P2P로 전송하기 위해서는 데이터를 적절한 크기로 분할하고, 다운로드된 각 조각이 전체 트리의 일부로서 유효한지 검증하는 Proof 생성/검증 로직이 필수적입니다.
  `geth` 내의 기존 검증 코드는 MPT의 노드 레이아웃과 증명 방식에 고도로 최적화(강결합)되어 있어, `zktrie` 환경에서는 그대로 사용할 수 없었습니다.
  이를 해결하기 위하여 `mpt` 맞춤형으로 구현된 증명 로직을 분석 및 재구현 했습니다. 키 공간을 기반으로 구간을 나누고,
  각 구간의 Proof를 생성하여 P2P 피어(Peer) 간의 데이터 무결성을 보장하도록 프로토콜 레이어를 완성했습니다.

### 스냅샷 자료구조 커스텀

이더리움의 `스냅샷 자료구조`에 `zktrie` 가 호환되도록 병합했습니다. 이 또한 범위 질의가 필수이며, `mpt` 를 기반으로 강결합 되어 있었습니다.
그러나 `Iterator Key` 덕분에 변경을 최소화 할 수 있었으며, 이는 역설적이게도 깊은 이해를 했기 때문에 가능했습니다.

```
이더리움 스냅샷 자료구조 모델링
push new head -> [memory[ head ], memory[ head - 1 ], ..., memory[ head - 127 ], disk[ head - 128 ]]
                                                                           \___ 병합 ___/
```

#### 핵심 구현 내용

- **`disk` 계층 `zktrie node` io 처리** : 메모리에 모든 데이터를 적재할 순 없기 때문에, `disk` 계층이 존재하며, 여기엔 `zktrie node` 를 저장해야 했습니다.
  변경을 최소화 하기 위하여, `bool` 파라미터 한개로 이를 해결했습니다.
- **스냅샷 자료구조 활성화** : `스냅샷 자료구조`는 `스냅싱크` 뿐만 아니라 다양한 용도로 활용됩니다.
    - `Cache` 활성화 : 이더리움의 주요 로직(블록 생성, 트랜잭션 시뮬레이션, 등등..) 은 대부분 현재 상태를 필요로 합니다. `스냅샷 자료구조`는 이를 완벽히 지원합니다.
    - `Pruning` 활성화 : 과거 데이터 삭제 기술 활성화로 디스크 사용량 절약합니다. 이는 `스냅샷 자료구조`가 128 블록까지의 상태를 저장한다라는 기술적 특징에 기반하여,
      128 블록보다 더 과거 데이터는 리프노드를 순회하며 disk 에서 삭제합니다.

### P2P 스냅싱크 프로토콜 통합 및 테스트 자동화

커스텀된 `trie` 검증 로직과 `스냅샷 자료구조`를 실제 P2P 통신 레이어에 병합했습니다.
`Iterator Key` 덕분에 변경을 최소화 할 수 있었으며, 이는 역설적이게도 깊은 이해를 했기 때문에 가능했습니다.

#### 핵심 구현 내용

- **분산 범위 기반 동기화** : `trie`를 16진수 공간 기반으로 분할하고, 다수의 피어에게 각 범위에 해당하는 `leaf node`를 병렬로 요청합니다.
  다운로드된 데이터는 `Iterator Key`를 통해 정렬 순서가 보장된 상태에서 Trie 구축 및 검증을 거쳐 디스크에 영속화됩니다.
- **추상화를 통한 테스트 가동성 확보** : 기존 테스트 코드가 `mpt` 구조에 강결합 되어 있어 `스냅싱크를` 검증하기 어려운 상태였습니다.
  `golang` 은 덕타이핑이 가능한 언어였기 때문에, 이를 활용하여 test 수정을 최소화 하며 `mpt`, `zktrie` 둘 다 테스트 했습니다.

## 결과

- Sync 시간 최적화: 신규 노드 기동 시 동기화 완료까지 소요되는 시간을 2주 이상에서 2일 이내로 단축했습니다.
- 기술 내재화: `geth` 프로토콜의 하위 레이어를 직접 제어함으로써 팀 내 블록체인 코어 기술 자립도를 학보했습니다.
- 조금씩 분석 및 구현하면서 얻은 지식과, 개인적으로 하고 있던 `geth` 구현 프로젝트를 통해, `zktrie` 는 심각한 성능 결함이 있음을 사전에 알 수 있었고,
  이는 [trie 재구현](./zktrie.md) 작업으로 이어지게 되었습니다.

### 용어집

- geth: `Go-Ethereum` 의 약자로, 이더리움 프로토콜의 공식 Go 구현체이자 가장 널리 쓰이는 클라이언트 노드 소프트웨어입니다.
- mpt: `merkle patricia trie` 의 줄임말로 미들 노드가 16개인 트리 입니다.
- zktrie: 이진 바이너리 해시 트리 입니다.