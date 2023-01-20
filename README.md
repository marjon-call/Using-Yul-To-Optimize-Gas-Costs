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
        gameLength = 28800; // 8 hours
    }

```
All that we are doing here is setting ```gameCost``` to 0.05 ether, and setting ```gameLength``` to 28800 (8 hours).


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
First thing we do is check if a game is in progress. If it is, we revert. Otherwise,  we set our storage variables to set up our game.


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

Ok, so the first thing we are going to do is use ```require()``` statements to verify that we are playing by the rules. The first ```require()``` checks if the game is locked. The second ```require()``` checks that a game has been created. The third ```require()``` is checking if the player has sent enough ether to play the game. The last ```require()``` checks if the player has sent a valid move. After that we lock the game. Then, we have an if statement checking if Player 1 is making their move. We also check if they have made a move already by checking if the value of ```player1Move``` is set to 0 (remember that Solidity initializes values to 0). Notice that we check the player first. We deem that it is more likely that condition will not be satisfied, by doing so we save gas by not needing to check if Player 1 has made their move yet (less operations, less gas costs). If the conditions are satisfied we set Player 1’s move. Then we do the same checks for Player 2. If neither of the if statements are satisfied, we revert. Otherwise, we check if both moves have been made. If they have, we call ```evaluateGame()```. Finally, we unlock the game. 

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



























