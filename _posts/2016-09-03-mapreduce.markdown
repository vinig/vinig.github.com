---
layout:     post
title:      "MapReduce"
subtitle:   "research paper by Jeff Deans and Sanjay Ghemawat, explained."
date:       2016-09-03
author:     "me"
header-img: "img/post-bg-02.jpg"
---

<p>I'm taking an advance distributed systems' class this fall by Dr. Wyatt Lloyd. This research paper <b>MapReduce: Simplified Data Processing on Large Clusters</b> was recently discussed. I had read this paper before when I first started working on the Big Data project in my previous organization but never gave it a much deeper thought. This time when it was discussed in the class, I discovered so many new angles to the problem that were all hidden in this paper but never caught my eye. I'd like to share some of these insights here.</p>

<h2 class="section-heading">What's the problem that this paper is trying to address ?</h2>
<p>If you have read the paper, it talks in depth about how to partition the data such that the results of the mapper workers are evenly distributed for the reducer workers to function efficiently. But the major contribution is the abstraction over the ugly details of parallelization, fault-tolerance, data distribution and load balancing, without loosing the generality of the code, performance on large clusters and re-usability. So the problem really was that there wasn't a programming model available that had more general application, would parallelize execution without making the code complex and can be easily used by everyone. And yes ! it's a real problem. Google is a company that indexes the internet, so we can only imagine the amount of data that it crunches each day.<p>

<h2 class="section-heading">Key insights</h2>
<ul>
	<li><b>Any program can be divided into map and reduce functions.</b> But what would be the worst program ? The one that produces just one key for all the partitions and then everything gets picked up by one reduce function. It would defeat the purpose of self load-balancing design. An interesting problem would be, to find the top k-words in a given set of books.</li>
	<li><b>Simple interface</b> where map/reduce function definitions are required and jobs can be run by anyone.</li>
	<li><b>Network bandwidth is saved</b> because of the locally written intermediate files. Only the metadata is sent to the master for book-keeping.</li>
	<li><b>Failures are inevitable.</b> Even if the probability of failure is very low, say 0.001 but because of the large cluster of machines being used, say 1000, then according to the probability, 1 machine in 1000 will fail. So, the paper talks about awesome techniques they implemented to make sure that data being crunched at a failing worker, gets redone. It is redistributed to the other workers. Incase of bad records, they are flagged so that they aren't processed by any other worker and save time.</li>
	<li><b>Stragglers exist.</b> The most common reason could be the slow I/O where the map task is writing to the intermediate file system and it's taking long on that worker.</li>
</ul>

<h2 class="section-heading">It's Read-Map-Combine-Shuffle-Sort-Reduce-Write !</h2>
<p>The paper discusses the extension to the program that'll help in improving efficiency of map/reduce tasks.
	<ul>
		<li><b>Combiner Function.</b> It is executed on each machine that performs a map task. It solves the problem of repetitive keys produced by each map task. A good example is a word-count program, where the most common word "The" is emitted multiple times by the same map task as <the, 1>. Instead of sending all the records to the reducer, the combiner function would add them up and emit just one record per map task per partition. Other such combined records from other partitions, will then be sent to the reducer and it'll come up with the final count for "The". Since the combiner function works on the local data, it saves a lot of network bandwidth.</li>
		<li><b>Shuffling Function.</b> This stage deals with the output produced by different mappers. The rows with the same reduce key are grouped together into a sequence by merge-sort.</li>
		<li><b>Sorting Function.</b> This is an important step after the shuffle because its output is picked by the reducer workers. To guarantee the order of keys, they are sorted such that a reducer will pick up, say second partition, from each input system.</li>
	</ul>
</p>

<h2 class="section-heading">Determinism of functions</h2>
<p>The paper makes a compelling case on why these functions should be deterministic and how the non-deterministic nature makes the semantics weaker.
	<ul>
		<li><b>Side effects.</b> MapReduce functions are known to emit auxillary files as additional outputs. Such side-effects should be atomic and idempotent, so that incase of failure when these functions are re-run, the same output is generated everytime, for the same set of data.</li>
		<li><b>Skipping Bad records.</b> The records that cause deterministic failures, can be flagged and made to skip in order to make forward progress.</li>
		<li><b>Non-deterministic Mapper.</b> It may or may not affect the reducer functions but can create troubles in MapReduce chains. Consider an example of two non-deterministic map tasks, M1 and M2, that are doing some computation which has a factor, say M1 = 3 and M2 = 12. If one map worker fails, the data is redistributed to other workers. Say M1 dies and all of its data goes to M2. Then the result by M2 for that data will have 4 times larger output than the original M1 data. That may not affect the immideate reducer function because after the failed M1 task, it's impact is null on the system, as if it never existed. But incase it comes back up and we are doing a chaining or re-run of the tasks, the outputs are going to be different, whereas we expect them to be idempotent.</li>
		<li><b>Non-deterministic Reducer.</b> A commutative reducer is a pure function and will produce the same result, for any sequence of input rows and any of its permutation. The paper on <a href="https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/icsecomp14seip-seipid15-p.pdf"><b>Nondeterminism in MapReduce Considered Harmful?</b></a> discusses different scenarios and its implications. In their investigation, they found over half of user-defined reducers (58%) were non-commutative. Many of them were found in well-tested recurring jobs. But it's unnecessarily restrictive to mark all non-commutative reducers as incorrect. It may not affect the program if the overall output is deterministic, as the intermediate outputs could be inputs to a MapReduce chain. The non-commutativity can go unnoticed for a long time, it can either be determined by pattern matching or checking implicit properties on real data as the programmer's assumptions could be inconsistent with the data. One of the reasons for the non-commutativity was data-shuffling as it introduces uncertainity in group row order. For some reducers, the divergence of input order doesn't matter but some are sensitive.</li>
	</ul>
</p>

<p>So, this was the collective summary and review of the paper. I might have miss out on some points and I'll add them when I find out. But it'll give you a good headstart on what points to read in depth and how to evaluate the paper. There are so many supplemental papers and google scholar citations of this paper, you can read them to understand certain points better. Hope this helps!
</p>

