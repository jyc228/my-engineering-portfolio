# keth: kotlin Ethereum (Re-engineering Geth Core with Kotlin)

[github](https://github.com/jyc228/keth)

## 한줄 요약

업무로 다루던 `geth`를 더 잘 이해하기 위해 `Kotlin`으로 `MPT`, `StateDB`, `EVM`을 다시 구현해 본 개인 프로젝트. 여기서 얻은 이해가 실제 업무의 성능 개선과 장애 대응에 도움이 되었습니다.

## 배경 & 도전

조직 업무로 [geth](https://github.com/ethereum/go-ethereum) 라는 ethereum 의 go 구현체를 유지보수 하고 있었습니다.
`geth`는 규모가 크고 복잡한 프로젝트였고, Go도 블록체인도 처음이었던 저에게는 어려운 업무였습니다.
코드를 읽는 것만으로는 이해가 부족하다고 느껴서, 가장 익숙한 Kotlin으로 `geth`의 핵심 부분을 직접 다시 구현해 보기로 했습니다.

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

이때 `mpt`를 이해한 덕분에 회사에서 쓰던 `zktrie`(mpt 대신 쓰던 trie 구현체)의 구조적인 병목을 발견할 수 있었습니다.
그 후로 `zktrie` 를 개선하는 작업에 착수 했습니다. 이와 관련된 이야기는 [blockchain.md](../lightscale/zktrie.md) 에 상세히 적었습니다.

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
- `OffchainStateDatabase`: 네트워크를 통해 데이터를 조회하는 db 입니다.
  - disk 사용량이 없으며 테스트, debugger 등 경량화된 사용을 목표로 만들었습니다.
  - `eth_getProof`로 계정을 조회하고, `storage`와 `code`는 실제 접근 시점에만 lazy하게 네트워크에서 가져옵니다. 
  - `origin`/`dirty` 분리로 `committed state`와 실행 중 변경을 구분합니다.

특히 `OffchainStateDatabase` 를 만들면서 debugger 를 만들수 있겠다. 라는 생각을 가지게 되었습니다.

### mid level : evm

프로젝트 계층상 `state database` 다음 목표는 `evm` 이 되었습니다.
`state database` 와 같은 mid level 로 묶긴 했지만 `state database` -> `evm` 이 좀 더 정확한 표현이 됩니다.

제가 가장 궁금했던 부분입니다. `evm`을 따라 구현하면서 VM의 동작 원리를 조금은 알게 되었습니다.

`evm`도 우선 그대로 옮긴 뒤, 제가 보기에 유지보수하기 좋은 구조로 리팩토링했습니다. 가장 많이 바뀐 곳은 opcode와 연산 부분입니다.

`geth` 는 opcode 의 주요 파트마다 전부 파일이 분리되어 있습니다 (`instructions.go`, `jump_table.go`, `memory_table.go`, `gas_table.go`).
흔한 구조이지만, 하나의 opcode를 이해하려면 여러 파일을 오가야 해서 파악하기 어렵다고 느꼈습니다.

Kotlin DSL로 opcode별 동작, 가스비, 스택 조작, 메모리 확장을 한 곳에 선언하도록 바꿨습니다.

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

그다음 인터프리터를 구현했습니다. `geth`는 tracer 같은 확장 기능을 코어 로직 안에 분기문으로 넣는 방식이라,
저는 **위임**을 써서 핵심 로직과 확장 로직을 분리했습니다.

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
아래처럼 코어 로직 안에 `if (evm.Config.Tracer != null)` 같은 분기문이 들어가 있습니다.

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

`Delegate` 의 실제 활용 예시로 `EVMStructLogger`가 있습니다.
`EVMInterpreterDelegate`와 `StateDatabase`를 동시에 구현하여, `opcode` 단위 실행 추적과 `storage` 접근 로깅을 코어 코드 변경 없이 외부에서 조립할 수 있습니다.

```kotlin
class EVMStructLogger : EVMInterpreterDelegate, AbstractStateDatabase() {
    // 인터프리터 흐름 가로채기
    override suspend fun execute(frame: EVMFrame, operation: Operation?, execute: ...) {
        val log = StructLog(pc = frame.pc, op = operation?.opCode, ...)
        return execute(frame, operation).apply { log.gasCost -= frame.remainGas }
    }
    // StateDB도 래핑하여 storage 접근까지 추적
    // DelegatedStateAccount가 storage get/set만 오버라이드하여 로깅을 끼워넣음
}
```

위의 `EVMStructLogger` 를 활용하면 `opcode` 내역과 실행한 값을 전부 추출하여, fixture 테스트도 수행할 수 있습니다.

```kotlin
fun test() {
    val logger = EVMStructLogger(enableStorage = true)
    evm.execute(eth.getTransactionByHash(txHash), logger)
    val structLogsFromRemote = eth.debug_traceTransaction(txHash).structLogs
    val structLogsFromLocal = logger.logs
    // structLogsFromLocal, structLogsFromRemote 일치 검증
}
```

실제로 조직 메인넷 트랜잭션을 대상으로 검증했습니다.
`OffchainStateDatabase` 덕분에 풀노드 없이 네트워크로 state를 조회하며 로컬에서 트랜잭션을 재현할 수 있었고,
`TransactionReceipt` 비교 → 불일치 시 `debug_traceTransaction`으로 `opcode` 단위 추적까지 3단계로 검증했습니다.

- [Operations](https://github.com/jyc228/keth/blob/dev/ethereum/vm/src/main/kotlin/com/github/jyc228/keth/vm/Operations.kt)
- [Operation](https://github.com/jyc228/keth/blob/dev/ethereum/vm/src/main/kotlin/com/github/jyc228/keth/vm/Operation.kt)
- [Interpreter](https://github.com/jyc228/keth/blob/dev/ethereum/vm/src/main/kotlin/com/github/jyc228/keth/vm/interpreter/EVMInterpreter.kt)

## 결과

- `trie`부터 `evm`까지 직접 구현하면서 `geth` 내부 동작을 훨씬 잘 이해하게 되었습니다.
- 이 이해가 업무에서 [zktrie 성능 개선](../lightscale/zktrie.md)과 장애 대응에 직접적인 도움이 되었습니다.
- 이후 [IntelliJ Plugin 프로젝트](intellij-ethereum-plugin.md)로 이어졌습니다.
