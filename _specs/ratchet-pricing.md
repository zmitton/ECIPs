---
ecip: <to be assigned>
title: Ratchet Pricing
author: Zac Mitton (@zmitton)
discussions-to: https://github.com/ethereumclassic/ECIPs/issues/148
status:  Draft
type: CONSENSUS
category: Core
created: 2019-10-24
replaces: eip-1559, ecip-1047
---

---

<h2>Abstract</h2>

---
<h3>Summary:</h3>

<p>A new gas market mechanism is proposed where:</p>

<ul>
  <li><a href="#max-spend-price">Senders specify the <i>maximum gas price they are willing to spend</i></a></li>
  <li><a href="#gas-price-earned">Miners are <i>paid</i> a flat gas-price when including transactions</a></li>
  <li>Senders are<a href="#exponential-charges"> <i>charged</i> exponentially </a>based on the<a href="#variable-block-size"> size of the block in which the tx is included</a></li>
  <li><a href="#recycling-burned-ether">The rest is <i>burned</i>, (more precisely recycled into future block rewards)</a></li>
</ul>

<h3>Results</h3>

<p>
  The mechanism can better prioritize important txs, ensure users pay for their approximate burden on the network (and no more), pay miners enough to justify computation and prioritzation of transactions (and no more), and allow flexibility in throughput during high-value periods, and provid mitigation against the known problems of a fee-only mining model. 
</p>

---

<h2>Motivation</h2>

---

<h3>Current Issues</h3>

<p>There are 3 issues with the current gas mechanism attempting to be solved or sufficiently mitigated</p>

<ol>
  <li>
    The UX Problem
    <ul>
      <li><p>The <b>confidence</b> issue of getting your tx confirmed</p>
      <p>Even when a sender selects a perfectly reasonable gas price, their tx can end up in limbo indefinitely. This happens whenever traffic increases (even by a little), because these prices <a href ="https://bitstein.org/blog/pierre-rochard-and-gavin-andresen-discuss-the-block-size-limit/"> tend to correlate exponentially with demand</a> and, tend to <a href="https://bitinfocharts.com/comparison/bitcoin-transactionfees.html">stay high for very long periods</a> when they do. Even after traffic dies down weeks or months later, the tx is likley to have been kicked out of mem-pools as too low-priority.</p>
      <p>This destroys confidence, and puts the user in a sort of limbo - even worse than having one outcome or the other. This proposal, if adopted, will restore senders' confidence in their txs being confirmed.</p></li>
      <li><p>The absence of a clear strategy to pick one's gas price</p>
      <p>For the current <code>gasPrice</code> property (set by tx sender), the goal is to find the minimum price that will be accepted; <a href="#gas-price-strategy">But doing so requires knowing what all others will do</a> (fundamentally unknowable).</p></li>
    </ul>
  </li>

  <li>
    The State Bloat Problem
    <ul>
      <li>The current mechanism ignores indirect <a href="#network-participants">network participant</a> costs (i.e. full-nodes)</li>
      <li>Not only is the state already large, it has <a href="https://twitter.com/ercwl/status/1159940020331040770">the potential to become much worse</a>, and will be impossible to reduce (in an immutable way) after-the-fact.</li>
    </ul>
  </li>
  <li>
    The known problem that <a href="http://randomwalker.info/publications/mining_CCS.pdf">fee-only mining</a> opens the door to Undercutting attacks / Selfish Mining <a href="#fee-only-mining">(see below)</a>
  </li>
</ol>

---

<h3>Incentive Requirements</h3>

<p>Although the tx-fee model creates beneficial incentives for both <i>senders</i> and <i>miners</i>, those incentives are <b>independent</b>, and can be decoupled through the use of burning.</p>

<div id="#desired-incentives">
  <ol>
    <li id="miner-incentive-prioritize">
      Miners must be incentivized to prioritize txs by <i>importance</i> (`maxSpendPrice`).
    </li>
    <li id="miner-incentive-max-spend-price">
      Miners must be incentivised such that the <a href="#the-uncling-problem">risk of including a tx (uncling)</a> is less then the fee earned.
    </li>
    <li id="sender-incentive-max-spend-price">
      Senders must be incentivised to <a href="#max-price-priority-dilemma">actually specify the <i>real</i> max price</a> that they are willing to spend. 
    </li>
    <li id="sender-disincentive-growth">
      Senders should be dis-incentivised such that the market sum of gas useage is less than the <a href="#ideal-blockchain-growthrate-rational"><i>ideal blockchain growthrate</i></a>.
    </li>
  </ol>
</div>

<p>The proposal below keeps or strengthens these incentives and adds new ones. For instance, in <a href="#miner-incentive-prioritize">[1]</a>. Miners currently can only prioritize by <code>gasPrice</code>, which is correlated much less with importance, due to <a href="#gas-price-strategy">pricing strategy problems</a> than <code>maxSpendPrice</code> described below.</p>

---

<div id="max-spend-price">

  <h3>Key Idea 1: Max Spend Price</h3>

  <p>Senders specify <i>the amount the transaction is worth to them</i> - their <code>maxSpendPrice</code>. Similar to <a href="https://medium.com/@avivzohar/rethinking-bitcoins-fee-market-ea9319e4ea57">Monopoly Pricing</a>, this can yield huge UX benefits.

  <p id="gas-price-strategy">For the current <code>gasPrice</code> property (set by tx sender), the goal is to find the minimum price that will be accepted; But doing so requires knowing what all others will do. (The theoretical best price is one such that there exist exactly enough higher priced transactions to fill an entire block minus the gas cost of yours. This dynamic does not have a <a href="https://en.wikipedia.org/wiki/Nash_equilibrium">Nash equilibrium</a> - i.e. you can always benefit from changing your price after knowing everyone else's; But changing yours, means that they again would want to change theirs, ad infinitum.)</p>

  <p id="max-spend-price-strategy"> Specifying a <code>maxSpendPrice</code> instead would eliminate the need for senders to do any market valuation <i>and</i> create a very high confidence that a given tx will get mined. This is due to most txs being "well worth it": There is usually a huge gap between the offer(<code>maxSpendPrice</code>) and the current market rates(<code>blockSpendPrice</code>). This can not be true for all txs of course, as some are bound to be right on the cusp where <code>maxSpendPrice</code> is roughly equal the current <code>blockGasPriceSpent()</code>. We think these txs however, will often be very low-value, non-finantial txs, likely from some automated proccess fair to be called <i>spam</i>.</p>
  
  <p>Take for example, a simple currency transfer of $20. I might set a <code>maxSpendPrice</code> of $1. Although I would hope not to pay that much, I would still much rather pay up to $1, then have the transaction completely fail. But since the market rate for a transfer (23,000 gas tx) would often only be 5 cents or less, it is extremely likely that the tx will get mined (market price would have to suddenly skyrocket over 2000% for it not to be). Compare this to the current system where an offer and the market price are generally within a much smaller margin of ~25%.

  <!--
    possible price grpah here
    (<a href="#price-graph-1 fixme">these results</a>).</p>
  -->

  <p>Under this system <i>if</i> your transaction does not get confirmed, then <i>you did not want it confirmed</i> because all txs that <i>got confirmed</i> in that time period would have paid a higher price then your <code>maxSpendPrice</code>, which means it would have cost more than it was worth to you.</p>

  <h4 id="max-price-priority-dilemma">Dilemma:</h4>

  <p>Although <a href="#miner-incentive-prioritize">[1]</a> getting miners to prioritize by amount seems simple, paying them the <code>maxSpendPrice</code> (or a fixed percentage of it), would directly dis-incentivize <a href="#sender-incentive-max-spend-price">[3]</a> senders from specifying their tx's true worth: If they know they will be charged more for specifying a higher <code>maxSpendPrice</code>, they will attempt to submit smaller values - degrading the system to the current one.</p>

  <p>Given the above hurdle, we must charge the sender a fee somewhat independent of <code>maxSpendPrice</code>; But then we need a new ways to <a href="#sender-disincentive-growth">[4]</a> dis-incentivize senders from spamming the network, and <a href="#miner-incentive-prioritize">[1]</a> incentivize miners to prioritize txs.</p>

</div>

---

<div id="gas-price-earned">
  <h3>Key Idea 2: Fixed Gas Price Earned</h3>

  <p>The miners don't necessarily need to be <i>paid</i> more just because the sender is willing to <i>pay</i> more (as is currently the case where the entire fee goes to the miner). Miners really only need to be paid enough to offset <a href="#the-uncling-problem">the cost of uncling</a>, and for the small overhead of prioritizing the transactions by <code>masxSpendPrice</code></p>

  <p>The best way to offset uncling is actually for miners to earn a gas price that is a fixed ratio of total block reward, because losing the block reward is precisely what is at risk by including more txs. This ensures that the incentive stays constant even as block rewards change.</p>

  <!-- 
  maybe put the following p in the appendix section under uncling
  -->

  <p>The risk of uncling is likely <a href="#the-uncling-problem">super-linear as gas used within a block increases</a>. This risk is also worse for smaller mining operations, giving bigger one a distinct advantage in expected uncling losses. This is reduced although not eliminated through use of the GHOST protocol. The only way to eliminate the disadvantage is to reduce uncling rates to near zero. A conservative <code>block-gas-limit</code> (or a defacto limit) will achieve this. </p>

  <p>We take the <code>gasPriceEarned()</code> from the tx sender who's <code>maxGasPrice</code> must be >= to it. It is constant ratio of the current <code>blockReward</code> (<code>blockReward / BLOCK_REWARD_TO_GAS_PRICE_EARNED_RATIO</code>).</p>

  <p>Still unsolved is the miner's incentive to order txs by <code>maxGasPrice</code> (why would they expend the effort). Without it, we have all tx treated equally, and supply shortages.
</div>

---

<div id="variable-block-size">
  <h3>Key Idea 3: Variable Block Size</h3>

  <p>A transaction's <i>cost to the network</i> is different than its <i>cost to the miner(s)</i>; The cost to the network is bloat - relevant to "overall <a href="#network-participants">network participants</a>" (including full-nodes), but the cost to the miner is an increased chance of uncling (or orphan). Although more transactions increases both (the concepts overlap), they are 2 separate things:</p>

  <ol>
    <li id="miner-incentive-1">Gas usage needs to be be given a strict limit <b>per block</b> to offset miner's <a href="#the-uncling-problem">uncling costs</a></li>
    <li id="miner-incentive-2">But it is only the <i>total amount of gas used in history</i> that is important to overall <a href="#network-participants">network participants</a>. Gas usage needs only to be below some <b>average</b> rate to offset network participant costs.</li>
    <!-- note there may be some repeated info here -->
  </ol>

  <p>So although a gas usage of 10 million might be reasonable for the <a href="#the-uncling-problem">uncling risk</a>, a lower <i>average</i> gas usage would surely be better for overall <a href="#network-participants">network participants</a>.</p>

  <p>If we can incentive blocks to have a low average (say 1 million), it is not a problem if <i>some</i> blocks are much higher (up to maybe 10 million).</p>

  <p>This fact is useful to combine with some of the above key concepts. Instead of a fixed block size, state bloat could be dis-incentivized by having <code>priceCharged</code> (to each tx in a given block) rise exponentially as <i>current traffic</i> (gas used within the given block) increases, (burning the extra so miners would get the fixed gas <code>gasPriceEarned()</code>).</p>

  <p>This satisfies miner incentive <a href="#miner-incentive-2">[2]</a>, because, picking higher <code>maxSpendPrice</code> txs would allow them to fit more txs in their block, thus linearly making more $ (note that this is easily sufficient for the very small amount of work for them to order the txs). Conveniently this eliminates the need for a <i>hard</i> block-gas-limit (Because the nature of exponential price increase would implicitly make one).</p>

  <p>This reasonably satisfies the requirement for senders to be incentivized to specify their real price, Because most txs will only pay it in times of very high traffic, and the amount they specify does nothing to change the current price <i>needed</i> to be included anyway.</p>
</div>

---

<div id="exponential-charges">
  <h3>Key Idea 4: Exponentially Increasing Gas Price Spend</h3>

  <p>The reason the price should raise <i>exponentially</i> with has not been satisfactorily justified. It has been observed, especially in Bitcoin from december 2017 (when <a href="https://bitinfocharts.com/comparison/bitcoin-transactionfees.html">fees skyrocketed above $20</a> and stayed there for months), that the market price of transactions often rises exponentially as demand approaches capacity. The mechanism for this was hypothesized years beforehand and <a href="https://bitstein.org/blog/pierre-rochard-and-gavin-andresen-discuss-the-block-size-limit/">explained quite well</a> by Pierre Rochard</p>

  <p>(Due to the current pricing mechanism in Bitcoin, its impossible to tell what the real demand is until that capacity is met. With this ECIP we will know the real demand (<code>maxSpendPrice</code>) at all times!</p>

  <p>By exponential increasing price with throughput it insures we hit a steep wall, limiting throughput, <i>despite</i> the exponential nature of demand. In fact, we can configure the increase such that <code>priceCharged * gasUsed</code> exceeds the total ETC supply when <code>gasUsed</code> is (for example) 10 million. This actually simplifies the design by removing a need for a hard cap, because its impossible to spend > the total ETC supply. In practice of course, the gas use will never get anywhere near that high (multiple senders would have to offer millions in gas each).</p>

  <p>This dynamic also allows us to make narrow conclusions of average gas use given extremely wide assumptions on price (demand).</p>

  <p>Its hard to describe the exact problems/solutions motivating each feature because some of the features fit together to solve multiple problems like a sort of puzzle piece: The idea that miners are paid a flat gas price also turned out to be imperative for this to work, otherwise miners would have incentive to withhold txs into sudden "bursts" of throughput in order to extract much higher average fees than flat throughput.</p>
</div>

---

<div id="recycling-burned-ether">
  <h3>Key Idea 5: Recycling Burned Ether to Extend Rewards</h3>

  <p>An earlier thought process had <code>gasPriceEarned</code> set as constant by the protocol. I later realized that it would have differing incentives as block rewards reduced, and that really what had to be made constant was the ratio of <code>blockReward</code> to <code>gasPriceEarned</code>. This simplified the reasoning such that the fee-only-mining problem could be attacked.

  <div id="fee-only-mining">
    <p>From the Princeton paper:</p>
    <p>"[in a tx-fee-only scenario] if there is a 1-block fork, it is more profitable for the next miner to break the tie by extending the block that leaves the most available transaction fees rather than the oldest-seen block. We call this strategy <code>PettyCompliant</code>. Once any non-zero fraction of miners is <code>PettyCompliant</code>, it enables various strategies that are more aggressive and harmful to Bitcoin consensus."</p> 
    <p> The summarize the more aggressive strategies <a href="https://www.cs.cornell.edu/~ie53/publications/btcProcFC.pdf">Selfish Mining</a> for instance, becomes easier and more profitable: When withholding a block, a miner can entice <code>PettyCompliant</code> miners to favor their block <b>even after another block is published</b> by leaving a few txs on the table.</p>
  </div>

  <p>With a small amendment: <b>"Recycling" the burned Ether back into the 210 Million capped supply (instead of straight burning it), by reducing the <code>blockReward</code> based on totalSupply instead of blockNumber</b>, it would allow the protocol to eventually reach an equilibrium where the amount being burned is approximately equal to the amount being rewarded. The block reward would not only always exists, but it would also <i>always be the majority</i> of the payment received for a given block, which would remove the incentive to be <code>PettyCompliant</code>, stopping the more aggressive strategies in their tracks</p>
</div>

---

<div id="specification">
  <h2>Specification</h2>

---

  <h4>Parameters</h4>
  <ul>
    <li><code>BLOCK_REWARD_TO_GAS_PRICE_EARNED_RATIO</code>: 100,000,000?</li>
    <li><code>GAS_AMOUUNT_PER_PRICE_DOUBLING</code>: 500,000?</li>
  </ul>

  <h4>Reduced Complexities</h4>

  <details>
    <summary>The block field <code>gasLimit</code> and the significantly complex voting mechanism surrounding it are no longer used.</summary>
    <p>This is a positive side-affect. There have been good reasons to remove the miner voting mechanism all along. It was a somewhat controversial "punt" in the first place, introduced as a way to mitigate a <a href="https://github.com/LeastAuthority/ethereum-analyses/blob/master/Appendix.md#summary-of-mitigations-made-in-response-to-our-recommendations">vulnerability found by Least Authority</a> (item 4); Allowing miners to affect the blockchain growth has been pointed out by Bitcoiners to be suboptimal and lead to insecure mining centralization.</p>
    <p>So in the case of Ethereum Classic: "Ethereum technology with Bitcoin philosophy" it makes a lot of sense to change that parameter as it solves 2 problems while eliminating complexity.</p>
  </details>

  <h4>Additional Complexities</h4>

  <details>
    <summary>In order to know the <code>priceCharged</code> the block must already be complete.</summary>
    <p>To pull this off, we must charge each sender their <i>full</i> <code>maxSpendPrice</code> during tx execution and then refund each one the difference once the block is full. This is roughly equivalent (in overhead) to updating half of a storage slot per tx.</p>
  </details>

  <h4>Proposal</h4>

  <pre><code>
    func blockGasPriceSpent(gasUsed) { 
      return (blockReward / BLOCK_REWARD_TO_GAS_PRICE_EARNED_RATIO) * 2**(gasUsed/GAS_AMOUUNT_PER_PRICE_DOUBLING)
    }

    func blockGasPriceEarned() { 
      return blockReward / BLOCK_REWARD_TO_GAS_PRICE_EARNED_RATIO
    }

    func txFeesEarned() {
      return blockGasPriceEarned() * blk.gasUsed
    }
  </code></pre>

  <ul>
    <li>Sender specifies the <code>maxSpendPrice</code> in the existing tx field <code>gasPrice</code> (index <code>1</code>).</li>
    <li>Txs MUST satisfy <code>tx.maxSpendPrice >= blockGasPriceSpent(tx.gasLimit)</code> to be valid</li>
    <li>During tx execution, sender is charged <code>maxSpendPrice</code> as they consume gas.</li>
    <li>The <code>lowestMaxSpendPrice</code> of the current block is tracked after each transactions and is REQUIRED to remain > the <code>blockGasPriceSpent()</code></li>
    <li>
      Before finalizing a block, each sender is refunded <code>(maxSpendPrice - blockGasPriceSpent(blk.gasUsed)) * tx.gasUsed</code>.
    </li>
    <li>Miners strategy would likely be to arrange txs by <code>maxSpendPrice</code> and execute them one by one until all remaining txs are such that <code>blockGasPriceSpent(blk.gasUsed+tx.gasLimit) > lowestMaxSpendPrice || blockGasPriceSpent(blk.gasUsed+tx.gasLimit) > tx.maxSpendPrice</code></li>
    <li>The <code>blk.gasLimit</code> field must now be <code>null</code></li>
  </ul>

  <h3 id="parameter-rational">Parameter bounds and Rationale:</h3>

  <ul>
    <li id="miner-earnings-rational"><b>Miners</b> "should get paid" to include txs: An amount that makes it economically advantageous <i>for the (reasonably assumed) smallest, least connected miner</i> to include the tx (given the added risk of "uncleing").</li>
    <li id="sender-payment-rational"><b>Senders</b> "should pay" a gas price such that the global demand for txs is less than or equal to the <b>ideal blockchain growthrate</b>.</li>
    <li id="ideal-blockchain-growthrate-rational">Because the only economic benefit of running a <b>full node</b> is the validation of one's own assets which could have arbitrarily small value, the <b>ideal blockchain growthrate</b> is such that the cost to run a full-node is <b>insignificant</b> for even them. It should be far lower than mental costs and other barriers on a reasonably low-end computer and reasonably low-end connection. i.e. If it were to cost > $100 to run a node, it would not make economic sense for anyone with < $100 to run one.</li>
  </ul>


  <p>Therefore constants should be chosen to create the following market conditions:</p>
  
  <ul>
    <li><code>blk.gasUsed > 10,000,000</code> less than 1% of the time (for centralized mining benefits to be ~ insignificant)</li>
    <li>When <code>blk.gasUsed >= 20,000,000</code> totalspent > 210,000,000ETC and is thus impossible (effective hard cap)</li>
    <li>When <code>blk.gasUsed < 500,000</code> <code>priceCharged ~= 20gWei</code> (so existing wallet software will continue to be affective during normal traffic, although wallets <b>should</b> upgrade the way they calculate gas price to a (much simpler!) equation such as <code>maxSpendPrice = (tx.value * 0.01 + 0.1e18)/23000)</code></li>
    <li>When <code>blk.gasUsed <= 5 million</code>, large price deterrents exist (i.e. tx cost is at least a few dollars). This will keep averages below 5 million (unless demand is so high that people are willing to pay 1000 times the current average, in which case I'd call that an emergency).</li>
  </ul>

</div>

<!-- fixme (should this stuff below go here though?) -->



<!-- 
This model should effectively squash near infinite demand (exponential), into linear blockchain growth by time (or constant growth rate). So, because this growth rate is both _reasonably small for current hardware_, and because hardware capability growth can be expected to continue _at least_ linearly at a rate faster than the constant blockchain growth rate ([no need to assume anything even near moore's law!]()), Therefore the economical cost to run a node _should get _only cheaper over time. That is, the current cost to run a node, is the most expensive it will ever be, and it is already _nearly free_. 

This is the conservative vision I believe will ensure the long term health of ETC (in at least the single metric of bloat). By erring on the conservative side here, the "scalability tradeoff" may be throughput reduction of maybe 5x. It really can not be stated loudly enough how little this actually changes about the nature of our intended interactions which will take place 99.99% on layer 2.
 -->
---

<div id="conclusion">
  <h3>Conclusion and Limitations</h3>
  <p>After all of the above research including several epiphanies that were made, there is an unfortunate unavoidable attack vector: An incentive for out-of-band bribery. Because the above system would have miners only earning a fraction of the amount paid by the sender, it would be possible for both parties to gain from coming to an agreement where the miner purposely includes less transactions (including the offending sender's tx), thus lowering the burn amount exponentially and then getting reimbursed by the sender out-of-band. This is because the miner can drop the cost to the sender <i>exponentially</i> while only dropping the cost to himself linearly (by including fewer txs than available).</p>
  <p>With rampant bribery, this system again degrades to the current one (but with a different blockchain growth rate). I believe the attack vector is high severety espessially when considering that the out-of-band bribery can easily be built into a contract, and thus be easily coordinated anonomously without trust.</p>
  <p>This attack exists in any system where</p>
  <ul>
    <li>A portion of the fee is burned</li>
    <li>A miner can affect the amount burned without equally affecting his amount earned</li>
  </ul>
  <p>A believe the severity of the above is sufficiently high to not adopt the proposal. Nonetheless there are a lot of valuable findings worth documenting here even though the proposal itself it deemed to have unsolved problems.</p>
  <p>There are however, interesting new directions being pursued from the learnings here which I continue to iterate on.</p>
  <!-- By having miners earning less than the amount being charged to the sender, there is negative property incentivizing out-of-band bribery. Because miners would only be earning a fraction of the amount paid by the sender, it would be possible for both parties to gain from coming to an agreement where the miner purposely includes less txs (therefore lowering the burn amount for the sender), and gets paid more than the difference by the sender. -->
  <!--
    <li>[EIP 1559](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1559.md)</li> 
    <li>[LeastAuthority](https://github.com/LeastAuthority/ethereum-analyses/blob/master/GasEcon.md#loose-bounds-of-the-current-gas-limit-system)</li>
  -->
</div>

---

<div id="appendix">
  <h2>Appendix:</h2>

---

  <div id="the-uncling-problem">
    <h3>The uncling problem</h3>
    <p>When a miner happens to mine a block, they must broadcast it to all the other miners as fast as possible. This requeres multiple hops. They broadcast it to all their dirrect peers, who then re-broadcast to all theirs - but not before validating the block. Therefore, the larger the block (and the more time it takes to validate it), the longer it will take for the network to receieve it. Once a miner recieves it, they will stop mining the last block, and start mining on top of this new one. while the block is being broadcast, other nodes might produce a block as well. If there block is broadcast faster, then the miners will mine on top of it insead, and the original miner will not end up recieving the full reward (block becomes an uncle or orphin). Because larger blocks take longer to propigate, they have increased risk of this happening (the risk likely grows faster than linearly with computation time). Thus miners are in a predicament, they can expect to make more money by including more txs, but if the block gets too big, they can expect less money due to increased uncle risk. The exact sweet spot differs based on how "well connected" the miner is. If they are direct peers to most of the big mining pool (or they are a mining pool), then they can probably include larger blocks and still have them propigate rather quickly. This creates a tendancy toward mining centrilization. The tendancy can be broken only by doing 2 things.</p>
    <ol> 
      <li>Sufficiently limiting the block size</li>
      <li>Putting a lower bound on the mining fee</li>
    </ol>
    <p>By setting those two parameters correctly, it can be made such that there is effectively no tradeoff at all. They simply fill their blocks until they are either full, or they have run out of available txs, because doing so, is _always_ more profitable then not (for even the _least connected_ miners).</p>
  </div>

  <div id="gas-price-stats">
    <h3>Gas Price Earned By Miners</h3>
    <p>The fee should be high enough so that even the smallest conceivable mining rig, would find it more profitable to include all available transactions, than to not. Since larger miners would benefit from lower numbers and thus one-by-one putting smaller operations out of business, it does not make snense to leave this parameter open to a vote, it should be a hard coded parameter. Also, since the chance of having a block "orphan" (or uncle) increases super-linearly with gas usage, we must use a value such that it is still worth including a tx <i>when the block is already at its fullest</i> (remember we will not have a defined block-gas-limit, but we will have a "de facto" one due to the super-linear <code>gasPricePaid</code> as a block increases). Lastly because "gas" is only rough approximation of the actual time needed to compute, we should again err on the side of safety by assuming the "worst case".</p>
  </div>

  <div id="gas-use-stats">
    <h3>Average Gas Usage</h3>
    <p id="network-participants">Since <i><b>network participants</b></i> (i.e. full nodes or anyone needing to validat data) can not profit from the fee system, the average gas usage should be low enough, such that anyone interested in running a full-node can do so without significant cost. Of course “significant” is a matter of debate but having users or miners affect this value is worse than developers, as their incentives are known to be opposed on this metric; However there is no way to get a sybil-proof opinion from network participants so as a consolation the engineers, having less of a conflict-of-interest (than senders or miners), decide.</p>
    <p>We might want blocks to be only 1 million gas on average, but thats a <i>full-node</i> constraint. Full nodes don't care if some blocks are huge as long as the average is low so the total rate of increase is reasonable. However for <i>miners</i> we might want a limit of 10 million gas for any single block. Above that and they will start to create uncles (which leads to centralization etc.). But its important to note that these are separate constraints. They exist for entirely different reasons but also aren't mutually exclusive. With exponentially ratcheting prices we get both the low average, and the high limit.</p>
  </div>
</div>
