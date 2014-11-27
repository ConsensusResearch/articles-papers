Inside a Proof-of-Stake Cryptocurrency part 3: A Local Ledger
=============================================================

Pre-Introduction: Consensus Research Group
------------------------------------------

After [previous article](http://chepurnoy.org/blog/2014/10/inside-a-proof-of-stake-cryptocurrency-part-2/) I started to work
with friend of mine, [andruiman](https://github.com/andruiman?tab=activity) on deeper investigation of proof-of-stake. We are
working now as Consensus Research group and have successful [Phase 1 crowdfunding almost done](https://trade.secureae.com/#5841059555983208287).

We have published [paper on multibranch forging](https://github.com/ConsensusResearch/articles-papers), while I continue
to describe a mode of 100% Proof-of-Stake single-branch forging.


Introduction
------------

Cryptocurrency ledger lives in p2p network and as any information shared in [AP](http://en.wikipedia.org/wiki/CAP_theorem) distributed environment it's a subject
 of inconsistency. So we will talk about local ledger as there's no global ledger in a network though algorithms behind
 a cryptocurrency try to build some common view under some assumptions.

Local Ledger
------------

In pt. 1 global network-wide structure System was defined slightly wrong for simplicity's sake:

    data System = System {
            nodes :: [Node],
            connections :: [Connection],
            accounts :: [Account]
        }

For now this simplification becomes too rude, so we're going to remove accounts and rename the structure
 to Network (`connections` definition also changed but just for performance's sake)

    data Network = Network {
        nodes :: [Node],
        connections :: Map Node [Node]
    }

New field to be injected into node to store it's local state is `localView`,

    data Node =
        Node {
            localView :: LocalView,
            unconfirmedTxs :: [Transaction],
            account :: Account          -- simplification - one account per node
            isForging :: Bool
        }

and consist of two entities:

    data LocalView =
        LocalView{
            nodeChain :: BlockChain,        -- simplification - one blockchain per node(but true for NRS)
            balances :: Map Account Int
        }

The latter entity containing balances for known accounts isn't strictly necessary but without it simple question
"what's the account balance now?" will need for whole local blockchain to be processed to answer.

To update `balances` during transaction processing we define 'applyTx' transformer function:

    applyTx :: Transaction -> Map Account Int -> Map Account Int
    applyTx tx blns = addMoney amt (recipient tx) $ addMoney (-amt) (sender tx) blns
        where amt = amount tx

where auxiliary function `addMoney` just adds some `diff` value to an account balance:

    addMoney ::  Int -> Account ->  Map Account Int -> Map Account Int
    addMoney diff acc blns = insert acc ((findWithDefault 0 acc blns) + diff) blns


To process a block, we process each transaction it contains then add sum of all transaction fees to a block generator balance:

    processBlock :: Block -> Map Account Int -> Map Account Int
    processBlock block priorBalances = appliedWithFees
        where
            txs = transactions block
            txApplied = foldl (\bs tx -> applyTx tx bs) priorBalances txs
            fees = sum(map fee txs)
            appliedWithFees = addMoney fees (generator block) txApplied



A Local Ledger & The Network Consensus
--------------------------------------

In a proof-of-stake environment chance to generate block & incoming block check result are depend on a balance of an account.
But information about balances is local, so all decisions are also made locally. E.g. If cluster of nodes have own
view of network, they will reject blocks from other nodes and build own blockchain. Is that good or bad? Well, the question is open,
as well as other consensus properties in proof-of-stake networks.


Further Work
------------

Next chapter will be about building blocks of an executable simulation of Nxt-like single-branch forging. Sources of
 the simulation tool will be published the same time.
