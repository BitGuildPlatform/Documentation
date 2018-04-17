## Documentation
BitGuild portal and platform docs

## Portal integration

### 1. Placing your game on our portal

We need developers to provide us the url of the game to be used from within BGP. For this url we require developers to not reference any of their own login, signup, or landing pages. You can test it via the sandbox url: (**TBD**)

### 2. Required game smart contract changes

Our sole requirement is that everywhere partners accept ETH for in-game sales, they must also accept PLAT (BitGuild token). To determine the current PLAT/ETH market exchange rate you can use our recommended (https://github.com/BitGuildPlatform/SampleIntegration/blob/master/contracts/PLATPriceOracle.sol) price oracle smart contract (so you can adjust your prices) or you can deploy your own version.

For the details on how to accept tokens as payment methods, see the *approveAndCall* method on BitGuild token. The idea is that
* Instead of calling your smart contract directly, you call BitGuild’s token smart contract with approveAndCall, passing your smart contract address and additional call parameters (of your smart contract) to this method.
* approveAndCall on our token smart contract will call receiveApproval on your smart contract with this additional data so you can process the payment.
* Also make sure to have a withdrawToken method on your smart contract so you have a way to retrieve those tokens. 

See example integration here: https://github.com/BitGuildPlatform/SampleIntegration/blob/game/shared/contracts/TestGame.sol


#### Code example

before
```
pragma solidity ^0.4.2;

contract Ownable {
  address public owner;

  function constructor() public {
    owner = msg.sender;
  }

  modifier onlyOwner() {
    require(msg.sender == owner);
    _;
  }

  function transferOwnership(address newOwner) public onlyOwner {
    if (newOwner != address(0)) {
      owner = newOwner;
    }
  }
}

contract YourGameContract is Ownable {
  function kill() public onlyOwner {
    selfdestruct(msg.sender);
  }
  
  function buyItem(bytes _id) public payable {
    require(msg.value != 0);
    require(_id.length != 0);
     _buyItem(_id)
  }
  
  function () external payable {
    revert();
  }
}

```

after
```
pragma solidity ^0.4.2;

contract BitGuildToken { // implements ERC20Interface
    function totalSupply() public constant returns (uint);
    function balanceOf(address tokenOwner) public constant returns (uint balance);
    function allowance(address tokenOwner, address spender) public constant returns (uint remaining);
    function transfer(address to, uint tokens) public returns (bool success);
    function approve(address spender, uint tokens) public returns (bool success);
    function transferFrom(address from, address to, uint tokens) public returns (bool success);

    event Transfer(address indexed from, address indexed to, uint tokens);
    event Approval(address indexed tokenOwner, address indexed spender, uint tokens);
}

contract Ownable {
  address public owner;

  function constructor() public {
    owner = msg.sender;
  }

  modifier onlyOwner() {
    require(msg.sender == owner);
    _;
  }

  function transferOwnership(address newOwner) public onlyOwner {
    if (newOwner != address(0)) {
      owner = newOwner;
    }
  }
}

contract YourGameContract is Ownable {
    BitGuildToken public tokenContract;

    function constructor(address _tokenContract) public {
        tokenContract = BitGuildToken(_tokenContract);
    }

    function kill() public onlyOwner {
        tokenContract.transferFrom(this, msg.sender, tokenContract.balanceOf(this));
        selfdestruct(msg.sender);
    }

    function buyItem(bytes _id) public payable {
      require(msg.value != 0);
      require(_id.length != 0);
      _buyItem(_id)
    }

    function receiveApproval(address _sender, uint256 _value, BitGuildToken _tokenContract, bytes _extraData) public {
        require(_tokenContract == tokenContract);
        require(_tokenContract.transferFrom(_sender, address(this), _value));
        require(_extraData.length != 0);
        _buyItem(_extraData)
    }

    withdrawToken() public {

    }

    function _buyItem(bytes _id) private {
        // do something with item
    }

    function () external payable {
      revert();
    }
}
```

### 3. Required game UI changes

Partners will need to change game UI to support rendering prices in PLAT. For that partners will need to: 
* Detect if the game was launched from BitGuild portal by checking it via the Portal SDK (url/docs https://github.com/BitGuildPlatform/BitGuildPortalSDK/blob/master/README.md). 
* If the game was launched from the portal use the previously mentioned oracle contract to determine the PLAT prices of items. 
* Change item prices in game UI from “<XX> ETH” to “<XX * ratio> PLAT”
  
See example integration here: https://github.com/BitGuildPlatform/SampleIntegration/blob/game/client/components/main/main.jsx

**TBD** Code example

contracts
```js
const YourGameAddr = "0xB54AFFF5dB6c165548808A9ecfb234bA24bDCeEb";
const YourGameABI = [
  {
    "constant": false,
    "inputs": [],
    "name": "kill",
    "outputs": [],
    "payable": false,
    "stateMutability": "nonpayable",
    "type": "function"
  },
  {
    "constant": false,
    "inputs": [
      {
        "name": "_id",
        "type": "bytes"
      }
    ],
    "name": "buyItem",
    "outputs": [],
    "payable": true,
    "stateMutability": "payable",
    "type": "function"
  },
  {
    "constant": true,
    "inputs": [],
    "name": "owner",
    "outputs": [
      {
        "name": "",
        "type": "address"
      }
    ],
    "payable": false,
    "stateMutability": "view",
    "type": "function"
  },
  {
    "constant": false,
    "inputs": [],
    "name": "constructor",
    "outputs": [],
    "payable": false,
    "stateMutability": "nonpayable",
    "type": "function"
  },
  {
    "constant": false,
    "inputs": [
      {
        "name": "newOwner",
        "type": "address"
      }
    ],
    "name": "transferOwnership",
    "outputs": [],
    "payable": false,
    "stateMutability": "nonpayable",
    "type": "function"
  },
  {
    "payable": true,
    "stateMutability": "payable",
    "type": "fallback"
  }
];

const BitGuildTokenAddr = "0x7E43581b19ab509BCF9397a2eFd1ab10233f27dE";
const BitGuildTokenABI = [
  {
    "constant": true,
    "inputs": [],
    "name": "name",
    "outputs": [
      {
        "name": "",
        "type": "string"
      }
    ],
    "payable": false,
    "stateMutability": "view",
    "type": "function"
  },
  {
    "constant": false,
    "inputs": [
      {
        "name": "_spender",
        "type": "address"
      },
      {
        "name": "_value",
        "type": "uint256"
      }
    ],
    "name": "approve",
    "outputs": [
      {
        "name": "success",
        "type": "bool"
      }
    ],
    "payable": false,
    "stateMutability": "nonpayable",
    "type": "function"
  },
  {
    "constant": true,
    "inputs": [],
    "name": "totalSupply",
    "outputs": [
      {
        "name": "",
        "type": "uint256"
      }
    ],
    "payable": false,
    "stateMutability": "view",
    "type": "function"
  },
  {
    "constant": false,
    "inputs": [
      {
        "name": "_from",
        "type": "address"
      },
      {
        "name": "_to",
        "type": "address"
      },
      {
        "name": "_value",
        "type": "uint256"
      }
    ],
    "name": "transferFrom",
    "outputs": [
      {
        "name": "success",
        "type": "bool"
      }
    ],
    "payable": false,
    "stateMutability": "nonpayable",
    "type": "function"
  },
  {
    "constant": true,
    "inputs": [],
    "name": "decimals",
    "outputs": [
      {
        "name": "",
        "type": "uint8"
      }
    ],
    "payable": false,
    "stateMutability": "view",
    "type": "function"
  },
  {
    "constant": false,
    "inputs": [
      {
        "name": "_value",
        "type": "uint256"
      }
    ],
    "name": "burn",
    "outputs": [
      {
        "name": "success",
        "type": "bool"
      }
    ],
    "payable": false,
    "stateMutability": "nonpayable",
    "type": "function"
  },
  {
    "constant": true,
    "inputs": [
      {
        "name": "",
        "type": "address"
      }
    ],
    "name": "balanceOf",
    "outputs": [
      {
        "name": "",
        "type": "uint256"
      }
    ],
    "payable": false,
    "stateMutability": "view",
    "type": "function"
  },
  {
    "constant": false,
    "inputs": [
      {
        "name": "_from",
        "type": "address"
      },
      {
        "name": "_value",
        "type": "uint256"
      }
    ],
    "name": "burnFrom",
    "outputs": [
      {
        "name": "success",
        "type": "bool"
      }
    ],
    "payable": false,
    "stateMutability": "nonpayable",
    "type": "function"
  },
  {
    "constant": true,
    "inputs": [],
    "name": "symbol",
    "outputs": [
      {
        "name": "",
        "type": "string"
      }
    ],
    "payable": false,
    "stateMutability": "view",
    "type": "function"
  },
  {
    "constant": false,
    "inputs": [
      {
        "name": "_to",
        "type": "address"
      },
      {
        "name": "_value",
        "type": "uint256"
      }
    ],
    "name": "transfer",
    "outputs": [],
    "payable": false,
    "stateMutability": "nonpayable",
    "type": "function"
  },
  {
    "constant": false,
    "inputs": [
      {
        "name": "_spender",
        "type": "address"
      },
      {
        "name": "_value",
        "type": "uint256"
      },
      {
        "name": "_extraData",
        "type": "bytes"
      }
    ],
    "name": "approveAndCall",
    "outputs": [
      {
        "name": "success",
        "type": "bool"
      }
    ],
    "payable": false,
    "stateMutability": "nonpayable",
    "type": "function"
  },
  {
    "constant": true,
    "inputs": [
      {
        "name": "",
        "type": "address"
      },
      {
        "name": "",
        "type": "address"
      }
    ],
    "name": "allowance",
    "outputs": [
      {
        "name": "",
        "type": "uint256"
      }
    ],
    "payable": false,
    "stateMutability": "view",
    "type": "function"
  },
  {
    "inputs": [],
    "payable": false,
    "stateMutability": "nonpayable",
    "type": "constructor"
  },
  {
    "anonymous": false,
    "inputs": [
      {
        "indexed": true,
        "name": "from",
        "type": "address"
      },
      {
        "indexed": true,
        "name": "to",
        "type": "address"
      },
      {
        "indexed": false,
        "name": "value",
        "type": "uint256"
      }
    ],
    "name": "Transfer",
    "type": "event"
  },
  {
    "anonymous": false,
    "inputs": [
      {
        "indexed": true,
        "name": "from",
        "type": "address"
      },
      {
        "indexed": false,
        "name": "value",
        "type": "uint256"
      }
    ],
    "name": "Burn",
    "type": "event"
  }
];

const OracleAddr = "0x2339a01f8424d116ff7cf0869c9c37b769ed274f";
const OracleABI = [
  {
    constant: true,
    inputs: [],
    name: "PLATprice",
    outputs: [
      {
        name: "",
        type: "uint256"
      }
    ],
    payable: false,
    stateMutability: "view",
    type: "function"
  },
  {
    constant: false,
    inputs: [
      {
        name: "_newAdmin",
        type: "address"
      },
      {
        name: "_value",
        type: "bool"
      }
    ],
    name: "setAdmin",
    outputs: [],
    payable: false,
    stateMutability: "nonpayable",
    type: "function"
  },
  {
    constant: false,
    inputs: [
      {
        name: "_newPrice",
        type: "uint256"
      }
    ],
    name: "updatePrice",
    outputs: [],
    payable: false,
    stateMutability: "nonpayable",
    type: "function"
  },
  {
    inputs: [],
    payable: false,
    stateMutability: "nonpayable",
    type: "constructor"
  },
  {
    anonymous: false,
    inputs: [
      {
        indexed: false,
        name: "newPrice",
        type: "uint256"
      }
    ],
    name: "PLATPriceChanged",
    type: "event"
  }
];
```

before
```js
class MyButton extends Component {
  state = {
    amount: this.props.amount,
    currency: "ETH"
  }
  
  onClick() {
    const yourContract = window.web3.eth.contract(YourGameABI).at(YourGameAddr);
    return yourContract.buyItem(this.props.itemId, {
      from: this.props.wallet,
      value: this.state.amount * 1e18,
      gas: window.web3.toHex(15e4),
      gasPrice: window.web3.toHex(1e10)
    }, cb);
  }
  
  render() {
    return (<Button>Pay {this.state.amount} {this.state.currency}</Button>);
  }
}
```

after
```js
class MyButton extends Component {
  state = {
    amount: this.props.amount,
    currency: "ETH"
  }
    
  componentDidMount() {
    sdk.isOnPortal()
      .then(isOnPortal => {
        if (isOnPortal) {
          const oracleContract = window.web3.eth.contract(OracleABI).at(OracleAddr);
          oracleContract.PLATprice.call((err, PLATprice) => {
            const ratio = 1 / PLATprice.toNumber() * 1e18; // 1 ETH = 50.000
            this.setState({
              amount: this.props.amount * ratio,
              name: "PLAT"
            });
          });
        }
      });
  }
  
  onClick() {
    sdk.isOnPortal()
      .then(isOnPortal => {
        if (isOnPortal) {
          const bitGuildContract = window.web3.eth.contract(BitGuildTokenABI).at(BitGuildTokenAddr);
          bitGuildContract.approveAndCall(YourGameAddr, this.state.amount * 1e18, this.props.itemId, {
            from: this.props.wallet,
            gas: window.web3.toHex(15e4),
            gasPrice: window.web3.toHex(1e10)
          }, cb);
        } else {
          const yourContract = window.web3.eth.contract(YourGameABI).at(YourGameAddr);
          return yourContract.buyItem(this.props.itemId, {
            from: this.props.wallet,
            value: this.state.amount * 1e18,
            gas: window.web3.toHex(15e4),
            gasPrice: window.web3.toHex(1e10)
          }, cb);
        }
      });
  }
  
  render() {
    return (<Button>Pay {this.state.amount} {this.state.currency}</Button>);
  }
}
```

For the moment we will allow partners to use their own marketplace solutions; in the future (time TBD) BitGuild will include standardized marketplace logic in our SDK, and will likely require partners to implement it.
  
### 4. Compliance with wider token standards

We want to make sure all the game tokens are properly standardized and can be used from within our services. Make sure you use the latest version of the ERC721 standard and support:
* tokenOfOwnerByIndex (see “Enumeration Extension” by the link above)
* safeTransferFrom, that also triggers the receiver protocol (ERC721TokenReceiver)
* ERC721MetaData with name and a tokenUri for json that has image url in it

For all the ERC20 tokens you use in your game (for resources and fungible items) make sure you support the transferAndCall function for the extended ERC20 standard, so tokens can be transferred to a smart contract in 1 transaction, not 2. We generally recommend using ERC20+ERC677 instead of ERC223.
