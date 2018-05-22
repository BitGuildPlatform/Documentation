# BitGuild Documentation

I. [Portal integration](#i-portal-integration)
  1. [Placing your game on the portal](#1-placing-your-game-on-the-portal)
  2. [Required game smart contract changes](#2-required-game-smart-contract-changes)
  3. [Required game UI changes](#3-required-game-ui-changes)
  4. [Testing](#4-testing)
  
II. [Compliance with token standards](#ii-compliance-with-token-standards)
  1. [Standards](#1-requirements)
  2. [Fallbacks](#2-fallbacks)
  
III. [BitGuild Portal SDK](#iii-bitguild-portal-sdk)

If you still have any questions after reading this doc, join our developer round table group: https://discord.gg/EJNgmD5

## I. Portal integration

### 1. Placing your game on the portal

We need developers to provide us the url of the game to be used from within the BitGuild Portal (BGP). On the page at this url we require developers to not include any of their own login, signup, or landing pages - assume the user is already connected via web3 bridge like MetaMask. Ideally this is not your default game page but rather portal-specific - since SDK load adds latency on page refresh and you probably don't want that when you're not running on the portal. You can test the placement via the sandbox url: https://bitguild.com/sandbox?url=GAME_PAGE_URL

### 2. Required game smart contract changes

Our key requirement is that wherever partners accept ETH for in-game purchases, they must also accept PLAT (BitGuild token). To determine the current PLAT/ETH market exchange rate (so you can convert your ETH prices to PLAT) you can use our [recommended price oracle smart contract](https://etherscan.io/address/0x3127be52acba38beab6b4b3a406dc04e557c037c) or you can [deploy your own version](https://github.com/BitGuildPlatform/Contracts/blob/master/contracts/PLATPriceOracle.sol).

For the details on how to accept tokens as payment methods, see *approveAndCall* method on BitGuild token. The idea is that:
* Instead of calling your smart contract directly with eth as a payment, you call BitGuild’s token smart contract with approveAndCall, passing your smart contract address, price and additional data parameter (to be passed to your smart contract) to this method.
* approveAndCall on our token smart contract will call receiveApproval on your smart contract so you can process the payment.
* Make sure to have a withdrawToken method on your smart contract so you have a way to retrieve those tokens. 

**Smart contract code example:**
```
contract YourGameContract {
  BitGuildToken public tokenContract = 0x7E43581b19ab509BCF9397a2eFd1ab10233f27dE; // Predefined PLAT token address

  // Function to accept ethereum as a payment (not needed for integration, shown as an example)
  function buyItem(bytes _id) public payable {
    require(msg.value != 0);
    require(_id.length != 0);
    _buyItem(_id)
  }

  // Function that is called when trying to use PLAT for payments from approveAndCall
  function receiveApproval(address _sender, uint256 _value, BitGuildToken _tokenContract, bytes _extraData) public {
    require(_tokenContract == tokenContract);
    require(_tokenContract.transferFrom(_sender, address(this), _value));
    require(_extraData.length != 0);
    _buyItem(_extraData)
  }

  function _buyItem(bytes _id) private {
     // Actual item purcase logic
  }
}
```

### 3. Required game UI changes

Partners will need to change game UI to support rendering prices in PLAT. For that partners will need to: 
* Use the last version of [BitGuild Portal SDK](https://github.com/BitGuildPlatform/BitGuildPortalSDK/): https://bitguild.com/sdk/BitGuildPortalSDK_v0.1.js
* Detect if the game was launched from the BitGuild portal by checking isOnPortal method.
* If the game was launched from the portal, use the previously mentioned oracle contract to determine the PLAT prices of items. 
* Change item prices in game UI from “<XX> ETH” to “<XX * ratio> PLAT”

**Frontend code example:**
```js
  // On page load:
  componentDidMount() {
    sdk.isOnPortal()
      .then(isOnPortal => {
        if (isOnPortal) {
          const oracleContract = window.web3.eth.contract(OracleABI).at(OracleAddr);
          oracleContract.PLATprice.call((err, PLATprice) => {
            const ratio = 1 / PLATprice.toNumber() * 1e18;
            this.setState({
              amount: this.props.amount * ratio,
              name: "PLAT"
            });
          });
        } else {
          this.setState({
            amount: this.props.amount,
            name: "ETH"
          });
        }
      });
  }
  
  // On button click, pay in eth or in PLAT dependinfg on where it's called from
  onClick() {
    sdk.isOnPortal()
      .then(isOnPortal => {
        if (isOnPortal) {
          const bitGuildContract = window.web3.eth.contract(BitGuildTokenABI).at(BitGuildTokenAddr);
          bitGuildContract.approveAndCall(YourGameAddr, this.state.amount * 1e18, this.props.itemId, {
            from: this.props.wallet
          });
        } else {
          const yourContract = window.web3.eth.contract(YourGameABI).at(YourGameAddr);
          return yourContract.buyItem(this.props.itemId, {
            from: this.props.wallet,
            value: this.state.amount * 1e18
          });
        }
      });
  }
```

For the moment we will allow partners to use their own marketplace solutions; in the future (timeline TBD) BitGuild will include standardized marketplace logic in our SDK, and will likely require partners to implement it.

See full integration example here: https://github.com/BitGuildPlatform/SampleIntegration

### 4. Testing

We have PLAT and price oracle contracts on both Mainnet and Rinkeby, here're the addresses:
* Mainnet
  * PLAT: [0x7E43581b19ab509BCF9397a2eFd1ab10233f27dE](https://etherscan.io/address/0x7E43581b19ab509BCF9397a2eFd1ab10233f27dE)
  * Price Oracle: [0x3127be52acba38beab6b4b3a406dc04e557c037c](https://etherscan.io/address/0x3127be52acba38beab6b4b3a406dc04e557c037c)
* Rinkeby
  * PLAT: [0x0f2698b7605fe937933538387b3d6fec9211477d](https://rinkeby.etherscan.io/address/0x0f2698b7605fe937933538387b3d6fec9211477d)
  * Price Oracle: [0x20159d575724b68d8a1a80e16fcb874883329114](https://rinkeby.etherscan.io/address/0x20159d575724b68d8a1a80e16fcb874883329114)
  
## II. Compliance with token standards

### 1. Requirements

We want to make sure all the game tokens are properly standardized and can be used from within our services. Make sure you use the latest version of the ERC721 standard and support:
* tokenOfOwnerByIndex (see “Enumeration Extension” [here](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-721.md))
* safeTransferFrom, that also triggers the receiver protocol (ERC721TokenReceiver)
* ERC721MetaData with name and a tokenUri for json that has image url in it (also [here](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-721.md))
* You also may want to support [attributes](https://medium.com/blockchain-manchester/evolving-erc-721-metadata-standards-44646c2eb332) in your metadata, to be used as categories for items on the portal.

For all the ERC20 tokens you use in your game (for resources and fungible items) make sure you support the transferAndCall function from the [extended ERC20 standard](https://github.com/ethereum/EIPs/issues/677), so tokens can be transferred to a smart contract in 1 transaction, not 2. We generally recommend using ERC20+ERC677 instead of ERC223.

### 2. Fallbacks

In the case you didn't implement enumeration or metadata extensions to ERC721 in your smart contract, there's still a way to provide our wallets necessary data - in a traditional centralized fashion. For that we need two APIs with the following schemas:

* `https://[YOURDOMAIN/PATH]/itemList/[address]` where addess is an Ethereum public address for a particular user. This should return an json with an array of token indices that belong to this user. Example: `{itemList: [142, 31, 3181]}`.
* `https://[YOURDOMAIN/PATH]/itemInfo/[index]` where index is the token index. This endpoint should return json data in the same format as ERC721MetaData described above would.

For localized names and descriptions add `translations` blob in the json, that would look the following way:

```
  name: "XXXX",
  description: "XXXX",
  image:"https://gameurl/XXXX.png"
  translations: {
    “fr”: {
      “name”:”xxx”,   
      “description”:”xxx”
    }
  }
```
Where the values from translations blob will be taken for specified languages and default for all the others.

## III. BitGuild Portal SDK

SDK methods:

* **init(): Promise\<Void>**, initializes SDK
* **isOnPortal(): Promise\<Boolean>**, determinates whether SDK was properly initialized by portal
* **getUser(): Promise\<User>**, returns portal user info if ran from the portal
* **getUsersByAddress(addresses: string[]): Promise\<Array\<User>>**, resolves addresses to portal users,
it is possible not all addresses are resolved, so you have to manually match input array to results

```typescript
interface User {
  language: string; // String 2-char language code, available languages are en|zh|pt|ja|fr|es|ru
  wallet: string; // ERC20 compatible wallet address 0x87efa7f59bAA8e475F181B36f77A3028494a2cf6
  nickName: string; // user defined string that matches /^[ a-z0-9_-]+$/i
}
```

Initialization:
```js
const sdk = require("BitGuildPortalSDK_v0.1.js");
sdk.init();
```

Getting current user
```js
sdk.isOnPortal()
  .then(isOnPortal => {
    if (isOnPortal) {
      return sdk.getUser()
        .then(user => {
          // do something with user
        })
    }
  })
```

Resolving Addresses
```js
sdk.getUsersByAddress(["0x87efa7f59bAA8e475F181B36f77A3028494a2cf6"])
  .then(users => {
    // do something with users
  })
```
