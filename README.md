## Documentation
BitGuild portal and platform docs

## Portal integration

### 1. Placing your game on our portal

We need developers to provide us the url of the game to be used from within BGP. For this url we require developers to not reference any of their own login, signup, or landing pages. You can test it via the sandbox url: (**TBD**)

### 2. Required game smart contract changes

Our sole requirement is that everywhere partners accept ETH for in-game sales, they must also accept PLAT (BitGuild token). To determine the current PLAT/ETH market exchange rate you can use our recommended (**TBD**) price oracle smart contract (so you can adjust your prices) or you can deploy your own version.

For the details on how to accept tokens as payment methods, see the *approveAndCall* method on BitGuild token. The idea is that
* Instead of calling your smart contract directly, you call BitGuild’s token smart contract with approveAndCall, passing your smart contract address and additional call parameters (of your smart contract) to this method.
* approveAndCall on our token smart contract will call receiveApproval on your smart contract with this additional data so you can process the payment.
* Also make sure to have a withdrawToken method on your smart contract so you have a way to retrieve those tokens. 

See example integration here: (**TBD** link to game smart contract code)

**TBD** Code example

### 3. Required game UI changes

Partners will need to change game UI to support rendering prices in PLAT. For that partners will need to: 
* Detect if the game was launched from BitGuild portal by checking it via the Portal SDK (url/docs **TBD**). 
* If the game was launched from the portal use the previously mentioned oracle contract to determine the PLAT prices of items. 
* Change item prices in game UI from “<XX> ETH” to “<XX * rate> PLAT”
  
See example integration here: (**TBD** link to game UI code)

**TBD** Code example

For the moment we will allow partners to use their own marketplace solutions; in the future (time TBD) BitGuild will include standardized marketplace logic in our SDK, and will likely require partners to implement it.
  
### 4. Compliance with wider token standards

We want to make sure all the game tokens are properly standardized and can be used from within our services. Make sure you use the latest version of the ERC721 standard and support:
* tokenOfOwnerByIndex (see “Enumeration Extension” by the link above)
* safeTransferFrom, that also triggers the receiver protocol (ERC721TokenReceiver)
* ERC721MetaData with name and a tokenUri for json that has image url in it

For all the ERC20 tokens you use in your game (for resources and fungible items) make sure you support the transferAndCall function for the extended ERC20 standard, so tokens can be transferred to a smart contract in 1 transaction, not 2. We generally recommend using ERC20+ERC677 instead of ERC223.
