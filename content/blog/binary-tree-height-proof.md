+++
title = "Proof for height of a binary tree"
date = "2024-04-26"

#
# description is optional
#
# description = "An optional description for SEO. If not provided, an automatically created summary will be used."

tags = ["algorithms","math",]
+++

I keep forgetting the standard result of the height of a binary tree. 
I figured, writing out the proof on my own might help me remember the result. 

We assume that the tree is a full binary tree for simplicity. 
A full binary tree is defined as follows[1]:
> A binary tree in which each node has exactly zero or two children.

A binary tree starts with the root node at level one(we consider levels to be 1-indexed).  
It's two children form the next level.  
For any given level \\(i\\)(where \\(i > 0\\)), the number of nodes at the level \\(n_i\\) can be defined as:

$$
n_i =\left\{
\begin{array}{ll}
2n_{i-1} &\text{if }i>1 \\ 
1 &\text{if } i = 1.
\end{array} 
\right.
$$

For any given height \\(h\\), the total number of nodes in the tree \\(n\\) is given by:
$$
n = \sum_{1}^{h}n_i = 1 + 2 + 4 + ... + 2^{h-1}
$$
The number of nodes in a tree is given by a geometric progression with common ratio \\(2\\) and \\(h\\) number of terms.    
\\(n = 2^h - 1\\)  
Thus, \\(h = \log_2 {(n + 1)}\\). \\(\blacksquare\\) 


### References 
1. https://xlinux.nist.gov/dads/HTML/fullBinaryTree.html