# Using-Yul-To-Optimize-Gas-Costs

In this article we will use Yul to optimize our smart contracts to save on gas. We will look at three different smart contracts developed to play the classic game Rock, Paper, Scissors. One contract will be written purely in Solidity, another contract will be written in Solidity with inline assembly, and the last contract will be written purely in Yul. If you are unfamiliar with Yul, please check out the article I wrote on Yul to get a better understanding of what it is and how it works.

Beginner’s Guide To Yul: https://medium.com/coinsbench/beginners-guide-to-yul-12a0a18095ef <br>
Github repository with Rock, Paper Scissors: https://github.com/marjon-call/Rock-Paper-Scissors

Ok Let’s dive into the game’s structure!


## Game Structure
To play the game a user needs to call ```createGame(address _player1, address _player2) external```. The two parameter addresses represent the two players playing the game. Once the game is created, the players can make their moves by calling ```playGame(uint8 _move) external payable```. This function takes in a ```uint8``` that must be in between 1 - 3. 1 represents rock, 2 represents paper, and 3 represents scissors. The cost to play is 0.05 ether and each player is only allowed to make one move. Once both players have made their move, ```evaluateGame() private``` is called to check the outcome of the game and distributes the ether to the winner, or splits the ether if the game is a tie. Only one game is allowed to be played at a time. Storage variable ```bool public gameInProgress``` is used to keep track of this. We also have storage variable ```uint256 public gameStart``` to keep track of when the game was started, and storage variable ```uint256 public gameLength``` to keep track of how long we want our game length to be. If the game is taking too long any spectator can call ```terminateGame() external``` so they can play a game. In this case, the ether is sent to a player who has made their move already. 

<br>

## Solidity

First let's take a look at our Solidity smart contract and outline the functionality. Here are our storage variables.

```
// slot 0
uint256 public gameStart;
// slot 1
uint256 public gameCost;
// slot 2
address public player1;
uint8 private player1Move;
// slot 3
address public player2;
uint8 private player2Move;
bool public gameInProgress;
bool public lockGame;
// slot 4
uint256 public gameLength;

event GameOver(address Winner, address Loser);
event GameStart(address Player1, address Player2);
event TieGame(address Player1, address Player2);
event GameTerminated(address Loser1, address Loser2)
```


|   Variable  |  Use  |
|  :---   | :--- |
|   gameStart   |  Used to keep track of when the game was started, using ```block.timestamp```   |
|   gameCost   |    Used to keep track of the cost to play the game in wei |
|   player1    |  The address of player 1 |
|   player1Move    |  Used to keep track of player 1’s move |
|   player2    |  The address of player 2 |
|   player2Move    |  Used to keep track of player 2’s move |
|   gameInProgress    |  Used to keep track of whether or not a game is currently being played |
|   lockGame    |   Locks the game while a player is making a move (to prevent re-entrancey) |
|   gameLength  |  Keeps track of when the earliest time a spectator can call ```terminateGame() external``` |


Take note that we are packing our variables to save on gas costs. We will dive deeper into this later on, but for now just be aware of it.

Great, now let's look at the constructor.

```
constructor() {
        gameCost = 0.05 ether;
        gameLength = 28800;
    }

```
All that we are doing here is setting ```gameCost``` to 0.05 ether, and setting ```gameLength``` to 28800 blocks.


Now let’s look at ```createGame()```.
```
// allows users to create a new game
function createGame(address _player1, address _player2) external {
    require(gameInProgress == false, "Game still in progress.");
   
    gameInProgress = true;
    gameStart = block.number;
    player1 = _player1;
    player2 = _player2;


}
```
First thing we do is check if a game is in progress. If it is, we revert. Otherwise, we set our storage variables to set up our game.


Let’s take a look at our storage to get a better understanding of what is happening under the hood. If you would like to follow along here is the function I am using to get the storage layout.
```
function getStorage() public view returns(bytes32, bytes32, bytes32, bytes32) {


    assembly {
        mstore(0x00, sload(0))
        mstore(0x20, sload(1))
        mstore(0x40, sload(2))
        mstore(0x60, sload(3))


        return(0x00, 0x80)
    }


}
```
<br>

|   Storage Slot  |  Value  |
|  :---:   | :--- |
|   0  |  ``` 0x0000000000000000000000000000000000000000000000000000000000000005``` (I am using remix so game start is set to a low value for ```block.timestamp```)  |
|   1  |   ```0x00000000000000000000000000000000000000000000000000b1a2bc2ec50000``` (0.05 ether) |
|   2   |  ```0x0000000000000000000000005b38da6a701c568545dcfcb03fcb875f56beddc4``` (```player1``` address)|
|   3   |  0x0000000000000070800001005b38da6a701c568545dcfcb03fcb875f56beddc4 (```player2``` address, ```gameInProgress```, and ```gameLength```|

Let’s talk about storage slot 3 before moving on. Packed from right to left, we see the first 20 bytes is the address of Player 2 (```0x5b38da6a701c568545dcfcb03fcb875f56beddc4```). Next we see 1 empty byte (```0x00```), this is where we will store Player 2’s move. After that we see ```0x01``` for ```gameInProgress```, which is equivalent to true. After that another empty byte (```0x00```), because ```lockGame``` is set to false for the time being. Then we see ```0x7080```, which is equivalent to 28800 in decimal (the value we set ```gameLength``` to in the constructor). 

Let’s also take a quick look at memory.

|   Memory Location |  Value  |
|  :---:   | :--- |
|   0x00  |   Empty  |
|   0x20  |   Empty  |
|   0x40  |  ```0x80``` (Free memory Pointer) |

We do not perform any operations in memory so we expect it to look like this.


Great, Now we will look at how ```playGame()``` works!
```
function playGame(uint8 _move) external payable {
    require(lockGame == false);
    require(gameInProgress == true, "Game not in progress.");
    require(msg.value >= gameCost, "Player has not sent enough ether to play.");
    require(_move > 0 && _move < 4, "Invalid move");


    lockGame = true;


    if (msg.sender == player1 && player1Move == 0) {
        player1Move = _move;
    } else if(msg.sender == player2 && player2Move == 0) {
        player2Move = _move;
    } else {
        require(1 == 0, "User not authorized to make move.");
    }
   
    if (player1Move != 0 && player2Move != 0) {
        evaluateGame();
    }
    lockGame = false;
}
```

Ok, so the first thing we are going to do is use ```require()``` statements to verify that we are playing by the rules. The first ```require()``` checks if the game is locked. The second ```require()``` checks that a game has been created. The third ```require()``` is checking if the player has sent enough ether to play the game. The last ```require()``` checks if the player has sent a valid move. After that we lock the game. Then, we have an ```if``` statement checking if Player 1 is making their move. We also check if they have made a move already by checking if the value of ```player1Move``` is set to 0 (remember that Solidity initializes values to 0). Notice that we check the player first. We deem that it is more likely that condition will not be satisfied, by doing so we save gas by not needing to check if Player 1 has made their move yet (less operations, less gas costs). If the conditions are satisfied we set Player 1’s move. Then we do the same checks for Player 2. If neither of the ```if``` statements are satisfied, we revert. Otherwise, we check if both moves have been made. If they have, we call ```evaluateGame()```. Finally, we unlock the game. 

Let’s look at what the storage looks like if Player 1 moves first with rock.

|   Storage Slot  |  Value  |
|  :---   | :--- |
|   0  |  ``` 0x0000000000000000000000000000000000000000000000000000000000000005``` (I am using remix so game start is set to a low value for ```block.timestamp```)  |
|   1  |   ```0x00000000000000000000000000000000000000000000000000b1a2bc2ec50000``` (0.05 ether) |
|   2   |  ```0x0000000000000000000000015b38da6a701c568545dcfcb03fcb875f56beddc4``` (```player1``` address, ```player1Move```)|
|   3   |  0x0000000000000070800001005b38da6a701c568545dcfcb03fcb875f56beddc4 (```player2``` address, ```gameInProgress```, and ```gameLength```|

Notice the only change is in slot 2. Reading right to left, after the Player 1’s address we see ```0x01``` (1 in decimal), which is equivalent to rock. It is also important to note that this is at the end of the transaction so ```lockGame``` is already set back to 0.

Again, memory is not used so it looks the same as when we called ```createGame()```.

Now let's look at what happens in storage right before ```evaluateGame()``` is called when Player 2 makes their move with scissors. 

|   Storage Slot  |  Value  |
|  :---   | :--- |
|   0  |  ``` 0x0000000000000000000000000000000000000000000000000000000000000005``` (I am using remix so game start is set to a low value for ```block.timestamp```)  |
|   1  |   ```0x00000000000000000000000000000000000000000000000000b1a2bc2ec50000``` (0.05 ether) |
|   2   |  ```0x0000000000000000000000015b38da6a701c568545dcfcb03fcb875f56beddc4``` (```player1``` address)|
|   3   |  0x000000000000007080010103ab8483f64d9c6d1ecf9b849ae677dd3315835cb2 (```player2``` address, ```player2Move```, ```gameInProgress```, ```lockGame```, and ```gameLength```|

The only difference here is in slot 3. We see that after ```player2```, ```player2Address``` is set to ```0x03``` (decimal 3), which is equal to scissors for our game. Also notice that in between ```gameInProgress``` and ```gameLength```, ```lockGame``` has been set to ```0x01``` (true as a boolean value).


Now that we know how ```playGame()``` works, let’s see how ```evaluateGame()``` works.

```
function evaluateGame() private {


    address _player1 = player1;
    uint8 _player1Move = player1Move;
    address _player2 = player2;
    uint8 _player2Move = player2Move;


    if (_player1Move == _player2Move) {
        _player1.call{value: address(this).balance / 2}("");
        _player2.call{value: address(this).balance}("");
        emit TieGame(_player1, _player2);
    } else if (
    (_player1Move == 1 && _player2Move == 3 ) || 
    (_player1Move == 2 && _player2Move == 1)  || 
    (_player1Move == 3 && _player2Move == 2)) {
    
        _player1.call{value: address(this).balance}("");
        emit GameOver(_player1, _player2);
    } else {
        _player2.call{value: address(this).balance}("");
        emit GameOver(_player2, _player1);
    }


    gameInProgress = false;
    player1 = address(0);
    player2 = address(0);
    player1Move = 0;
    player2Move = 0;
    gameStart = 0;
}
```

The first thing we do in ```evaluateGame()``` is store storage variables to the stack. To understand why we are doing this let’s talk about the gas costs of storage for a moment. When referring to storage slots there are two different states; cold storage and warm storage. Cold storage is when a slot has not been previously accessed in the transaction (we already accessed these variables in ```playGame()```). Accessing cold storage is very expensive and it costs 2,100 gas. Once you access a storage slot in a transaction it is called warm storage. Accessing warm storage costs 100 gas, which is still pretty expensive. This is one of the advantages of packing variables, since you can access warm storage instead of cold storage even if it's your first time touching that particular variable. Accessing a variable from the stack can have different costs, but in the above example it is around 10 gas per read. This is a significant saving, so keep this in mind when writing smart contracts. 

The next section of the code checks who wins the game. The first ```if``` statement is checking if there is a tie. In that case the contract sends half of the ether to both players. Next we use a lengthy ```if``` statement to check the situations that Player 1 wins. If any of those conditions are met we send Player 1 the ether. Otherwise, Player 2 must have won, and Player 2 gets rewarded the ether.

The last section of this function resets the smart contract state to allow users to play a new game. If you call ```getStorage()```, you will see that the layout is the same as before we called ```creatGame()```. Notice how we are resetting slot 0 and slot 2 to being empty. This actually gives us a gas refund! For each slot set to 0, you receive 15,000 gas. However, the maximum refund is ⅕ of the transaction’s gas cost.

The last function we need to go over in our Solidity contract is ```terminateGame()```.
```
function terminateGame() external {
    require(gameStart + gameLength < block.number, "Game has time left.");
    require(gameInProgress == true, "Game not started");




    if(player1Move != 0) {
        player1.call{value: address(this).balance}("");
    } else if(player2Move != 0) {
        player2.call{value: address(this).balance}("");
    }


    gameInProgress = false;
    player1 = address(0);
    player2 = address(0);
    player1Move = 0;
    player2Move = 0;
    gameStart = 0;


    emit GameTerminated(player1, player2);
}
```
Our ```require()``` statements check if the game has gone past its allocated time, and whether there is a game to terminate. Next we check if either player has made a move yet. If they have, we give them the contracts ether. Finally we reset the contract state the way we did in ```evaluateGame()```.



## Hybrid Contract
Now let’s see how we can use Yul to save on some gas costs!

The hybrid contract is going to have the same storage layout as our pure Solidity contract. The constructor will also remain the same.

Here is our updated ```creatGame()``` function.
```
// allows users to create a new game// allows users to create a new game
function createGame(address _player1, address _player2) external {
    require(gameInProgress == false, "Game still in progress.");


    assembly {
        sstore(0, number())
        sstore(2, _player1)
        let glAndGip := or(0x0000000000000000000001000000000000000000000000000000000000000000, sload(3))
        sstore(3, or(glAndGip, _player2))
    }
   
   
}


```
The ```require()``` is the same, but after that we dive into some Yul. First, we store the block number to storage slot 0. This is setting ```gameStart```. Next we store Player 1’s address in slot 2. After, we need to format our data to pack it into slot 3. Storage slot 3 is already storing ```gameLength``` at this time. So all we have to do it load slot 3 and ```or()``` it with a 32 byte value that has true set for ```gameInProgress``` (```0x0000000000000000000001000000000000000000000000000000000000000000
```). Our last step is to ```or()``` this value with the address of Player 2 and store it. If you call ```getStorage()```, you will see the same results as when we use the Solidity smart contract. 


```playGame()``` is going to look identical to as it did in the Solidity contract. However, ```evaluateGame()``` has one big change in it.
```
function evaluateGame() private {


    address _player1 = player1;
    uint8 _player1Move = player1Move;
    address _player2 = player2;
    uint8 _player2Move = player2Move;




    if (_player1Move == _player2Move) {
        _player1.call{value: address(this).balance / 2}("");
        _player2.call{value: address(this).balance}("");
        emit TieGame(_player1, _player2);
    } else if ((_player1Move == 1 && _player2Move == 3 ) || (_player1Move == 2 && _player2Move == 1)  || (_player1Move == 3 && _player2Move == 2)) {
        _player1.call{value: address(this).balance}("");
        emit GameOver(_player1, _player2);
    } else {
        _player2.call{value: address(this).balance}("");
        emit GameOver(_player2, _player1);
    }


    assembly {
        sstore(0,0)
        sstore(2, 0)
        let gameLengthShifted := shr(mul(23,8), sload(3))
        let gameLengthVal := shl(mul(23,8), gameLengthShifted)
        sstore(3, gameLengthVal)
    }
   
}


```
The first part of the contract is the same, but the last assembly block is where we optimize our code. We will go over how this saves us gas in a moment, but in the meantime let's finish going over the hybrid contract.

Lastly, let's look at ```terminateGame()```.
```
function terminateGame() external {
    require(gameStart + gameLength < block.number, "Game has time left.");
    require(gameInProgress == true, "Game not started");




    if(player1Move != 0) {
        player1.call{value: address(this).balance}("");
    } else if(player2Move != 0) {
        player2.call{value: address(this).balance}("");
    }
   
    assembly {
        sstore(0,0)
        sstore(2, 0)
        let gameLengthShifted := shr(mul(23,8), sload(3))
        let gameLengthVal := shl(mul(23,8), gameLengthShifted)
        sstore(3, gameLengthVal)
    }


    emit GameTerminated(player1, player2);
}


```
You probably notice the change is very similar. Again, we only change one section, the assembly code block. So why did we do this? Well the answer is, when we are clearing our storage slots this gives us the power to clear two packed variables with one operation instead of manually having to clear each one like in the Solidity contract.

Now that we have two contracts let’s compare the gas costs of calling each one.

|  Function  |   Solidity Min | Hybrid Min | Solidity Max  |  Hybrid Max | Solidity Avg  |  Hybrid Avg |  Avg Gas Savings  |
|  :---   | :--- | :--- |  :--- |   :--- |   :--- |   :--- |   :--- | 
|   ```createGame()``` |  72,362  | 71,954  |   72,362 |  71,954  |  72,362 | 71,954 |  592   |
|   ```playGame()```  |  32,907 | 32,818  |   52,085  |  50,853 |  37,858  |  37,298 |  560 |
|   ```terminateGame()```  |  32,279 |  31,382  |   40,473  |  39,352  |  36,882  |  35,911  |  971  |



We made every function call less expensive, including deployment costs (1,240,438 vs 1,171912)! It is also worth noting that for ```playGame()```, if the second player is calling the function it calls ```evaluateGame()```. So the 1,232 gas difference between both max’s gas is a big deal for users.

Now that we wrapped up this section we are ready to write our contract purly in Yul!


## Yul
I want to start this section off by saying if you are following along, I recommend you use remix. Hardhat does not support pure Yul contracts, so you will have issues compiling your smart contract. Additionally, when working with a contract purely written in Yul, you can not just call a function like you would in Solidity. For example, instead of calling ```playGame()``` you would have to structure your call data manually and call the function selector ```0x985d4ac3```. So first let’s look at how we need to structure our calldata so we can call our contract.

```createGame()```’s function selector is ```0xa6f979ff```. If you do not remember how to derive a function selector, please check out my “Beginner's Guide to Yul,” which is linked at the beginning of this article. Next, we need to look at how we pass our two address arguments from before. If you recall, memory works in serieses of 32 bytes. So you need to format both addresses from 20 bytes to 32 bytes by padding the left side with 12 empty bytes. Here is an example of what our calldata will look like: ```0xa6f979ff0000000000000000000000005b38da6a701c568545dcfcb03fcb875f56beddc4000000000000000000000000ab8483f64d9c6d1ecf9b849ae677dd3315835cb2```
```Function Selector```: ```0xa6f979ff```
```_player1```: ```0000000000000000000000005b38da6a701c568545dcfcb03fcb875f56beddc4```
```_player2```: ```000000000000000000000000ab8483f64d9c6d1ecf9b849ae677dd3315835cb2```

```playGame()``` is a little bit simpler because we only have one variable. Although it is a ```uint8```, since there's only one variable, and we are not packing it, it takes up 32 bytes. Here is what our calldata looks like if we use rock: ```0x985d4ac30000000000000000000000000000000000000000000000000000000000000001```
```Function Selector```: ```0x985d4ac3```
```_move```: ```0000000000000000000000000000000000000000000000000000000000000001```

Last is ```terminateGame()```. It has no parameters so we only need to use the function selector for our calldata: ```0x97661f31```

Great, now we are ready to dive into the code!

```
object "RockPaperScissorsYul" {


    code {
        sstore(1, 0xB1A2BC2EC50000)
        sstore(3, 0x000000000000000B400000000000000000000000000000000000000000000000)
        datacopy(0, dataoffset("runtime"), datasize("runtime"))
        return(0, datasize("runtime"))
    }


    object "runtime" {


        // Storage Layout:
        // slot 0 : gameStart
        // slot 1 : gameCost
        // slot 2 : bytes 13 - 32 : player1
        // slot 2 : bytes 12 : player1Move
        // slot 3 : bytes 13 - 32 : player2
        // slot 3 : bytes 12 : player2Move
        // slot 3 : bytes 11 : gameInProgress
        // slot 3 : bytes 10  : lockGame
        // slot 3 : bytes 9 - 8 : gameLength


        // rest of code
    }
}
```

```object "RockPaperScissorsYul.sol" {}``` is declaring our smart contract. All code will run within it. The first ```code {}``` is what a constructor would be in solidity. Inside we are storing ```gameCost``` as 0.05 ether in slot 1. Then, we are storing ```gameLength```. Notice I set a different value this time, feel free to set it to whatever you prefer. The next two lines are copying the runtime code and returning it. Next, ```object "runtime" {}``` is where we put our code that we want to call at runtime. Finally, I put the storage layout. I recommend doing this when writing contracts in Yul, because it helps you keep track of where all of your variables are stored.

Since this code is long and formatted differently then Solidity, we will be going over it in chunks.
```
code {
            let callData := calldataload(0)
            let selector := shr(0xe0, callData)


            switch selector


            // createGame(address, address)
            case 0xa6f979ff {


                // get gameInProgress from storage
                let gameInProgress := and(0xff, shr( mul( 21, 8), sload(3) ) )
                // if game in progress set, revert
                if eq(gameInProgress, 1) {
                    revert(0,0)
                }


                // copies calldata to memory without function selector
                calldatacopy(0, 4, calldatasize())
                // gets address 1 & 2
                let address1 := and(0x000000000000000000000000ffffffffffffffffffffffffffffffffffffffff, mload(0x00))
                let address2 := and(0x000000000000000000000000ffffffffffffffffffffffffffffffffffffffff, mload(0x20))


                // stores block of game start & player1 to storage
                sstore(0, number())
                sstore(2, address1)


                // packs gameLength, gameInProgress, and player2 into slot 3
                let gameLengthShifted := shr(mul(23,8), sload(3))
                let gameLengthVal := shl(mul(23,8), gameLengthShifted)
                let glAndGip := or(0x0000000000000000000001000000000000000000000000000000000000000000, gameLengthVal)
                sstore(3, or(glAndGip, address2))


            }
            // more cases
}
```

```code{}``` is the code we will be running, we will wrap the rest of our code within it.  ```calldataload(0)``` is loading the first 32 bytes of call data. Then we are shifting right by 28 bytes to isolate the function selector.























