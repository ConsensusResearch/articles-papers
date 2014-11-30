Inside a Proof-of-Stake Cryptocurrency part 2: Forging Algo
============================================================

Introduction
--------------

Welcome to the second part of "Inside a Proof-of-Stake Cryptocurrency" series.
[First chapter](http://chepurnoy.org/blog/2014/10/inside-a-proof-of-stake-cryptocurrency-part-1/) defined basic
structures specific to proof-of-stake(but not only) p2p cryptocurrencies.
We're going deeper now into details of a forging algo. Provided information is more specific to
[Nxt](http://nxt.org/) and Nxt-like 100% PoS currencies. It will be more code in this chapter so in the first place
let's have a closer look at some of its properties.


Functions & Types
------------------

_(This section could be skipped while reading for the first time)_

_Everything_ is a function in Haskell. E.g. even `txTimestamp` "field" of `Transaction` record is a function in fact.
So having value `tx` of `Transaction` type, you can get it's timestamp by calling the function as `txTimestamp tx`.

Every function definition(except of implicitly defined functions e.g. record fields) starts with signature describing
types of arguments and result. E.g. `validate :: Transaction -> Bool` signature means "function `validate` accepts
an argument of `Transaction` type and returns boolean value(i.e. true or false). In the same manner `calcGenerationSignature :: Block -> Account -> ByteString`
literally means "function `calcGenerationSignature` is producing ByteString value (i.e. array of bytes) from instances of Block and Account types".

A function in Haskell is pure means the output is determined by inputs only. In case of purity violation output
should be wrapped into [a monad](http://www.haskell.org/haskellwiki/Monad) to make function pure again.
Hopefully, no knowledge of monads is needed for this chapter.

Why does that matter in the context of the article?

* Even from signatures provided in the article it's obvious the algo is deterministic
* Before reading function body try to understand its signature i.e. entities going in and out
* All the code provided below is about field getters and pretty simple self-explanatory functions mostly. I hope the code
 is understandable even without Haskell knowledge.


A Chance to Be Richer
----------------------

The goal of forging algo is to choose accounts(the only one in best case) having right to generate a block.
Obviously, an algo should be:

* verifiable
* deterministic (or it will be not verifiable in cryptocurrency environment)


Target and Hit
---------------

A forging algorithm is reminiscent of a rock-paper-scissors game. Every account willing and able to forge generates a pseudo-random value called `hit` every second.
If it's less than the stake-dependent `target`value, a node the account is running against is going to generate a block.
Other parties prevent cheating by checking both values against the incoming block, as `hit` and `target` are both
deterministic.


Hit
---

Our first goal is to produce a deterministic account-dependent `hit` value with fair distribution.
Do you remember `generationSignature :: ByteString` field in Block structure? There are two purposes for it:

1. To link a block with a previous one.
2. It is also being using to generate `hit`

The formula for `generationSignature` is pretty simple: it's the hash from `generationSignature` of previous block
attached with generator's public key. Following code calculates it using previous block and candidate account:


    calcGenerationSignature :: Block -> Account -> ByteString
    calcGenerationSignature prevBlock acct = bytestringDigest(sha256(append gsPrev pk))
        where pk = publicKey acct
              gsPrev = generationSignature prevBlock

Then `hit` is just a number constructed from first 8 bytes of generation signature:

    calculateHit :: Block -> Account -> Integer
    calculateHit prevBlock account =  fromIntegral $ runGet getWord64le first8
        where first8 = take 8 (calcGenerationSignature prevBlock account)

For the genesis block generationSignature is set to some predefined value e.g. filled with zeros.


Target & Hit Verifying
----------------------

To generate the `target` value for our rock-paper-scissors game we use last block again. More precisely,
we multiply its `baseTarget` field value by generator stake (i.e. effective balance) and the elapsed time
since last block generation timestamp (in seconds). Then the calculated `target` is being used in the `verifyHit` function which has
two use cases:

* Honest forging account decides whether it has a right to forge a block by calling the function.
Checking is to be done each second (and each second `target` increases along with a chance to generate a block by
satisfying `hit < target` condition).
* Other parties ensure the incoming block is forged by a proper account with the right to do it

    verifyHit :: Integer -> Block -> Timestamp -> Integer -> Bool
    verifyHit hit prevBlock timestamp effBalance =  (hit < target) && (eta > 0)
        where eta = timestamp - blockTimestamp prevBlock
              target = effBalance*(baseTarget prevBlock)*eta

Again, if a result of function call is true, a generator is going to form a block and push it to the network.
Other nodes can check whether block is generated properly by calling the same function.


BaseTarget & Block Constructing Function
----------------------------------------

Now it's time to define `baseTarget :: Integer` field of a block used as an initial value for a forger to generate
its own `target` in previous section.
As well as `generationSignature`, `baseTarget` also depends on a previous block's field value
with some predefined constant for genesis block. In the case of Nxt the base target value for the genesis block is:

    initialBaseTarget :: Integer
    initialBaseTarget = 153722867

Why 153722867? Well, chapter 3 of this series will give precise formula describing that. The `baseTarget` of
a block is `baseTarget` of a previous block multiplied by elapsed time since previous block generation timestamp,
in minutes. It's also bounded to be not less than half of a previous block field value nor more than twice as high as it.
Also it couldn't be less than 1 and more than `maxBaseTarget` which is:

    maxBaseTarget :: Integer
    maxBaseTarget = initialBaseTarget * systemBalance

Having rules to calculate `generationSignature` and `baseTarget` we can define the function to form a block based on previous block,
generator account, block generation timestamp and transactions to be included:

    formBlock :: Block -> Account -> Timestamp -> [Transaction] -> Block
    formBlock prevBlock gen timestamp txs =
        Block{transactions = txs, blockTimestamp = timestamp, baseTarget = bt, generator = gen, generationSignature = gs}
        where prevTarget = baseTarget prevBlock
              maxTarget = min (2*prevTarget)  maxBaseTarget
              minTarget = max (prevTarget `div` 2)  1
              candidate = prevTarget*(timestamp - (blockTimestamp prevBlock)) `div` 60
              bt = min (max minTarget candidate) maxTarget
              gs = calcGenerationSignature prevBlock gen


Difficulty
----------------------

Within blocktree the canonical blockchain is the path having the max value e.g. height. We will use sum of `baseTarget`
 to select canonical blockchain from possible options and call this function `cumulativeDifficulty`:

    cumulativeDifficulty :: BlockChain -> Integer
    cumulativeDifficulty BlockChain {blocks=bs} = sum(map baseTarget bs)


Transparent Forging
-------------------

There's a lot of buzz about transparent forging these days. The transparent forging concept is about a forger of
a next block is to be known for all online network members in prior. It's possible because the algo is deterministic
but in practice there are some issues to be resolved:

* A forger could miss its turn (there are some workarounds to make block generation quicker for a next forger though)
* List of forging accounts for the whole network is needed. And it's not possible to have it consistent constantly.
* Even in case of pretty weak consistency it could be a tricky task to implement gossiping about forgers in privacy-friendly
way.



Conclusion & Further Work
--------------------------

This chapter of the series defined functional building blocks and whole forging algo as well.
Next chapter will be kind of academic paper about statistical modelling of Nxt forging algo
with some interesting results. After that we will go deeper into forging simulation with Haskell.