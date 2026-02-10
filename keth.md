# keth: kotlin Ethereum (Re-engineering Geth Core with Kotlin)

[github](https://github.com/jyc228/keth)

## 배경 & 도전

조직 업무로 [geth](https://github.com/ethereum/go-ethereum) 라는 ethereum 의 go 구현체를 유지보수 하고 있었습니다.
`geth` 라는 매우 방대하고 복잡한 프로젝트는 go lang 도 처음, 블록체인도 처음 접하는 저에겐 매우 도전적인 업무였습니다.
단순 유지보수를 넘어 코어 레벨의 동작 원리를 바이트 단위까지 완벽하게 이해할 필요성을 느꼈습니다.
이를 위해 가장 익숙한 언어인 Kotlin으로 **이더리움 프로토콜 스펙을 밑바닥부터 재구현(Cleanroom Implementation)** 하는 R&D 프로젝트를 시작했습니다.

## 과제

### 프로젝트 구조

geth 는 크게 보면 하단과 같은 구조로 되어 있습니다.

```
low level
   - disk: 실제 블록 / 스냅샷 / 데이터 파일 보관소
   - trie: 상태 루트/해시를 위한 핵심 자료구조 (mpt)
   
mid level
   - state database: trie 기반으로 정리된 계정 / 스마트컨트랙트 상태 저장소
   - evm: 트랜잭션과 스마트컨트랙트 코드 실행 엔진
   - p2p: 다른 노드와 데이터를 주고받는 네트워크 계층

high level
   - api: 외부(지갑, dApp, 사용자)가 노드와 소통하는 인터페이스 (JSON-RPC 등)
   - sync: 네트워크를 통해 블록체인 상태를 최신으로 맞추는 동기화 로직
   - chain: 블록 저장, 포크 처리, 체인 상태 관리
   - consensus: 어떤 블록을 정답으로 채택할지 결정하는 알고리즘(예: PoS)
   - miner / validator: 블록 생성(또는 검증)을 담당 (consensus의 행위자)
```

하나하나가 매우 방대하고 복잡한 모듈입니다.
저는 low level 부터 차근차근 구현해 나갔으며, `trie`, `state database`, `evm` 을 구현하고 과제 종료했습니다.

### low level : trie (merkle patricia trie / mpt)

`geth` 에 있던 `mpt` 를 클론코딩 했습니다. 클론코딩 이후 리팩토링을 하기 위하여, 모든 로직을 이해하려고 노력했으며 `mpt` 의 동작 원리를 좀 더 자세히 알게 되었습니다.
하단은 핵심 api 를 추출하여 인터페이스로 구현한 결과물 입니다.

```kotlin
interface MerkleTree {
    fun rootHash(): ByteArray?
    fun collectDirties(includeLeaf: Boolean): MerkleTreeDirtyNodes?
    operator fun get(key: ByteArray): ByteArray?
    operator fun set(key: ByteArray, value: ByteArray)
    operator fun minusAssign(key: ByteArray)

    companion object
}
```

인터페이스로 추출한 이유는 제가 했던 프로젝트만 해도 `zktrie` 라는 `mpt` 가 아닌 다른구현체를 썻습니다.
미래엔 어떤 구현체가 또 생길지 알 수 없었기 때문에 다형성을 확보하고 public api 식별을 용이하게 하기 위하여 인터페이스 위주로 작업 했습니다.

- [geth mpt](https://github.com/ethereum/go-ethereum/blob/f4817b7a5326a14b5648904a5881396f22fcbc37/trie/trie.go)
- [MerkleTree](https://github.com/jyc228/keth/blob/dev/collections/src/main/kotlin/com/github/jyc228/keth/collections/MerkleTree.kt)

#### zktrie 최적화

이때 얻은 지식 덕분에 조직에서 쓰던 `zktrie (trie 구현체 중 하나, mpt 사용 안했었습니다.)` 에 구조적 병목을 발견하고 최적화 하는 결정적 통찰을 제공했습니다.
그 후로 `zktrie` 를 개선하는 작업에 착수 했습니다. 이와 관련된 이야기는 [blockchain.md](blockchain.md) 에 상세히 적었습니다.

### mid level : state database

`geth` 의 `StateDatabase` 는 블록체인에서 사용자가 발생시키는 모든 데이터 변경을 읽고, 쓰는 역할을 합니다. 절차지향적으로 구현되어 있으며, 파악이 어렵고, 유지보수 난이도가 높다고 생각했습니다.
이를 개선하기 위하여 도메인 객체를 정의하고 응집력을 높였습니다.

모든 액션은 `ManagedStateAccount` 를 통해 이루어집니다. `StateDatabase` 는 `Account` 와 관련된 연산은 읽기만 허용합니다.

차이점을 확인하고 싶으시다면 아래 링크를 확인해 보세요. 기능적으론 아래 kotlin 인터페이스와 완전히 동일합니다.

- [statedb.go](https://github.com/ethereum/go-ethereum/blob/f4817b7a5326a14b5648904a5881396f22fcbc37/core/state/statedb.go)

```kotlin
interface StateDatabase {
    suspend fun createAccount(address: Address, callback: (suspend (ManagedStateAccount) -> Unit)? = null): StateAccount
    suspend fun findAccount(address: Address): StateAccount?
    suspend fun applyAccount(address: Address, callback: suspend (ManagedStateAccount) -> Unit): StateAccount?

    suspend fun commit(): StateRoot?
    suspend fun intermediateRoot(): StateRoot?

    fun snapshot(): Int
    fun revertSnapshot(id: Int)
}

interface StateAccount {
    val nonce: ULong
    val balance: BigInteger
    val root: StorageRoot?
    val codeHash: CodeHash?
}

interface ManagedStateAccount : StateAccount {
    override var nonce: ULong
    override var balance: BigInteger
    val address: Address
    val storage: Storage

    suspend fun getCode(): ByteArray?
    suspend fun setCode(code: ByteArray?)

    interface Storage {
        suspend fun get(key: ByteArray): ByteArray?
        suspend fun getCommittedState(key: ByteArray): ByteArray?
        suspend fun set(key: ByteArray, value: ByteArray?)
    }
}
```

`StateDatabase` 는 2가지의 구현체가 있습니다.

- `OnchainStateDatabase`: `trie` 에 접근하는 db 입니다. 일반적인 블록체인 풀노드 (디스크 사용량 : 수백 gb 이상) 에서 사용하는 구현체 입니다.
- `OffchainStateDatabase`: 네트워크를 통해 데이터를 조회하는 db 입니다. disk 사용량이 없으며 테스트, debugger 등 경량화된 사용을 목표로 만들었습니다.

특히 `OffchainStateDatabase` 를 만들면서 debugger 를 만들수 있겠다. 라는 생각을 가지게 되었습니다.

### mid level : evm

프로젝트 계층상 `state database` 다음 목표는 `evm` 이 되었습니다.
`state database` 와 같은 mid level 로 묶긴 했지만 `state database` -> `evm` 이 좀 더 정확한 표현이 됩니다.

이부분은 특히 제가 제일 궁금해 했던 파트였습니다. `evm` 클론 코딩을 통해 vm 의 동작 원리를 조금이라도 알 수 있게 되었습니다.

`evm` 구현 역시 단순 클론코딩 수준으로 끝내지 않았습니다. 일차적으로 클론 코딩후, 제가 생각하는 유지보수 하기 좋은 구조로 리팩토링을 진행했으며, 가장 영향을 많이 받는 곳은 opcode 와 연산쪽이었습니다.

`geth` 는 opcode 의 주요 파트마다 전부 파일이 분리되어 있습니다 (`instructions.go`, `jump_table.go`, `memory_table.go`, `gas_table.go`).
매우 표준적인 구조이지만, 저는 이 구조가 파악 및 유지보수를 매우 어렵게 한다고 생각했습니다.

Kotlin DSL을 활용하여, EVM Opcode의 명세(가스비, 스택 조작, 메모리 확장)를 선언적 코드로 구현했습니다.
이는 복잡한 Switch-Case 로직을 '실행 가능한 문서' 형태로 승화시켜, 코드의 가독성과 안정성을 획기적으로 높였습니다.

```kotlin
fun OperationBuilder.withOpCode(opCode: OpCode): OperationBuilder {
    when (opCode) { // https://ethervm.io/
        OpCode.STOP -> execute { result = EVMReturn.success(byteArrayOf()) }
        OpCode.ADD -> poppush { (a, b) -> a + b }.gas3()
        OpCode.MUL -> poppush { (a, b) -> a * b }.gas5()
        OpCode.SUB -> poppush { (a, b) -> a - b }.gas3()
        OpCode.DIV -> poppush { (a, b) -> a / b }.gas5()
        OpCode.SDIV -> poppush { (a, b) -> a.signed() / b.signed() }.gas5()
        OpCode.MOD -> poppush { (a, b) -> a % b }.gas5()
        OpCode.RETURN -> pop { (offset, size) ->
            result = EVMReturn.success(memory.read(offset.int, size.int))
        }.memorySize { (offset, size) -> memorySize(offset.int, size.int) }
        // 생략..
    }; return this
}
```

그 후 인터프리터를 클론 코딩 했습니다. `geth` 의 인터프리터 확장 방식은 코어 소스 코드를 오염시키면서 하고 있었습니다. 저는 핵심 로직과 확장 로직을 분리하기 위하여
저는 **위임** 을 통해 이를 분리하여, 코어의 순수성을 지키면서도 기능을 유연하게 확장할 수 있는 구조를 만들었습니다.

```kotlin
open class EVMInterpreter(private val instructionSet: InstructionSet) {
    open suspend fun execute(frame: EVMFrame): EVMReturn {
        frame.interpreter = this
        while (true) return execute(frame, instructionSet[frame.contract.code[frame.pc]]) ?: continue
    }

    suspend fun execute(frame: EVMFrame, opCode: OpCode) = execute(frame, instructionSet[opCode.v])

    protected open suspend fun execute(frame: EVMFrame, operation: Operation?): EVMReturn? {
        ...
    }

    class Delegate(set: InstructionSet, private val delegate: EVMInterpreterDelegate) : EVMInterpreter(set) {
        override suspend fun execute(frame: EVMFrame) = delegate.execute(frame) { super.execute(it) }
        override suspend fun execute(
            frame: EVMFrame,
            operation: Operation?
        ) = delegate.execute(frame, operation) { f, o -> super.execute(f, o) }
    }
}
```

참고차 `geth` 가 인터프리터를 확장하는 방식을 추가합니다.
하단과 같이 코어 로직 내부에 `if (evm.Config.Tracer != null)` 같은 분기문을 삽입하여, **핵심 비즈니스 로직과 부가 로직이 강하게 결합** 되어 있었습니다.

```golang
func (evm *EVM) DelegateCall(originCaller common.Address, caller common.Address, addr common.Address, input []byte, gas uint64, value *uint256.Int) (ret []byte, leftOverGas uint64, err error) {
	// Invoke tracer hooks that signal entering/exiting a call frame
	if evm.Config.Tracer != nil {
		...
	}
    ...
	// It is allowed to call precompiles, even via delegatecall
	if p, isPrecompile := evm.precompile(addr); isPrecompile {
		ret, gas, err = RunPrecompiledContract(p, input, gas, evm.Config.Tracer)
	} else {
		...
	}
	if err != nil {
		evm.StateDB.RevertToSnapshot(snapshot)
		if err != ErrExecutionReverted {
			if evm.Config.Tracer != nil && evm.Config.Tracer.OnGasChange != nil {
				evm.Config.Tracer.OnGasChange(gas, 0, tracing.GasChangeCallFailedExecution)
			}
			gas = 0
		}
	}
	return ret, gas, err
}
```

- [Operations](https://github.com/jyc228/keth/blob/dev/ethereum/vm/src/main/kotlin/com/github/jyc228/keth/vm/Operations.kt)
- [Operation](https://github.com/jyc228/keth/blob/dev/ethereum/vm/src/main/kotlin/com/github/jyc228/keth/vm/Operation.kt)
- [Interpreter](https://github.com/jyc228/keth/blob/dev/ethereum/vm/src/main/kotlin/com/github/jyc228/keth/vm/interpreter/EVMInterpreter.kt)

## 결과

- Core Level Deep Dive: 블록체인의 가장 밑바닥인 `trie`부터 실행 엔진인 `evm`까지 직접 구현하며, 시스템의 동작 원리를 바이트 단위까지 이해했습니다.
- Problem Solving: 이 프로젝트에서 얻은 통찰력은 실무에서의 **대규모 성능 최적화(1000% 향상)** 와 **장애 대응(6시간 중단 사고 해결)** 등 블록체인 코어를 유지보수 할 수 있는 핵심
  원동력이 되었습니다.
- Expansion: 이 경험은 단순한 학습을 넘어, 이더리움 개발 환경을 혁신하는 [IntelliJ Plugin 개발 프로젝트](intellij-ethereum-plugin.md)로 이어졌습니다.
