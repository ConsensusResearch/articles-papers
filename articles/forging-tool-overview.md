
Forging Simulation Tool For a Nxt-like Proof-of-Stake Cryptocurrency
====================================================================

Introduction
------------

I'm the proud member of [Consensus Research](http://consensusresearch.org/). Recently we published two
executable simulations of forging(term for generating blocks activity in Proof-of-Stake). [Multibranch forging simulation](https://github.com/ConsensusResearch/MultiBranch)
has [the paper about it](https://github.com/ConsensusResearch/articles-papers/tree/master/multibranch). I'm going to
describe [Nxt-like single-branch forging simulation tool](https://github.com/ConsensusResearch/ForgingSimulation) in series of posts, some blogposts about multibranch will be
published later also to give simplified description of the tool.

The Tool
--------

The simulation tool is [published on GitHub](https://github.com/ConsensusResearch/ForgingSimulation) along with
short description. Some internal details described in the [Inside a Proof-of-Stake Cryptocurrency Part 4: The Executable Forging Simulation](http://chepurnoy.org/blog/2014/12/inside-a-proof-of-stake-cryptocurrency-part-4/) article.
I don't want to repeat myself here, so let's take a closer look on how the tool works and what results it dumps out.

After launching the program it prints out "Starting cryptocurrency simulation..." then does its job silently and dumps out
few results to the console when simulation is done:

![Alt text](http://chepurnoy.org/images/launcher.png "forging simulation tool screenshot")

So it dumps out a balance of every node from it's own balances sheet and common sub-chain length of a node with each
 other node. The screenshot above shows that most of nodes are agreeing on a chain with 121 or 122 blocks, while there's
 a cluster of nodes having 130 blocks in common chain and one node stuck in fork at the moment of the end of an experiment
 with 116 common blocks.

A lot of detailed data goes into output folder, readme describes files going there.

Forging Quality
---------------

Forging quality seems to be worse than in real-world implementations e.g. Nxt mainnet blockchain: big forgers
producing few blocks in a row more often(especially after connecting to a network with a genesis state), common chain is
behind the current moment. I see some reasons for that:

* in Nxt a node waits for blockchain to be downloaded then starts to forge, in simulation not
* there are no latencies, clocks de-synchronization, also all processes are tied to the world clocks,
so in case two nodes generate blocks at the same time a hard battle between forks has begun
* Nxt Reference Software has some additional checks and patches not being presenting in the model

But our goal wasn't to replicate Nxt forging and propagation processes, so it's okay for now to have what we have.



Performance
-----------

Performance is bad at the moment. And I did not optimizations for the reason: for now understandability is much more important.


Simulating Different Models
---------------------------

Using the simulation tool a researcher can make experiments and get results quickly. For example, to change cumulative
 difficulty measure from `sum of 1/baseTarget` to `sum of 1/(baseTarget^2` just one line modification needed:

  `cumulativeDifficulty chain = sum(map (\bl -> two64 / ((fromIntegral $ baseTarget bl)**2) ) chain)`

And from first look things are getting worse after that.

Further Improvements
--------------------

* Better output, e.g. CSV format usage to work with results in Excel or statistics tools
* Some aggregated metrics, e.g. number of forks living in a network, overall network consensus quality etc
* Latency modelling
* For now nodes stay online forever, better to model going offline at random moments
* Better performance

Usage
-----

* We already using the mix of this tool with multibranch forging simulation to play with Nothing-at-Stake issues(a paper
on that is ready and will be published soon)

* We can make experiments in very quickly (see "Simulating Different Models" section above)

* As result, community can compare different forging models without long debates, e.g. compare Nxt & Qora forging, or
see improvement proposals in action








