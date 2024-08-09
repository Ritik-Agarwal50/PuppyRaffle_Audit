### [M-#] Looping to players array to check for duplicate in `PuppyRaffle:enterRaffle` is potential DOS attack, incremneting gas cost for future entrants.

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
<details>
<Summary>POC</Summary>

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

 </details>
 <br>
 
**Recommended Mitigation** There are few recommendation.
    1. Consider the allowing then duplicate. User can make new wallwt address anyways to enter raffle, so duplicate check doesnt prevent the samw person from entering multiple time, only the same wallet address.
    2. Consider using mapping to check for duplicates. This would allow constant time loopup of wheather a user has already entered or not.
