### [H-1] Reentrancy attack  in `PuppyRaffle::refund` allows entrant to darin raffle fund.

**Description** The `PuppyRaffle::refund` function doesnot allow CEI (Checks, effects and interaction) and as a result it drain the entire raffle funds.

In the `PuppyRaffle::refund` function , we make the external call to the `msg.sender` address and only after making this external call do we update the `players` array. This allow msg.sender to call the refund function and drain the raffle funds.

```javascript
    function refund() external {
        require(
            block.timestamp > raffleEnd,
            "PuppyRaffle: Raffle has not ended"
        );
        require(
            !isRaffleEnded,
            "PuppyRaffle: Raffle has already ended"
        );
        isRaffleEnded = true;
        for (uint256 i = 0; i < players.length; i++) {
            payable(players[i]).transfer(entranceFee);
        }
        players = new address[](0);
    }
```

**Impact** An attacker can drain the entire raffle funds by calling the `PuppyRaffle::refund` function.

**Proof of Concept** 

```javascript

function test_Rentrancy_refund() public {
        address[] memory players = new address[](4);
        players[0] = playerOne;
        players[1] = playerTwo;
        players[2] = playerThree;
        players[3] = playerFour;
        puppyRaffle.enterRaffle{value: entranceFee * 4}(players);
        ReentranctyAttackContract attackerContract = new ReentranctyAttackContract(
                puppyRaffle
            );
        address attackerUser = makeAddr("attackerUser");
        vm.deal(attackerUser, 1e18);

        uint256 startingAttackerContractBalance = address(attackerContract)
            .balance;
        uint256 startingContractBalance = address(puppyRaffle).balance;

        vm.prank(attackerUser);
        attackerContract.attack{value: entranceFee}();
        console.log(
            "attacker Contract Balance",
            startingAttackerContractBalance
        );
        console.log("starting contract balance", startingContractBalance);
        console.log(
            "ending attcker balance",
            address(attackerContract).balance
        );
        console.log("ending contract balance", address(puppyRaffle).balance);
    }
    
```

**Recommended Mitigation** The recommended mitigation is to follow the Checks-Effects-Interactions pattern. This means that all the checks should be done before any effects or interactions are made. In this case, the `players` array should be updated before making the external call to the `msg.sender` address.

```javascript
    function refund() external {
        require(
            block.timestamp > raffleEnd,
            "PuppyRaffle: Raffle has not ended"
        );
        require(
            !isRaffleEnded,
            "PuppyRaffle: Raffle has already ended"
        );
        isRaffleEnded = true;
        address[] memory playersTemp = players;
        players = new address[](0);
        for (uint256 i = 0; i < playersTemp.length; i++) {
            payable(playersTemp[i]).transfer(entranceFee);
        }
    }
```

### [M-1] Looping to players array to check for duplicate in `PuppyRaffle:enterRaffle` is potential DOS attack, incremneting gas cost for future entrants.

**Description** The `PuppyRaffle::enterRaffle` loops to the player arrays to check dup. However the longer the `PuppyRaffle::players` array is the more checks a new player will have to make. This means the gas cost for palyers who enter at start has too pay too low gas fee as comparre to who enter last has to pay alot of gasFees.

```javascript
//@DOS attack
         for (uint256 i = 0; i < players.length - 1; i++) {
            for (uint256 j = i + 1; j < players.length; j++) {
                require(
                    players[i] != players[j],
                    "PuppyRaffle: Duplicate player"
                );
            }
        }
```

**Impact** The gas cost of for raffle entrant will greatly increase as more player enter thr raffle. Discouraging later user form entering and causing a rush at the start of a raffle to be one of the first entrant on the queue.

Anattacker might fill up the raffle array so big, that no one else enter, gurantee themself the win.

**Proof of Concept** 

If we have 2 set of number of player enter in raffle the gas cost will be high for the 2set for the people.
- 1st batch of gas used- 6252039
- 2st batch of gas used- 18068129
This is more than a 3x of the used from the 1st batch.

```javascript
 function test_breakingforDosc() external {
        vm.txGasPrice(1);
        //Forst 100 Batch
        uint256 playerNum = 100;
        address[] memory players = new address[](playerNum);
        for (uint256 i = 0; i < playerNum; i++) {
            players[i] = address(i );
        }
        uint256 gasStart = gasleft();
        puppyRaffle.enterRaffle{value: entranceFee * playerNum}(players);
        uint256 gasEnd = gasleft();
        uint256 gasUsedFirst = gasStart - gasEnd;
        console.log("gasUsed", gasUsedFirst);

        //Sencond 100 Batch

        address[] memory playersSecond = new address[](playerNum);
        for (uint256 i = 0; i < playerNum; i++) {
            playersSecond[i] = address(i + playerNum);
        }
        uint256 gasStartSecond = gasleft();
        puppyRaffle.enterRaffle{value: entranceFee * playerNum}(playersSecond);
        uint256 gasEndSecond = gasleft();
        uint256 gasUsedSecond = gasStartSecond - gasEndSecond;
        console.log("gasUsed", gasUsedSecond);
        assert(gasUsedFirst < gasUsedSecond);
    }
 ```   
 <br>
 
**Recommended Mitigation** There are few recommendation.
    1. Consider the allowing then duplicate. User can make new wallwt address anyways to enter raffle, so duplicate check doesnt prevent the samw person from entering multiple time, only the same wallet address.
    2. Consider using mapping to check for duplicates. This would allow constant time loopup of wheather a user has already entered or not.


## L-2: Solidity pragma should be specific, not wide

Consider using a specific version of Solidity in your contracts instead of a wide version. For example, instead of `pragma solidity ^0.8.0;`, use `pragma solidity 0.8.0;`


- Found in src/PuppyRaffle.sol [Line: 2](src/PuppyRaffle.sol#L2)

	```solidity
	pragma solidity ^0.7.6;
	```


## I-1 Using older version is not recommended.
Solc releases new version of compiler verison. USing older version prevent access to new solidity security checks. We also recommended using new version like `0.8.18`.


# Gas

## [G-1] Unchanged state variable should be declared constant or immutablw.

Reading from sttoragw is much more expensive than constant.
 
Instance :
- `PupptRaffle::raffleDuration` should be `immutable`
- `PuppyRaffle:commonIamgeURI` should be `constant`
- `PuppyRaffle::rareImageUri` should be `constant`
- `PuppyRaffle::legendaryImageUri` should be `constant`


### [G-2] Storage variable in a loop should be cached.

Everytime you use `players.length` you read from storage, as opposed to memory which is more gas efficient.

```diff
+     uint256 playerLength = players.length
-     for (uint256 i = 0; i < players.length - 1; i++) {
+     for (uint256 i = 0; i < playerLength - 1; i++) {
-            for (uint256 j = i + 1; j < players.length; j++) {
+            for (uint256 j = i + 1; j < playerLegth; j++) {
                require(
                    players[i] != players[j],
                    "PuppyRaffle: Duplicate player"
                );
            }
        }
```        

## L-3: Missing checks for `address(0)` when assigning values to address state variables

Check for `address(0)` when assigning values to address state variables.



- Found in src/PuppyRaffle.sol [Line: 62](src/PuppyRaffle.sol#L62)

	```solidity
	        feeAddress = _feeAddress;
	```

- Found in src/PuppyRaffle.sol [Line: 174](src/PuppyRaffle.sol#L174)

	```solidity
	        feeAddress = newFeeAddress;
	```

