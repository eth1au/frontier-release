
## The Coin 

What is a coin? Coins are much more interesting and useful than they seem, they are in essence just a tradeable token, but can become much more, depending on how you use them. Its value depends on what you do with it: a token can be used to control access (**an entrance ticket**), can be used to represent debt owed by an organization (**a bond**), can be placeholders for an asset held by a third party (**a certificate of ownership**) or even simply be used as an exchange of value within a community (**a currency**).

You could do all those things by creating a centralized server, but using an Ethereum token contract comes with some free functionalities: for one, it's a decentralized service and tokens can be still exchanged even if the original service goes down for any reason. The code can guarantee that no tokens will ever be created other than the ones set in the original code. Finally, by having each user hold their own token, this eliminates the scenarios where one single server break-in can result in the loss of funds from thousands of clients. This makes the service ethereum offers effectively censorship proof.

You could create your own token on a different blockchain, but creating on ethereum is easier and, as you'll see in the next few tutorials, more flexible. This means you can focus your energy on the innovation that will make your coin stand out - and it's more secure, as your security is provided by all the miners who are supporting the ethereum network. Finally, by creating your token in Ethereum, your coin will be compatible with any other contract running on ethereum.

### The Code

This is the code for the contract we're building:
 
    contract token { 
        mapping (address => uint) public coinBalanceOf;
        event CoinTransfer(address sender, address receiver, uint amount);
      
      /* Initializes contract with initial supply tokens to the creator of the contract */
      function token(uint supply) {
            if (supply == 0) supply = 10000;
            coinBalanceOf[msg.sender] = supply;
        }
      
      /* Very simple trade function */
        function sendCoin(address receiver, uint amount) returns(bool sufficient) {
            if (coinBalanceOf[msg.sender] < amount) return false;
            coinBalanceOf[msg.sender] -= amount;
            coinBalanceOf[receiver] += amount;
            CoinTransfer(msg.sender, receiver, amount);
            return true;
        }
    }

If you have ever programmed, you won't find it hard to understand what it does: it is a contract that generates 10 thousand tokens to the creator of the contract, and then allows anyone with enough balance to send it to others. These tokens are the minimum tradeable unit and cannot be subdivided, but for the final users could be presented as a 100 units subdividable by 100 subunits, so owning a single token would represent having 0.01% of the total. If your application needs more fine grained atomic divisibility, then just increase the initial issuance amount.

In this example we declared the mapping "coinBalanceOf" to be public, this will automatically create a 'get' function that returns any account's balance.

### Compile and Deploy

**So let's run it!**

    var tokenSource = 'contract token { mapping (address => uint) public coinBalanceOf; event CoinTransfer(address sender, address receiver, uint amount); function token(uint supply) { if (supply == 0) supply = 10000; coinBalanceOf[msg.sender] = supply; } function sendCoin(address receiver, uint amount) returns(bool sufficient) { if (coinBalanceOf[msg.sender] < amount) return false; coinBalanceOf[msg.sender] -= amount; coinBalanceOf[receiver] += amount; CoinTransfer(msg.sender, receiver, amount); return true; } }'

    var tokenCompiled = eth.compile.solidity(tokenSource)

Now let’s set up the contract, just like we did in the previous section. Change the "initial Supply" to the amount of non divisible tokens you want to create. If you want to have divisible units, you should do that on the user frontend but keep them represented in the minimum unit of account. 

    var supply = 10000;
    var tokenContract = web3.eth.contract(tokenCompiled.token.info.abiDefinition);
    var token = tokenContract.new(
      supply,
      {
        from:web3.eth.accounts[0], 
        data:tokenCompiled.token.code, 
        gas: 1000000
      }, function(e, contract){
        if(!e) {

          if(!contract.address) {
            console.log("Contract transaction send: TransactionHash: " + contract.transactionHash + " waiting to be mined...");

          } else {
            console.log("Contract mined! Address: " + contract.address);
            console.log(contract);
          }

        }
    })

#### Online Compiler

**If you don't have solC installed, you can simply use the online compiler.** Copy the contract code to the [online solidity compiler](https://chriseth.github.io/browser-solidity/), if there are no errors on the contract you should see a text box labeled **Geth Deploy**. Copy the content to a text file so you can change the first line to set the initial supply, like this:  

    var supply = 10000;

Now you can paste the resulting text in your geth window. Wait up to thirty seconds and you'll see a message like this:

    Contract mined! address: 0xdaa24d02bad7e9d6a80106db164bad9399a0423e 

This has deployed the contract, and it is ready to be interacted with.

### Check balance watching coin transfers

If everything worked correctly, you should be able to check your own balance with:

    token.coinBalanceOf(eth.accounts[0]) + " tokens"

It should have all the 10 000 tokens that were created once the contract was published. Since there is not any other defined way for new coins to be issued, these are all that will ever exist. 

You can set up a **Watcher** to react whenever anyone sends a coin using your contract. Here's how you do it:

    var event = token.CoinTransfer({}, '', function(error, result){
      if (!error)
        console.log("Coin transfer: " + result.args.amount + " tokens were sent. Balances now are as following: \n Sender:\t" + result.args.sender + " \t" + token.coinBalanceOf.call(result.args.sender) + " tokens \n Receiver:\t" + result.args.receiver + " \t" + token.coinBalanceOf.call(result.args.receiver) + " tokens" )
    });

### Sending coins

Now of course those tokens aren't very useful if you hoard them all, so in order to send them to someone else, use this command:

    token.sendCoin.sendTransaction(eth.accounts[1], 1000, {from: eth.accounts[0]})

If a friend has registered a name on the registrar you can send it without knowing their address, doing this:

    token.sendCoin.sendTransaction(registrar.addr("Alice"), 2000, {from: eth.accounts[0]})


Note that our first function **coinBalanceOf** was simply called directly on the contract instance and returned a value. This was possible since this was a simple read operation that incurs no state change and which executes locally and synchronously. Our second function **sendCoin** needs a **.sendTransaction()** call. Since this function is meant to change the state (write operation), it is sent as a transaction to the network to be picked up by miners and included in the canonical blockchain. As a result the consensus state of all participant nodes will adequately reflect the state changes resulting from executing the transaction. Sender address needs to be sent as part of the transaction to fund the fuel needed to run the transaction. Now, wait until the transaction is included in a block and check both accounts balances:

    token.coinBalanceOf.call(eth.accounts[0])/100 + "% of all tokens"
    token.coinBalanceOf.call(eth.accounts[1])/100 + "% of all tokens"
    token.coinBalanceOf.call(registrar.addr("Alice"))/100 + "% of all tokens"


### Improvement suggestions

Right now this "cryptocurrency" is quite limited as there will only ever be 10,000 coins and all are controlled by the coin creator, but you can change that. You could for example reward ethereum miners, by creating a transaction that will reward who found the current block:

    mapping (uint => address) miningReward;
    function claimMiningReward() {
      if (miningReward[block.number] == 0) {
        coinBalanceOf[block.coinbase] += 1;
        miningReward[block.number] = block.coinbase;
      }
    }

You could modify this to anything else: maybe reward someone who finds a solution for a new puzzle, wins a game of chess, install a solar panel—as long as that can be somehow translated to a contract. Or maybe you want to create a central bank for your personal country, so you can keep track of hours worked, favours owed or control of property. In that case you might want to add a function to allow the bank to remotely freeze funds and destroy tokens if needed. 


### Register a name for your coin

**The name registrar contract has changed in Frontier and the following section is only valid for the Olympic test net. Please wait while we update it.**

The commands mentioned only work because you have token javascript object instantiated on your local machine. If you send tokens to someone they won't be able to move them forward because they don't have the same object and wont know where to look for your contract or call its functions. In fact if you restart your console these objects will be deleted and the contracts you've been working on will be lost forever. So how do you instantiate the contract on a clean machine? 

There are two ways. Let's start with the quick and dirty, providing your friends with a reference to your contract’s ABI:

    token = eth.contract([{constant:false,inputs:[{name:'receiver',type:'address'},{name:'amount',type:'uint256'}],name:'sendCoin',outputs:[{name:'sufficient',type:'bool'}],type:'function'},{constant:true,inputs:[{name:'',type:'address'}],name:'coinBalanceOf',outputs:[{name:'',type:'uint256'}],type:'function'},{inputs:[{name:'supply',type:'uint256'}],type:'constructor'},{anonymous:false,inputs:[{indexed:false,name:'sender',type:'address'},{indexed:false,name:'receiver',type:'address'},{indexed:false,name:'amount',type:'uint256'}],name:'CoinTransfer',type:'event'}]).at('0x4a4ce7844735c4b6fc66392b200ab6fe007cfca8')

Just replace the address at the end for your own token address, then anyone that uses this snippet will immediately be able to use your contract. Of course this will work only for this specific contract so let's analyze step by step and see how to improve this code so you'll be able to use it anywhere.

All accounts are referenced in the network by their public address. But addresses are long, difficult to write down, hard to memorize and immutable. The last one is specially important if you want to be able to generate fresh accounts in your name, or upgrade the code of your contract. In order to solve this, there is a default name registrar contract which is used to associate the long addresses with short, human-friendly names.

Names have to use only alphanumeric characters and, cannot contain blank spaces. In future releases the name registrar will likely implement a bidding process to prevent name squatting but for now, it works on a first come first served basis: as long as no one else registered the name, you can claim it.


First, if you register a name, then you won't need the hardcoded address in the end. Select a nice coin name and try to reserve it for yourself. First, select your name:

    var tokenName = "MyFirstCoin"

Then, check the availability of your name:

    registrar.addr(tokenName)

If that function returns "0x00..", you can claim it to yourself:

    registrar.reserve.sendTransaction(tokenName, {from: eth.accounts[0]});


Wait for the previous transaction to be picked up. Wait up to thirty seconds and then try:

    registrar.owner(tokenName)

If it returns your address, it means you own that name and are able to set your chosen name to any address you want:

    registrar.setAddress.sendTransaction(tokenName, token.address, true,{from: eth.accounts[0]});

_You can replace **token.address** for **eth.accounts[0]** if you want to use it as a personal nickname._

Wait a little bit for that transaction to be picked up too and test it:

    registrar.addr("MyFirstCoin")

You can send a transaction to anyone or any contract by name instead of account simply by typing 

    eth.sendTransaction({from: eth.accounts[0], to: registrar.addr("MyFirstCoin"), value: web3.toWei(1, "ether")})

**Tip: don't mix registrar.addr for registrar.owner. The first is to which address that name is pointed at: anyone can point a name to anywhere else, just like anyone can forward a link to google.com, but only the owner of the name can change and update the link. You can set both to be the same address.**

This should now return your token address, meaning that now the previous code to instantiate could use a name instead of an address.

    token = eth.contract([{constant:false,inputs:[{name:'receiver',type:'address'},{name:'amount',type:'uint256'}],name:'sendCoin',outputs:[{name:'sufficient',type:'bool'}],type:'function'},{constant:true,inputs:[{name:'',type:'address'}],name:'coinBalanceOf',outputs:[{name:'',type:'uint256'}],type:'function'},{inputs:[{name:'supply',type:'uint256'}],type:'constructor'},{anonymous:false,inputs:[{indexed:false,name:'sender',type:'address'},{indexed:false,name:'receiver',type:'address'},{indexed:false,name:'amount',type:'uint256'}],name:'CoinTransfer',type:'event'}]).at(registrar.addr("MyFirstCoin"))

This also means that the owner of the coin can update the coin by pointing the registrar to the new contract. This would, of course, require the coin holders trust the owner set at  registrar.owner("MyFirstCoin")

Of course this is a rather unpleasant big chunk of code just to allow others to interact with a contract. There are some avenues being investigated to upload the contract ABI to the network, so that all the user will need is the contract name. You can [read about these approaches](https://github.com/ethereum/go-ethereum/wiki/Contracts-and-Transactions#natspec) but they are very experimental and will certainly change in the future.


### Learn More 

* [Meta coin standard](https://github.com/ethereum/wiki/wiki/Standardized_Contract_APIs) is a proposed standardization of function names for coin and token contracts, to allow them to be automatically added to other ethereum contract that utilizes trading, like exchanges or escrow.

* [Formal proofing](https://github.com/ethereum/wiki/wiki/Ethereum-Natural-Specification-Format#documentation-output) is a way where the contract developer will be able to assert some invariant qualities of the contract, like the total cap of the coin. *Not yet implemented*.

