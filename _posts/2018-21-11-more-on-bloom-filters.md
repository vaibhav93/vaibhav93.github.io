---
layout: post
title:  More on Bloom filters
date:   2018-11-21 16:40:16
description: Algorithms for Data Science
---
# The problem
Given a large set of elements, how can we answer queries about set membership in
- Constant time
- Small space
- Small margin for false positives

Let $$S = \{x_1,x_2, x_3,...,x_n\}$$ be a set in universe $$U$$ on which we which to answer queries of the form *Does* $$y \in S$$ ? 

Bloom filter (BF) is an extremely simple solution to this problem.
They have been widely adopted in network routers, web caches and packet inspection. Bloom filters are also at work in [Uber's M3 platform](https://eng.uber.com/m3/){:target="_blank"}, [Blockchains](https://medium.com/@midnight426/bloom-filters-are-also-used-in-the-bitcoin-digital-currency-peer-to-peer-communication-d2e8aa124f68){:target="_blank"} and URL shortning service [Bitly](https://github.com/bitly/dablooms){:target="_blank"}. In the past [Google Chrome](https://bugs.chromium.org/p/chromium/issues/detail?id=71832){:target="_blank"} used Bloom filters to detect malicious URLs but have now moved on to use Prefix-Set.

# Solution: Standard Bloom filter (SBF)
Take a bit array $$B$$ of size $$m$$. Let $$H = \{h_1(x), h_2(x) ..., h_k(x)\}$$ be a family of k independent hash functions such that $$h_i(x)$$ is an integer in $$[0,m-1]$$. For every element in $$S$$, get k indices using all functions in $$H$$ and set those indices to 1 in our bit array. If a bit is already set to 1 by previous element, do nothing.

To check if an element $$y \in S$$, obtain k indices using $$\{h_1(y), h_2(y), .. h_k(y)\}$$.\\
$$y \in S$$ if and only if the bit is set to 1 at all k indices i.e

$$y \in S \iff B[h_i(y)]=1 \quad \forall i \in \{0,1,2,..k-1\}$$.

It is easy to see that this construction leaves the possibility for false positives. $$B[h_i(y)]$$ and $$B[h_j(y)]$$ may be set to 1 by distinct elements $$x_m$$ and $$x_n$$. Probablity of false positive is given by

$$P_{fp} = \bigg( 1 - e^{-kn/m}\bigg)^k$$

Derivation is straighforward and can be easily found online. [Wikipedia](https://en.wikipedia.org/wiki/Bloom_filter#Probability_of_false_positives){:target="_blank"}

> Interesting Fact: Instead of k hash function, we can start with two independent hash functions $$h_1(x)$$ and $$h_2(x)$$ and generate other hash functions using $$g_l(x) = h_1(x) + ih_2(x)$$ without any loss in asymptotic false positive probability. More details here [[5]](https://www.eecs.harvard.edu/~michaelm/postscripts/rsa2008.pdf){:target="_blank"}

# Problem with SBF
Bloom filters perform well when the set $$S$$ is static. For many real world applications, this is not the case and $$S$$ changes over time with addition and deletion of elements, example routers monitoring network flows witness new connections being established and others getting closed. 
> Deletion of elements from the set $$S$$ may lead to **False Negatives** in a Standard Bloom filter.

The reason is self evident. If we set $$B[h_i(x')]=0$$ for $$i=\{1,2,...,k\}$$ while deleting $$x'$$, we may also be setting $$B[h_m(x'')]=0$$ where $$h_m(x'')=h_i(x')$$ for some $$i$$. Subsequent membership queries for $$x''$$ will wrongly output *false*. This has prompted researchers to come up with several solutions which allow delete operation in a BF. Three of these are discussed below.

# Solution: Counting Bloom Filter (CBF)
We can make a small change to bit array $$B$$ to overcome the problem of false negatives on deletion. Instead of creating a bit array, each cell becomes an *n* bit counter. When $$x$$ gets hashed to a cell, we increment the counter by 1. For deletion, each $$B[h_i(x)]$$ is decremented by 1. A query $$y$$ is believed to be in the set if all $$B[h_i(y)]$$ are greater than zero. Probability of false positives remain the same as a SBF. The size of counter is usually 4 bits. The probability of overflow with 4 bits is extremely low [1]

$$P(max(c)>16) \leq 1.37\times 10^{-15} \times m$$

# Problem with CBF
Each cell in a CBF takes up 4 bits even though most of the values are zero. We can see that a CBF has poor utilization of space and reaches atleast 4x the size of a SBF.

# Solution: d-left Counting Bloom Filter (dl-CBF)
d-left hashing is an "almost perfect" dynamic hash function [2]. We need a dynamic hash function because elements can be added and deleted from our set $$S$$ and a static hash function does not perform well for a changing set.

Steps for dl-CBF construction:
1. Take a hash table with $$n \times d$$ buckets. Each cell in a bucket stores a fingerprint and count.
2. Divide the hash table into $$d$$ subtables. Each subtable has $$n$$ buckets.
3. For each element $$X_i$$ in the set, use hash function of the form $$H: U \rightarrow [B]^d \times R$$
   1.  Let $$ H(X_i) = \{b_1, b_2, ..b_d\} \times r$$
	   1. $$b_i \in B$$ denote bucket index where $$B = \{0,1,2,...n-1\}$$.
	   2. $$r \in R$$ is the fingerprint of size $$k$$ bits.
   4. Search $$d$$ buckets to see if $$r$$ exists in any bucket. If yes, increment the count of that cell.
   5. If the fingerprint $$r$$ is not found, select the least loaded bucket from our $$d$$ choices. If two or more buckets have the same load, select the left most bucket to add the fingerprint.
   
> Always-go-left is a biased choice in case of clash. This may seem counter-intuitive to our objective of balancing load uniformly across buckets. Curious readers can read this [[2]](https://dl.acm.org/citation.cfm?id=792546){: target="_blank"} interesting paper.

![cbf](/img/aww-board.png){: width="100%"}
*{1011} is fingerprint of $$X1$$ which is put into the first bucket since it was the left-most least loaded*

To check membership of an element $$y$$, get its target $$\{b_1,b_2,..b_d\} \times r_y$$ using the hash function and search $$d$$ buckets parallely. If $$r_y$$ is found, we say $$y \in S$$\\
To delete an element, we follow similar procedure to CBF. Use the hash function to find the bucket which holds $$r_y$$ and decrement the count.

**Comparision with CBF**:
Since elements are finally hashed to $$log_2(n) \times k$$ bits (after choosing the bucket) instead of a single index in CBF, the probability of collisions in d-left hashing is very low (upper bounded by $$2^{-log_2(n)\times k}$$). This allows us to have **smaller 2 bit counters** as compared to the 4 bit counters necessary in CBF to prevent overflow. Experiments by Bonomi et.al [1] reveal that
1. dl-CBF of the same size as CBF achieves 100 times smaller false positive probability.
2. For the same false positive probability, dl-CBF utilizes half the space of a conventional CBF.

**Probability of False positive**

$$ P_{fp} = 1 - \bigg( 1 - \frac{1}{nd \times |R|} \bigg)^{|S|} \approx \frac{m}{nd \times |R|} $$

Result is easy to derive. Refer to [1] for more details.

# Cuckoo Filter (CF)
A Cuckoo filter is based on Cuckoo Hashing. It is a simple yet powerful algorithm which can achieve upto 95% utilization of the hash table. It it 3-4 times more space efficient than a CBF and upto 2 times more space efficient than a  dl-CBF.\\
**Cuckoo Hashing**: We start off with a hash table with n buckets. For every element to be added to the hash table, we have two buckets to choose from. These two buckets are dervied from two hash functions $$h_1(x)$$ and $$h_2(x)$$. If either of the two buckets is empty, we insert $$x$$ into that empty bucket. If both are occupied, choose either one and move its element to its alternative bucket in the table.
![cuckoo](/img/cuckoo.png){: width="100%"}
*An illustration of Cuckoo hashing. $$x$$ is hashed to the buckets containing $$a$$ and $$b$$. We move $$a$$ to its alternate position to accomodate $$x$$. At $$a$$'s alternate position, $$c$$ is already present. $$c$$ is moved to its alternate position to make space for $$a$$. Source [4]*

Some changes to the standard Cuckoo Hashing technique are required to make it work as a set membership datastructure. Instead of storing the element as it is in the hash table, we store fixed length fingerprints. This helps us save space but also leads to another issue. In the image above, when $$x$$ wants to take the place of $$a$$, we need to calculate the alternate bucket for $$a$$. This means we need access to the element $$a$$ so that we can calculate its alternative position $$h_1(a)$$ or $$h_2(a)$$. Since we are only storing fingerprints, we need a way to calculate the new bucket index without access to the original element itself. This is accomplished by a simple idea. Two candidate buckets are calculated as following:

$$h_1(x) = hash(x) \qquad \quad h_2(x) = h_1(x) \oplus hash(fingerprint(x))$$

If we have to move an element in bucket $$i$$, we can calculate its new position with the help of *XOR* operation $$ j = i \oplus hash(fingerprint(x))$$

There is a trade off between table occupancy and size of fingerprint. If we want higher table occupancy, we have to increase size of fingerprints to maintain the same probability of false positives. We need larger fingerprints to reduce probability of collisions.
![cuckoo_algo](/img/cuckoo_algo.png){: width="100%"}
*Algorithms to insert, lookup and delete elements from a Cuckoo filter. It is relatively simple once we grasp Cuckoo hashing and the 2 modifications discussed above. Source [4]*

# Summary
We looked at 3 different variants of the Standard Bloom Filter (SBF) which handle deletion of elements. Counting Bloom Filter (CBF) is extremely simple in construction but it has very poor utilization of space. With some added complexity, d-left Counting Bloom Filter (dl-CBF) improves upon CBF by using d-left hashing and storing element fingerprints to save space and reduce probability of false positive. Finally, Cuckoo Filter goes further in saving space by getting rid of counters. There are alse other variants of Bloom Filter such as Bloomier Filter or Quotient Filter and it is upto the user to evaluate which is best suited for their application. 

# References
[1] F. Bonomi, M. Mitzenmacher, R. Panigrahy, S. Singh, G. Varghese, *"An improved construction for counting bloom filters"*, Proc. ESA'06, pp. 684-695.\\
[2] Vöcking, Berthold. *“How asymmetry helps load balancing.”* J. ACM 50 (2003): 568-589.\\
[3] Rasmus Pagha, Flemming FricheRodler *"Cuckoo Hashing"*, Journal of Algorithms, Volume 51, Issue 2, May 2004, Pages 122-144\\
[4] Fan, . Cuckoo Filter: *"Practically Better Than Bloom"*. Proceedings of the 10th ACM International on Conference on emerging Networking Experiments and Technologies
Pages 75-88\\
[5] Kirsch A., Mitzenmacher M. (2006) *Less Hashing, Same Performance: Building a Better Bloom Filter*. In: Azar Y., Erlebach T. (eds) Algorithms – ESA 2006. ESA 2006. Lecture Notes in Computer Science, vol 4168. Springer, Berlin, Heidelberg
