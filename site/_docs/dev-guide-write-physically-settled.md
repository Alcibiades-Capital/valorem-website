---
date: 2023-04-12 00:00:00
title: How to write physically settled options
description: Developer guide for writing physically settled options on digital assets with the Valorem Options Clearinghouse subgraph.
---

TODO remove code tabs for now
TODO cast commands

In this developer guide, we'll walk through how to write physically settled options on digital assets using the Valorem Options Clearinghouse.

We will use Alice and Bob as our two addressess. Alice will write a call option on the XYZ asset, transfer it to Bob, Bob will exercise the option (if it's in-the-money), and finally Alice will redeem her claim on the underlying/exercise assets from the Clearinghouse.

All code examples are provided in TypeScript // ethers-js, Python // web3.py, Rust // ethers-rs, and Solidity // forge.

## Creating an option type

First we'll set up our environment with a connection, a signer, and the Clearinghouse interface, and create a new option type in the Clearinghouse. This is done by calling the [`newOptionType`](/docs/clearinghouse-contracts/#newOptionType) function on the ABI.

For this tutorial, we'll create an American call option on ETH, with the following details:

- Underlying asset: WETH
- Underlying amount: 1
- Exercise asset: LUSD
- Exercise amount: 2100
- Exercise timestamp: Now
- Expiry timestamp: 1 week from now

#### Code Examples

{% tabs log %}

{% tab log js %}
```typescript
import { ethers } from 'ethers';

// Define the addresses for the Clearinghouse and the underlying assets
const CLEARINGHOUSE_ADDRESS = '0x...';
const WETH_ADDRESS = '0x...';
const LUSD_ADDRESS = '0x...';

// Connect to a provider (e.g., a local or remote Ethereum node)
const provider = new ethers.providers.JsonRpcProvider('http://localhost:8545');

// Set up a signer (e.g., a wallet) to interact with the Ethereum network
const signer = new ethers.Wallet('0x...').connect(provider);

// Define the ABI (Application Binary Interface) for the ValoremOptionsClearinghouse contract
const clearinghouseABI: ethers.ContractInterface = [
  // ...
  {
    "inputs": [
      {
        "internalType": "address",
        "name": "underlyingAsset",
        "type": "address"
      },
      {
        "internalType": "uint256",
        "name": "underlyingAmount",
        "type": "uint256"
      },
      // ... other input fields
    ],
    "name": "newOptionType",
    "outputs": [],
    "stateMutability": "nonpayable",
    "type": "function"
  }
  // ...
];

// Instantiate the ValoremOptionsClearinghouse contract
const clearinghouse = new ethers.Contract(CLEARINGHOUSE_ADDRESS, clearinghouseABI, signer);

// Define the option parameters
const underlyingAmount = ethers.utils.parseEther('1'); // Convert 1 ether to wei
const strike = ethers.utils.parseUnits('1000', 18); // Convert 1000 LUSD to its smallest unit (18 decimals)
const earliestExercise = Math.floor(Date.now() / 1000) + 60 * 60 * 24 * 7; // One week from now
const expiry = earliestExercise + 60 * 60 * 24 * 30; // One month after the earliest exercise

// Create a new option type and get the optionId
async function createOptionTypeAndGetId(): Promise<number> {
  const tx = await clearinghouse.newOptionType(
    WETH_ADDRESS,
    underlyingAmount,
    LUSD_ADDRESS,
    strike,
    earliestExercise,
    expiry
  );

  // Wait for the transaction to be mined
  const receipt = await tx.wait();
  const optionId = receipt.events?.[0]?.args?.optionId.toNumber();
  console.log('Option type created, optionId:', optionId);
  return optionId;
}

// Call the createOptionTypeAndGetId function
const optionId = await createOptionTypeAndGetId();
```
{% endtab %}

{% tab log python %}
```python
from web3 import Web3
import time

# Define the addresses for the Clearinghouse and the underlying assets
CLEARINGHOUSE_ADDRESS = '0x...'
WETH_ADDRESS = '0x...'
LUSD_ADDRESS = '0x...'

# Connect to a provider (e.g., a local or remote Ethereum node)
w3 = Web3(Web3.HTTPProvider('http://localhost:8545'))

# Set up a signer (e.g., a wallet) to interact with the Ethereum network
private_key = '0x...'
account = w3.eth.account.privateKeyToAccount(private_key)

# Define the ABI (Application Binary Interface) for the ValoremOptionsClearinghouse contract
clearinghouse_abi = [
    # ...
    {
        'inputs': [
            {
                'internalType': 'address',
                'name': 'underlyingAsset',
                'type': 'address'
            },
            {
                'internalType': 'uint256',
                'name': 'underlyingAmount',
                'type': 'uint256'
            },
            # ... other input fields
        ],
        'name': 'newOptionType',
        'outputs': [],
        'stateMutability': 'nonpayable',
        'type': 'function'
    }
    # ...
]

# Instantiate the ValoremOptionsClearinghouse contract
clearinghouse = w3.eth.contract(address=CLEARINGHOUSE_ADDRESS, abi=clearinghouse_abi)

# Define the option parameters
underlying_amount = w3.toWei(1, 'ether')  # Convert 1 ether to wei
strike = w3.toWei(1000, 'ether')  # Convert 1000 LUSD to its smallest unit (18 decimals)
earliest_exercise = int(time.time()) + 60 * 60 * 24 * 7  # One week from now
expiry = earliest_exercise + 60 * 60 * 24 * 30  # One month after the earliest exercise

# Create a new option type and get the optionId
def create_option_type_and_get_id():
    tx = clearinghouse.functions.newOptionType(
        WETH_ADDRESS,
        underlying_amount,
        LUSD_ADDRESS,
        strike,
        earliest_exercise,
        expiry
    ).buildTransaction({
        'from': account.address,
        'gas': 3000000,
        'gasPrice': w3.toWei('30', 'gwei'),
        'nonce': w3.eth.getTransactionCount(account.address),
    })

    signed_tx = w3.eth.account.signTransaction(tx, private_key)
    tx_hash = w3.eth.sendRawTransaction(signed_tx.rawTransaction)
    tx_receipt = w3.eth.waitForTransactionReceipt(tx_hash)

    option_id = tx_receipt['logs'][0]['topics'][1]
    print('Option type created, optionId:', option_id)
    return option_id

# Call the create_option_type_and_get_id function
option_id = create_option_type_and_get_id()
```
{% endtab %}

{% tab log rust %}
```rust
use ethers::{
    contract::Contract,
    prelude::{Address, Eth, Middleware, PrivateKey, SignerMiddleware, TransactionRequest, H256, U256, U64},
};
use std::{convert::TryFrom, str::FromStr, time::SystemTime};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Define the addresses for the Clearinghouse and the underlying assets
    let clearinghouse_address = Address::from_str("0x...")?;
    let weth_address = Address::from_str("0x...")?;
    let lusd_address = Address::from_str("0x...")?;

    // Connect to a provider (e.g., a local or remote Ethereum node)
    let provider = ethers::providers::Provider::<ethers::providers::Http>::try_from("http://localhost:8545")?;

    // Set up a signer (e.g., a wallet) to interact with the Ethereum network
    let private_key = PrivateKey::from_str("0x...")?;
    let signer = SignerMiddleware::new(provider, private_key);

    // Define the ABI (Application Binary Interface) for the ValoremOptionsClearinghouse contract
    let clearinghouse_abi = r#"[{"inputs": [{"internalType": "address","name": "underlyingAsset","type": "address"},{"internalType": "uint256","name": "underlyingAmount","type": "uint256"}],"name": "newOptionType","outputs": [],"stateMutability": "nonpayable","type": "function"}]"#;

    // Instantiate the ValoremOptionsClearinghouse contract
    let clearinghouse = Contract::from_json(signer, clearinghouse_address, clearinghouse_abi.as_bytes())?;

    // Define the option parameters
    let underlying_amount = U256::exp10(18); // Convert 1 ether to wei
    let strike = U256::from(1000) * U256::exp10(18); // Convert 1000 LUSD to its smallest unit (18 decimals)
    let earliest_exercise = U64::from(
        SystemTime::now()
            .duration_since(SystemTime::UNIX_EPOCH)?
            .as_secs() + 60 * 60 * 24 * 7,
    ); // One week from now
    let expiry = earliest_exercise + 60 * 60 * 24 * 30; // One month after the earliest exercise

// Create a new option type and get the optionId
async fn create_option_type_and_get_id() -> Result<U256, Box<dyn std::error::Error>> {
    let tx = clearinghouse
        .method::<_, H256>(
            "newOptionType",
            (weth_address, underlying_amount, lusd_address, strike, earliest_exercise, expiry),
            None,
            TransactionRequest::new(),
        )
        .send()
        .await?;

    let receipt = tx.await_receipt().await?;
    let option_id = U256::from_big_endian(
        &receipt
            .logs
            .get(0)
            .ok_or("No logs found")?
            .topics
            .get(1)
            .ok_or("No optionId found")?
            .as_fixed_bytes(),
    );

    println!("Option type created, optionId: {:?}", option_id);
    Ok(option_id)
}

// Call the create_option_type_and_get_id function
let option_id = create_option_type_and_get_id().await?;
```
{% endtab %}

{% tab log solidity %}
```solidity
// Instantiate the IValoremOptionsClearinghouse interface
IValoremOptionsClearinghouse clearinghouse = new IValoremOptionsClearinghouse(CLEARINGHOUSE_ADDRESS);

// Create a new option type and get the optionId
uint256 optionId = clearinghouse.newOptionType({
    underlyingAsset: WETH_ADDRESS,
    underlyingAmount: 1 ether,
    exerciseAsset: LUSD_ADDRESS,
    exerciseAmount: strike,
    exerciseTimestamp: earliestExercise,
    expiryTimestamp: expiry
});
```
{% endtab %}

{% endtabs %}


## Writing an option

Next we will write 10 options for this option type. In return, we will receive 10 fungible option tokens, representing our long position, and 1 claim NFT, representing our short position. This is done by calling the [`write`](/docs/clearinghouse-contracts/#write) function.

{% tabs log %}

{% tab log js %}
```typescript
// Write the option with a specified optionId and quantity
async function writeOption(optionId: number, quantity: number): Promise<void> {
  const tx = await clearinghouse.write(optionId, quantity);

  // Wait for the transaction to be mined
  const receipt = await tx.wait();
  console.log('Option written:', receipt);
}

// Call the writeOption function with the obtained optionId
writeOption(optionId, 10);
```
{% endtab %}

{% tab log python %}
```python
# Write the option with a specified optionId and quantity
def write_option(option_id, quantity):
    tx = clearinghouse.functions.write(option_id, quantity).buildTransaction({
        'from': account.address,
        'gas': 3000000,
        'gasPrice': w3.toWei('30', 'gwei'),
        'nonce': w3.eth.getTransactionCount(account.address),
    })

    signed_tx = w3.eth.account.signTransaction(tx, private_key)
    tx_hash = w3.eth.sendRawTransaction(signed_tx.rawTransaction)
    tx_receipt = w3.eth.waitForTransactionReceipt(tx_hash)

    print('Option written:', tx_receipt)

# Call the write_option function with the obtained optionId
write_option(option_id, 10)
```
{% endtab %}

{% tab log rust %}
```rust
// Write the option with a specified optionId and quantity
async fn write_option(option_id: U256, quantity: u64) -> Result<(), Box<dyn std::error::Error>> {
    let tx = clearinghouse
        .method::<_, H256>(
            "write",
            (option_id, quantity),
            None,
            TransactionRequest::new(),
        )
        .send()
        .await?;

    let receipt = tx.await_receipt().await?;
    println!("Option written: {:?}", receipt);

    Ok(())
}

// Call the write_option function with the obtained optionId
write_option(option_id, 10).await?;
```
{% endtab %}

{% tab log solidity %}
```solidity
// Write the option with a specified optionId and quantity
clearinghouse.write(optionId, 10);
```
{% endtab %}

{% endtabs %}

## Transferring a long position

Now that we have written the option, we can transfer our long position to another address. This is done by calling the [`safeTransferFrom`](https://eips.ethereum.org/EIPS/eip-1155#specification) function, native to the IERC1155 interface.

{% tabs log %}

{% tab log js %}
```typescript
// Define the address of the recipient, Bob
const BOB_ADDRESS = '0x...';

// Transfer options to Bob's address
async function transferOptionsToBob(optionId: number, quantity: number): Promise<void> {
  const tx = await clearinghouse.transferFrom(await signer.getAddress(), BOB_ADDRESS, optionId, quantity);

  // Wait for the transaction to be mined
  const receipt = await tx.wait();
  console.log('Options transferred to Bob:', receipt);
}

// Call the transferOptionsToBob function with the obtained optionId
transferOptionsToBob(optionId, 10);
```
{% endtab %}

{% tab log python %}
```python
# Define the address of the recipient, Bob
BOB_ADDRESS = '0x...'

# Transfer options to Bob's address
def transfer_options_to_bob(option_id, quantity):
    tx = clearinghouse.functions.transferFrom(
        account.address, BOB_ADDRESS, option_id, quantity
    ).buildTransaction({
        'from': account.address,
        'gas': 3000000,
        'gasPrice': w3.toWei('30', 'gwei'),
        'nonce': w3.eth.getTransactionCount(account.address),
    })

    signed_tx = w3.eth.account.signTransaction(tx, private_key)
    tx_hash = w3.eth.sendRawTransaction(signed_tx.rawTransaction)
    tx_receipt = w3.eth.waitForTransactionReceipt(tx_hash)

    print('Options transferred to Bob:', tx_receipt)

# Call the transfer_options_to_bob function with the obtained optionId
transfer_options_to_bob(option_id, 10)
```
{% endtab %}

{% tab log rust %}
```rust
// Define the address of the recipient, Bob
let bob_address = "0x...".parse::<Address>()?;

// Transfer options to Bob's address
async fn transfer_options_to_bob(
    option_id: U256,
    quantity: u64,
) -> Result<(), Box<dyn std::error::Error>> {
    let tx = clearinghouse
        .method::<_, H256>(
            "transferFrom",
            (signer.address(), bob_address, option_id, quantity),
            None,
            TransactionRequest::new(),
        )
        .send()
        .await?;

    let receipt = tx.await_receipt().await?;
    println!("Options transferred to Bob: {:?}", receipt);

    Ok(())
}

// Call the transfer_options_to_bob function with the obtained optionId
transfer_options_to_bob(option_id, 10).await?;
```
{% endtab %}

{% tab log solidity %}
```solidity
// Define the address of the recipient, Bob
address BOB_ADDRESS = address(0xB0B);

// Transfer options to Bob's address
clearinghouse.transferFrom(address(this), BOB_ADDRESS, optionId, 10);
```
{% endtab %}

{% endtabs %}

## Exercising an option

Now that we have written the option and transferred our long position to Bob, as long as we are within the exercise window defined on the option type (on or after `exerciseTimestamp`, before `expiryTimestamp), holders of option tokens can exercise their options.

Let's imagine for this tutorial the position is finishing in-the-money, and Bob wants to exercise his options. This is done by calling the [`exercise`](/docs/clearinghouse-contracts/#exercise) function.

{% tabs log %}

{% tab log js %}
```typescript
// Define a provider that supports the required methods for time manipulation
const provider = new ethers.providers.JsonRpcProvider('http://localhost:8545');

// Warp forward to the exercise timestamp and pretend to be Bob
async function warpAndExercise(optionId: number, quantity: number): Promise<void> {
  // Increase the time on the local blockchain
  await provider.send('evm_increaseTime', [earliestExercise]);
  await provider.send('evm_mine', []);

  // Connect to the clearinghouse contract as Bob
  const bob = new ethers.Wallet(BOB_PRIVATE_KEY, provider);
  const clearinghouseAsBob = clearinghouse.connect(bob);

  // Bob exercises his options
  const tx = await clearinghouseAsBob.exercise(optionId, quantity);

  // Wait for the transaction to be mined
  const receipt = await tx.wait();
  console.log('Options exercised by Bob:', receipt);
}

// Call the warpAndExercise function with the obtained optionId
warpAndExercise(optionId, 10);
```
{% endtab %}

{% tab log python %}
```python
# Warp forward to the exercise timestamp and pretend to be Bob
def warp_and_exercise(option_id, quantity):
    # Increase the time on the local blockchain
    w3.provider.make_request("evm_increaseTime", [earliest_exercise])
    w3.provider.make_request("evm_mine", [])

    # Connect to the clearinghouse contract as Bob
    bob_account = w3.eth.account.privateKeyToAccount(BOB_PRIVATE_KEY)
    clearinghouse_as_bob = w3.eth.contract(
        address=clearinghouse.address,
        abi=clearinghouse.abi,
        DefaultAccount=bob_account.address,
    )

    # Bob exercises his options
    tx = clearinghouse_as_bob.functions.exercise(option_id, quantity).buildTransaction({
        'from': bob_account.address,
        'gas': 3000000,
        'gasPrice': w3.toWei('30', 'gwei'),
        'nonce': w3.eth.getTransactionCount(bob_account.address),
    })

    signed_tx = w3.eth.account.signTransaction(tx, BOB_PRIVATE_KEY)
    tx_hash = w3.eth.sendRawTransaction(signed_tx.rawTransaction)
    tx_receipt = w3.eth.waitForTransactionReceipt(tx_hash)

    print('Options exercised by Bob:', tx_receipt)

# Call the warp_and_exercise function with the obtained optionId
warp_and_exercise(option_id, 10)
```
{% endtab %}

{% tab log rust %}
```rust
use ethers::prelude::*;
use std::time::Duration;
use tokio::time::sleep;

// Define a custom transport that supports the required methods for time manipulation
pub struct TimeManipulatingTransport<M: Middleware>(pub M);

impl<M: Middleware> TimeManipulatingTransport<M> {
    // Increase the time on the local blockchain
    pub async fn increase_time(&self, seconds: u64) -> Result<(), M::Error> {
        let _ = self.0.send("evm_increaseTime", vec![serde_json::json!(seconds)]).await?;
        let _ = self.0.send("evm_mine", vec![]).await?;
        Ok(())
    }
}

// Warp forward to the exercise timestamp and pretend to be Bob
async fn warp_and_exercise(
    provider: Arc<TimeManipulatingTransport<Provider<Http>>>,
    option_id: U256,
    quantity: u64,
) -> Result<(), Box<dyn std::error::Error>> {
    // Increase the time on the local blockchain
    provider.increase_time(earliest_exercise).await?;

    // Connect to the clearinghouse contract as Bob
    let bob = Wallet::from_private_key(BOB_PRIVATE_KEY, provider.clone());
    let clearinghouse_as_bob = clearinghouse.connect_with(bob);

    // Bob exercises his options
    let tx = clearinghouse_as_bob
        .method::<_, H256>(
            "exercise",
            (option_id, quantity),
            None,
            TransactionRequest::new(),
        )
        .send()
        .await?;

    let receipt = tx.await_receipt().await?;
    println!("Options exercised by Bob: {:?}", receipt);

    Ok(())
}

// Call the warp_and_exercise function with the obtained optionId
let provider_with_time_manipulation = Arc::new(TimeManipulatingTransport(provider));
warp_and_exercise(provider_with_time_manipulation, option_id, 10).await?;
```
{% endtab %}

{% tab log solidity %}
```solidity
        // Warp forward to the exercise timestamp.
        vm.warp(earliestExercise);

        // Bob exercises his options.
        vm.prank(BOB_ADDRESS);
        clearinghouse.exercise(optionId, 10);
```
{% endtab %}

{% endtabs %}

## Redeeming a Claim

Finally, we will redeem our claim NFT, which returns the `underlyingAsset` for unexercised positions, the `exerciseAsset` for fully exercised positions, and a mix of both assets in the correct proportions for partially exercised positions. This is done by calling the [`redeem`](/docs/clearinghouse-contracts/#redeem) function.

{% tabs log %}

{% tab log js %}
```typescript
// Warp to the expiry timestamp and redeem the claim NFT
async function warpAndRedeem(claimId: number): Promise<void> {
  // Increase the time on the local blockchain
  await provider.send('evm_increaseTime', [expiry]);
  await provider.send('evm_mine', []);

  // Alice redeems her claim
  const tx = await clearinghouse.redeem(claimId);

  // Wait for the transaction to be mined
  const receipt = await tx.wait();
  console.log('Claim redeemed by Alice:', receipt);
}

// Call the warpAndRedeem function with the obtained claimId
warpAndRedeem(claimId);
```
{% endtab %}

{% tab log python %}
```python
# Warp to the expiry timestamp and redeem the claim NFT
def warp_and_redeem(claim_id):
    # Increase the time on the local blockchain
    w3.provider.make_request("evm_increaseTime", [expiry])
    w3.provider.make_request("evm_mine", [])

    # Alice redeems her claim
    tx = clearinghouse.functions.redeem(claim_id).buildTransaction({
        'from': account.address,
        'gas': 3000000,
        'gasPrice': w3.toWei('30', 'gwei'),
        'nonce': w3.eth.getTransactionCount(account.address),
    })

    signed_tx = w3.eth.account.signTransaction(tx, private_key)
    tx_hash = w3.eth.sendRawTransaction(signed_tx.rawTransaction)
    tx_receipt = w3.eth.waitForTransactionReceipt(tx_hash)

    print('Claim redeemed by Alice:', tx_receipt)

# Call the warp_and_redeem function with the obtained claimId
warp_and_redeem(claim_id)
```
{% endtab %}

{% tab log rust %}
```rust
// Warp to the expiry timestamp and redeem the claim NFT
async fn warp_and_redeem(
    provider: Arc<TimeManipulatingTransport<Provider<Http>>>,
    claim_id: U256,
) -> Result<(), Box<dyn std::error::Error>> {
    // Increase the time on the local blockchain
    provider.increase_time(expiry).await?;

    // Alice redeems her claim
    let tx = clearinghouse
        .method::<_, H256>(
            "redeem",
            claim_id,
            None,
            TransactionRequest::new(),
        )
        .send()
        .await?;

    let receipt = tx.await_receipt().await?;
    println!("Claim redeemed by Alice: {:?}", receipt);

    Ok(())
}

// Call the warp_and_redeem function with the obtained claimId
warp_and_redeem(provider_with_time_manipulation, claim_id).await?;
```
{% endtab %}

{% tab log solidity %}
```solidity
// Warp to the expiry timestamp.
vm.warp(expiry);

// Alice redeems her claim.
vm.prank(ALICE_ADDRESS);
clearinghouse.redeem(claimId);
```
{% endtab %}

{% endtabs %}

## Conclusion

There we have it, a physically settled American option using the Valorem Clearinghouse. We learned how to setup our environment, create a new option type, transfer the long position to another address, exercise this long position, and finally redeem our claim representing the collateral of the short position.

Please get in touch on our [Discord server](#) if you have any questions or feedback. We are always looking for ways to improve our documentation and tutorials. Good luck building!
