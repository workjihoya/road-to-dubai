# Query

## 0. Query
`Query`(쿼리)는 메시지와 더불어 스마트 컨트랙트와 상호작용하는 중요한 방법 중 하나이다. 쿼리는 데이터베이스에서 데이터를 읽거나 상태를 조회하는 방법으로 생각할 수 있다. 이는 외부 클라이언트(CLI를 사용하여)뿐만 아니라 컨트랙트를 실행하는 동안에도 사용된다. 예를 들어, 이전 섹션에서 "Alice"나 "Bob"과 같은 이름을 해석하는 방법을 논의했는데, 이는 다른 컨트랙트에 쿼리를 해야 한다. 


먼저 raw 쿼리와 커스텀 쿼리의 두 가지 유형을 다루고, 외부 클라이언트를 통한 쿼리와 내부 클라이언트(다른 컨트랙트)에서의 쿼리의 의미를 살펴보고자 한다.

## 1. Raw Query
가장 간단한 쿼리는 키-값 저장소에 대한 raw 읽기 접근입니다. 호출자(외부 클라이언트 또는 다른 컨트랙트)가 컨트랙트의 스토리지에 사용되는 raw 바이너리 키를 전달하면, 해당 raw 바이너리 값을 쉽게 반환할 수 있습니다. 이 접근 방식의 장점은 구현이 매우 쉽고 보편적이라는 것입니다. 단점은 호출자가 스토리지 구현에 연결되며, 실행되는 정확한 컨트랙트에 대한 지식이 필요하다는 것입니다.

이 쿼리는 wasmd 런타임 내부에 구현되어 있으며 VM을 우회합니다. 따라서 CosmWasm 컨트랙트의 지원이 필요하지 않으며 모든 컨트랙트 상태가 공개됩니다. 이러한 query_raw 함수는 모든 호출자(외부 및 내부)에게 노출됩니다.

## 2. 커스텀 Query
대부분의 쿼리는 커스텀 쿼리로, 이는 컨트랙트의 데이터 저장소에 읽기 전용 모드로 접근한다. 예를 들어, 특정 주소의 잔액을 조회하거나 토큰 정보를 조회하는 같은 경우가 있다. 이렇게 하면 인터페이스에 의존할 수 있어 구현에 강하게 결합되지 않는다는 장점이 있다. 각 컨트랙트는 쿼리 함수를 노출하여 이러한 커스텀 쿼리를 처리할 수 있다.

#### 커스텀 Query 사용 예시 
쿼리에 사용 가능한 메시지는 일반적으로 `msg.rs` 또는 `query.rs` 파일에 정의되어 있으며, 이는 컨트랙트 작성자가 코드를 어떻게 구조화했는지에 따라 달라진다. 쿼리는 외부 클라이언트(API나 CLI를 통해) 또는 내부 클라이언트(컨트랙트 내에서 다른 컨트랙트로)로 수행할 수 있다. 커스텀 쿼리는 [`QueryMsg`](./05_message.md#0-messages) 열거형에 항목으로 정의되고, 컨트랙트의 [query 함수](./04_entrypoint.md#0-entrypoint)에서 처리된다.

[nameservice 컨트랙트의 `QueryMsg`](./nameservice/src/msg.rs)의 형태는 다음과 같다:
```rust
#[cw_serde]
pub enum QueryMsg {
    ResolveRecord { name: String },
    Config {},
}
```

컨트랙트는 이를 [query 함수](./nameservice/src/contract.rs)에서 다음과 같이 처리한다:
```rust
#[cfg_attr(not(feature = "library"), entry_point)]
pub fn query(deps: Deps, env: Env, msg: QueryMsg) -> StdResult<Binary> {
    match msg {
        QueryMsg::ResolveRecord { name } => query_resolver(deps, env, name),
        QueryMsg::Config {} => to_binary(&config_read(deps.storage).load()?),
    }
}
```
여기서 `query_resolver`는 또 다른 함수이며, `config_read`는 데이터 저장소에 접근하는 헬퍼 함수이다. 커스텀 쿼리는 `query 함수`를 통해 외부에 노출된다. 이를 통해 스마트 컨트랙트는 다양한 데이터 조회 요구에 응답할 수 있다.

## 3. 외부 Query
외부 쿼리는 웹 및 CLI 클라이언트가 블록체인과 작업하는 일반적인 방법이다. 이들은 CometBFT RPC를 호출하고, 이는 Cosmos SDK의 abci_query로 호출된다. 쿼리에는 무한 가스 한도가 있으며, 이는 한 노드에서만 실행되므로 전체 블록체인을 느리게 할 수 없다. 그러나 무한 루프가 있는 wasm 컨트랙트를 업로드하고 이를 사용하여 쿼리를 노출하는 공개 RPC 노드를 DoS 공격하는 문제를 방지하기 위해 query_custom 트랜잭션에 대해 고정 가스 한도를 정의해야 한다. 이는 요금을 부과하지 않지만 남용을 제한하는 데 사용된다.

## 4. 내부 Query 
컨트랙트 간의 상호작용은 메시지를 보내는 것으로 쉽게 모델링할 수 있지만, 상태를 변경하지 않고 다른 모듈을 동기적으로 쿼리하고 싶은 경우가 있다. 예를 들어, 이름을 주소로 해석하거나 다른 컨트랙트에서 KYC 상태를 확인하는 경우이다. 이를 일련의 메시지로 모델링할 수 있지만 매우 복잡해진다.

사실 해당 설계는 액터 모델의 기본 원칙 중 하나인 각 컨트랙트가 자신의 내부 상태에 독점적으로 접근할 수 있다는 원칙을 위반하기 때문에, 이는 동시성 및 재진입 문제로 이어질 수 있다. 
- 동시성 문제를 해결하기 위해 현재 CosmWasm 메시지의 실행 직전에 상태 스냅샷에 대한 읽기 전용 접근을 Querier에 제공한다. 스냅샷을 찍고, 실행 중인 컨트랙트와 쿼리된 컨트랙트 모두가 컨트랙트 실행 전의 데이터에 대해 읽기 전용 접근을 가지므로 안전한다. 현재 컨트랙트는 캐시에만 쓰며, 성공 시 플러시된다.
- 또 다른 문제는 재진입 문제는 이러한 쿼리는 동기적으로 호출되므로, 호출 컨트랙트로 다시 호출될 수 있어 문제가 발생할 수 있다. 쿼리는 읽기 전용 접근만 가지며 부작용을 가질 수 없으므로 큰 위험은 아니지만, 여전히 고려해야 할 문제이다.


## Resources
- https://docs.cosmwasm.com/docs/smart-contracts/message/submessage
- https://docs.cosmwasm.com/docs/architecture/query/