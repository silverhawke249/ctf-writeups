# Uwusignatures

# The problem

Get the flag by forging a signature for a given message. We may obtain a sample signature for any
message that is not the target message.

# The observation

For each instantiation of the signature scheme, we have a 2048-bit prime $p$, a random value $g$,
the private key $x$, and the public key $y \equiv g^x \mod p$. Additionally, a random value $k$ is
chosen such that $k$ and $p - 1$ is coprime. The values $p, g, y, k$ are public knowledge.

In order to sign, the message $m$ is hashed and treated as a number, $h$. We calculate
$r \equiv g^k \mod p$ and $s \equiv (h - rx) k^{-1} \mod (p - 1)$, and publish $(r, s)$ as the
signature corresponding to $m$. Note that $k$ is always invertible since $k$ and $p - 1$ are
coprime.

Verification is done by computing $h$ from $m$, and then making sure that $g^h - y^r r^s \equiv 0
\mod p$. This is true because $y^r r^s \equiv g^{xr + k(h - rx)k^{-1}} \equiv g^h \mod p$.

This signature scheme is, in fact, the ElGamal signature scheme. In this signature scheme, it is
important that the value of $k$ is chosen randomly for every message that is signed. Additionally,
the value of $k$ must not leak in any way. In this implementation, the value of $k$ is chosen at
instantiation, reused for other messages, and published, which breaks the signature scheme.

# The execution

Let $h$ be the hash value corresponding to the target message `gib flag pls uwu`, and $h'$ be the
hash value corresponding to the message that we control. That means, we want to get $r \equiv g^k
\mod p$ and $s \equiv (h - rx) k^{-1} \mod (p - 1)$.

Choosing option 2, we can get the signature corresponding to $h'$, which would be $s' \equiv
(h' - rx) k^{-1} \mod (p - 1)$. Note that $r' = r$, since $k$ did not change.

Since we have $k$ and $p$ as public knowledge, we can easily compute $h' - s'k \equiv rx \mod
(p - 1)$. Additionally, since we know the target message, and thus, $h$, we can obtain the value
$s \equiv (h - rx) k^{-1} \mod (p - 1)$.

Putting it all together, the attack proceeds as follows:

1. Fix some message $m' \neq m$, compute its corresponding hash $h'$.
2. Request for the signature $(r', s')$ corresponding to $m'$.
3. Compute $k^{-1} \mod (p - 1)$.
4. Compute $s \equiv (h - h') k^{-1} + s' \mod (p - 1)$.
5. Send $(r', s)$ as the forged signature to $m$.
