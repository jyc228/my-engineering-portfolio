# 미션 크리티컬 시스템 경험: 블록체인 코어 최적화

## 프로젝트 개요

- 기간: 2023.10 - 2024.2 (약 4개월)
- 인원: 2명
- 핵심 기술: golang
- 주요 역할: 설계, 개발, 지식 전파
- 상위 문서: [maintenance.md](maintenance.md)

## 한 줄 요약

성능 결함을 6개월 전에 예견하고, 실제 장애 발생 시 원인을 즉시 진단하여 코어 자료구조를 전면 재설계 → 플랫폼 처리량 300% 향상, 6시간 체인 중단 사고 근본 해결

## 배경 & 도전

조직의 블록체인 코어에서 사용 중인 자료구조(`zktrie`)가 트래픽이 증가하거나 데이터가 누적될수록 지수적으로 성능이 저하되는 구조적 결함을 발견했습니다.

당시 공식 업무는 [스냅싱크](./snapsync.md) 구현이었고, 코어 트리 재구현은 제 업무가 아니었습니다.
하지만 장기적으로 시스템 전체가 위험하다고 판단하여 팀을 설득, 전면 재구현으로 방향을 바꿨습니다.
그러나 리스크를 최소화하기 위해 상대적으로 안전한 스냅싱크 기능에서만 사용하는 것으로 결정했습니다.

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
지수적으로 증가하는 연산 부하가 발생하여 전체 체인의 성능을 저해하는 시스템 취약점을 발견했습니다.

이를 해결하기 위해 자식 노드를 인터페이스화하여 해시 재계산 시점을 커밋(Commit) 단계로 지연시키는 구조로 트리를 전면 재설계하였고, 결과적으로 시스템의 고가용성을 확보했습니다.

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

2024년 1월경 트래픽 폭증으로 체인이 tps 10을 견디지 못하고 약 6시간 중단되었습니다.
팀 내 원인 파악이 안 되는 상황에서 저는 즉시 zktrie 성능 이슈로 진단, 인스턴스 스펙 상향으로 임시 복구하고 재구현 중이던 트리를 빠르게 완성해 적용했습니다.

## 2차 장애: 신규 트리 버그로 인한 합의 실패

> 기술적인 배경 설명: 블록체인은 모두가 master 노드이지만, L2 라고 불리우는 체인들은 master, slave 구조라고 보셔도 좋습니다.
> 그리고 master 노드는 (보통 시퀀서라고 부릅니다) 회사가 운영하게 됩니다.

약 2개월 뒤, 아침 10시에 체인은 합의에 실패했습니다. 문제 예상이 안되는 상황에서 저는 **업그레이드된 우리 노드에서만 문제가 발생했다**는 사실에 집중했습니다.
제가 만든 트리를 적용한 패치 때문에 db 가 오염됐을 가능성을 배제할 수 없었고,
조직에서 운영중인 모든 노드는 업그레이드를 한 상태였기 때문에 **업그레이드를 하지 않은 파트너사의 DB로 교체하여 재기동하자**고 제안하여 긴급 대응을 주도했습니다.
여기까지 약 3시간 소요되었으며 이 이후부턴 팀원들과 같이 모니터링, 원인분석을 하면서 체인을 정상화 할 수 있도록 노력했습니다. 약 18시간 후 다행히 체인은 정상화 되었습니다.

버그의 원인은 **깊이 1의 노드를 삭제할 때 발생하는 로직 결함**으로 밝혀졌습니다.
이 장애를 해결하는 과정에서 저희 팀은 실패를 투명하게 공유하고 함께 배우는 문화를 바탕으로, 모든 팀원이 참여하여 원인을 분석하고 재발 방지 대책을 수립했습니다.
당시의 치열했던 고민과 기술적 분석을 담은 **저희 팀의 공식 사후 분석 보고서(Post-Mortem)** 는 아래 에서 확인하실수 있습니다.

- [zktrie-postmortems.md](zktrie-postmortems.md)
- [zktrie-postmortems github](https://github.com/kroma-network/kroma/blob/acd78a9bafb79ba1b34b1d1d9b4ad8f96b31dea3/postmortems/2024-03-12-zktrie-hardfork.md)

수정후 재 적용할땐 훨씬 조심스럽게 적용했습니다. 문제가 없다고 판단이 되자, 시퀀서에도 적용을 했고 차후 다른 zktrie 기반 체인들은 성능 저하 문제를 겪을때, 우리는 아무런 문제도 없게 되었습니다.

## 결과

- 플랫폼 전체 처리량 300% 향상
- commit 미수행 시 최대 성능 1000% 향상
- 디스크 I/O, 블록 생성 속도 등 전 지표에서 개선

## 이후

저는 이 후 조직의 백엔드로 포지션 변경했습니다. 그 뒤 조직은 `tree` 를 `zktrie` -> `mpt` 로 변경을 검토하는 매우 중요한 판단을 하려고 했습니다.
저는 여기에서 `mpt` 로 변경하는게 좋겠다. 라고 의견을 다음과 같은 근거를 들며 제시했습니다.

- 이더리움은 앞으로도 계속 개선될 것인데.. 이 패치를 온전히 받기 매우 어렵다.
- 이더리움의 모든 테스트는 기반이 `mpt` 이다. 테스트는 다양한 구현체를 받을 수 없도록 되어 있었기 때문에 실행되는 테스트 중 약 80% 이상의 테스트 케이스는 의미가 없다.
- 내가 개선한 트리로 인하여 성능 문제는 해결하였지만 근본적으로 구조차이로 인한 물리적인 한계를 뛰어넘을순 없다. (자식 개수, depth 깊이, 디스크 사용량)

이런 의견까지 더해져서 결국 조직은 db 마이그레이션을 하기로 결정했습니다.
그리고 저는 마이그레이션 전략, 기반 코드를 작성해서 팀원에게 전달 했고, 중간중간 구현 방향을 가이드 했습니다. 그 팀원은 이더리움에 성공적으로 병합하여 최종적으로 마이그레이션에 성공했습니다.

## 참고

- tree 구현 : https://github.com/kroma-network/go-ethereum/pull/45
- tree 버그 수정 : https://github.com/kroma-network/go-ethereum/pull/81
- snap sync 구현 : https://github.com/kroma-network/go-ethereum/pull/19