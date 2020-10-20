# Protocol levers

The protocol provides a number of settings controllable by the DAO. They are listed below, grouped
by the contract.

## [DePool.sol](/contracts/0.4.24/DePool.sol)

### Oracle

The address of the oracle contract.

* Accessor: `getOracle() returns (address)`
* Mutator: `setOracle(address)` (requires `SET_ORACLE` permission)

This contract serves as a bridge between ETH 2.0 -> ETH oracle committee members and the rest of the protocol,
implementing quorum between the members. The oracle committee members report balances controlled by the DAO
on the ETH 2.0 side, which can go up because of reward accumulation and can go down due to slashing.


### Fee

The total fee, in basis points (`10000` corresponding to `100%`).

* Accessor: `getFee() returns (uint16)`
* Mutator: `setFee(uint16)` (requires `MANAGE_FEE` permission)

The fee is taken on staking rewards and distributed between the treasury, the insurance fund, and
staking providers.


### Fee distribution

Controls how the fee is distributed between the treasury, the insurance fund, and staking providers.
Each fee component is in basis points; the sum of all components must add up to 1 (`10000` basis points).

* Accessor: `getFeeDistribution() returns (uint16 treasury, uint16 insurance, uint16 sps)`
* Mutator: `setFeeDistribution(uint16 treasury, uint16 insurance, uint16 sps)` (requires `MANAGE_FEE` permission)


### ETH 2.0 withdrawal Credentials

Credentials to withdraw ETH on ETH 2.0 side after phase 2 is launched.

* Accessor: `getWithdrawalCredentials() returns (bytes)`
* Mutator: `setWithdrawalCredentials(bytes)` (requires `MANAGE_WITHDRAWAL_KEY` permission)

The pool uses these credentials to register new ETH 2.0 validators.


### Deposit loop iteration limit

Controls how many ETH 2.0 validators can be registered in a single transaction.

* Accessor: `getDepositIterationLimit() returns (uint256)`
* Mutator: `setDepositIterationLimit(uint256)` (requires `SET_DEPOSIT_ITERATION_LIMIT` permission)
* [Scenario test](/test/scenario/depool_deposit_iteration_limit.js)

When someone submits Ether to the pool, the received Ether gets buffered in the pool contract. If the amount
of the buffered Ether becomes larger than the ETH 2.0 deposit size (32 ETH), then the pool tries to register
as many ETH 2.0 validators as it can. The limiting factors here are the amount of the buffered Ether (obviously),
the number of spare validator keys (provided by staking providers), and the deposit loop iteration limit,
controlled by this lever.

The limit is needed to prevent the submit transaction from [failing due to the block gas limit](https://github.com/ConsenSys/smart-contract-best-practices/blob/8f99aef/docs/known_attacks.md#gas-limit-dos-on-a-contract-via-unbounded-operations).
This is possible if the amount of the buffered Ether becomes sufficiently large, which, in turn, can occur due to:

* a user sending a large amount of Ether in a single transaction;
* temporary lack of new validator keys.

In a situation when some Ether that could otherwise be used to register validators gets left buffered in
the pool due to this limit, one can resume registering new validators by submitting zero Ether, which is
allowed exactly for this purpose.

Currently, the submit function, compiled with `200` optimizer iterations, consumes around `577000 gas` plus
`107000 gas` for each registered validator, so the default limit is set by deploy scripts to `16` validators
to make the submit transaction occupy no more than `20%` of a block. See [this file](/estimate_deposit_loop_gas.js)
for the related calculations; you can run them with `yarn estimate-deposit-loop-gas`.


### Pausing

Allows pausing pool routine operations.

* Accessor: `isStopped() returns (bool)`
* Mutators: `stop()`, `resume()` (requires `PAUSE_ROLE`)


### TODO

* Treasury (`getTreasury() returns (address)`; mutator?)
* Insurance fund (`getInsuranceFund() returns (address)`; mutator?)
* Transfer to vault (`transferToVault()`)


## [StakingProvidersRegistry.sol](/contracts/0.4.24/sps/StakingProvidersRegistry.sol)

### Pool

Address of the pool contract.

* Accessor: `pool() returns (address)`
* Mutator: `setPool(address)` (requires `SET_POOL` permission)


### Staking providers list

* `addStakingProvider(string _name, address _rewardAddress, uint64 _stakingLimit)`,
  requires `ADD_STAKING_PROVIDER_ROLE`
* `setStakingProviderName(uint256 _id, string _name)`,
  requires `SET_STAKING_PROVIDER_NAME_ROLE`
* `setStakingProviderRewardAddress(uint256 _id, address _rewardAddress)`,
  requires `SET_STAKING_PROVIDER_ADDRESS_ROLE`
* `setStakingProviderStakingLimit(uint256 _id, uint64 _stakingLimit)`,
  requires `SET_STAKING_PROVIDER_LIMIT_ROLE`

Staking providers act as validators on the Beacon chain for the benefit of the protocol. Each
staking provider submits no more than `_stakingLimit` signing keys that will be used later
by the pool for registering the corresponding ETH 2.0 validators. As oracle committee
reports rewards on the ETH 2.0 side, the fee is taken on these rewards, and part of that fee
is sent to staking providers’ reward addresses (`_rewardAddress`).


### Deactivating a staking provider

* `setStakingProviderActive(uint256 _id, bool _active)`, requires `SET_STAKING_PROVIDER_ACTIVE_ROLE`

The DAO can deactivate misbehaving staking providers by calling this function. The pool skips
deactivated staking providers during validator registration; also, deactivated providers don’t
take part in fee distribution.


### Managing staking provider’s signing keys

* `addSigningKeys(uint256 _SP_id, uint256 _quantity, bytes _pubkeys, bytes _signatures)`
* `removeSigningKey(uint256 _SP_id, uint256 _index)`

Allow the DAO to manage signing keys for the given staking provider. All functions require
`MANAGE_SIGNING_KEYS` permission.


### Reporting new stopped validators

* `reportStoppedValidators(uint256 _id, uint64 _stoppedIncrement)`

Allows the DAO to report that `_stoppedIncrement` more validators of a staking provider
have become stopped. Requires `REPORT_STOPPED_VALIDATORS_ROLE`.


## [DePoolOracle.sol](/contracts/0.4.24/oracle/DePoolOracle.sol)

### Pool

Address of the pool contract.

* Accessor: `pool() returns (address)`
* Mutator: `setPool(address)` (requires `SET_POOL` permission)


### Members list

The list of oracle committee members.

* Accessor: `getOracleMembers() returns (address[])`
* Mutators: `addOracleMember(address)`, `removeOracleMember(address)`;
  require `MANAGE_MEMBERS` permission


### The quorum

The number of oracle committee members required to form a data point.

* Accessor: `getQuorum() returns (uint256)`
* Mutator: `setQuorum(uint256)`, requires `MANAGE_QUORUM` permission

The data point for a given report interval is formed when:

1. No less than `quorum` oracle committee members have reported their value
   for the given report interval;
2. Among these values, there is some value that occurs more frequently than
   the others, i.e. the set of reported values is unimodal. This value is
   then used for the resulting data point.