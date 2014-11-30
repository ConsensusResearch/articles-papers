Inside a Proof-of-Stake Cryptocurrency part 1: Basic Structures
===============================================================

Introduction
------------

Pure and hybrid Proof-of-Stake cryptocurrencies are on the rise today. The most known pure PoS example, Nxt, has tens of clones
at the moment (some of them are quite promising), the Ethereum team thinks about hybrid (PoS/PoW) mining principle, and so on.
At the same time, there are many concerns about the concept which are not formalized well enough, unfortunately. For example even
the famous "nothing-at-stake" attack has not a good description. The sad thing is both good and bad aspects about proof-of-stake
have no good formal description and model behind. The challenge of this series is to make some progress in this area.


Prerequisites
-------------

I assume you know some common things about Bitcoin and proof-of-work mining. It's better to have basic coding skills
 to understand code snippets. Haskell programming language is being using for them (though Haskell knowledge isn't required).
We will start with model as simple as possible. I prefer to make a simple thing harder than to work with complicated case from day one.

Time
----

Wikipedia defines time as "a measure in which events can be ordered from the past through the present into the future,
and also the measure of durations of events and the intervals between them".
For distributed environments working with time means headache because of clock synchronization, events ordering etc.
Fortunately blockchain-based networks make some problems easier.  Due to this our model could be even easier: we assume all clocks
in the system are synchronized! Fortunately, we will lose not too much in modelling quality because of that. Later we can introduce
local clocks. The problems of time in distributed systems was first considered in Lamport's article from [1978](http://research.microsoft.com/en-us/um/people/lamport/pubs/time-clocks.pdf).

Time in our approach is just a number of seconds from genesis block, so our first entity definition is pretty simple:

`type Timestamp = Integer`


Transaction Model
-----------------

Unlike the [Bitcoin-like model](https://en.bitcoin.it/wiki/Transaction) where transaction has inputs connected with
other transaction's outputs as well as own outputs(possibly unspent), we will work with simpler approach. First, we
introduce the Account entity:

    data Account =
        Account {
            balance :: Integer,
            publicKey :: ByteString,
            isForging :: Bool
        }

It has an explicitly defined balance. Many simplifications made in comparison with real currencies, in particular:

* We use no addresses. In real currencies the address is hash(publicKey), while we use publicKey itself as an address.

* We even don't use the private key! The private key is needed for transaction signing, and that is out of scope of our study.


Having Timestamp and Account entities described, we can define transaction as:

    data Transaction =
        Transaction {
            sender :: Account,
            recipient :: Account,
            amount :: Integer,
            fee :: Integer,
            txTimestamp :: Timestamp
        }

Again, it's far away from practical use. There's no id, no even signature, so anyone could print money easily.


Forging
-------

In proof-of-stake currencies there are no third-party "workers", instead, stake holders are block generators.
Instead of term *mining* we'll use *forging*. The meaning is the same though: forging is a process of blocks generation.
Remember the Account definition? The isForging field there shows whether account is participating in forging or not.

Blockchain
----------

In every cryptocurrency I know transactions are grouped into blocks. With the following block definition Proof-of-Stake
 nature of our model starts to appear:

    data Block =
        Block {
            transactions :: [Transaction],
            generator :: Account,
            blockTimestamp :: Timestamp
            baseTarget :: Integer,
            generationSignature :: ByteString
        }

`transactions` and `blockTimestamp` fields are self-describing.
`generator` is account generated this block.
`baseTarget` and `generationSignature` fields are related to forging algorithm and will be described in a next chapter of the tutorial.


The Blockchain is just a sequence of blocks:

`data BlockChain = BlockChain { blocks :: [Block] }`


Effective Balance
-----------------

In proof-of-stake the probability of block generation  depends on the account’s balance.
In Nxt-like currencies, it is a function of the effective balance .
E.g. in Nxt itself the effective balance is the balance 1440 blocks ago from last one minus all spendings
for that period plus leasing forging balance given by other accounts etc. We can simplify again:

    effectiveBalance :: Account -> Integer
    effectiveBalance acc = balance acc

So the effective balance in our model is just the current balance.


The Network
-----------

Nothing has been said about whole p2p network yet. The first entity to be defined here is a node in the network:

    data Node =
        Node {
            nodeChain :: BlockChain,
            unconfirmedTxs :: [Transaction],
            account :: Account
        }

Again, we're  dealing with simplifications:

* Only one blockchain per node(this is true though e.g. for current Nxt Reference Software implementation)

* Only one account per node

Having the Connection entity defined as just a tuple of Nodes:

    type Connection = (Node, Node)

we can define biggest type in our model, System:

    data System =
        System {
            nodes :: [Node],
            connections :: [Connection],
            accounts :: [Account]
        }

In proof-of-stake currencies all the money is usually being “printed” in a genesis block.
The Nxt system has ~ 1 Billion balance from the genesis block:

    systemBalance :: Integer
    systemBalance = 1000000000

Forks
-----

Forks are possible as well as with proof-of-work mining. How could we define fork? Well, a blockchain is stored in a node,
and System has a list of nodes([Node]), so tree of blocks could be exctracted from System by function with following
signature(implementation is missed for now).

     blockTree :: System -> BlockTree

Where Blocktree is just the alias for a list of Blockchain: `type BlockTree = [BlockChain]`

Why is a list called a tree? All blockchains have some common prefix(genesis block at least), so in fact
  we're dealing with a tree structure. However, not enforcing tree structure by type system is not the best design
definitely, but for simplicity's sake we leave it as is for now.


Properties Analysis
--------------------

How can Haskell code helps us with formal model analysis? Let's start with a property given in
[Gavin Wood's Ethereum YellowPaper](http://gavwood.com/Paper.pdf) :

`The canonical blockchain is a path from root to leaf
 through the entire block tree. In order to have consensus
 over which path it is, conceptually we identify the path
 that has had the most computation done upon it, or, the
 heaviest
 path.`

Canonical blockchain could be defined with a function having signature:

 `canonicalBlockchain :: System -> Maybe BlockChain`

(again, no implementation for now). Maybe BlockChain could be Just blockchain(so defined), or Nothing(means undefined).
So if function result is defined a system has some canonical blockchain. We can then enforce more strict conditions, e.g
canonical blockchain for some time ago should be the prefix of current canonical blockchain.

There are two ways to make conclusions about the property:

1. We can make executable forging algo model then gather statistics about function result.

2. We can translate our Haskell model some Haskell-like language with dependent types(e.g. Coq) then, thanks to [Curry–Howard correspondence](https://en.wikipedia.org/wiki/Curry%E2%80%93Howard_correspondence),
theorems could be formulated and proven about model properties.

Both ways will be shown in action in next chapters!


Conclusion & Further Work
-------------------------

Data structures needed for model (Timestamp, Account, Transaction, Block, BlockChain, Node, Connection, System, BlockTree) are described
along with simplest functions over them. In next chapter forging algo functions will be defined, then executable imitation and
formal analysis will be provided.



