---
layout: post
title:  "Rescuing ENS names from compromised wallets."
date:   2022-05-18 20:28:22 -0400
categories: web3
---
![image](/assets/img/lost.jpeg)

Recently [we](https://ens.vision) had a client message our support channel asking about strange transactions.  

Shortly after we determined the users wallet had been compromised and a drainer/sweeper script attached to the address.  

Understandably the user was quite frantic but patient and allowed us to come up with a plan to try and help rescue his ENS names.  

# The "drainer" problem:

Generally when a wallet is compromised, an attacker will leave some tokens as bait for the owner to send funds to try and transfer.  

The trap works as follows: 

1) Wallet is compromised & drained of ETH. The attacker leaves some tokens in the compromised wallet.  

2) This means the owner can not transfer the tokens in the wallet until they send it more ether to cover the gas fee.

3) Wait for the legitimate owner to try and save their precious tokens by sending more ETH to cover the gas fees of transfer methods and and steal that ETH also.   

A "sweeper" loads a private key of a hacked account and attempts to send the full balance in a loop.  

If there is no balance it simply reverts before broadcasting. 

If it has a balance the funds are sent instantly.  

This kind of attack is low effort and cost as it only requires a web3 rpc connection to query an accounts balance and broadcast transactions.  

Enabling it to run indefinately for basically free.  

Below is an example of a basic drainer script that attempts to send all ETH every 60 seconds if it has enough balance to cover the cost of gas:  

```python
from web3 import Web3, HTTPProvider
import time

# Connect to Ethereum node
w3 = Web3(HTTPProvider('INFURA'))  # Replace with your node address

# Replace with your account details
account = "YOUR_ACCOUNT_ADDRESS"
private_key = "YOUR_PRIVATE_KEY"
destination_account = "DESTINATION_ACCOUNT_ADDRESS"

def send_all_eth(account, private_key, destination_account):
    # Get account balance
    balance = w3.eth.getBalance(account)

    if balance > 0:
        # Estimate gas
        gas_estimate = w3.eth.estimateGas({
            "from": account,
            "to": destination_account,
            "value": balance
        })
        # Check if balance is sufficient for gas
        if balance > gas_estimate * w3.eth.gasPrice:
            # Build transaction
            transaction = {
                "from": account,
                "to": destination_account,
                "value": balance - gas_estimate * w3.eth.gasPrice,
                "gas": gas_estimate,
                "gasPrice": w3.eth.gasPrice,
                "nonce": w3.eth.getTransactionCount(account)
            }
            # Sign transaction
            signed_transaction = w3.eth.account.signTransaction(transaction, private_key)
            # Send transaction
            tx_hash = w3.eth.sendRawTransaction(signed_transaction.rawTransaction)
            print(f"Transaction hash: {tx_hash.hex()}")
        else:
            print("Balance is not sufficient to cover gas costs.")
    else:
        print("Balance is zero.")

while True:
    # Send all Ethereum from account to destination_account
    send_all_eth(account, private_key, destination_account)
    # Sleep for 60 seconds
    time.sleep(60)
```

# The solution: Flashbots

[Flashbots](https://docs.flashbots.net/) on a high level enables a client to submit Bundles of transactions.  

[Understanding bundles](https://docs.flashbots.net/flashbots-auction/searchers/advanced/understanding-bundles)  

Bundles are one or more transactions that are grouped together and executed in the order they are provided.  

Note: if a transaction in a bundle would revert then the whole bundle will be disgarded and not be included in a block. Only bundles of successful transactions will be included unless reverting transactions are specified when submitting the bundle which explicitaly state which transactions are allowed to revert.    

All of this sounds great and allows technical people to submit bundles to a Flashbots relay through the use of specifically written code that usually loads the accounts private key directly. As demonstrated above with the web3.py example.  

What about people who are simply not technical enough to run specialized python or node scripts to build and submit Flashbots bundles?

Flashbots have recently updated the RPC to enable transaction or [Bundle Caching](https://docs.flashbots.net/flashbots-protect/rpc/bundle-cache).  

A client simply needs to generate a special id that works as a bundle id with the flashbots RPC in Metamask.  
Example: `https://rpc.flashbots.net/bundle?id=111-222-333-444`  

# Building a better bundler

The Scaffold ETH bundler was fairly basic and only allowed Sending ETH and entering a contracts ABI to add a transaction from.  
-picture-  

That works well for fungible tokens such as ERC20 or ERC1155 that can transfer multiple tokens or the full balance in a single transaction.  

ERC721 tokens have no methods to transfer multiple tokens in a single transaction. Tokens such as ENS would require adding 1x safeTransferFrom or transferFrom call per tokenId to a bundle.  

In the past there were size limitations on bundles. Unsure of the current size limitations of a bundle I decided to write a small contract to aid in transferring all of this clients ENS names.  

A basic `transferProxy` contract was written and used for this client.  

The contract needs to be approved by the _from address on the collection address.  

The contract also requires the caller be approved by the `from` address as an authentication check to prevent users transferring other peoples names who have approved the contract.  

### Basic transferProxy.sol  
```java
contract Transfer {

    function bulkTransfer(uint256[] calldata tokens, address from, address to, address collection ) public {
        uint256 length = tokens.length;

        // check if caller is approved by the owner on collection to transfer tokens
        (,bytes memory dataApproved ) = collection.staticcall(abi.encodeWithSignature("isApprovedForAll(address,address)", from, msg.sender));
        bool isApproved = abi.decode(dataApproved, (bool));
        require(isApproved, "Caller Not approved by Owner");

        unchecked {
            for (uint256 i = 0; i < length;) {
                // transfer token from the owner to the receiver
                collection.call(abi.encodeWithSignature("safeTransferFrom(address,address,uint256)", from, to, tokens[i]));
                ++i;
            }
        }
    }
}
```
The contract was refined and rewritten in yul to be more effecient :  

### Optimized transferProxy.sol
```java
contract transferProxy {

    // custom error = 0x383462e29
    error notApproved();

    function transfer(bytes[] calldata _data, address _contract, address _from) external {
        assembly {
            // checks if the caller is approved by _from on _contract
            //
            // store the selector for isApprovedForAll at memory offset 0x00-0x04
            mstore(0x00, shl(224, 0xe985e9c5)) 
            // store the _from parameter in the calldata 0x04-0x24
            mstore(0x04, _from) 
            // store the msg.sender as the operator parameter at 0x24-0x44
            mstore(0x24, caller())
            // perform staticcall on the contract with the stored calldata from memory 0x00 - 0x44 
            let success := staticcall(gas(), _contract, 0x00, 0x44, 0x00, 0x00)
            // call is finished, overwrite the calldata at 0x00 with returndata
            returndatacopy(0x00, 0x00, returndatasize())
            // if call failed then revert
            if iszero(success) {
                revert(0x00, returndatasize())
            }
            // if the caller isnt approved revert with custom error 'notApproved'
            if iszero(mload(0x00)) {
                // store the selector for the notApproved error at 0x00 - 0x4
                mstore(0x00, shl(224, 0x383462e29))
                // revert with the custom error selector at 0x00-0x4
                revert(0x00, 0x04)
            }

            let i := 0
            for {} 1 { i:= add(i, 1) } {
                // i starts at 0 while length starts at 1
                if eq(i, _data.length){ break }
                // elements start at _data.offset
                // +0x00: length
                // +0x20: data
                // shl(5, i) = mul(0x20, i)
                let data := calldataload(add(_data.offset, shl(5, i)))
                let len := calldataload(add(_data.offset, data))
                // copy the bytes string from the array to 0x00 
                calldatacopy(0x00, add(_data.offset, add(data, 0x20)), len)
                // call the _contract with the calldata copied to 0x00    
                success := call( gas(), _contract, 0x00, 0x00, len, 0x00, 0x00)
                // if a call reverts then revert the entire call.
                if iszero(success) {
                    // copy the revert reason to 0x00
                    returndatacopy(0x00, 0x00, returndatasize())
                    // return the revert from 0x00
                    revert(0x00, returndatasize())
                }
            }
        }
    }
}
```
# Building a better bundler .. cont.

With a more effecient ERC721 transfer contract I then added the code to the UI to check if the specified contract was an ERC721 or ERC1155.  

This is accomplished by checking if the contract implements the `supportsInterface(ERC721InterfaceId)` or `supportsInterface(ERC1155InterfaceId)`.  

Where `ERC1155InterfaceId = "0xd9b67a26"` and `ERC721InterfaceId = "0x80ac58cd"`.  

Now the function will use the built-in ERC1155 safeBatchTransferFrom method for ERC1155 tokens and the transferProxy for ERC721 tokens.  

Some additional functionality added and UI changes include:  

1) Added Transfer NFTs tab that checks and selects between using the transferProxy for ERC721 or the built-in safeBatchTransferFrom for ERC1155 tokens.  

2) Added the ERC20 tab that will check the connected wallets balance for the provided contract and offer a 1-click button to add a transaction to a bundle that transfers the connected wallets entire balance to a new address.  

3) Added a New Bundle tab and button for generating a new UUID / Bundle ID for easily submitting multiple bundles.  

4) Added a button for automatically adding and changing to the bundle RPC in MetaMask.  

5) Added a Submit Bundle tab for submitting finalized bundles and crafting bundles easier.  

# Rescue  

With all of the code ready I decided to ask the client in this instance to provide the compromised private key to make sure everything went smooth and execute the transactions from my wallet for gas.  

1) Imported the private key of the compromised wallet in Metamask.  
2) Changed to clean address in Metamask with ETH.  
3) Clicked the New Bundle tab.  
4) Generate a new bundle UUID.  
5) Click the button to add the RPC to Metamask.  
6) Click the Send ETH tab.  
7) Click to estimate the cost.    
8) As ENS is an ERC721 we need to do two approvals. Copy the cost for two approvals.  
9) Click Send Eth to add the transaction to the bundle.  
10) Change to the hacked wallet in metamask.  
11) Click the Set NFT Approvals tab.  
12) Enter the ENS contract address.  
13) Enter the hacked address that holds the ENS names.   
14) Enter the unhacked address that will receive the names.   
15) Click to add the first approval tx to the bundle.  This approves the new address for the hacked address.  
16) Click to add the second approval tx to the bundle. This approves the transferProxy contract.  
17) Click the submit bundle tab. Wait for the bundle to be included.  
18) Optional: Disconnect the flashbots RPC and connect to Ethereum Mainnet in Metamask.  
19) Click the Transfer NFT's tab.  
20) Enter the ENS contract address.  
21) Enter the hacked address that holds the ENS names.  
22) Enter the unhacked address that will receive the names.   
23) Enter the tokenIds 1 per line in the input area.  
24) Click to broadcast the transaction.    

# PoC
