### [M-#] There can be a denial of servies (DoS) attack in the function `PuppyRaffle::enterRaffle`.

**Description:** The function `PuppyRaffle::enterRaffle` loops through the `players` to check for duplicates. Which is a issue as longer the `PuppyRaffle::players` array becomes the more it has to loop through the array and check for duplicates. This means that the gas cost for the  players who entered right after the raffle starts will be less as complared to the players entering after and after. So more the players gets added to the `PuppyRaffle::players` array the more gas will be used.

**Impact:** The gas cost for the raffle will increase as more and more players enters the raffle which will definitely give advantage to the people entering as the start of the raffle and there might be a rust of people trying to enter the raffle as the starting.

An attacker can even make the `PuppyRaffle::entrants` so big that no one else can enter and gauranteeing themselves the win.

**Proof of Concept:** The below test shows that the first 100 people entering the raffle costs a total of `6503275` gas and the next 100 people entering the raffle will cost a total of `18995515` gas which is a huge difference. 

<details>
<summary>PoC</summary>
Add this below test in your `PuppyRaffleTest.t.sol`.

```javascript

function testDosAttackPossible() external{
        uint256 playersNum = 100;
        address[] memory players = new address[](playersNum);
        for(uint256 i = 0; i < players.length; i++){
            players[i] = address(i);
        }
        uint256 startingGas = gasleft();
        puppyRaffle.enterRaffle{value: entranceFee * players.length}(players);
        uint256 endingGas = gasleft();
        uint256 gasUsed = startingGas - endingGas;
        console.log("the gas used by first 100 players is", gasUsed);
        // now for the next 100 players
        address[] memory nextPlayers = new address[](100);
        for(uint256 j = 0; j < nextPlayers.length; j++){
            nextPlayers[j] = address(j+100);
        }
        uint256 startingGas2 = gasleft();
        puppyRaffle.enterRaffle{value: entranceFee * nextPlayers.length}(nextPlayers);
        uint256 endingGas2 = gasleft();
        uint256 gasUsed2 = startingGas2 - endingGas2;
        console.log("the gas used by last 100 players is", gasUsed2);
        assert(gasUsed < gasUsed2);
    }

```

</details>

**Recommended Mitigation:** The below steps are the recomendations from my side that you should follow: 

1. Instead of using a nested for loop which takes a lot of gas, you should use mapping let me show you how.

```diff

mapping(address => bool) public isAdded;

function enterRaffle(address[] memory newPlayers) public payable {
        require(msg.value == entranceFee * newPlayers.length, "PuppyRaffle: Must send enough to enter raffle");
        for (uint256 i = 0; i < newPlayers.length; i++) {
            // Check for duplicates
            require(!isAdded[newPlayers[i]], "PuppyRaffle: Duplicate player")
            isAdded[newPlayers[i]] = true;
            players.push(newPlayers[i]);
        }
        // @audit DOS attack:
-        for (uint256 i = 0; i < players.length - 1; i++) {
-            for (uint256 j = i + 1; j < players.length; j++) {
-               require(players[i] != players[j], "PuppyRaffle: Duplicate player");
-            }
-        }
        emit RaffleEnter(newPlayers);
    }


```

2. Allow duplicates to enter the raffle, as if any user wants to enter more than once he could just use different metamask accounts and enter from them.