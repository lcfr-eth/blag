---
layout: post
title:  "Bounty: Using a Web2 bug to duplicate ENS names"
date:   2022-05-01 20:28:22 -0400
categories: web3
---
![image](/assets/img/lost.jpeg)

##### Quick Links 
[About ENS: Decentralized Domain Name System](#about-ens)  
[ENS Registration & Metadata explained](#ens-registration--metadata-explained)  
[ENS Subgraph Service](#ens-subgraph-service)  
[ENS Metadata Service](#ens-metadata-service)  
[ZWJ, ZWNJ, ZWSP ?](#zwj-zwnj-zwsp-)  
[Null-bytes & String termination](#null-bytes--string-termination)  
[Idea](#idea)  
[Results](#results)  
[The Patche(s)](#the-patchs)    
[Revisited 12/2022](#revisited-122022)  
[Reward](#reward)  

# About ENS: Decentralized Domain Name System

ENS (Ethereum Name Service) is a decentralized domain name system built on Ethereum. Its goal is to make the process of accessing decentralized websites, applications, and resources much easier and user-friendly.

ENS replaces complicated addresses such as "0x742d35Cc6634C0532925a3b844Bc454e4438f44e" with human-readable names like "myaddress.eth". This allows users to remember and share their Ethereum addresses more easily, just like with traditional domain name. 

[Learn more about ENS](https://docs.ens.domains/)

# ENS Registration & Metadata explained

When registering an ENS name you are registering the keccak256 "hash" of the name. The hash of the name and owner are stored as a mapping in the contract. An event is emitted that is eventually used to create the metadata entry for a given name. As the data is hashed its not possible to reverse to determine the plain text name directly. This keccak256 hash of the name when converted to a uint256 is the names TokenID.  

ENS Metadata is two parts. The Subgraph and the Metadata services. The subgraph services is responsible for catching contract events and adding them to a queryable database. 

The metadata service queries this database to return a json object from the database results of a name or tokenId.  

dApps query the metadata service URL for information on a name/tokenId.  

### ENS Subgraph Service

The subgraph catches events emitted from the various ENS contracts and adds them to the subgraph database for easy querying from web UI or by direct API. 

[Public ENS Subgraph instance via TheGraph](https://thegraph.com/hosted-service/subgraph/ensdomains/ens)  
[ENS Subgraph Daemon code](https://github.com/ensdomains/ens-subgraph)  
[ENS Subgraph documentation](https://docs.ens.domains/contract-api-reference/subgraphdata)  

subgraph.yaml snippet which defines a contracts, events and event handler functions.  

```
  - kind: ethereum/contract
    name: EthRegistrarController
    network: mainnet
    source:
      address: '0x283Af0B28c62C092C9727F1Ee09c02CA627EB7F5' <--- this is the contract address to watch
      abi: EthRegistrarController
    mapping:
      kind: ethereum/events
      apiVersion: 0.0.5
      language: wasm/assemblyscript
      file: ./src/ethRegistrar.ts
      entities:
        - Registration
      abis:
        - name: EthRegistrarController
          file: ./abis/EthRegistrarController.json
      eventHandlers:
        - event: >-
            NameRegistered(string,indexed bytes32,indexed address,uint256,uint256) <--- this is the event to catch for
            handler: handleNameRegisteredByController <--- this is the function that handles the events from the contract

```  
Below is an example event handler to handle registration events from the Controller contract.  

```typescript
export function handleNameRegisteredByController(event: ControllerNameRegisteredEvent): void {
  setNamePreimage(event.params.name, event.params.label, event.params.baseCost.plus(event.params.premium))
}
```

### ENS Metadata Service
This is where a names metadata is stored per name/tokenID. This is the endpoint in which services query to return information about a supplied name or tokenId. 

https://metadata.ens.domains

This queiries theGraph instance to return the metadata for a given name/tokenID. 

# ZWJ, ZWNJ, ZWSP ?

ZWJ, ZWNJ, and ZWSP are all individual byte sequences to join or space unicode characters properly for display purposes that essentially end up "render" as an invisible character in a string. 

[ZWJ - Zero Width Joiner](https://en.wikipedia.org/wiki/Zero-width_joiner).  
[ZWNJ - Zero With Non Joiner](https://en.wikipedia.org/wiki/Zero-width_non-joiner).  
[ZWSP = Zero Width Space](https://en.wikipedia.org/wiki/Zero-width_space).

Each of these are actually three individual bytes in sequence as seen below:  

[ZWJ - 0xE2 0x80 0x8D](https://www.fileformat.info/info/unicode/char/200d/index.htm)  
[ZWNJ - 0xE2 0x80 0x8C](https://www.fileformat.info/info/unicode/char/200c/index.htm)  
[ZWSP - 0xE2 0x80 0x8B](https://www.fileformat.info/info/unicode/char/200b/index.htm)  

Appending or prepending any of these byte sequences to an ENS name results in an invisible character in-place. To create a name with a ZWNJ for example one could register an ENS name such as "testzwsp"+u"\u200B" in python to create an ENS name with an invisible character appended to the end of the name. The characters are invisible but still show in the names length field on metadata.  

An example of a name with a ZWSP was registered "testzwsp"+u"\u200B". This will create an ENS name of length 9 total. 8 for ("testzwsp") + 1 for the invisible unicode zero width space we inserted after the text.

[testzwsp.eth metadata](https://metadata.ens.domains/mainnet/0x57f1887a8bf19b14fc0df6fd9b2acc9af147ea85/87168065001888073846593886985622493818274710011410357743988130633182109658955)


```json
{
  "is_normalized":false,
  "name":"[0xc0b7...6f4b].eth",
  "description":"[0xc0b7...6f4b].eth, an ENS name. (testzwsp​.eth is not in normalized form) ⚠️ ATTENTION: This name contains non-ASCII characters as shown above. Please be aware that there are characters that look identical or very similar to English letters, especially characters from Cyrillic and Greek. Also, traditional Chinese characters can look identical or very similar to simplified variants. For more information: https://en.wikipedia.org/wiki/IDN_homograph_attack",

  "attributes":[
    {"trait_type":"Created Date","display_type":"date","value":1672119599000},
    {"trait_type":"Length","display_type":"number","value":9},
    {"trait_type":"Segment Length","display_type":"number","value":9},
    {"trait_type":"Character Set","display_type":"string","value":"mixed"},
    {"trait_type":"Registration Date","display_type":"date","value":1672119599000},
    {"trait_type":"Expiration Date","display_type":"date","value":1674538799000}
  ],
  "name_length":9,
  "segment_length":9,
  "url":null,
  "version":0,
  "image":"https://metadata.ens.domains/mainnet/0x57f1887a8bf19b14fc0df6fd9b2acc9af147ea85/0xc0b7605c7c38d15e9babd8edbde1427b6fe8d926104130267dc6e8acdcd66f4b/image",
  "image_url":"https://metadata.ens.domains/mainnet/0x57f1887a8bf19b14fc0df6fd9b2acc9af147ea85/0xc0b7605c7c38d15e9babd8edbde1427b6fe8d926104130267dc6e8acdcd66f4b/image"
}
```

![image](/assets/img/duplicateensbounty/zwsp.PNG)

As pictured you can see the length of the name is X which is the length of "" + the 3 bytes for the ZWNJ sequence. Knowing this a smart ENS user can avoid scams by checking the length of a name to verify if it is the intended name. 

# Null-bytes & String termination

placeholder  

# Idea

Instead of adding a ZWJ or a space character to mint a 'duplicate' name. I'll try a null-byte to mint a duplicate name as its a single byte vs a sequence.  

# Results

[real souls.eth metadata](https://metadata.ens.domains/mainnet/0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85/21408738346033573782297054604318283394034443274101412154976884646337787450239)  

```json
{
  "is_normalized":true,
  "name":"souls.eth",
  "description":"souls.eth, an ENS name.",
  "attributes":[
    {"trait_type":"Created Date","display_type":"date","value":1573148261000},
    {"trait_type":"Length","display_type":"number","value":5},
    {"trait_type":"Segment Length","display_type":"number","value":5},
    {"trait_type":"Character Set","display_type":"string","value":"letter"},
    {"trait_type":"Registration Date","display_type":"date","value":1646418606000},
    {"trait_type":"Expiration Date","display_type":"date","value":1677975558000}
  ],
  "name_length":5,
  "segment_length":5,
  "url":"https://app.ens.domains/name/souls.eth",
  "version":0,
  "background_image":"https://metadata.ens.domains/mainnet/avatar/souls.eth",
  "image":"https://metadata.ens.domains/mainnet/0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85/0x2f54ea9f8401e3f2ecb79b9bf6ca7581eb0100529a3283083b959dbb792bd37f/image",
  "image_url":"https://metadata.ens.domains/mainnet/0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85/0x2f54ea9f8401e3f2ecb79b9bf6ca7581eb0100529a3283083b959dbb792bd37f/image"
}
```

[fake souls.eth metadata](https://metadata.ens.domains/mainnet/0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85/3952704116753594465626453052297191666829096611916680535498635886104442511571)  

```json
{
  "is_normalized":true,
  "name":"souls.eth",
  "description":"souls.eth, an ENS name.",
  "attributes":[
    {"trait_type":"Created Date","display_type":"date","value":1671569999000},
    {"trait_type":"Length","display_type":"number","value":5},
    {"trait_type":"Segment Length","display_type":"number","value":5},
    {"trait_type":"Character Set","display_type":"string","value":"letter"},
    {"trait_type":"Registration Date","display_type":"date","value":1671569999000},
    {"trait_type":"Expiration Date","display_type":"date","value":1703126951000}
  ],
  "name_length":5,
  "segment_length":5,
  "url":"https://app.ens.domains/name/souls.eth",
  "version":0,
  "background_image":"https://metadata.ens.domains/mainnet/avatar/souls.eth",
  "image":"https://metadata.ens.domains/mainnet/0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85/0x08bd26b83793d0d667a993e5c2499c959245f0232c3a6c39d0dfb7d365cf20d3/image",
  "image_url":"https://metadata.ens.domains/mainnet/0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85/0x08bd26b83793d0d667a993e5c2499c959245f0232c3a6c39d0dfb7d365cf20d3/image"}
```  

[Fake vitalik.eth metadata](https://metadata.ens.domains/mainnet/0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85/24200900942460299581294029538738684782160326766579606917767243798520494602098)  

```json
{
  "is_normalized":true,
  "name":"vitalik.eth",
  "description":"vitalik.eth, an ENS name.",
  "attributes":[
    {"trait_type":"Created Date","display_type":"date","value":1673268731000},
    {"trait_type":"Length","display_type":"number","value":7},
    {"trait_type":"Segment Length","display_type":"number","value":7},
    {"trait_type":"Character Set","display_type":"string","value":"letter"},
    {"trait_type":"Registration Date","display_type":"date","value":1673268731000},
    {"trait_type":"Expiration Date","display_type":"date","value":1675687931000}
  ],
  "name_length":7,
  "segment_length":7,
  "url":"https://app.ens.domains/name/vitalik.eth",
  "version":0,
  "background_image":"https://metadata.ens.domains/mainnet/avatar/vitalik.eth",
  "image":"https://metadata.ens.domains/mainnet/0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85/0x3581397a478dcebdc1ee778deed625697f624c6f7dbed8bb7f780a6ac094b772/image",
  "image_url":"https://metadata.ens.domains/mainnet/0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85/0x3581397a478dcebdc1ee778deed625697f624c6f7dbed8bb7f780a6ac094b772/image"
}
```


The name is registered and the events are emitted. The name is added to the subgraph intance which results in a valid metadata object being returnable from the metadata service. Checking the metadata reveals two discoveries.  

The first is that the length of the name is misrepresented and does not account for the additional null-byte.  

The second is that there is no visible warning about "unicode" characters as a single null-byte is not considered unicode. It appears a null-byte bypasses the normalization check that would otherwise display a visible alert to users viewing the name on NFT marketplaces.  

Thus names minted with an appended null-byte create a hard-to-distinguish duplicate token / name as the length is wrong and no visible warning is displayed that users are used to when looking for duplicate/fake names.  

As an example of this bug I originally registered two different ```sload.eth``` names. Which can be viewed at the metadata links below.  

Screen shots provided as well displaying both names length record as displaying only 5 and not 6 as well as both names being displayable on Opensea, LooksRare and X2Y2. 

We can see two "identical" names are created in the metadata service using the null-byte technique causing the names lengths to be misrepresented. We say identical as the length is misrepresented as only the length of the name without the additional null-byte as the string appears to have been terminated before being processed by the subgraph ENS service.  

The string termination also bypasses the normalization check's so no visual alerts are added to the name similar to the visible alerts with unicode characters detected.  

# The patch(s)

Reviewing github commits for the ENS subgraph these two commits appeared relevent:  

[Fix label hashing check](https://github.com/ensdomains/ens-subgraph/commit/5f97205606a239a6db657d97394dfadf81510a20)  
[Fix hash check, add dot check](https://github.com/ensdomains/ens-subgraph/commit/a7ec9017e0498ab82e09f0de6157923d0aec9057)  

The patch also fixes another [vulnerability](https://medium.com/@hacxyk/how-we-spoofed-ens-domains-52acea2079f6) reported around the same time which allowed a user to create / mint spoofed ERC721 "subdomains" from names they didn't own.  

TLDR of the other vulnerability: by prepending your chosen subdomain and a "." character to create a new spoofed name name that looks like a subdomain.  

for example coke.eth - they could have registered drink.coke.eth while not owning coke.eth. At current time ENS subdomains are non transferable and this should be a huge red flag that the name is "fake" on markets.


```typescript
function setNamePreimage(name: string, label: Bytes, cost: BigInt): void {
  const labelHash = crypto.keccak256(ByteArray.fromUTF8(name));
  if(!labelHash.equals(label)) {
    log.warning(
      "Expected '{}' to hash to {}, but got {} instead. Skipping.",
      [name, labelHash.toHex(), label.toHex()]
    );
    return;
  }

  if(name.indexOf(".") !== -1) {
    log.warning("Invalid label '{}'. Skipping.", [name]);
    return;
  }

  let domain = Domain.load(crypto.keccak256(concat(rootNode, label)).toHex())!
  if(domain.labelName !== name) {
    domain.labelName = name
    domain.name = name + '.eth'
    domain.save()
  }

  let registration = Registration.load(label.toHex());
  if(registration == null) return
  registration.labelName = name
  registration.cost = cost
  registration.save()
}

export function handleNameRegisteredByControllerOld(event: ControllerNameRegisteredEventOld): void {
  setNamePreimage(event.params.name, event.params.label, event.params.cost);
}
```

# Revisited 12/2022
It seems an oversight of the subgraph instances configuration subgraph.yaml file and eventhandler definitions lead to this bug still existing and being "exploitable".  

It appears there are two handlers for NameRegistered events - One from the Registrar controller which input is passed to SetPreimage() correctly. How ever there is another definition for NameRegistered events from the baseRegistrar contract which registers & emits events first before other contracts. Allowing the malformed name to be inserted into the metadata service regardless of the patch.

The first definition for the NameRegistered event / handler calls a handler which is not calling SetPreimage() correctly.

```
  - kind: ethereum/contract
    name: BaseRegistrar
    network: mainnet
    source:
      address: "0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85"
      abi: BaseRegistrar
      startBlock: 9380410
    mapping:
      kind: ethereum/events
      apiVersion: 0.0.5
      language: wasm/assemblyscript
      file: ./src/ethRegistrar.ts
      entities:
        - Registration
        - NameRegistered
        - NameRenewed
        - NameTransferred
      abis:
        - name: BaseRegistrar
          file: ./abis/BaseRegistrar.json
      eventHandlers:
        - event: "NameRegistered(indexed uint256,indexed address,uint256)"
          handler: handleNameRegistered
```

The event handler:  

```javascript
export function handleNameRegistered(event: NameRegisteredEvent): void {
  let account = new Account(event.params.owner.toHex())
  account.save()

  let label = uint256ToByteArray(event.params.id)
  let registration = new Registration(label.toHex())
  let domain = Domain.load(crypto.keccak256(concat(rootNode, label)).toHex())!

  registration.domain = domain.id
  registration.registrationDate = event.block.timestamp
  registration.expiryDate = event.params.expires
  registration.registrant = account.id

  let labelName = ens.nameByHash(label.toHexString())
  if (labelName != null) {
    domain.labelName = labelName
    registration.labelName = labelName
  }
  domain.save()
  registration.save()

  let registrationEvent = new NameRegistered(createEventID(event))
  registrationEvent.registration = registration.id
  registrationEvent.blockNumber = event.block.number.toI32()
  registrationEvent.transactionID = event.transaction.hash
  registrationEvent.registrant = account.id
  registrationEvent.expiryDate = event.params.expires
  registrationEvent.save()
}
```

I registered a copy of an ENS name I already owned "souls.eth" to verify the bug was indeed still alive. 

The metadata links and screen shots are provided below:  


# Reward    
Awarded $45000 USDC originally  
Awarded $20000 USDC re-reporting  
Total: $65,000 USDC  
