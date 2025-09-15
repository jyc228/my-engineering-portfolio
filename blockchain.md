# 미션 크리티컬 시스템 경험: 블록체인 코어 최적화

## 배경 & 도전

회사에서 운영중인 블록체인은 low level tree 를 표준과는 다른 구현체를 쓰고 있었습니다. 그러다 보니 몇몇 중요한 기능을 사용할 수 없었으며 대표적으로 스냅샷 동기화(스냅싱크)가 있었습니다.

스냅싱크는 새롭게 실행된 노드가 최신 상태에 빠르게 동기화 할 수 있도록 도와주며, 디스크 사용량도 약 절반정도 줄여주는 중요한 기능입니다.
저는 스냅싱크를 구현하라는 과제를 받았고, 분석 설계 및 구현을 조금씩 하다 보니.. 사용중인 tree 가 심각한 성능 문제를 발생시킬 수 있다라고 판단하게 되었습니다.
팀에게 이 사실을 알렸고, 저의 의견은 전체 재구현 해야 한다. 였습니다. 결과적으론 재구현을 하는 방향으로 설정되었습니다만.. 당시엔 스냅싱크에서만 사용할 것을 가정했습니다.

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
이러한 구조로 인하여 tps 가 낮거나, db 에 데이터가 쌓이면 쌓일수록 체인이 받는 부하가 월등히 높아질 있다고 저는 예견했습니다.

핵심 개선사항은 child node 를 인터페이스로 바꾸고, hash 계산을 commit 할때만 하는것입니다.

```go
type TreeNode interface {
	Hash() *zkt.Hash

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
	hash   *zkt.Hash
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

## 결과

성능 문제에 대한 저의 예측은 옳았습니다. 2024년 1월경, 기존 시스템은 tps 10을 견디지 못하고 체인이 약 6시간 동안 중단되는 심각한 사태가 발생했습니다.
단기적으로는 인프라적으로 해결했으나 이건 근본적인 해결책은 되지 못하였습니다. 제가 구현중인 트리가 근본적인 해결책이었기 때문에 마무리 단계에 있던 트리를 구현 완료하고
내부 검증, 외부 업체 코드 감사 등 모든 만반의 준비를 거친 후 메인넷에 적용했습니다.

새롭게 구현된 트리의 성능은 commit 안 할 경우, 최대 성능이 1000% 향상 되었으며 플랫폼 전체적인 처리 속도도 300% 증가하게 되었습니다.
disk io, 블록 생성 속도 등 모든 지표에서 우월함이 증명되었습니다.

## 위기 및 극복

> 기술적인 배경 설명: 블록체인은 모두가 master 노드이지만, L2 라고 불리우는 체인들은 master, slave 구조라고 보셔도 좋습니다.
> 그리고 master 노드는 (보통 시퀀서라고 부릅니다) 회사가 운영하게 됩니다.

약 2개월 뒤, 아침 10시에 체인은 합의에 실패했습니다. 저 포함 모든 팀원이 문제 예상이 안되는 상황에서 저는 회사가 돌린 노드만 장애가 발생한 것을 보며 우리 시퀀서가 문제다.
라고 파악했습니다. 트리 구조를 변경한 패치 때문에 db 가 오염됐을 가능성을 배제할 수 없었고, 회사의 모든 노드는 업그레이드를 한 상태였기 때문에 파트너사의 db 를 가져와서 재기동 하자고 제안을 했습니다.
여기까지 약 3시간 소요되었으며 이 이후부턴 팀원들과 같이 모니터링, 원인분석을 하면서 체인을 정상화 할 수 있도록 노력했습니다. 약 18시간 후 다행히 체인은 정상화 되었습니다.

버그의 원인은 **트리 노드의 얕은 복사로 인한 멀티 스레드 간 데이터 동기화 문제**로 밝혀졌습니다.

수정후 재 적용할땐 훨씬 조심스럽게 적용했습니다. 문제가 없다고 판단이 되자, 시퀀서에도 적용을 했고 차후 다른 zktrie 기반 체인들은 성능 저하 문제를 겪을때, 우리는 아무런 문제도 없게 되었습니다.

## 이후

저는 이 후 블록체인 커리어를 포기하면서 조직의 백엔드로 포지션 변경했습니다. 그 뒤 조직은 tree 를 zktrie -> mpt 로 변경을 검토하는 매우 중요한 판단을 하려고 했습니다.
저는 여기에서 mpt 로 변경하는게 좋겠다. 라고 의견을 다음과 같은 근거를 들며 제시했습니다.

- 이더리움은 앞으로도 계속 개선될 것인데.. 이 패치를 온전히 받기 매우 어렵다.
- 이더리움의 모든 테스트는 기반이 mpt 이다. 테스트는 다양한 구현체를 받을 수 없도록 되어 있었기 때문에 실행되는 테스트 중 약 80% 이상의 테스트 케이스는 의미가 없다.
- 내가 개선한 트리로 인하여 성능 문제는 해결하였지만 근본적으로 구조차이로 인한 물리적인 한계를 뛰어넘을순 없다. (자식 개수, depth 깊이, 디스크 사용량)

이런 의견까지 더해져서 결국 조직은 db 마이그레이션을 하기로 결정했습니다.
그리고 저는 마이그레이션 전략, 기반 코드를 작성해서 팀원에게 전달 했고, 중간중간 구현 방향을 가이드 했습니다. 그 팀원은 이더리움에 성공적으로 병합하여 최종적으로 마이그레이션에 성공했습니다.
