+++
title = "Arriving at Kadane's algorithm"
date = "2024-05-26"
draft = true

tags = ["study", "algorithms"]
+++

I found the explanation of Kadane's algorithm on Wikipedia[1] too dense for a normie like me. So, I asked myself, if the algorithm were to be lost to the sands of time, how would I arrive at it naturally?

Kadane's algorithm is a solution to the **maximum subarray problem**.

What is the **maximum subarray problem**? We take help from Wikipedia here.
> In [computer science](https://en.wikipedia.org/wiki/Computer_science "Computer science"), the **maximum sum subarray problem**, also known as the **maximum segment sum problem**, is the task of finding a contiguous subarray with the largest sum, within a given one-dimensional [array](https://en.wikipedia.org/wiki/Array_data_structure "Array data structure") A[1...n] of numbers.
> For example, for the array of values [−2, 1, −3, 4, −1, 2, 1, −5, 4], the contiguous subarray with the largest sum is [4, −1, 2, 1], with sum 6.

Try solving the [Leetcode problem](https://leetcode.com/problems/maximum-subarray/)[2] for this.

A brute-force solution would simply enumerate all subarrays and check their sum.
```cpp
#include <climits>
#include <vector>
int maxSubarraySum(const std::vector<int> & arr) {
	int max_sum = INT_MIN, subarr_sum;
        for(int i = 0; i < arr.size(); i++) {
            subarr_sum = 0;
            for(int j = i; j < arr.size(); j++) {
                subarr_sum += arr[j];
                if(subarr_sum > max_sum)
                    max_sum = subarr_sum;
            }
        }
	return max_sum;
}
```
The time complexity for this is \\(O(n^2)\\). Terrible, I know!
But, this gives us some insight into the problem.

**Intuition**: To calculate the contiguous subarray sum, we will need to calculate a running sum(`subarr_sum` in the example above).

**Observation**:
Consider the same example \\(A = [-6, 4, -2, -3]\\).
The subarray sum between \\(A[1..2] < A[2]\\).
When calculating the running sum, if the array element being considered happens
to be greater than the current running sum, we can conclude that current running
sum will not contribute towards the maximum subarray sum since the single element is already the maximum subarray sum so far.

To incorporate the observation, we reset the running sum to the current element each time `running_sum < arr[i]`.

The algorithm then becomes:
```cpp
#include <climits>
#include <vector>
int maxSubarraySum(const std::vector<int> & arr) {
	int max_sum = INT_MIN, running_sum = 0;
	for(int i = 0; i < arr.size(); i++) {
		if(running_sum + arr[i] < arr[i])
			running_sum = arr[i];
		else
			running_sum += arr[i];

		if(running_sum > max_sum)
			max_sum = running_sum;
	}
	return max_sum;
}
```
This is gets us to \\(O(n)\\) time!

Making the code a little cleaner:
```cpp
#include <climits>
#include <vector>
int maxSubarraySum(const std::vector<int> & arr) {
	int max_sum = INT_MIN, running_sum = 0;
	for(int i = 0; i < arr.size(); i++) {
		running_sum = max(arr[i], running_sum + arr[i]);
		max_sum = max(max_sum, running_sum);
	}
	return max_sum;
}
```
This makes it closer to the solution on Wikipedia.
Finally, I understand Kadane's algorithm. 

## References
1. https://en.wikipedia.org/wiki/Maximum_subarray_problem
2. https://leetcode.com/problems/maximum-subarray/
