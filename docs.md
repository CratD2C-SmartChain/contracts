# Smart contracts documentation for CRAT blockchain validators & delegators system

This repo contains following contracts:
1. CratStakeManager.sol - stake manager contract for validators and delegators accounting;
2. CratVesting.sol - vesting contract for sheduled unlocking funds for several system wallets.

## CRATStakeManager contract description
The contract is a system for accounting the list of active/inactive validators and their delegators. This staking contract has two reward mechanisms: APR, according to staked amount share.

### Variables

`bytes32 public constant DISTRIBUTOR_ROLE = keccak256("DISTRIBUTOR_ROLE");` - a constant to keep distributor role's value.

`bytes32 public constant SWAP_ROLE = keccak256("SWAP_ROLE");` - a constant to keep swap contract role's value.

`uint256 public constant PRECISION = 100_00;` - a constant to keep a denominator value for percents (2 decimal places; an example: 2% == 200).

`uint256 public constant YEAR_DURATION = 365 days;` - a constant to keep a year duration in seconds.

`GeneralSettings public settings;` - general settings of the contract in GeneralSettings struct format (see below in Structs section).

`uint256 public totalValidatorsPool;` - total sum of staked CRAT coins by active validators.

`uint256 public totalDelegatorsPool;` - total sum of staked CRAT coins by active delegators.

`uint256 public stoppedValidatorsPool;` - total sum of staked CRAT coins by stopped validators.

`uint256 public stoppedDelegatorsPool;` - total sum of staked CRAT coins by stopped delegators.

`uint256 public forFixedReward;` - sum of CRAT coins on the contract available for APR% payments (increases when receive is triggered, decreases when the reward is withdrawn).

### Structs

```
struct ValidatorInfo { - struct to keep general info for validator's deposit

uint256 amount; - sum of staked coins

uint256 commission; - percent that validator takes from its delegators

uint256 lastClaim; - timestamp of the last claim transaction (to claim cooldown calculation)

uint256 calledForWithdraw; - 0 - validator is active; > 0  - validator is stopped (keeps timestamp of validatorCallForWithdraw call)

uint256 vestingEnd; - 0 - if staked by himself; > 0 - if staked by swap contract (keeps timestamp of the vesting funds process end)

FixedReward fixedReward; - fixed reward info in FixedReward struct format (see below)

VariableReward variableReward; - variable reward info in VariableReward struct format (see below)

SlashPenaltyCalculation penalty; - penalties info in SlashPenaltyCalculation struct format (see below)

uint256 delegatedAmount; - sum of active delegators deposits

uint256 stoppedDelegatedAmount; - sum of stopped delegators deposits

uint256 delegatorsAcc; - variable reward accumulator value for delegators

EnumerableSet.AddressSet delegators; - an array of delegators' addresses list (even if someone is stopped)
}
```

```
struct ValidatorInfoView { - same struct; not to keep, but to return validators info in getValidatorInfo

// same fields
uint256 amount;
uint256 commission;
uint256 lastClaim;
uint256 calledForWithdraw;
uint256 vestingEnd;
FixedReward fixedReward;
VariableReward variableReward;
SlashPenaltyCalculation penalty;
uint256 delegatedAmount;
uint256 stoppedDelegatedAmount;
uint256 delegatorsAcc;

// fields that differ
address[] delegators; - an array of delegators' addresses list (even if someone is stopped)

uint256 withdrawAvailable; - calculated timestamp since validator is able to withdraw

uint256 claimAvailable; - calculated timestamp since validator is able to claim
}
```

```
struct SlashPenaltyCalculation {

uint256 lastSlash; - last slashing call timestamp

uint256 potentialPenalty; - calculated potential penalty fot the future (calculated as earned fixed reward (APR) since previous slashing to the next one; due to the slashing, this sum will be substracted from the validators deposit additional to the fixed slashing penalty; 0 - before the first slashing, > 0 - after the first slashing)
}
```

```
struct DelegatorInfo {

EnumerableSet.AddressSet validators; - keeps the list of validators this delegator has deposited for

mapping(address => DelegatorPerValidatorInfo) delegatorPerValidator; - for each validator address keeps general delegator's deposit info in DelegatorPerValidatorInfo struct format (see below)
}
```

```
struct DelegatorPerValidatorInfo { - struct to keep general info for delegator's deposit per one validator

uint256 amount; - sum of delegated coins for this validator

uint256 storedValidatorAcc; - last stored accumulator value

uint256 calledForWithdraw; - 0 - delegator is active OR stopped because its validator became stopped; > 0 - stoplisted by himself (keeps timestamp of delegatorCallForWithdraw call)

uint256 lastClaim; - timestamp of the last claim transaction (to claim cooldown calculation)

FixedReward fixedReward; - fixed reward info in FixedReward struct format (see below)

VariableReward variableReward; - variable reward info in VariableReward struct format (see below)
}
```

```
struct FixedReward { - struct to keep fixed reward info (for validators and delegators)

uint256 apr; - %APR

uint256 lastUpdate; - timestamp of the last update (calculation) for fixed reward

uint256 fixedReward; - sum of CRAT coins calculated as fixed reward

uint256 totalClaimed; - sum of CRAT coins claimed for all the time as fixed reward
}
```

```
struct VariableReward { - struct to keep varible reward info (for validators and delegators)

uint256 variableReward; - sum of CRAT coins calculated as variable reward

uint256 totalClaimed; - sum of CRAT coins claimed for all the time as variable reward
}
```

```
struct GeneralSettings { - struct to keep general contract's settings

uint256 validatorsLimit; - total validators limit

address slashReceiver; - an address of the penalties (slashing results) receiver

RoleSettings validatorsSettings; - validators' settings in RoleSettings struct format (see below)

RoleSettings delegatorsSettings; - delegators' settings in RoleSettings struct format (see below)
}
```

```
struct RoleSettings { - validators & delegators settings

uint256 apr; - %APR

uint256 toSlash; - for validators - sum (in wei) of staked coins for what its deposit will be decreased (due to the slashing); for delegators - % of the deposit by which it will be decreased (due to the slashing).

uint256 minimumThreshold; - minimum staked amount (in wei)

uint256 claimCooldown; - number of seconds that limits claim call frequency

uint256 withdrawCooldown; - number of seconds that limits momental deposit's withdraw (there is an exit in two stages: firstly, it's necessary to be stoplisted; secondly, call withdraw after cooldown was passed)
}
```

### Events

`event ValidatorDeposited(address validator, uint256 amount, uint256 commission);` - emits in depositForValidator, depositAsValidator, restake; returns validator's address, staked amount and percent of reward that validator takes from its delegators

`event ValidatorClaimed(address validator, uint256 amount);` - emits in claim & restake (if txn call is from validator), withdrawAsValidator, withdrawForValidator; returns validator's address and claimed reward sum (fixed + variable)

`event ValidatorCalledForWithdraw(address validator);` - emits in callForWithdrawAsValidator and slash (if validator's deposit has become less than minimum threshold and it has been stoplisted); returns validator's address

`event ValidatorRevived(address validator);` - emits in reviveAsValidator; returns revived validator's address (removed from stoplist and became active)

`event ValidatorWithdrawed(address validator);` - emits in withdrawAsValidator and withdrawForValidator; returns validator's address has left the staking

`event DelegatorDeposited(address delegator, address validator, uint256 amount);` - emits in depositAsDelegator, restake; returns delegator's address, its validator's address, delegated amount

`event DelegatorClaimed(address delegator, uint256 amount);` - emits in claim и restake (if txn call is from delegator), withdrawAsDelegator, withdrawForDelegator; returns delegator's address and claimed reward sum (fixed + variable)

`event DelegatorCalledForWithdraw(address delegator);` - emits in callForWithdrawAsDelegator and slash (if delegator's deposit has become less than minimum threshold and it has been stoplisted); returns delegator's address

`event DelegatorRevived(address delegator);` - emits in reviveAsDelegator; returns returns revived delegator's address (removed from stoplist and became active)

`event DelegatorWithdrawed(address delegator);` - emits in withdrawAsValidator (when validator withdraw its deposit, loop begins for all its delegators to withdraw their deposits too), withdrawAsDelegator, withdrawForDelegator; returns delegator's address has left the staking

### Functions

#### For DEFAULT_ADMIN_ROLE

`function setSlashReceiver(address receiver) external` - set slash receiver address

`function setValidatorsLimit(uint256 value) external` - set maximum validators available

`function setValidatorsWithdrawCooldown(uint256 value) external` - set withdraw cooldown duration for validators

`function setDelegatorsWithdrawCooldown(uint256 value) external` - set withdraw cooldown duration for delegators

`function setValidatorsMinimum(uint256 value) external` - set minimum deposit amount for validators

`function setDelegatorsMinimum(uint256 value) external` - set minimum deposit amount for delegators per one validator

`function setValidatorsAmountToSlash(uint256 value) external` - set fixed penalty amount for slashed validators (in wei)

`function setDelegatorsPercToSlash(uint256 value) external` - set fixed penalty percent for slashed delegators (in %)

`function setValidatorsAPR(uint256 value) external` - set APR for validators

`function setDelegatorsAPR(uint256 value) external` - set APR for delegators

`function setValidatorsClaimCooldown(uint256 value) external` - set claim cooldown duration for validators

`function setDelegatorsClaimCooldown(uint256 value) external` - set claim cooldown duration for delegators

`function withdrawExcessFixedReward(uint256 amount) external` - withdraw excess funds from `forFixedReward` reserve

#### For DISTRIBUTOR_ROLE

```
function distributeRewards(
address[] calldata validators, - validators addresses
uint256[] calldata amounts - reward amounts (in wei) for this validators list
) external payable
```
- to distribute variable rewards between several validators (and its delegators automatically); necessary to set msg.value that won't be lower than `amounts` sum

`function slash(address[] calldata validators) external` - to slash several validators (and its delegators automatically)

#### For SWAP_ROLE

```
function depositForValidator(
address sender, - validator's address
uint256 commission, - percent that validator will take from its delegators
uint256 vestingEnd - timestamp of vesting ends
) external payable
```
- to call deposit for current validator direclty from Swap contract (in this future contract CRAT coins will be vested; so users will be able do not wait and stake before vesting ends, but they won't be able withdraw their deposits earlier than vesintg ends)

#### For users

```
function depositAsValidator(
uint256 commission - percent that validator will take from its delegators
) external payable
```
- to become a validator

`function depositAsDelegator(address validator) external payable` - to delegate coins for chosen validator

`function claimAsValidator() external` - claim rewards as validator

`function claimAsDelegatorPerValidator(address validator) external` - claim rewards from one chosen validator

`function restakeAsValidator() external` - restake (claim rewards + deposit) as validator

`function restakeAsDelegator(address validator) external` - рестейк (claim rewards + deposit) as delegator per one chosen validator

`function validatorCallForWithdraw() external` - become stoplisted validator

`function delegatorCallForWithdraw(address validator) external` - become stoplisted as delegator per one chosed validator

`function withdrawAsValidator() external` - final validator's withdraw call after cooldown (validator calls by himself)

`function withdrawAsDelegator(address validator) external` - final delegator's withdraw per one validator call after cooldown (delegator calls by himself)

`function withdrawForValidator(address validator) external` - final validator's withdraw call after cooldown (anyone calls)

`function withdrawForDelegator(address delegator, address validator) external` - final delegator's withdraw per one validator call after cooldown (anyone calls)

`function reviveAsValidator() external payable` - ability to become an active validator again (if validator is stoplisted)

`function reviveAsDelegator(address validator) external payable` - ability to become an active delegator again (if delegator is stoplisted and validator is active)

#### View functions

`function validatorEarned(address validator) public view returns (uint256 fixedReward, uint256 variableReward)` - get fixed and variable reward for validator

`function delegatorEarnedPerValidator(address delegator, address validator) public view returns (uint256 fixedReward, uint256 variableReward)` - to get delegator's fixed and variable reward earned per one validator

`function delegatorEarnedPerValidators(address delegator, address[] calldata validatorsArr) external view returns (uint256[] memory fixedRewards, uint256[] memory variableRewards)` - to get delegator's fixed and variable reward earned per several validators

`function isValidator(address account) public view returns (bool)` - true - address is validator (even if its stoplisted), else false

`function isDelegator(address account) public view returns (bool)` - true - address is delegator (even if its stoplisted), else false

`function getActiveValidators() external view returns (address[] memory validators, uint256[3][] memory amounts)` - to get active validators list and their amounts(amounts[0] - deposit of validator, amounts[1] - delegated amount for this validator (by active delegators), amounts[2] - delegated amount for this validator (by stopped delegators))

`function getStoppedValidators() external view returns (address[] memory validators, uint256[3][] memory amounts)` - to get stoplisted validators list and their amounts(amounts[0] - deposit of validator, amounts[1] - delegated amount for this validator (by active delegators), amounts[2] - delegated amount for this validator (by stopped delegators))

`function getValidatorInfo(address validator) external view returns (ValidatorInfoView memory info)` - to get info per one validator in ValidatorInfoView struct format (see in Structs section)

```
function getDelegatorInfo(
address delegator - delegator's address
) external view returns (

address[] memory validatorsArr, - validators' addresses list delegated for

DelegatorPerValidatorInfo[] memory delegatorPerValidatorArr - infos per each validator in DelegatorPerValidatorInfo struct format (see in Structs section)

uint256[] memory withdrawAvailable, - when withdraw in each validator is available

uint256[] memory claimAvailable - when claim rewards in each validator is available
) 
```
- to get delegator's info (for all its validators)

```
function getDelegatorsInfoPerValidator(
address validator - validator's address
) external view returns (

address[] memory delegators, - all delegators of this validator list

DelegatorPerValidatorInfo[] memory delegatorPerValidatorArr) - infos per each delegator in DelegatorPerValidatorInfo struct format (see in Structs section)
```
- to get validator's info (for all its delegators)



`function totalValidatorsRewards() external view returns (uint256 fixedReward, uint256 variableReward)` - to get approximately calculations of fixed and variable reward earned by all validators for all time

`function totalDelegatorsRewards() external view returns (uint256 fixedReward, uint256 variableReward)` - to get approximately calculations of fixed and variable reward earned by all delegators for all time

`function totalValidatorReward(address validator) external view returns (uint256 fixedReward, uint256 variableReward)` - to get approximately calculations of fixed and variable reward earned by one validator for all time

`function totalDelegatorRewardPerValidator(address delegator, address validator) external view returns (uint256 fixedReward, uint256 variableReward)` - to get approximately calculations of fixed and variable reward earned by one delegator per one chosen validator for all time

### Errors list

1. `no revert reason` - zero address as an input (functions: initialize, setSlashReceiver, depositForValidator)
2. `no revert reason` - input value is lower that current active validators number (functions: setValidatorsLimit) 
3. `no revert reason` - input value is larger than 100% (functions: setDelegatorsPercToSlash)
4. `no revert reason` - `withdrawExcessFixedReward` - try to withdraw more than available funds from `forFixedReward` reserve; `distributeRewards` - `amounts` sum is larger than provided `msg.value`
5. `CRATStakeManager: wrong length` - different length of arrays (functions: distributeRewards)
6. `CRATStakeManager: wrong vesting end` - vesting is already ended or it's an active validator (functions: depositForValidator)
7. `CRATStakeManager: wrong input amount` - input amount == 0 OR input amount is less that minimum (functions: depositForValidator, depositAsValidator, depositAsDelegator)
8. `CRATStakeManager: limit reached` - validators limit reached (functions: depositForValidator, depositAsValidator, reviveAsValidator)
9. `CRATStakeManager: validators only` - active delegator tries to become a validator (functions: depositForValidator, depositAsValidator)
10. `CRATStakeManager: delegators only` - active validator tries to become a delegator (functions: depositAsDelegator)
11. `CRATStakeManager: not validator` - user isn't a validator (functions: claimAsValidator, restakeAsValidator)
12. `CRATStakeManager: zero` - no rewards to reinvest it (functions: restakeAsValidator, restakeAsDelegator)
13. `CRATStakeManager: not active validator` - sender is not a validator OR validator stoplisted (functions: validatorCallForWithdraw)
14. `CRATStakeManager: not active delegator` - sender is not a delegator OR delegator stoplisted (functions: delegatorCallForWithdraw)
15. `CRATStakeManager: no withdraw call` - not a validator OR `validatorCallForWithdraw` didn't called (functions: reviveAsValidator)
16. `CRATStakeManager: too low value` - additional value + existed deposit amount is lower than minimum (functions: reviveAsValidator)
17. `CRATStakeManager: can not revive` - not a delegator OR additional value + existed deposit amount is lower than minimum OR `delegatorCallForWithdraw` didn't called OR validator stoplisted (functions: reviveAsDelegator)
18. `CRATStakeManager: in stop` - validator/delegator stoplisted (functions: depositAsValidator, depositForValidator, depositAsDelegator, restakeAsValidator, restakeAsDelegator)
19. `CRATStakeManager: commission` - wrong commission value (less then 5% or more then 30%) (functions: depositAsValidator, depositForValidator, restakeAsValidator)
20. `CRATStakeManager: wrong validator` - `depositAsDelegator`, `restakeAsDelegator` - wrong or unactive validator; `claimAsDelegatorPerValidator`, `restakeAsDelegator` - delegator didn't stak to this validator
21. `CRATStakeManager: not enough coins for fixed rewards` - not enough funds to pay fixed rewards (functions: claimAsValidator, claimAsDelegatorPerValidator, restakeAsValidator, restakeAsDelegator, withdrawAsValidator, withdrawAsDelegator)
22. `CRATStakeManager: claim cooldown` - claim cooldown not passed (functions: claimAsValidator, claimAsDelegatorPerValidator, restakeAsValidator, restakeAsDelegator, withdrawAsValidator, withdrawAsDelegator)
23. `CRATStakeManager: withdraw cooldown` - withdraw cooldown not passed (functions: withdrawAsValidator, withdrawAsDelegator)
24. `CRATStakeManager: no call for withdraw` - `delegatorCallForWithdraw` didn't called (functions: withdrawAsDelegator)

## CRATVesting

Vesting contract for this [shedule](https://docs.google.com/spreadsheets/d/1ilPoSqK1W3Uh3NHdqvhwooLh8tNPCKZKBhdeuQQIxmU/edit?gid=38530163#gid=38530163).

### Variables

`uint256 public constant PRECISION = 10 ** 26;` - constant to keep precision for calculating percentages of a number (percentages are calculated from `TOTAL_SUPPLY` value)

`uint256 public constant TOTAL_SUPPLY = 300_000_000 * 10 ** 18;` - constant to keep total amount of coins to be distributed according to the shedule

`address public initializer;`  - address that is able to call `startDistribution`

### Events

`event DistributionStarted(address[10] allocators);` - emits in `startDistribution`; returns 10 allocation addresses

`event Claimed(address allocator, uint256 amount);` - emits in `claim`; returns receiver address and amount of transferred coins

### Functions

#### For DEFAULT_ADMIN_ROLE

`function claim(address to,uint256 amount) external` - partial claim (to - receiver address (from 0 to 9 according to the order in column B in shedule (see table from description)), amount - amount of coins to transfer)

function claimAll(address to) external - claim all available coins (equals to pending) (to - receiver address (from 0 to 9 according to the order in column B in shedule (see table from description)))

#### View functions

function pending(address user) public view returns (uint256 unlocked) - available amount for claim (user - receiver address (from 0 to 9 according to the order in column B in shedule (see table from description)))

function getAddressInfo(address account) external view returns (bool hasShedule, uint256 claimed, uint256[8] memory shedule) - to get by address: has this address any shedule, how many coins it has claimed and its shedule percents

function getAllocationAddresses() external view returns (address[10] memory) - addresses in order from column B (see table from description)

### Errors

1. `CRATVesting: wrong sender` - wrong sender in `startDistribution` OR not first call of this funciton
2. `CRATVesting: wrong vesting supply` - wrong `msg.value` provided to the `startDistribution` call (should be equal to 300_000_000)
4. `CRATVesting: 0x00` - zero address as an input address in `startDistribution`
5. `CRATVesting: wrong amount` - `amount` == 0 OR larger than `pending` (`claim` call)
6. `CRATVesting: nothing to claim ` - `pending` == 0 (`claimAll` call)
