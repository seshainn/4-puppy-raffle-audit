DoS
### [M-1] Looping through players array to check for duplicates in PuppyRaffle::enterRaffle is a potential DoS vector, incrementing gas costs for future entrants

** Description: **
The PuppyRaffle::enterRaffle function loops through the players array to check for duplicates. However, the longer the PuppyRaffle:players array is, the more checks a new player will have to make. This means that the gas costs for players who enter right when the raffle starts will be dramatically lower than those who enter later. Every additional address in the players array, is an additional check the loop will have to make.

** Impact: **
The impact is two-fold: 
The gas costs for raffle entrants will greatly increase as more players enter the raffle.
Front-running opportunities are created for malicious users to increase the gas costs of other users, so their transaction fails.

** Proof of Concept: **
If we have 2 sets of 100 players enter, the gas costs will be as such:

1st 100 players: 6252039
2nd 100 players: 18067741
This is more than 3x as expensive for the second set of 100 players!

This is due to the for loop in the PuppyRaffle::enterRaffle function.

```javascript
// @audit DoS attack
    for (uint256 i = 0; i < players.length - 1; i++) {
        for (uint256 j = i + 1; j < players.length; j++) {
            require(players[i] != players[j], "PuppyRaffle: Duplicate player");
        }
    }
```

<details>
<summary>Proof of Code</summary>
Place the following test into 'PuppyRffleTest.t.sol'.
'''javascript
    function test_DoS() public {
        vm.txGasPrice(1);
        uint256 playersNum = 100;

        address[] memory players = new address[](playersNum);
        for (uint256 i = 0; i < playersNum; i++)  {
            players[i] = vm.addr(i+1);
        }
        uint256 gasStart = gasleft();
        puppyRaffle.enterRaffle{value: entranceFee * players.length}(players);
        uint256 gasEnd = gasleft();
        uint256 gasUsedFirst100 = (gasStart - gasEnd) * tx.gasprice;
        console.log("Gas cost of the first 100 players", gasUsedFirst100);

        address[] memory players2 = new address[](playersNum);
        for (uint256 i = 0; i < playersNum; i++)  {
            players2[i] = vm.addr(i+101);
        }
        gasStart = gasleft();
        puppyRaffle.enterRaffle{value: entranceFee * players2.length}(players2);
        gasEnd = gasleft();
        uint256 gasUsedSecond100 = (gasStart - gasEnd) * tx.gasprice;
        console.log("Gas cost of the second 100 players", gasUsedSecond100);
        assert(gasUsedFirst100 < gasUsedSecond100);
    }
'''
</details>

** Recommended Mitigation: **
There are a few recommended mitigations.
1. Consider allowing duplicates. Users can make new wallet addresses anyways, so a duplicate check doesn't prevent the same person from entering multiple times, only the same wallet address.
2. Consider using a mapping to check duplicates. This would allow you to check for duplicates in constant time, rather than linear time. You could have each raffle have a uint256 id, and the mapping would be a player address mapped to the raffle Id.
```javascript
+    mapping(address => uint256) public addressToRaffleId;
+    uint256 public raffleId = 0;
    .
    .
    .
    function enterRaffle(address[] memory newPlayers) public payable {
        require(msg.value == entranceFee * newPlayers.length, "PuppyRaffle: Must send enough to enter raffle");
        for (uint256 i = 0; i < newPlayers.length; i++) {
            players.push(newPlayers[i]);
+            addressToRaffleId[newPlayers[i]] = raffleId;            
        }

-        // Check for duplicates
+       // Check for duplicates only from the new players
+       for (uint256 i = 0; i < newPlayers.length; i++) {
+          require(addressToRaffleId[newPlayers[i]] != raffleId, "PuppyRaffle: Duplicate player");
+       }    
-        for (uint256 i = 0; i < players.length; i++) {
-            for (uint256 j = i + 1; j < players.length; j++) {
-                require(players[i] != players[j], "PuppyRaffle: Duplicate player");
-            }
-        }
        emit RaffleEnter(newPlayers);
    }
.
.
.
    function selectWinner() external {
+       raffleId = raffleId + 1;
        require(block.timestamp >= raffleStartTime + raffleDuration, "PuppyRaffle: Raffle not over");
```
3. Alternatively, you could use OpenZeppelin's EnumerableSet library.