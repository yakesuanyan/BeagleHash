## Solver attacks

For certain simple types of hash functions, an *SMT solver* can be used to solve for the seed given a set of keys and their respective hashes. However, all members of the BeagleHash family are more than complex enough to evade short-input solver attacks. On one core of an Apple M1 processor, the Z3 Solver only finished within 10 minutes when most or all of the "finalizer" has been removed:

| Hash | Finalizer attacked |
| ---- | -------- |
| BeagleHash | 0 of 18 lines |
| Zaphod64 | 9 of 27 lines |
| StadtX | 6 of 13 lines |
| Zaphod32 | 9 of 21 lines |
| Phat4 | 5 of 17 lines |

Certain techniques can be used to speed up solver attacks and potentially break more of these hash functions. However, in light of the SMT problem being NP-hard, the attacker is at a severe disadvantage: a linear increase in hash function complexity, and thus an incremental decrease in hashing speed, will lead to a super-polynomial increase in attack complexity with any known SMT solving algorithm. I therefore believe that these hash functions are virtually immune to solver-based seed discovery attacks.

## Cryptanalysis

All members of the BeagleHash family use simpler mixing functions than SipHash. This arguably makes them more susceptible to various forms of cryptanalysis, which could allow attackers to find collisions faster than brute force. These collisions are likely to be seed-dependent, which means that two colliding keys are unlikely to collide after changing the seed. Therefore, such an attack is only plausible if the seed is not kept secret or the output of the hash function is exposed.

With a realistic network and a good hash table implementation, I believe that single collisions will be hard to detect, and thus this weakness is not relevant for BeagleHash's target application. In other words, finding hash collisions will be annoying enough to discourage HashDoS attacks.