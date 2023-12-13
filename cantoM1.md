#  Inaccurate fee calculation on bonding curve

*mark0v*

**severity: medium**

The calculation of fees incurs a 3% to 20% relative error, which walks in both positive and negative directions from the quantity if it were calculated normally(the value anyone would expect it to be at least approximating, and docs would reflect). Anyone trying to account for this later could get a serious problem, though I don't see a manifestation of this yet. It is at bare minimum going to result in a very misrepresented fee structure due to negligence. 

[1155tech-contracts/src/bonding_curve/LinearBondingCurve.sol Line 20 of code-423n4/2023-11-canto ``main``](https://github.com/code-423n4/2023-11-canto/tree/335930cd53cf9a137504a57f1215be52c6d67cb3/1155tech-contracts/src/bonding_curve/LinearBondingCurve.sol#L20)
```solidity
// File: 1155tech-contracts/src/bonding_curve/LinearBondingCurve.sol
// Lines: 20-24

20         for (uint256 i = shareCount; i < shareCount + amount; i++) {
21             uint256 tokenPrice = priceIncrease * i;
22             price += tokenPrice;
23             fee += (getFee(i) * tokenPrice) / 1e18;
24         }
```


[1155tech-contracts/src/bonding_curve/LinearBondingCurve.sol Line 30 of code-423n4/2023-11-canto ``main``](https://github.com/code-423n4/2023-11-canto/tree/335930cd53cf9a137504a57f1215be52c6d67cb3/1155tech-contracts/src/bonding_curve/LinearBondingCurve.sol#L30)
```solidity
// File: 1155tech-contracts/src/bonding_curve/LinearBondingCurve.sol
// Lines: 30-35

30             divisor = log2(shareCount);
31         } else {
32             divisor = 1;
33         }
34         // 0.1 / log2(shareCount)
35         return 1e17 / divisor;
```

[1155tech-contracts/src/bonding_curve/LinearBondingCurve.sol Line 38 of code-423n4/2023-11-canto ``main``](https://github.com/code-423n4/2023-11-canto/tree/335930cd53cf9a137504a57f1215be52c6d67cb3/1155tech-contracts/src/bonding_curve/LinearBondingCurve.sol#L38)
```solidity
// File: 1155tech-contracts/src/bonding_curve/LinearBondingCurve.sol
// Lines: 38-42

38     /// @dev Returns the log2 of `x`.
39     /// Equivalent to computing the index of the most significant bit (MSB) of `x`.
40     /// Returns 0 if `x` is zero.
41     /// @notice Copied from Solady: https://github.com/Vectorized/solady/blob/main/src/utils/FixedPointMathLib.sol
42     function log2(uint256 x) internal pure returns (uint256 r) {
```


Proof of concept
----------------

This simulates it and shows the error walking around and how high it can be when the numbers are on the smaller end when the supply is low. it is not necessarily perfect but the main term introducing error is the log and it should come out as the floor of the exact value so i am basing it off of that. how it will manifest IRL is harder to guess, and not necessarily as predictable  



```julia
function getpriceandfee_real(L,amt)
    price = 0
    fee=0
    for s = L:(L+amt)
        getfee = (10^(k-1)/(log2(s)))
        tokenPrice=s
        price += s
        fee +=((getfee*tokenPrice)/(10^k))
    end
    return price,fee
end


function getpriceandfee(L,amt)
    price = 0
    fee=0
    for s = L:(L+amt)
        getfee = floor(10^(k-1)/floor(log2(s)))
        tokenPrice=s
        price += s
        fee +=floor((getfee*tokenPrice)/(10^k))
    end
    return price,fee
end

count=50
for jj=100:100:10000
    price,fee =  getpriceandfee_real(jj+1,count+1)
    priceR,feeR =  getpriceandfee(jj+1,count+1)
    print((feeR-fee)/feeR,'\n')
end
```


mitigation
----------

you can use this: $log(a x) = log(a)+log(x)$ or this relationship $a log(x) = log(x^a)$ to translate the calculations into a range where the truncation isnt going to curb stomp accuracy, and remove the extra magnitude later

in terms of what is already there, 
$$
\frac{10^{17}}{log_2(x)} = \frac{10^{18}}{10 log_2(x)} = \frac{10^{18}}{log_2(x^{10})}
$$

This would be good if we have a bound on $x$ (shares) that is small

but its not safe at all so its probably best to do $log_2(x) := log_2(2^{32} x) - 32$ or something like this, and pay attention to the distribution of values sampled in the function(from solady) to keep operations low.  theres a lot of ways to fix this so it should be taylored to whatever lends efficiency and safety with regards to problems like overflows. 

