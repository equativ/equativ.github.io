[[_TOC_]]

# Caching Serie 02 - Optimizing bounded cache eviction policies

## Introduction

Caching has been a hot research topic in computer science for decades and is still today, with many new papers on cache eviction policies published every year. Because of this, it can be difficult for a backend engineer to make an enlightened decision about which policy to use for a performance-critical use case.

One approach could be to arbitrarily implement a bunch of them, run some benchmarks with some production input data, and pick the one that performs the best. However, there is no guarantee that the efficiency of the chosen policy remains good after a subtle change affecting inputs. This is why we think it's crucial to learn a bit of theory, which we tried to address in the first article of the Equativ caching blog series. If you haven't seen it, please have a look before diving into this second part because we are going to reuse some technical jargon introduced in part 1.

In a research endeavor, in this article, we're going to put theory into practice by benchmarking various caching policies in different scenarios.

Let's dive in!

## Methodology

As stated in the introduction, we won't throw in some arbitrarily chosen eviction policies into a benchmark, but instead, start from a simple policy, study its pros and cons, propose improvements, and iterate until we think we have covered most issues.

### The framework

A benchmark ought to be rigorous to not take any biased conclusions.

Here are some important aspects:

* Frame the implementations so that we compare apples to apples.
    * **All implementations benchmarked must be of linear time complexity**. For instance, we won't benchmark a policy that must use a binary tree. Many implementations are doing some small tradeoffs to guarantee O(1) performance, so it would be unfair to make them compete with one that is O(logn).  
    * **The size constraint will be the number of entries**. It means that memory overhead per entry is not taken into account nor is the size taken by probatory entries (where only the key is retained, not the associated value).  
        We will make sure however we mention specificities that imply more memory usage. 
* Benchmark in a variety of scenarios
    * **Experiment with different cache sizes.**  
    * **Cover common access patterns.**  
* **Use cache efficiency** **as our performance indicator**, which is the percentage of cache hits among the total number of calls.

Finally, we use some "real world data" to see how good or bad our benchmark and reasoning were and to see if any of the new policies we can up with can compete with industry-proven ones.

### Generating input data

At the end of this article we will be using a few real-world datasets to compare the different implementations we came up with in "real world" cases, however, these datasets are generally a mix between different access patterns, making it difficult to conclude on the performance of some implementation against a specific access pattern. Instead, what we propose is to generate input data using what we called a **generator**.

Let's see what it looks like with some code.

A generator must implement the interface `IGenerator<T>` and implement `T Generate()`. An important implicit rule for a generator is to be deterministic, so that generated values when benchmarking different cache implementations are the same.

For our benchmarks, we'll be using `long` as our key type. Being an integer, it simplifies the process of generating keys that follow some mathematical rules. Also, 64 bits is enough for the spectrum we need, while still being relatively compact (we want to use as little memory as possible so that we can increase the number of iterations and even parallelize benchmark cases).

Two basic generators we can start with are:
* `SparseLongGenerator`: Generates random (but determinist using a seed) longs uniformly, given some min/max bounds
* `GuaussianLongGenerator`: Generates random (but determinist using a seed) longs given a set mean and standard deviation. We are using a Box-Muller transform to do this. Check out [this great article](https://medium.com/mti-technology/how-to-generate-gaussian-samples-3951f2203ab0) if you're interested in what it is.
    

Then we can implement some higher-level generators:
* `SwitchableGenerator<T>`: Swaps between several generators every N generated values. Useful for creating "true" scan access patterns for instance.
* `MultimodalGenerator<T>`: Randomly picks between several generators for every generated value. Useful for creating simple mixed access patterns like a bimodal Gaussian pattern for instance.
    

Here is an example: Let's say we have two Gaussian (also known as normal) distributions:
* A with a mean of 0 and a standard deviation of 1 (in blue on the graph)  
    `A = new GuaussianLongGenerator(µ: 0, σ: 1)`
* B with a mean of 12 and a standard deviation of 0.8 (in red on the graph)  
    `B = new GuaussianLongGenerator(µ: 12, σ: 0.8)`

Then we can generate a bimodal gaussian distribution (in green on the chart)  
`new MultimodalGenerator(A, B)`

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1678460731809/a815dcac-91b2-4295-99bc-f981bf2c8d8a.png)

With these different generators, we should be able to artificially reproduce some common access patterns in a comprehensive, deterministic, and scalable manner.

## Simple cache eviction with LRU

### Principle

The **LRU** (Least Recently Used) eviction policy is by far the most commonly used one because it is relatively simple to implement (compared to other policies) and performs well in many scenarios. As a quick drop-in, an LRU cache is better than no caching at all.

The idea is quite simple: whenever a key is requested, it is put on the top of a list. If the key was already present (cache it), it is moved from its current place in the list to the top of the list. When the cache is full, the policy is to evict the keys from the bottom of the least, which are the "Least Recently Used" keys.

### Implementation

Since we want to be able to remove from the bottom of that list, but also move any key to the top, we usually use a doubly-linked list for the O(1) performance on these operations. Thousands of implementations can be found online (this is even sometimes used as [an algorithmic exercise](https://leetcode.com/problems/lru-cache/solutions)).

For real-world usage, I'd recommend the one that is based on a hashmap (or dictionary in C#) and a doubly-linked list. On top of this, to avoid the memory fragmentation of a "classic" doubly linked list, I recommend [this optimized implementation](https://github.com/ogxd/optimized-linked-list).

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1678569500275/308a4d1a-9616-4909-b1ac-8ff5fb0d89ac.png)

Our implementation is made of two data structures:
* A dictionary, to be able to access any cache entry in linear time from its key.
* A doubly linked list, to store entries (with the actual entry value) **ordered by recency**.

### Pros/cons

Firstly, we can say that the LRU policy is quite simple to implement. But what about its performance?

Let's simulate such an access pattern. We're going to generate sparse integer keys (evenly distributed) within a range of 0-100,000.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1678571807440/6af165dc-6fd5-494a-b237-ebac1ab67cab.png)

What we can see is that when the cache has a maximum size of 50K entries, it has an efficiency of 50%, which is the maximum we could achieve in theory given that we are generating 100K uniformly scattered keys (100K keys / 50K cache size = 0.5 or 50%, which is what we observe). The same goes will all cache sizes, meaning that LRU is quite optimal in the scenario.

Let's complexify things a bit and throw in another 100K sparse integers but on a different range, and switch between the two ranges every 50K iterations. After the first 50K iterations, the keys would change. Thus, if the cache size is also 50K, in theory, the maximum efficiency we could have is 25%. Let's see how LRU performs in this scenario.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1678572469152/5d11ecf9-c579-4dd3-a48b-3185bfd9751f.png)

You can see it is doing great when the maximum cache size is 50K, at 23% efficiency which is close to the maximum we could in theory achieve.

Because it constantly moves accessed entries at the end of the list, entries that are not accessed are quickly evicted from the cache. It makes it very dynamic and efficient in scenarios where the hot entries (entries accessed often) change, such as this one.

But how well does it performs against a "scanning" access pattern? Given how easily it evicts entries, this is a legitimate question. Let's simulate such an access pattern by switching between 0-100K sparse integers and this time 100K-10M sparse integers. The second range is so large that is it very likely that every key requested in that range will be requested not more than once.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1678573412933/59b179e5-df05-4a9d-b12b-7d1bc4219435.png)

We can see that the cache efficiency drops when the cache maximum size is at 50K entries. That's because whenever we switch to the second range, every key, although it is requested only once, will push all first-range entries out of the cache, even though these were requested a lot more than once.

This is an issue because it can make LRU weak for complex and volatile access patterns and introduces a security vulnerability, where external factors can trigger a short scanning pattern and suddenly result in many keys being evicted from the cache, hindering its performance and the stability of the platform with it.

## Addressing scan vulnerability of LRU with SLRU and MSLRU

### Principle

Because the issue is that "hot" keys are evicted too quickly, a quick fix could be to bring in some entropy by splitting the list of entries into several segments. On a cache miss, we would insert the new entry and the end of the first segment, thus, it would require that key to be accessed multiple times to end up at the end of the last segments (as many times as we have segments).

We call this policy **SLRU** **for Segmented Least Recently Used**. This is what [MySQL uses internally](https://medium.com/@arpitbhayani/what-makes-mysql-lru-cache-scan-resistant-a73364f286d7).

### Implementation

Instead of having a single doubly-linked list to store entries ordered by recency, we use 2 doubly-linked lists. SLRU introduces the concept of midpoint to parameterize the size of the two segments. For instance, a midpoint of 0.2 means that the first segment would store 20% of the number of entries while the second segment would occupy 80%

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1678460631916/829a3fd3-74e9-4b12-9de5-0b6089c171f2.png)

A variant of SLRU would be **MSLRU for Multi-Segmented Least Recently Used**. Instead of using a midpoint, we would set several evenly spaced segments. An MSLRU with 2 segments would be the same as an SLRU with a midpoint of 0.5.

The benefit of MSLRU over SLRU is that it can be easier to parametrize and also bring in even more entropy (because we can have more than two LRU segments), making it even more scan resistant, at the cost of being less dynamic.

### Pros/cons

Let's use the same scenario as the one we used last on LRU:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1678574995619/61d0eeb7-d697-4d83-9e37-a91c045613f5.png)

This time you can see that we don't observe any drop in efficiency for SLRU and MSLRU. The MSLRU with 3 segments is the one that performs the best, with up to twice more cache hits compared to LRU.

Note however that SLRU with a midpoint of 0.1 performs worse than LRU. Here it begins to be a little tricky to understand why (probably some kind of threshold effect given iterations/cache size/ranges) but it shows also that introducing parametrization can be a problem.

## Generalization of (M)SLRU with LU

### Principle

By segmenting the LRU list in SLRU and MSLRU, we're indirectly starting to take into account the number of hits of an entry as an eviction criterion. We can generalize this idea by counting the hits count of an entry for real.

This kind of eviction policy is usually called LFU (Least Frequently Used). However, since we're counting hits and not measuring any frequency here (frequency is time-related), I think the name is incorrect, despite the term being widely spread as such ([Wikipedia](https://en.wikipedia.org/wiki/Least_frequently_used), [LFU on LeetCode](https://leetcode.com/problems/lfu-cache/), ...). For this reason, please allow me to call this policy **LU for Least Used**, which I think better emphasizes the counting aspect.

### Implementation

The <s>LFU</s> LU eviction policy is more difficult to implement than LRU. Very few implementations online are of linear time complexity and, when they are, they usually rely on some memory-inefficient data structure such as allocating a doubly linked list for each group of entries that share the name number of hits count.

As I've never been satisfied with any of the implementations I found, I came up with my own which I think is worth sharing.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1678460638992/a60c14e3-177f-4c1a-8618-2720f6a27cdc.png)

It consists in using two doubly linked lists:
* The first (in orange) stores entries ordered by hits count. The leftmost entry is the entry with the least hits count, while the rightmost entry has the most hits counts (likely it is requested a lot).  
    That is not enough however, because, in case of a cache hit, you need to increase the hits count of that entry by one, however, to keep the linked list sorted, you'll need to move that entry to the right, but not necessarily by one slot, but quite possibly more since there might be other entries with the same hits count on its right. That implies the usage of a for loop, which breaks our linear time complexity constraint.
* We introduce a second doubly linked list (in olive color) where each element represents a group of entries sharing the same hits count. Instead of referencing every entry of that group, it references the leftmost entry with that hits count in the first list plus the number of entries sharing that hits count. Thanks to this, when promoting an item, we can simply take the next hits count group and move that entry right before the first entry of that second group.

### Pros/cons

Let's use the same scenario as the one we used last on LRU and (M)SLRU:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1678615159459/6dc95798-3298-4a1d-abe7-a8f6a0d7f76b.png)

You can see how well LU performs against that access pattern that tends to "scan" entries. Not only it performs better than SLRU and MSLRU in that scenario, but it also does it while being parametrization free.

However, increasing counters come with two majors downsides:
* The first downside is the inherent limitations of integers. While it works in theory, in practice **the integer will overflow** at some timepoint, like a timebomb. You can get around this with wider integers but ultimately the problem is still there and using more bits to store counters means more memory usage.
* The second downside is **an aging problem** inherent to the usage of counters:  
    Consider an entry in memory that is accessed a lot and thus as a lot of hits counts in LU. If suddenly this entry stops being accessed, that entry will remain in the cache even though it is now useless, until other entries reach higher hit counts and push that first entry to the left until it gets evicted.

We can easily simulate a pattern where the second issue would arise. Let's warm up a cache with sparse keys in the range 0-100K for many iterations (warmup) and then suddenly change the range to 100K-200K and start measuring:

![Results will vary depending on the number of warmup and benchmark iterations](https://cdn.hashnode.com/res/hashnode/image/upload/v1678657053944/d685568a-6baa-4879-9a67-05bff18540a0.png)

*Results will vary depending on the number of warm-up and benchmark iterations*

You can see LU is completely flat here, as it is full of entries with high hits counts from the warmup, while not being accessed anymore after the warmup, preventing entries from the new range to get a place in the cache. For comparison, LRU quickly adapts and is quite efficient.

## Addressing LU's aging problem with LUDA

### Principle

We introduce the concept of "Dynamic Aging" to avoid a range of hits counts that widens over time. We'll call this policy **LUDA for Least Used with Dynamic Aging**.

Instead of assigning new entries a hits count of 1, one solution is to assign an existing hits count, preferably not the lowest one but the one after so that it allows low hits count buckets to slowly get evicted over time.

As it's getting tricky to explain, an example might ease things:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1678660674430/57e5f56b-58ba-462f-9f17-2fadcdaea3b3.png)

You can see that when inserting F into the cache, with LUDA we assign it the hits counts from the second to last entry which was C. Thus F gets a hits count of F. Thanks to this, if F were to suddenly be a hot key, it wouldn't need 100 hits to catch up with A and get his way to the front of the list.

### Implementation

Implementation is very similar to the LU implementation we proposed above. The only difference is what hits count bucket we assign to a new entry.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1678460647453/a6356f18-de6f-4f1b-ab3c-038057542dec.png)

### Pros/cons

Here we are reusing the same setup as when highlighted LU's aging weakness above.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1678657142768/ecbde523-3e47-47fc-bd19-3c4ee0750ab7.png)

On this run, we can see that LUDA does not suffer from LU's aging problem and even does almost as well as LRU, but slightly worse at higher cache sizes because there are still some old keys to evict and it takes more iterations to get them out compared to LRU which is inherently very dynamic. With more benchmark iterations, we can expect it to perform as good as LRU.

The integer overflow on counters however remains an issue. Let's see how we can address that.

## Preventing count overflow with LFU and LFURA

### Principle

We previously said that LFU was misnamed because, in the literature, it was not about frequency but rather counting hits. We assumed this and decided to call it LU instead. However, what if we implemented a "real" LFU? Meaning that we would compute frequencies in the physical sense.

We'll call this policy **LFU for Least Frequently Used**, but remember this is different from the LFU you'll probably encounter outside of this article.

By using frequencies instead of counting hits, we shouldn't be worried about integer overflow. However, in order not to lose the benefits of dynamic aging we introduced above since LFU is going to suffer from a similar aging problem (a key with a high frequency is likely to remain in the cache even if it suddenly stops being accessed).

To fix the aging problem in the context of LFU, we propose to mix in an LRU to get a notion of recency for each key. Thanks to this, we should be able to detect keys with high frequency but low recency, and actualize their frequencies (we can for instance arbitrarily decrease its frequency, until after some cycles it finally gets evicted). We'll call this policy **LFURA for Least Frequently Used Recency Actualized**.

### Implementation

LFU implementation looks very similar to LU. The main difference is that we'll store buckets of frequencies instead of counts.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1678460653359/20c4c7e6-7bae-45d1-b199-e5f7e214a489.png)

Computing a frequency is less trivial than incrementing a counter. Here we'll throw in some heuristics which we think to work well.

What we do in our LFU implementation is store the last accessed timestamp of each entry, and compute the frequency whenever it gets accessed a second time. The more a key is accessed, the more accurate will be the computed frequency.

Since a frequency is a floating value and we still want to benefit from our buckets structure from LU, we chose to transform frequencies to `log10`. Also, because with frequencies we are losing the incremental behavior of counters, we chose to promote/un-promote an entry frequency onto the next or the previous frequency instead of jumping directly to the buckets that match the newly computed frequency (because then it can't be O(1) anymore).

LFURA implementation is the same as LFU, expected that it has an LRU segment on top of it to track recency.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1678460658232/0ff75f3d-10dc-4692-b0b6-0e30366e018b.png)

todo

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1678466993674/c35cb39b-90fc-45a6-b9eb-a5d8d4a75e91.png)

LFURA vs LUDA

LFURA is sloppier because it takes more time to evict keys that used to be very hot. LUDA is more "neutral". But benefit of that for LFURA is that if the pattern is rare it will be doing better than LUDA on the long run.

## Improving tracking efficiency with PLUDA and PLFURA

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1678467918047/1b609a1b-f4a7-41ff-b517-2e618b365065.png)

LUDA and LFURA are addressing the "stuck" issues but as you can see we lost some in this case

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1678460684841/16574488-8f30-4f7c-878d-db2b44b2cb4c.png)

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1678468100647/888ab5ae-5616-4812-bbed-e630f40c5115.png)

## Other solutions (ARC, LIRS, ...)

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1678460691968/cd5c259f-8bdd-4a72-8d6b-53c7a0fdc7cb.png)

## Wrap up

### Benchmark with some real data

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1678468746866/006f1d4a-9c28-4667-8cf2-882ebd00e8c7.png)

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1678468762675/8b8034b2-deae-4049-ab6c-a32ce0777a43.png)

### Conclusion

---

1. Footnote content here.
    

---

Source code