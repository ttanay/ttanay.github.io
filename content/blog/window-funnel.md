+++
title = "How windowFunnel works in ClickHouse"
date = "2024-06-12T13:06:55+05:30"

tags = ["algorithms","clickhouse","study",]
+++

The `windowFunnel` function in ClickHouse has proved useful many times for me. I wondered how it actually works. I try to explain the algorithm it uses as well as why it works.


### Background
[Funnels](https://en.wikipedia.org/wiki/Funnel_analysis)[1] are a type of visualization that track how many users progress through a set of steps.
They are used to model customer journeys in different contexts like product, marketing, etc. Each step is visualized as a stage of a narrowing funnel. Hence, the name.

## Problem
Consider a dataset that contains the event stream of events performed by a user on a product.
The dataset contains the columns:

| Name         | Description                                                                                        |
| ------------ | -------------------------------------------------------------------------------------------------- |
| `user_id`    | The unique identifier for a user                                                                   |
| `event_name` | The name of the event performed by the user. Eg: "login", "signup", "add_to_cart", "checkout" etc. |
| `timestamp`  | The timestamp at which the action was performed(in epoch seconds).                                 |
Say for a given user, the event stream contains events the rows with columns `[event_name, timestamp]`:
```
[
	[0, "landing_page"],
	[1, "signup"],
	[2, "welcome"],
	[3, "product_page"],
	[4, "add_to_cart"],
	[5, "related"],
	[6, "product_page"],
	[7, "add_to_cart"],
	[8, "checkout"]
]
```

How would we figure out a breakdown of users who signed up on the website, added an item to cart and then checked out for purchase all within an hour?


### Formalizing the problem
To better understand the problem, let's express it more formally.

Let the events be described by a totally-ordered set \\(E\\) containing events \\(e \in E\\).
Let \\(timestamp\\) be a function that maps an event to its timestamp
$$timestamp(e): E \rightarrow \mathbb{Z}^{+}$$

Let the type of an event be given by a totally-ordered finite set \\(T\\) where \\(T \subset \mathbb{Z}^{+}\\)
Let \\(type\\) be a function that maps an event in the event stream to its type.
$$type(e): E \rightarrow T$$
Let the event stream \\(S\\) be defined as a sequence drawn from \\(E\\), such that \\(e_i \le e_j\\) where \\(i\\) and \\(j\\) are the ordinal numbers of the sequence. The binary relation \\(\le\\) on \\(E\\) is defined as:
$$e_i \le e_j \iff (timestamp(e_i) \le timestamp(e_j) \wedge (type(e_i) \le type(e_j)))$$
We are given a sequence \\(P\\) drawn from the set \\(T\\) of length \\(n\\) that defines that pattern of events that the funnel should match.
We are given a window \\(w\\) that defines the maximum difference of the first and last timestamps of a funnel.

A funnel can be thought of as a function that evaluates the length of the maximum match found in the event stream \\(S\\) for a given pattern \\(P\\) and window \\(w\\). 
Consider a sub-sequence \\(m\\) drawn from \\(S\\) -- \\(m = (e_1, ..., e_i, ..., e_k)\\). The sequence \\(m\\) is said to be a match of \\(P\\) iff the following hold:
1. \\(type(e_i) = p_i\\) where \\(p_i\\) is the \\(i\\)-th element of the sequence \\(P\\).
2. \\(timestamp(e_k) - timestamp(e_1) \le w\\) where \\(1 \le k \le n\\). 

Let \\(M\\) be the set of all possible matches such that \\(m \in M\\). Then, the level \\(L\\)  of a funnel is defined as the maximum length of all possible matches in \\(M\\).
$$L = max(|m| \mid m \in M) $$
## Solution
We can break down the problem a bit to move towards a solution.
We need the following:
1. Consider events of interest. Eg: `event_name IN ("signup", "add_to_cart", "checkout")`
2. Limit the timeframe/window. Eg: 1 hour in this case.
3. Consider the order of the events. Eg: `"signup" -> "add_to_cart" -> "checkout"`

The `windowFunnel` aggregate function does the job. For each user, it gives us the `level` up to which the user has converted.
It takes as input the following:
1. `window`: The timeframe/window in which the events should have been performed. It's unit is seconds.
2. `timestamp` column: The column to be considered as the timestamp column.
3. `condN`: The ordered conditions that make up the funnel. This makes up the pattern event chain.

So, in our case, this query would give us the desired result:
```sql
SELECT user_id,
	windowFunnel(3600)(timestamp, event_name = 'signup', event_name = 'add_to_cart', event_name = 'checkout') AS level
FROM table;
```
For a better example checkout the [documentation](https://clickhouse.com/docs/en/sql-reference/aggregate-functions/parametric-functions#windowfunnel)[2].

It also supports three options - `strict_order`, `strict_deduplication`, and `strict_increase`. But, for this post, we will limit ourselves to the default algorithm.

### The algorithm
Now that we know what [`windowFunnel`](https://github.com/ClickHouse/ClickHouse/blob/abb88e4d607fb927e2d444a3f5b1928d5dc0b962/src/AggregateFunctions/AggregateFunctionWindowFunnel.cpp)[3] can do, let's open the black box to see what lies inside.
Although `windowFunnel` supports multiple options, if we look at the default behavior, the pseudocode looks something like:
```cpp
using DateTime = std::uint64_t;
using EventIndex = std::uint8_t; // The event index as defined by event chain
std::uint8_t windowFunnel(std::uint64_t window, std::vector<DateTime, EventType> data, std::uint8_t events_chain_size) {
	std::vector<std::optional<DateTime>> events_timestamp(events_chain_size);
	for(size_t i = 0; i < data.size(); ++i) {
		auto [timestamp, event_idx] = data[i];
		if(event_idx == 0)
			events_timestamp[0] = timestamp;
		else if(events_timestamp[event_idx - 1].has_value() &&
			timestamp <= events_timestamp[event_idx - 1] + window) {
			events_timestamp[event_idx] = events_timestamp[event_idx - 1];
			if(event_idx == events_chain_size - 1)
				return events_chain_size;
		}
	}

	for(size_t i = events_chain_size; i > 0; --i) {
		if(events_timestamp[i - 1].has_value())
			return i;
	}
}
```
If we don't bother with types, it would look something like:
```python
def window_funnel(window, data, events_chain_size):
	events_timestamp = [-1] * events_chain_size
	for row in data:
		timestamp = row.timestamp
		event_idx = row.event_idx
		if event_idx == 0:
			events_timestamp[0] = timestamp
		elif events_timestamp[event_idx - 1] >= 0 and \
			timestamp <= events_timestamp[event_idx - 1] + window:
			events_timestamp[event_idx] = events_timestamp[event_idx - 1]
			if event_idx == events_chain_size - 1:
				return events_chain_size

	for i in range(events_chain_size, 0, -1):
		if events_timestamp[i - 1] >= 0:
			return i
```

Let's dig into why it works.

**Intuition**: While scanning through the events list, we evaluate potential matches that can exist w.r.t the sliding window, tracking them in the `events_timestamp` array.

It maintains an array - `events_timestamp`, of the same size as the pattern that needs to be matched. This array stores the start timestamp of possible matches in the event stream. When evaluating a new event for the funnel, it checks the following:
1. If it is the first event in the funnel's event pattern, a new window is opened by setting the `event_idx` of the `events_timestamp` array as the timestamp of the event.
2. For other events in the events chain, it checks if it has seen the predecessor event according to the funnel's event pattern. If it has seen its predecessor, it evaluates whether the event is within the defined the window by checking the difference of the start of the window stored in the `events_timestamp` of the predecessor event.
The `level` returned by the funnel is then evaluated using the index of the last non-empty value of the `events_timestamp` array.
In the `events_timestamp` array, it only keeps track of the timestamp of the start of the window because it relies on the fact that for an event for the same index of the funnel's pattern event chain, if there is an event with a higher timestamp, a successor event will be within the window even if evaluated w.r.t to the event with a higher timestamp.

The time complexity for this algorithm is \\(O(n)\\). But, since the event stream needs to be sorted by `(event_timestamp, event_type)` first, it is bounded to \\(O(\log n)\\).


## References
1. https://en.wikipedia.org/wiki/Funnel_analysis
2. https://clickhouse.com/docs/en/sql-reference/aggregate-functions/parametric-functions#windowfunnel
3. https://github.com/ClickHouse/ClickHouse/blob/abb88e4d607fb927e2d444a3f5b1928d5dc0b962/src/AggregateFunctions/AggregateFunctionWindowFunnel.cpp
