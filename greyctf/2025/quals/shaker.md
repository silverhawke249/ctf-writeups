# Shaker

# The problem

Reverse the flag from a black box that alters the flag with each shake. We may only shake the black
box no more than 200 times.

# The observation

We should deconstruct how the shaker works first.

On initialization, it sets the state $S$ to the flag's value, which we can consider as a vector of
64 bytes. It then assigns random values to another vector of 64 bytes, $X$. Finally, it
generates some permutation $\pi$ that permutes $[0, 1, 2, \dots, 63]$.

The shaker defines several operations:
* Permute: apply $\pi$ on $S$. i.e., if $S = [s_0, s_1, \dots, s_{63}]$, then after this
  operation, $S = [s_{\pi(0)}, s_{\pi(1)}, \dots, s_{\pi(63)}]$.
* XOR: replace $S$ with the value of $S \oplus X = [s_0 \oplus x_0, s_1 \oplus x_1, \dots, s_{63} \oplus x_{63}]$, where $\oplus$ denotes the XOR operation.
* Shake: perform XOR, and then permute.
* Reset: assign a new permutation to $\pi$, and then shake.
* Open: perform XOR, and then return $S$.

The user can either (1) shake, which... performs a shake on the shaker, or (2) see inside, which
performs an open and a reset, in that order.

A key insight for this challenge is the realization that most permutations generate small orbits
of elements -- permutations that cycle the elements around in big orbits are rare, and if they
exist, they tend to force smaller cycles to exist alongside them. Additionally, since $X$ is
constant, getting two XORs at the same position would cancel it out.

Suppose that on successive applications of $\pi$, we obtain the cycle $0 \rightarrow 5 \rightarrow
14 \rightarrow 2 \rightarrow 0$. After 4 shakes, the first element of $S$ would be $s_0 \oplus x_0
\oplus x_2 \oplus x_{14} \oplus x_5$. If we do 4 more shakes, the first element of $S$ would be
$s_0$. This means that for $S[0], S[2], S[5]$, and $S[14]$, shaking the shaker 8 times and opening
yields the same result as opening the shaker straight away.

More importantly, seeing inside the shaker causes two XORs to happen, and then a permutation. If
we see inside without shaking at all, the original permutation does not get to affect $S$, so we
can safely ignore that. This means that we would have a sequence such as $\oplus x_2 \rightarrow
\oplus x_{14} \rightarrow \oplus x_5 \rightarrow \oplus x_0 \rightarrow \oplus x_2 \rightarrow
\dots$. That means, doing 7 shakes yields $s_0 \oplus x_0$ at $S[0]$, which means that seeing inside
one more time should yield $s_0$! (and $s_2$, and $s_5$, and $s_{14}$)

# The execution

To maximize the likelihood that we obtain parts of the flag, we shake by $2 \times 60 - 1 = 119$
times. This is because 60 is a highly composite number, meaning we would cover the most cycle sizes
within the allowed number of shakes. The next highly composite number is 120, and we would have to
shake 239 times, which is beyond the maximum shakes allowed.

Effectively, the strategy goes like this:

1. Connect to challenge server.
2. See inside the shaker.
3. Shake 119 times.
4. See inside the shaker again. Take note of the result this time.
5. Check candidate flag by seeing the most common value for each byte over all runs, and compute
   the MD5 hash.
   * If MD5 hash matches, flag is found!
   * Else, go back to step 1.

Alternatively, we can skip tracking the most common values of each byte, and hope that we hit a
permutation where all the cycles are of some size that divides 60. According to the challenge
author, this happens ~1% of the time.

P.S. I was curious and calculated that there are $\approx 2.417 \times 10^{87}$ permutations where
all the cycle sizes divides 60. Taking this over the number of all permutations of 64 elements
(which is $64!$) yields $\approx 0.01905$ -- i.e., given a random 64-permutation, about 1.9% of the
time you get a permutation where all the cycle sizes divides 60.

P.P.S. The exact number of such "good" permutations is
2,416,970,844,847,973,521,750,237,877,211,729,378,473,197,297,208,879,191,228,030,371,848,739,084,042,240,000,000,000,
if my calculations are correct.
