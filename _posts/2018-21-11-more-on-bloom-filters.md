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

Bloom filters (BF) is an extremely simple solution to this problem.
They have been widely adopted in network routers, web caches and packet inspection. Bloom filters are also at work in [Uber's M3 platform](https://eng.uber.com/m3/), [Blockchains](https://medium.com/@midnight426/bloom-filters-are-also-used-in-the-bitcoin-digital-currency-peer-to-peer-communication-d2e8aa124f68) and URL shornening service [Bitly](https://github.com/bitly/dablooms). In the past [Google Chrome](https://bugs.chromium.org/p/chromium/issues/detail?id=71832) used Bloom filters to detect malicious URLs but have moved on to use Prefix-Set.

# Bloom filter construction
Take a bit array $$B$$ of size $$m$$. Let $$H = \{h_1(x), h_2(x) ..., h_k(x)\}$$ be a family of k independent hash functions such that $$h(x)$$ is an integer in $$[0,m-1]$$. For every element in $$S$$, get k indices using all functions in $$H$$ and set those indices to 1 in our bit array. If a bit is already set to 1 by previous element, do nothing.

To check if an element $$y \in S$$, obtain k indices using $$\{h_1(y), h_2(y), .. h_k(y)\}$$.\\
$$y \in S$$ if and only if the bit is set to 1 at all k indices i.e

$$y \in S \iff B[h_i(y)]=1 \quad \forall i \in \{0,1,2,..k-1\}$$.

It is easy to see that this construction leaves the possibility for false positives. $$B[h_i(y)]$$ and $$B[h_j(y)]$$ may be set to 1 by set distinct elements $$x_m$$ and $$x_n$$. Probablity of false positive is given by

$$P_{fp} = \bigg( 1 - e^{kn/m}\bigg)^k$$

Derivation is straighforward and can be easily found online. [Wikipedia](https://en.wikipedia.org/wiki/Bloom_filter#Probability_of_false_positives)

# The problem: Part 2
Bloom filters perform well when the set $$S$$ is static. For many real world applications, this is not the case and $$S$$ changes over time with addition and deletion of elements, example routers monitoring network flows witness new connections being established and others getting closed. 
> Deletion of elements from the set $$S$$ may lead to **False Negatives** in a Bloom filter.

This has prompted researchers to come up with several solutions which allow delete operation in a BF. Three of these are discussed below.

# Counting Bloom Filter (CBF)


