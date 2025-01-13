# Benqi 

### Prize Pool

- Total Pool - $33,750
- H/M -  $30,375
- Low - $3,375

- Starts: January 13, 2025 Noon UTC
- Ends: January 27, 2025 Noon UTC

- nSLOC: 1367

[//]: # (contest-details-open)

## About the Project

Ignite by BENQI is a permissionless liquid staking protocol that subsidizes the stake requirement for validating on
Avalanche. Zeeve is a partner validator hosting provider that assists in setting up and configuring nodes for Ignite
users. Registrations made with hosted validators, paid in QI, are subject to additional QI rewards, proportional to
the duration staked (one of 2, 4, 8, 12 weeks).

## Actors

- BENQI (Trusted): the liquid staking provider and creator of Ignite
- Zeeve (Trusted): the partner validator hosting provider
- Users: the end users of the system of Ignite contracts

[//]: # (contest-details-close)

[//]: # (scope-open)

## Scope (contracts)

- Ignite.sol
- IgniteStorage.sol
- staking.sol
- ValidatorRewarder.sol

## Compatibilities

- Blockchains:
  - Avalanche
- Tokens:
  - AVAX
  - ERC20
    - Rebasing, and fee-on-transfer tokens are excluded.
    - USDC explicitly supported (blacklisted addresses will result in failed transfers)

[//]: # (scope-close)

[//]: # (getting-started-open)

## Setup

Build:
```bash
npm install
npx hardhat build
```

Tests:
```bash
npx hardhat test
```

[//]: # (getting-started-close)

[//]: # (known-issues-open)

## Known Issues

Findings as per Cyfrin's original audit. Please, see the report for further details.

[Previous audit report](https://2452785816-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2F-MViz_ikDZy4OemUj_VI%2Fuploads%2Ftlv40KMOxjgE1ua46HAr%2FBENQI%20Ignite%20-%20Mar%20'23.pdf?alt=media&token=24962f50-64e8-4c9a-86a7-c5846a5aa5e2)

### StakingContract refunds are affected by global parameter updates

When StakingContract::refundStakedAmount is called by the BENQI admin, the following 
validation is performed using the globally-defined refundPeriod:

```
require(
  block.timestamp > record.timestamp + refundPeriod,
  "Refund period not reached"
);
```

The StakingContract::StakeRecord struct does not have a corresponding member and so does not store the
value of refundPeriod at the time of staking; however, if StakingContract::setRefundPeriod is called with an
updated period then that of an existing record could be shorter/longer than expected.


### Insufficient validation of Chainlink price feeds

Validation of the price and updatedAt values returned by Chainlink
AggregatorV3Interface::latestRoundData is performed within the following functions:

- StakingContract::_validateAndSetPriceFeed
- StakingContract::_getPriceInUSD
- Ignite::_initialisePriceFeeds
- Ignite::registerWithStake
- Ignite::registerWithErc20Fee
- Ignite::registerWithPrevalidatedQiStake
- Ignite::addPaymentToken
- Ignite::configurePriceFeed

However, there is additional validation shown below that is recommended but currently not present:

```
(uint80 roundId, int256 price, , uint256 updatedAt, ) = priceFeed.latestRoundData();
if(roundId == 0) revert InvalidRoundId();
if(updatedAt == 0 || updatedAt > block.timestamp) revert InvalidUpdate();
```

### Lack of user-defined slippage and deadline parameters in StakingContract::swapForQI may result in unfavorable QI token swaps

When a user interacts with StakingContract to provision a hosted node, they can choose between
two methods:StakingContract::stakeWithAVAX or StakingContract::stakeWithERC20. If the staked token is
not QI, StakingContract::swapForQI is invoked to swap the staked token for QI via Trader Joe. Once created,
the validator node is then registered with Ignite, using QI, via StakingContract::registerNode.

Within the swap to QI, amountOutMin is calculated using Chainlink price data and a slippage parameter defined by
the protocol:

```
// Get the best price quote
uint256 slippageFactor = 100 - slippage; // Convert slippage percentage to factor
uint256 amountOutMin = (expectedQiAmount * slippageFactor) / 100; // Apply slippage
```

If the actual amount of QI received is below this amountOutMin, the transaction will revert; however, users are
restricted by the protocol-defined slippage, which may not reflect their preferences if they desire a smaller slippage
tolerance to ensure they receive a more favorable swap execution.

Additionally, the swap deadline specified as block.timestamp in StakingContract::swapForQI provides no 
protection as deadline validation will pass whenever the transaction is included in a block:

```
uint256 deadline = block.timestamp;
```

This could expose users to unfavorable price fluctuations and again offers no option for users to provide their own
deadline parameter.


### Unnecessary amount parameter in StakingContract::stakeWithERC20

When provisioning a node through StakingContract::stakeWithERC20, users can pay with a supported ERC20 token.
The totalRequiredToken is calculated based on the avaxStakeAmount (needed to register the node in Ignite) and
the hostingFee (paid to Zeeve for hosting), before being transferred from the user to the contract:

```
require(isTokenAccepted(token), "Token not accepted");
uint256 hostingFee = calculateHostingFee(duration);

uint256 totalRequiredToken = convertAvaxToToken(
  token,
  avaxStakeAmount + hostingFee
);

require(amount >= totalRequiredToken, "Insufficient token");

// Transfer tokens from the user to the contract
IERC20Upgradeable(token).safeTransferFrom(
  msg.sender,
  address(this),
  totalRequiredToken
);
```

The amount parameter provided by the user is only used to validate that it covers the totalRequiredToken, but
since execution will revert if the user has not given the contract sufficient allowance for the transfer, the amount
parameter becomes redundant.


### Staking amount in QI should be calculated differently

Currently, if the stake token is QI, stakingAmountInQi is calculated as shown below:

```
stakingAmountInQi = totalRequiredToken - convertAvaxToToken(token, hostingFee);
```

However, this can result in a precision loss of 1 wei.


### Tokens with more than 18 decimals will not be supported

Currently, tokens with more than 18 decimals are not supported due to the decimals handling logic 
in `StakingContract::_getPriceInUSD`

```
uint256 decimalDelta = uint256(18) - tokenDecimalDelta;
```

and `Ignite::registerWithErc20Fee:`

```
uint tokenAmount = uint(avaxPrice) * registrationFee / uint(tokenPrice) / 10 ** (18 - token.decimals());
```


### Placeholder recipient constants in Ignite should be updated before deployment

While it is understood that the FEE_RECIPIENT and SLASHED_TOKEN_RECIPIENT constants in Ignite
have been modified for testing purposes, it is important to note that they should be reverted to valid values before
deployment to ensure that fees and slashed tokens are not lost.

``` 
address public constant FEE_RECIPIENT = 0xaAaAaAaaAaAaAaaAaAAAAAAAAaaaAaAaAaaAaaAa; // @audit-info -
update placeholder values,
address public constant SLASHED_TOKEN_RECIPIENT = 0xbBbBBBBbbBBBbbbBbbBbbbbBBbBbbbbBbBbbBBbB;
```


### Magic numbers should be replaced by constant variables

The magic numbers 10_000, 2000e18, 201/201e18 are used throughout the Ignite contract but
should be made constant variables instead.


### Misalignment of pause() and unpause() access controls across contracts

All three contracts, Ignite, ValidatorRewarder, and StakingContract, have pausing functionality
that can be triggered by accounts with special privileges; however, they all implement the access control differently:

- In Ignite, pause() can only be called by accounts granted the ROLE_PAUSE role and similarly for unpause() it is the ROLE_UNPAUSE role.
- In ValidatorRewarder, both pause() and unpause() can only be called by accounts granted the ROLE_PAUSE role. The role ROLE_UNPAUSE is defined but not used.
- In StakingContract, both pause() and unpause() are limited to accounts granted the role DEFAULT_ADMIN_ROLE.


### Inconsistent price validation in Ignite::registerWithStake

In `Ignite::registerWithErc20Fee`, `Ignite::registerWithPrevalidatedQiStake`, and `StakingContract::_getPriceInUSD`,
prices are validated to be greater than 0; however, in `Ignite::registerWithStake`,
the AVAX price is validated to be greater than the QI price. While the AVAX price is currently significantly higher than
the QI price and so will not result in any unwanted reverts, this validation is inconsistent with the other instances
and should be modified.


### Unnecessary validation in StakingContract::registerNode

When a new validator node has been created on behalf of a user, the Zeeve admin
reports this by calling StakingContract::registerNode which performs some validation before invoking
Ignite::registerWithPrevalidatedQiStake to register the node according to the requirements in Ignite.

Some of this validation done in StakingContract::registerNode, shown below, is unnecessary and can be
removed.

```
require(
  bytes(nodeId).length > 0 && blsProofOfPossession.length > 0,
  "Invalid node or BLS key"
);

require(
  igniteContract.registrationIndicesByNodeId(nodeId) == 0,
  "Node ID already registered"
);
```

All of this validation around nodeId, blsProofOfPossesion, and the registration index is performed again in `Ignite::_register`.

```
// Retrieve the staking details from the stored records
require(stakeRecords[user].stakeCount > 0, "Staking details not found");
require(index < stakeRecords[user].stakeCount, "Index out of bounds"); // Ensures the index is valid

StakeRecord storage record = stakeRecords[user].records[index]; // Access the record by index
```

If these requirements were removed, an invalid index or zero stake count would result in an uninitialized StakeRecord
being returned. Thus, execution would revert on all of the subsequent requirements:

```
require(record.timestamp != 0, "Staking details not found");
require(isValidDuration(record.duration), "Invalid duration");

// Ensure the staking status is Provisioning
require(
  record.status == StakingStatus.Provisioning,
  "Invalid staking status"
);
```

Even still, the timestamp validation is superfluous as there is no way for an existing record to have an uninitialized
timestamp, and the record is guaranteed to exist by the subsequent check on status. This means that the
duration validation is also unnecessary, as it is not needed to guarantee the existence of a record and is performed
again in `Ignite::_regiserWithChecks`.

- Blacklisted addresses in the cases of tokens such as USDC will have failed transfers. This is expected.

**Additional Known Issues as detected by LightChaser can be found [here](https://github.com/Cyfrin/2025-01-benqi/issues/1)**

[//]: # (known-issues-close)
