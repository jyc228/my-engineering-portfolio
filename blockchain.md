# 미션 크리티컬 시스템 경험: 블록체인 코어 최적화

## 배경 & 도전

저에게 주어진 초기 과제는, 새로운 서버가 최신 데이터를 빠르게 따라잡고 디스크 사용량을 절반으로 줄여주는 '스냅싱크'라는 중요 기능 구현이었습니다. 하지만 분석 과정에서 저는 이 기능이 의존하는 코어 데이터 구조의 처리 방식이 장기적인 안정성에 큰 위험이 될 수 있음을 발견했습니다.

당시 제게 주어진 공식적인 업무는 '스냅싱크 구현'이었지, '코어 트리 재구현'이 아니었습니다. 이 상황에서 저는 두 가지 선택지를 두고 깊이 고민했습니다. 하나는 주어진 과제인 스냅싱크의 목표 달성에만 집중하는 맞춤형 구현이었고, 다른 하나는 시스템 전체에 적용될 범용적인 코어 트리의 전면 재구현이었습니다. 후자는 훨씬 더 많은 시간과 노력이 필요한 길이었습니다.

저는 기능 하나를 구현하는게 아닌 조직이 운영하는 블록체인의 미래를 예상했습니다. 그러니 전면 재구현이 필수라고 판단했고, 지속적으로 코어 트리의 성능 문제를 예견하며 팀을 설득했습니다. 그 결과 프로젝트는 '전면 재구현'으로 방향이 설정되었지만, 초기에는 리스크를 최소화하기 위해 상대적으로 안전한 스냅싱크 기능에서만 사용하는 것으로 결정했습니다.

이후 저의 우려는 생각보다 빠르게 현실이 되었습니다.

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
이러한 구조로 인하여 tps 가 높거나, tps 가 낮아도 db 에 데이터가 누적될수록 (평균 트리 depth 증가) 체인이 받는 부하가 월등히 높아질 있다고 저는 예견했습니다.

핵심 개선사항은 child node 를 인터페이스로 바꾸고, hash 계산을 commit 할때만 하는것입니다.

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

## 결과

성능 문제에 대한 저의 예측은 옳았습니다. 2024년 1월경, 기존 시스템은 tps 10을 견디지 못하고 체인이 약 6시간 동안 중단되는 심각한 사태가 발생했습니다.
단기적으로는 인프라적으로 해결했으나 이건 근본적인 해결책은 되지 못하였습니다. 제가 구현중인 트리가 근본적인 해결책이었기 때문에 마무리 단계에 있던 트리를 구현 완료하고
내부 검증, 외부 업체 코드 감사 등 모든 만반의 준비를 거친 후 메인넷에 적용했습니다.

새롭게 구현된 트리의 성능은 commit 안 할 경우, 최대 성능이 1000% 향상 되었으며 플랫폼 전체적인 처리 속도도 300% 증가하게 되었습니다.
disk io, 블록 생성 속도 등 모든 지표에서 우월함이 증명되었습니다.

## 위기 및 극복

> 기술적인 배경 설명: 블록체인은 모두가 master 노드이지만, L2 라고 불리우는 체인들은 master, slave 구조라고 보셔도 좋습니다.
> 그리고 master 노드는 (보통 시퀀서라고 부릅니다) 회사가 운영하게 됩니다.

약 2개월 뒤, 아침 10시에 체인은 합의에 실패했습니다. 저 포함 모든 팀원이 문제 예상이 안되는 상황에서 저는 **업그레이드된 우리 노드에서만 문제가 발생했다**는 사실에 집중했습니다.
제가 만든 트리를 적용한 패치 때문에 db 가 오염됐을 가능성을 배제할 수 없었고, 조직에서 운영중인 모든 노드는 업그레이드를 한 상태였기 때문에 **업그레이드를 하지 않은 파트너사의 DB로 교체하여 재기동하자**고 제안하여 긴급 대응을 주도했습니다.
여기까지 약 3시간 소요되었으며 이 이후부턴 팀원들과 같이 모니터링, 원인분석을 하면서 체인을 정상화 할 수 있도록 노력했습니다. 약 18시간 후 다행히 체인은 정상화 되었습니다.

버그의 원인은 **깊이 1의 노드를 삭제할 때 발생하는 로직 결함**으로 밝혀졌습니다.
이 장애를 해결하는 과정에서 저희 팀은 실패를 투명하게 공유하고 함께 배우는 문화를 바탕으로, 모든 팀원이 참여하여 원인을 분석하고 재발 방지 대책을 수립했습니다. 당시의 치열했던 고민과 기술적 분석을 담은 **저희 팀의 공식 사후 분석 보고서(Post-Mortem)** 는 아래 링크에서 확인하시거나, 링크 유실을 대비하여 제일 아래에 별도 첨부 하였습니다.

[zktrie hardfork postmortems](https://github.com/kroma-network/kroma/blob/acd78a9bafb79ba1b34b1d1d9b4ad8f96b31dea3/postmortems/2024-03-12-zktrie-hardfork.md)

수정후 재 적용할땐 훨씬 조심스럽게 적용했습니다. 문제가 없다고 판단이 되자, 시퀀서에도 적용을 했고 차후 다른 zktrie 기반 체인들은 성능 저하 문제를 겪을때, 우리는 아무런 문제도 없게 되었습니다.

## 이후

저는 이 후 블록체인 커리어를 포기하면서 조직의 백엔드로 포지션 변경했습니다. 그 뒤 조직은 tree 를 zktrie -> mpt 로 변경을 검토하는 매우 중요한 판단을 하려고 했습니다.
저는 여기에서 mpt 로 변경하는게 좋겠다. 라고 의견을 다음과 같은 근거를 들며 제시했습니다.

- 이더리움은 앞으로도 계속 개선될 것인데.. 이 패치를 온전히 받기 매우 어렵다.
- 이더리움의 모든 테스트는 기반이 mpt 이다. 테스트는 다양한 구현체를 받을 수 없도록 되어 있었기 때문에 실행되는 테스트 중 약 80% 이상의 테스트 케이스는 의미가 없다.
- 내가 개선한 트리로 인하여 성능 문제는 해결하였지만 근본적으로 구조차이로 인한 물리적인 한계를 뛰어넘을순 없다. (자식 개수, depth 깊이, 디스크 사용량)

이런 의견까지 더해져서 결국 조직은 db 마이그레이션을 하기로 결정했습니다.
그리고 저는 마이그레이션 전략, 기반 코드를 작성해서 팀원에게 전달 했고, 중간중간 구현 방향을 가이드 했습니다. 그 팀원은 이더리움에 성공적으로 병합하여 최종적으로 마이그레이션에 성공했습니다.

## 참고 pr
- tree 구현 : https://github.com/kroma-network/go-ethereum/pull/45
- tree 버그 수정 : https://github.com/kroma-network/go-ethereum/pull/81
- snap sync 구현 : https://github.com/kroma-network/go-ethereum/pull/19

## 별첨

### 2024-03-12 Chain Halt due to ZKTrie Upgrade Post-Mortem

# Incident Summary

On March 12, 2024, at 15:16:25 UTC, a fork occurred at block number [8171899](https://kromascan.com/block/8171899).

Some nodes encountered a `BAD BLOCK` error due to a hash discrepancy in that block compared to the block hash of the 
sequencer. This discrepancy was caused by a bug introduced in `KromaZKTrie` in kroma-geth 
[`v0.4.4`](https://github.com/kroma-network/go-ethereum/releases/tag/v0.4.4).

Since the sequencer was using kroma-geth `v0.4.4`, it was recovered by rollback using the chain data of nodes running 
kroma-geth [`v0.4.3`](https://github.com/kroma-network/go-ethereum/releases/tag/v0.4.3).

Block generation was halted for about 7 and a half hours due to this issue, and it took 2 more hours for the chain to be
normalized.

# Background

On Thu, Mar 07, 2024, at 05:00:00 UTC, an upgrade to kroma 
[`v1.3.2`](https://github.com/kroma-network/kroma/releases/tag/v1.3.2) and kroma-geth `v0.4.4` was conducted to enhance 
ZKTrie, which is to store the state of accounts and storage. The upgrade was first tested on the internal devnet and 
Kroma sepolia to validate backward compatibility before being applied to the Kroma mainnet. Nodes with the improved 
ZKTrie were also tested by syncing the canonical chain from the genesis block to approximately 7 million blocks to 
ensure proper functionality. The ZKTrie has also been audited by [Chainlight](https://github.com/kroma-network/go-ethereum/blob/main/docs/audits/2024-02-23_KromaZKTrie_Security_Audit_ChainLight.pdf).

# Causes

There was a bug in the process of deleting nodes in the `KromaZKTrie`. When deleting a node with a depth of 1, the node 
should be removed, leaving an empty node. However, in this case, after deleting the node, another child node was 
mistakenly set as the root node, altering the state tree and resulting in a different state root value.

![zktrie-deletion.svg](assets/zktrie-deletion.svg)

This issue is first discovered when executing a specific transaction. As a result of execution of the 
[transaction](https://kromascan.com/tx/0x50580775422fee57c8ec78dce4a3598e2246c12d7a73756f874dd117bed0ad72), the 
structure of the tree changed, leading to the calculation of different state roots and thus causing a fork with 
different block hashes.

# Recovery

Nodes running kroma-geth `v0.4.3` had the correct state root. Therefore, the chain was rolled back using the chain data 
of these nodes. All nodes, including the sequencer, were downgraded to kroma `v1.3.1` and kroma-geth `v0.4.3` to 
continue generating correct blocks.

## Timeline (UTC)

- 2024-03-12 0616: fork occurred at 8171899
- 2024-03-12 0619: received alerts from some RPC nodes about discrepancies of latest block
- 2024-03-12 0620: started investigating the fork
- 2024-03-12 0649: announced fork occurrence on discord
- 2024-03-12 0814: requested canonical chain snapshot data from Wemade
- 2024-03-12 0907: announced rollback of Kroma mainnet on discord
- 2024-03-12 0942: set up a new sequencer using the snapshot data provided by Wemade
- 2024-03-12 0949: Chainlight notified
- 2024-03-12 1329: announced that the recovery of Kroma mainnet is on going on discord
- 2024-03-12 1348: restarted a new sequencer
- 2024-03-12 1521: provided rollback snapshot and instructions to security council members
- 2024-03-12 1533: provided rollback snapshot and instructions to etherscan
- 2024-03-12 1548: announced the recovery of Kroma mainnet (RPC, P2P, validator) on discord
- 2024-03-13 0441: found a bug in `KromaZKTrie` and proposed workaround by Chainlight
- 2024-03-13 1212: completed test of rollback for node operators
- 2024-03-13 1239: provided rollback snapshot and instructions on discord
- 2024-03-14 0626: opened PR to fix a bug in `KromaZKTrie`

# How it is fixed

The logic for deleting node with a depth of 1 was modified to ensure proper deletion. This was achieved by removing a 
separate case handling for depth 1 that caused incorrect deletion.

Related PR: https://github.com/kroma-network/go-ethereum/pull/81

# Lessons learned

## No tests for deletion

The absence of tests for node deletion made it impossible to prevent this issue in advance. In the future, more tests 
will be implemented to thoroughly examine the functionality of all functions, thereby preventing such issues in advance.

# Future plan

The bug will be fixed in this [PR](https://github.com/kroma-network/go-ethereum/pull/81), and once merged, it will 
undergo sufficient test on the internal devnet and Kroma sepolia. Additionally, we will re-execute the problematic 
transaction using the `KromaZKTrie` and see if it results in the correct state root. Once all tests are completed, it 
will be applied to the Kroma mainnet, and relevant instructions will be provided via 
[kroma-up](https://github.com/kroma-network/kroma-up).
