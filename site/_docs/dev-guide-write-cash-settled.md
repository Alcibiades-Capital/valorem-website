---
date: 2023-04-12 00:00:00
title: How to write cash settled options
description: Developer guide for writing cash settled options on digital assets with the Valorem Options Clearinghouse subgraph.
---

TODO Exercising a leveraged long with a flash swap

Picking up from where we left off in the previous guide to writing [physically settled options](/docs/dev-guide-write-asset-settled), let's now walk through how to achieve cash settlement of your positions with the Clearinghouse.

## Writing an option 

XYZ

{% tabs log %}

{% tab log js %}
```typescript
// Create a new option type for a MAGIC-LUSD call option
const optionId = await clearinghouse.newOptionType({
    underlyingAsset: MAGIC,
    underlyingAmount: ethers.utils.parseEther("1"),
    exerciseAsset: LUSD,
    exerciseAmount: ethers.utils.parseEther("1.5"),
    exerciseTimestamp: exerciseTimestamp,
    expiryTimestamp: exerciseTimestamp + 60 * 60 * 24 * 7, // 1 week
});

// Write 10 options on a new claim
const claimId = await clearinghouse.write(optionId, 10);

// Transfer 5 options to Bob
await clearinghouse.transferFrom(signer.address, bob, optionId, 5);
```
{% endtab %}

{% tab log python %}
```python
# Create a new option type for a MAGIC-LUSD call option
option_id = clearinghouse.functions.newOptionType({
    "underlyingAsset": MAGIC,
    "underlyingAmount": int(1e18),
    "exerciseAsset": LUSD,
    "exerciseAmount": int(1.5e18),
    "exerciseTimestamp": exercise_timestamp,
    "expiryTimestamp": exercise_timestamp + 60 * 60 * 24 * 7, # 1 week
}).call()

# Write 10 options on a new claim
claim_id = clearinghouse.functions.write(option_id, 10).call()

# Transfer 5 options to Bob
clearinghouse.functions.transferFrom(account.address, bob, option_id, 5).call()
```
{% endtab %}

{% tab log rust %}
```rust
// Create a new option type for a MAGIC-LUSD call option
let option_id = clearinghouse
    .method::<_, U256>(
        "newOptionType",
        (
            MAGIC,
            U256::exp10(18),
            LUSD,
            U256::from(15) * U256::exp10(17),
            exercise_timestamp,
            exercise_timestamp + 60 * 60 * 24 * 7, // 1 week
        ),
        None,
        TransactionRequest::new(),
    )
    .send()
    .await?
    .await_receipt()
    .await?;

// Write 10 options on a new claim
let claim_id = clearinghouse
    .method::<_, U256>(
        "write",
        (option_id, 10),
        None,
        TransactionRequest::new(),
    )
    .send()
    .await?
    .await_receipt()
    .await?;

// Transfer 5 options to Bob
clearinghouse
    .method::<_, H256>(
        "transferFrom",
        (signer.address(), bob, option_id, 5),
        None,
        TransactionRequest::new(),
    )
    .send()
    .await?
    .await_receipt()
    .await?;
```
{% endtab %}

{% tab log solidity %}
```solidity
// Create a new option type for a MAGIC-LUSD call option.
uint256 optionId = clearinghouse.newOptionType({
    underlyingAsset: MAGIC,
    underlyingAmount: 1e18,
    exerciseAsset: LUSD,
    exerciseAmount: 1.50e18,
    exerciseTimestamp: exerciseTimestamp,
    expiryTimestamp: exerciseTimestamp + 1 weeks
});

// Write 10 options on a new claim.
uint256 claimId = clearinghouse.write(optionId, 10);

// Transfer 5 options to Bob.
clearinghouse.transferFrom(address(this), bob, optionId, 5);
```
{% endtab %}

{% endtabs %}

## Exercising a long position with cash settlement

XYZ

{% tabs log %}

{% tab log js %}
```typescript
// Warp forward to the exercise timestamp
await provider.send("evm_increaseTime", [earliestExercise]);
await provider.send("evm_mine", []);

// Bob exercises his options
await clearinghouse.connect(bob).exercise(optionId, 5);

// Bob swaps his MAGIC for LUSD via Uniswap
const bobMagicBalance = await MAGIC.balanceOf(bob.address);
const swapRouter = new ethers.Contract(SWAP_ROUTER_ADDRESS, ISwapRouterABI, bob);

const params = {
  tokenIn: MAGIC.address,
  tokenOut: LUSD.address,
  fee: 3000,
  recipient: bob.address,
  deadline: Math.floor(Date.now() / 1000) + 1,
  amountIn: bobMagicBalance,
  amountOutMinimum: 0,
};

await swapRouter.exactInputSingle(params);

// Log the amount of LUSD received for cash-settled exercise
const bobLusdBalance = await LUSD.balanceOf(bob.address);
console.log("LUSD received:", bobLusdBalance.toString());
```
{% endtab %}

{% tab log python %}
```python
# Warp forward to the exercise timestamp
w3.provider.make_request("evm_increaseTime", [earliest_exercise])
w3.provider.make_request("evm_mine", [])

# Bob exercises his options
clearinghouse.functions.exercise(option_id, 5).transact({"from": bob})

# Bob swaps his MAGIC for LUSD via Uniswap
bob_magic_balance = MAGIC.functions.balanceOf(bob).call()
swap_router = w3.eth.contract(address=SWAP_ROUTER_ADDRESS, abi=ISwapRouterABI)

params = {
    "tokenIn": MAGIC.address,
    "tokenOut": LUSD.address,
    "fee": 3000,
    "recipient": bob,
    "deadline": int(time.time()) + 1,
    "amountIn": bob_magic_balance,
    "amountOutMinimum": 0,
}

swap_router.functions.exactInputSingle(params).transact({"from": bob})

# Log the amount of LUSD received for cash-settled exercise
bob_lusd_balance = LUSD.functions.balanceOf(bob).call()
print("LUSD received:", bob_lusd_balance)
```
{% endtab %}

{% tab log rust %}
```rust
async fn warp_and_swap(
    provider: Arc<TimeManipulatingTransport<Provider<Http>>>,
    clearinghouse: Contract<TimeManipulatingTransport<Provider<Http>>>,
    option_id: U256,
) -> Result<(), Box<dyn std::error::Error>> {
    // Warp forward to the exercise timestamp
    provider.increase_time(earliest_exercise).await?;

    // Bob exercises his options
    let bob = Wallet::from_private_key(BOB_PRIVATE_KEY, provider.clone());
    let clearinghouse_as_bob = clearinghouse.connect_with(bob.clone());
    let tx = clearinghouse_as_bob
        .method::<_, H256>(
            "exercise",
            (option_id, 5),
            None,
            TransactionRequest::new(),
        )
        .send()
        .await?;

    let _ = tx.await_receipt().await?;

    // Bob swaps his MAGIC for LUSD via Uniswap
    let magic_contract = Contract::new(MAGIC_ADDRESS, MAGIC_ABI.to_vec(), provider.clone());
    let bob_magic_balance = magic_contract
        .connect_with(bob.clone())
        .method::<_, U256>("balanceOf", bob.address(), None, CallOptions::new())
        .call()
        .await?;

    let swap_router = Contract::new(SWAP_ROUTER_ADDRESS, ISwapRouterABI.to_vec(), provider.clone());
    let swap_router_as_bob = swap_router.connect_with(bob);

    let params = (
        MAGIC.address(),
        LUSD.address(),
        3000,
        bob.address(),
        (Utc::now() + Duration::seconds(1)).timestamp() as u64,
        bob_magic_balance,
        U256::zero(),
    );

    let tx = swap_router_as_bob
        .method::<_, H256>(
            "exactInputSingle",
            params,
            None,
            TransactionRequest::new(),
        )
        .send()
        .await?;

    let _ = tx.await_receipt().await?;

    // Log the amount of LUSD received for cash-settled exercise
    let lusd_contract = Contract::new(LUSD_ADDRESS, LUSD_ABI.to_vec(), provider.clone());
    let bob_lusd_balance = lusd_contract
        .connect_with(bob.clone())
        .method::<_, U256>("balanceOf", bob.address(), None, CallOptions::new())
        .call()
        .await?;

    println!("LUSD received: {}", bob_lusd_balance);

    Ok(())
}
```
{% endtab %}

{% tab log solidity %}
```solidity
// Warp forward to the exercise timestamp.
vm.warp(earliestExercise);

// Bob exercises his options.
vm.prank(BOB_ADDRESS);
clearinghouse.exercise(optionId, 5);

// Bob swaps his MAGIC for LUSD via Uniswap.
uint256 bobMagicBalance = MAGIC.balanceOf(address(this));
ISwapRouter.ExactInputSingleParams memory params = ISwapRouter.ExactInputSingleParams({
    tokenIn: address(MAGIC),
    tokenOut: address(LUSD),
    fee: 3000,
    recipient: address(this),
    deadline: block.timestamp + 1,
    amountIn: bobMagicBalance,
    amountOutMinimum: 0
});

// Log the amount of LUSD we received for cash settled exercise.
uint256 bobLusdBalance = LUSD.balanceOf(address(this));
```
{% endtab %}

{% endtabs %}

## Redeeming a short position with cash settlement

XYZ

{% tabs log %}

{% tab log js %}
```typescript
// Warp to the expiry timestamp
await provider.send("evm_increaseTime", [expiry]);
await provider.send("evm_mine", []);

// Check the position of our claim -- 5 options should be exercised, meaning
// 5 * 1e18 MAGIC and 1.5e18 LUSD should be available to redeem from the claim
const position = await clearinghouse.position(claimId);
const underlyingAmount = position[1];
const exerciseAmount = position[3];

// Redeem our claim
await clearinghouse.redeem(claimId);

// Swap any remaining MAGIC for LUSD via Uniswap
const magicBalance = await MAGIC.balanceOf(alice.address);
const swapRouter = new ethers.Contract(SWAP_ROUTER_ADDRESS, ISwapRouterABI, alice);

const params = {
  tokenIn: MAGIC.address,
  tokenOut: LUSD.address,
  fee: 3000,
  recipient: alice.address,
  deadline: Math.floor(Date.now() / 1000) + 1,
  amountIn: magicBalance,
  amountOutMinimum: 0,
  sqrtPriceLimitX96: 0,
};

await swapRouter.exactInputSingle(params);

// Log the amount of LUSD received for cash-settled writing
const lusdBalance = await LUSD.balanceOf(alice.address);
console.log("LUSD received from swap:", lusdBalance.toString());
console.log("LUSD received from claim:", exerciseAmount.toString());
console.log("Total LUSD received:", lusdBalance.add(exerciseAmount).toString());
```
{% endtab %}

{% tab log python %}
```python
import time

# Warp to the expiry timestamp
provider.time_travel(expiry)
provider.mine_block()

# Check the position of our claim -- 5 options should be exercised, meaning
# 5 * 1e18 MAGIC and 1.5e18 LUSD should be available to redeem from the claim
position = clearinghouse.functions.position(claimId).call()
underlyingAmount = position[1]
exerciseAmount = position[3]

# Redeem our claim
clearinghouse.functions.redeem(claimId).transact({'from': alice.address})

# Swap any remaining MAGIC for LUSD via Uniswap
magic_balance = MAGIC.functions.balanceOf(alice.address).call()
swap_router = web3.eth.contract(address=SWAP_ROUTER_ADDRESS, abi=ISwapRouterABI)

params = {
    'tokenIn': MAGIC.address,
    'tokenOut': LUSD.address,
    'fee': 3000,
    'recipient': alice.address,
    'deadline': int(time.time()) + 1,
    'amountIn': magic_balance,
    'amountOutMinimum': 0,
    'sqrtPriceLimitX96': 0,
}

swap_router.functions.exactInputSingle(params).transact({'from': alice.address})

# Log the amount of LUSD received for cash-settled writing
lusd_balance = LUSD.functions.balanceOf(alice.address).call()
print("LUSD received from swap:", lusd_balance)
print("LUSD received from claim:", exerciseAmount)
print("Total LUSD received:", lusd_balance + exerciseAmount)
```
{% endtab %}

{% tab log rust %}
```rust
use ethers::prelude::*;
use std::sync::Arc;
use tokio::time::Duration;
use chrono::Utc;

async fn redeem_and_swap(
    provider: Arc<TimeManipulatingTransport<Provider<Http>>>,
    clearinghouse: Contract<TimeManipulatingTransport<Provider<Http>>>,
    alice: Wallet<TimeManipulatingTransport<Provider<Http>>>,
    claim_id: U256,
) -> Result<(), Box<dyn std::error::Error>> {
    // Warp to the expiry timestamp
    provider.increase_time(expiry).await?;

    // Check the position of our claim -- 5 options should be exercised, meaning
    // 5 * 1e18 MAGIC and 1.5e18 LUSD should be available to redeem from the claim
    let position = clearinghouse
        .method::<_, (U256, i256, U256, i256)>("position", claim_id, None, CallOptions::new())
        .call()
        .await?;
    let underlying_amount = position.1;
    let exercise_amount = position.3;

    // Redeem our claim
    let tx = clearinghouse
        .connect_with(alice.clone())
        .method::<_, H256>("redeem", claim_id, None, TransactionRequest::new())
        .send()
        .await?;

    let _ = tx.await_receipt().await?;

    // Swap any remaining MAGIC for LUSD via Uniswap
    let magic_contract = Contract::new(MAGIC_ADDRESS, MAGIC_ABI.to_vec(), provider.clone());
    let magic_balance = magic_contract
        .connect_with(alice.clone())
        .method::<_, U256>("balanceOf", alice.address(), None, CallOptions::new())
        .call()
        .await?;

    let swap_router = Contract::new(SWAP_ROUTER_ADDRESS, ISwapRouterABI.to_vec(), provider.clone());
    let swap_router_as_alice = swap_router.connect_with(alice);

    let params = (
        MAGIC.address(),
        LUSD.address(),
        3000,
        alice.address(),
        (Utc::now() + Duration::seconds(1)).timestamp() as u64,
        magic_balance,
        U256::zero(),
        U256::zero(),
    );

    let tx = swap_router_as_alice
        .method::<_, H256>(
            "exactInputSingle",
            params,
            None,
            TransactionRequest::new(),
        )
        .send()
        .await?;

    let _ = tx.await_receipt().await?;

    // Log the amount of LUSD received for cash-settled writing
    let lusd_contract = Contract::new(LUSD_ADDRESS, LUSD_ABI.to_vec(), provider.clone());
    let alice_lusd_balance = lusd_contract
        .connect_with(alice.clone())
        .method::<_, U256>("balanceOf", alice.address(), None, CallOptions::new())
        .call()
        .await?;

    println!("LUSD received from swap: {}", alice_lusd_balance);
    println!("LUSD received from claim: {}", exercise_amount);
    println!(
        "Total LUSD received: {}",
        alice_lusd_balance.checked_add(exercise_amount).unwrap()
    );

    Ok(())
}
```
{% endtab %}

{% tab log solidity %}
```solidity
// Warp to the expiry timestamp.
vm.warp(expiry);

// Check the position of our claim -- 5 options should be exercised, totaling
// 5 * 1e18 MAGIC and 1.5e18 LUSD should be available to redeem from the claim.
(, int256 underlyingAmount,, int256 exerciseAmount) = clearinghouse.position(claimId);

// Redeem our claim.
vm.prank(ALICE_ADDRESS);
clearinghouse.redeem(claimId);

// Swap any remaining MAGIC for LUSD via Uniswap.
uint256 magicBalance = MAGIC.balanceOf(address(this));
ISwapRouter.ExactInputSingleParams memory params = ISwapRouter.ExactInputSingleParams({
    tokenIn: address(MAGIC),
    tokenOut: address(LUSD),
    fee: 3000,
    recipient: address(this),
    deadline: block.timestamp + 1,
    amountIn: magicBalance,
    amountOutMinimum: 0,
    sqrtPriceLimitX96: 0
});

// Log the amount of LUSD we received for cash settled writing.
uint256 lusdBalance = LUSD.balanceOf(address(this));
emit LogUint("LUSD received from swap", lusdBalance);
emit LogUint("LUSD received from claim", uint256(exerciseAmount));
emit LogUint("Total LUSD received", lusdBalance + uint256(exerciseAmount));
```
{% endtab %}

{% endtabs %}

## Conclusion

There we have it, a cash settled American option using the Valorem Clearinghouse. We picked up from the [physical settlement developer guide](/docs/dev-guide-write-asset-settled) and learned how to exercise a long position and redeem a short position into cash.

Please get in touch on our [Discord server](#) if you have any questions or feedback. We are always looking for ways to improve our documentation and tutorials. Good luck building!
