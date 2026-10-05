# 블록체인 코어 트리(zktrie) 성능 개선

## 프로젝트 개요

- 기간: 2023.10 - 2024.2 (약 4개월)
- 인원: 2명
- 핵심 기술: golang
- 주요 역할: 설계, 개발, 지식 전파
- 상위 문서: [maintenance.md](maintenance.md)

## 한 줄 요약

zktrie의 구조적인 성능 문제를 미리 발견해 재구현을 진행. 트래픽 급증으로 체인이 약 6시간 중단됐을 때 재구현한 트리를 적용해 처리량 300% 향상 (적용 과정에서 삭제 로직 버그로 2차 장애 발생)

## 배경 & 도전

회사 블록체인 코어에서 쓰던 자료구조(`zktrie`)가 트래픽이 늘거나 데이터가 쌓일수록 성능이 크게 떨어지는 구조라는 걸 발견했습니다.

당시 공식 업무는 [스냅싱크](./snapsync.md) 구현이었고, 코어 트리 재구현은 제 업무가 아니었습니다.
하지만 장기적으로 위험하다고 판단해서 팀에 재구현을 제안했고, 리스크를 줄이기 위해 처음에는 상대적으로 안전한 스냅싱크 기능에서만 쓰기로 했습니다.

## 과제

표준 구현체와 당시 회사에서 사용중인 트리는 다음과 같은 차이가 있습니다.

|                | mpt     | zktrie        |
|----------------|---------|---------------|
| child count    | 16      | 2             |
| max depth      | 32      | 256           |
| node structure | pointer | binary db key |

특히, node structure 에서 binary db key 만 트리에 유지시키는 구조로 인하여 다음과 같은 성능 저하가 발생하게 됩니다.

- node read -> disk io 는 캐싱이 되나, node structure 를 매번 디코딩 하므로 cpu, memory 를 계속 사용함.
- node write, delete -> 매 함수 호출마다 리프노드 까지 가는 경로의 모든 미들 노드 hash 재계산, db commit 할때 버려지는 middle node 도 같이 write 됨.
- commit 단계 부재 -> write, delete 할 때 마다 커밋 하는것과 비슷한 효과가 발생. 이더리움에선 write, delete 는 매우 가벼워야 하는데 플랫폼과 맞지 않음.

zktrie 는 바이너리 트리 이므로 자식노드 개수도 적고, 미들 노드의 깊이도 훨씬 길어지므로, write, delete 연산이 여러번 있을 경우, 중복된 경로가 발생할 확률이 월등히 높아졌습니다.
이러한 구조로 인하여 tps 가 높거나, tps 가 낮아도 db 에 데이터가 누적될수록 (평균 트리 depth 증가)
연산 부하가 급격히 늘어나 체인 전체 성능이 떨어지는 구조였습니다.

그래서 자식 노드를 인터페이스로 만들어 메모리에 유지하고, 해시 재계산을 커밋(Commit) 시점으로 미루는 방식으로 트리를 다시 구현했습니다.

```go
type TreeNode interface {
	Hash() [32]byte

	// CanonicalValue returns the byte form of a node required to be persisted, and strip unnecessary fields
	// from the encoding (current only KeyPreimage for Leaf node) to keep a minimum size for content being
	// stored in backend storage
	CanonicalValue() []byte
}

type HashNode [32]byte

// childL, childR 은 최초 디스크에서 로드시 HashNode 입니다. 한번이라도 접근시 디코딩 된 후 (MiddleNode, LeafNode) 그 구조가 계속 메모리에 유지됩니다.
type MiddleNode struct {
	childL TreeNode 
	childR TreeNode
	hash   [32]byte
}

type LeafNode struct { ... }

// 기존 구현 입니다. 
// 이 구조체가 MiddleNode, LeafNode 둘 다 처리하며, Type 필드로 유형을 확인합니다.
// 보시다시피 ChildL, ChildR 이 단지 [32]byte hash 인 db key 입니다. 
// 이로 인하여 트리 구조가 변경될 때 마다 (update, delete) leaf node 까지 경로에 있는 모든 middle node 의 hash 를 재계산 해야 합니다.  
type Node struct {
	// Type is the type of node in the tree.
	Type NodeType
	// ChildL is the node hash of the left child of a parent node.
	ChildL [32]byte
	// ChildR is the node hash of the right child of a parent node.
	ChildR [32]byte
	...
}
```

## 1차 장애 : 메인넷 성능 하락

2024년 1월경 트래픽이 급증하면서 체인이 tps 10을 버티지 못하고 약 6시간 중단되었습니다.
원인이 바로 드러나지 않던 상황에서 zktrie 성능 문제로 판단했고, 인스턴스 스펙을 올려 임시 복구한 뒤 재구현 중이던 트리를 마무리해 적용했습니다.

## 2차 장애: 신규 트리 버그로 인한 합의 실패

> 기술적인 배경 설명: 블록체인은 모두가 master 노드이지만, L2 라고 불리우는 체인들은 master, slave 구조라고 보셔도 좋습니다.
> 그리고 master 노드는 (보통 시퀀서라고 부릅니다) 회사가 운영하게 됩니다.

약 2개월 뒤, 아침 10시에 체인이 합의에 실패했습니다. 원인을 모르는 상황에서 **업그레이드한 우리 노드에서만 문제가 발생했다**는 점을 보고,
제가 만든 트리가 들어간 패치 때문에 DB가 오염됐을 가능성이 있다고 판단했습니다.
회사가 운영하는 노드는 모두 업그레이드된 상태였기 때문에 **업그레이드하지 않은 파트너사의 DB로 교체해서 재기동하자**고 제안했습니다.
여기까지 약 3시간이 걸렸고, 이후 팀원들과 모니터링과 원인 분석을 같이 하면서 약 18시간 만에 체인이 정상화되었습니다.

버그의 원인은 **깊이 1의 노드를 삭제할 때 발생하는 로직 결함**으로 밝혀졌습니다.
삭제 로직에 대한 테스트가 없어서 미리 잡지 못했습니다. 팀 전체가 원인 분석과 재발 방지 대책에 참여했고,
당시 팀에서 공개한 사후 분석 보고서(Post-Mortem)는 아래에서 보실 수 있습니다.

- [zktrie-postmortems.md](zktrie-postmortems.md)
- [zktrie-postmortems github](https://github.com/kroma-network/kroma/blob/acd78a9bafb79ba1b34b1d1d9b4ad8f96b31dea3/postmortems/2024-03-12-zktrie-hardfork.md)

수정 후에는 훨씬 신중하게 단계적으로 적용했고, 문제가 없는 걸 확인한 뒤 시퀀서에도 적용했습니다. 이후 다른 zktrie 기반 체인들이 성능 문제를 겪을 때 저희 체인은 같은 문제가 없었습니다.

## 결과

- 체인 전체 처리량 300% 향상
- commit을 하지 않는 연산 기준 최대 1000% 향상
- 디스크 I/O, 블록 생성 속도 개선

## 이후

이후 저는 백엔드로 포지션을 옮겼고, 그 뒤 회사는 `zktrie`를 `mpt`로 바꿀지 검토하게 되었습니다.
저는 아래 근거로 `mpt`로 바꾸는 게 좋겠다는 의견을 냈습니다.

- 이더리움은 앞으로도 계속 개선될 것인데.. 이 패치를 온전히 받기 매우 어렵다.
- 이더리움의 모든 테스트는 기반이 `mpt` 이다. 테스트는 다양한 구현체를 받을 수 없도록 되어 있었기 때문에 실행되는 테스트 중 약 80% 이상의 테스트 케이스는 의미가 없다.
- 내가 개선한 트리로 인하여 성능 문제는 해결하였지만 근본적으로 구조차이로 인한 물리적인 한계를 뛰어넘을순 없다. (자식 개수, depth 깊이, 디스크 사용량)

결국 팀은 마이그레이션을 하기로 결정했습니다.
저는 마이그레이션 방식과 기반 코드를 작성해서 팀원에게 넘기고 중간중간 구현 방향을 같이 봤고, 그 팀원이 마이그레이션을 마무리했습니다.

## 참고

- tree 구현 : https://github.com/kroma-network/go-ethereum/pull/45
- tree 버그 수정 : https://github.com/kroma-network/go-ethereum/pull/81
- snap sync 구현 : https://github.com/kroma-network/go-ethereum/pull/19