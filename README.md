# CSCI 1102 Computer Science 2

### Fall 2018

------

## Lecture Notes

### Week 10: Hash Tables

#### Topics:

1. Basics
2. Collision Resolution
3. Hash Functions

---

> Credit: These lecture notes are cribbed from Cormen, Leiserson & Rivest.

## 1. Basics

As we've noted, one-dimensional arrays are stored in contiguously allocated words of memory. The process of computing the address of a particular array cell is simple, requiring one multiplication and one addition
```bash
loc = baseAddress + wordSize * index
```
No matter how large the array, we can always address an element of the array with just two operations.

This fact is the basis of the most longstanding and still most commonly used way to represent maps -- the [hash table](https://en.wikipedia.org/wiki/Hash_table). Hash tables are quite fast on average, $\Omicron(1)$ in both the worst and average cases. Since they are allocated sequentially, hash tables also tend to have good locality. Finally, they do not require an ordered set of keys. If there are drawbacks, it is that they sometimes need to be resized and they're naturally mutable structures, it takes a bit of work to implement thread-safe immutable versions (though see [HAMT](https://en.wikipedia.org/wiki/Hash_array_mapped_trie)s.

For the purposes of this discussion, we'll assume the following API for maps:

```java
public interface Map<K, V> {
    V get(K key);
    void put(K key, V value);
    void remove(K key);
    boolean isEmpty();
    int size();
    String toString();
}
```

We're primarily concerned with the performance of `get` but also with `put` and `remove`.

The basic idea is to define a "hash function" -- a function mapping keys to natural number array indices. If $m$ is the size of the array representing the hash table, we can understand a hash function $h$ as having type:
$$
h : U \to \{0, \ldots, m - 1 \}
$$
where $U$ is the set of keys.

> In Java, every reference type inherits a hash function `hashCode` from the root class `Object`.

From here on, we won't worry about the storage of the map's value, we'll only be concerned with keys. 

##### Example

```bash
   Key[] keys = {"Robert", "Alice", "Mei"};
   int hashCode(String key) { return key.length % 4; }

                          +-----+
   "Robert" -----+      0 |     |
                 |        +-----+
   "Alice"  -----|----> 1 |     |
                 |        +-----+
   "Mei"    --+  +----> 2 |     |   
              |           +-----+
              +-------> 3 |     |  
                          +-----+
```

It is very common to use the mod function `%` to ensure that the hash is a valid array index.

#### Collisions

The main challenges that arise with hashing is colliding keys. If the example above included the key `"Bo"`, its hash would have collided with that of `"Robert"`, i.e., both would hash to location 2 of the array.

```bash
   Key[] keys = {"Robert", "Alice", "Mei"};
   int hashCode(String key) { return key.length % 4; }

                          +-----+
   "Robert" -----+      0 |     |
                 |        +-----+
   "Alice"  -----|----> 1 |     |
                 |        +-----+
   "Mei"    --+  o----> 2 |  ?  |   
              |  |        +-----+
              +--|----> 3 |     |  
                 |        +-----+
    "Bo"    -----+
```

### Separate Chaining

One simple way to deal with hash collisions is to use a linked list of key/value nodes -- a *chain* -- for each index. When a key hashes to an array index, its entry may be found on the linked chain.

```bash
   Key[] keys = {"Robert", "Alice", "Mei"};
   int hashCode(String key) { return key.length % 4; }

                          +-----+
   "Robert" -----+      0 |     |
                 |        +-----+
   "Alice"  -----|----> 1 |     |
                 |        +-----+     +----------+---+    +------+---+
   "Mei"    --+  o----> 2 |  o--+---> | "Robert" | o-+--> | "Bo" | o-+--+
              |  |        +-----+     +----------+---+    +------+---+  =
              +--|----> 3 |     |  
                 |        +-----+
    "Bo"    -----+
```
Let's say the table has $n$ entries. The **load factor** of the table is the ration $n / m$. With chaining, $n$ may be smaller, equal to or larger than $m$. When the load factor is very small, many of the $m$ cells are being wasted. When it is too large, i.e., when $n$ is much greater than $m$, the chains are (on average) too long. In the extreme, if every key hashed to the same index, then the chain for that index would contain all $n$ keys.

### Open Addressing

In open addressing, all of the keys (and values) are stored directly in the table. Since collisions may occur, multiple table locations may require inspection. In theory, one might have to probe all of the table's $m$ cells. To support this, we'll reformulate our hash functions to accept an integer indicating the probe attempt:
$$
h : U \times \{0, \ldots, m - 1\} \to \{0, \ldots, m - 1\}
$$
The definition of a the two-argument hash function employs a one-argument auxiliary hash function as described above.

### Probe Sequences

Let $m$ be the size of the table. A *probe sequence* is a sequence of $m$ table indices
$$
\langle h(k, 0), h(k, 1), \ldots, h(k, m - 1)\rangle
$$
that is a permutation of the indices
$$
\langle 0, 1, \ldots, m - 1 \rangle
$$
What this means is that in the sequence of probes in a probe sequence, *all* of the table's $m$ locations will eventually be inspected.

### Linear Probing

The simplest probing method is to simply look in the next available spot.
$$
h(k, i) = (h'(k) + i)\ \%\ m
$$
```bash
 Key[] keys = {"Robert", "Alice", "Mei"};
   int h'(String key) { return key.length % 4; }

                          +----------+
   "Robert" -----+      0 |   "Bo"   | <---+
                 |        +----------+     |
   "Alice"  -----|----> 1 |  "Alice" |     |
                 |        +----------+     |
   "Mei"    --+  o----> 2 | "Robert" |     |
              |  ^        +----------+     |
              +--|----> 3 |   "Mei"  |     |
                 ^        +----------+     |
    "Bo"    -----+-------------------------+
```

After colliding with "Robert" at hash 2, index 3 was considered for "Bo". Since cell 3 was also occupied, index 0 was considered. It was open.

##### Primary Clustering

The main problem with this simple method is that sequences of full slots tend to grow. Assume that every hash is equally likely with probability $1/m$. If location $k$ is occupied, then location $(k + 1)\;\%\;m$ has probability $(1 / m + 1 / m) = 2/m$. If locations $k$ and  $(k + 1)\;\%\;m$ are both occupied, then the following slot has probability $3/m$ etc. When searching for a key in such a sequence, one has to run through these sequentially, which slows down the search.

### Quadratic Probing

With quadratic probing, the index "jumps" further and further away with each probe.
$$
h(k, i) = (h'(k) + c_1i + c_2i^2)\ \%\ m
$$

For $i$ in $\{0, 1, 2, 3, \ldots \}$, $i^2$ is in $\{0, 1, 4, 9, \ldots \}$.

##### Secondary Clustering

This can lead to sequence formation at the more remote cells.

### Double Hashing

An approach that works quite well in practice is to employ a second has function, $h_2$ below:
$$
h(k, i) = (h_1(k) + i h_2(k))\ \%\ m
$$

For example, with $m = 13$ 
$$
h_1(k) = k\;\%\; 13
$$
and
$$
h_2(k) = 1 + (k\;\%\; 11)
$$

```bash
   0    1    2    3    4    5    6    7    8    9   10   11   12
+----+----+----+----+----+----+----+----+----+----+----+----+----+
|    |    |    |    |    |    |    |    |    |    |    |    |    |
+----+----+----+----+----+----+----+----+----+----+----+----+----+
```

Insertion of 69, 72, 50 and 79 all find open cells.

```bash
   0    1    2    3    4    5    6    7    8    9   10   11   12
+----+----+----+----+----+----+----+----+----+----+----+----+----+
|    | 79 |    |    | 69 |    |    | 72 |    |    |    | 50 |    |
+----+----+----+----+----+----+----+----+----+----+----+----+----+
```

Attempting to insert 98 gives $h_1(98) = 7$, a collision with 72. The second probe gives

 $$(h_1(98) + h_2(98))\;\%\;13 = (7 + 11)\;\%\; 13 = 18\;\%\;13 = 5$$. 

```bash
   0    1    2    3    4    5    6    7    8    9   10   11   12
+----+----+----+----+----+----+----+----+----+----+----+----+----+
|    | 79 |    |    | 69 | 98 |    | 72 |    |    |    | 50 |    |
+----+----+----+----+----+----+----+----+----+----+----+----+----+
```

Attempting to insert 14 gives $h_1(14) = 1$, a collision with 79. Since $h_2(14) = 4$, the second probe gives

 $$(h_1(14) + h_2(14))\;\%\;13 = (1 + 4)\;\%\; 13 = 5\;\%\;13 = 5, \mathrm{another\ collision.}$$

```bash
   0    1    2    3    4    5    6    7    8    9   10   11   12
+----+----+----+----+----+----+----+----+----+----+----+----+----+
|    | 79 |    |    | 69 | 98 |    | 72 |    |    |    | 50 |    |
+----+----+----+----+----+----+----+----+----+----+----+----+----+
```

Finally the third probe (with $i = 2$) finds an open slot, position 9.

$$(h_1(14) + 2\cdot h_2(14))\;\%\;13 = (1 + 2 \cdot 4)\;\%\; 13 = 9\;\%\;13 = 9.$$

```bash
   0    1    2    3    4    5    6    7    8    9   10   11   12
+----+----+----+----+----+----+----+----+----+----+----+----+----+
|    | 79 |    |    | 69 | 98 |    | 72 |    | 14 |    | 50 |    |
+----+----+----+----+----+----+----+----+----+----+----+----+----+
```

In order for double-hashing to implement a probe sequence, $h_2(k)$ and $m$ should be [co-coprime](https://en.wikipedia.org/wiki/Coprime_integers), that is, if their only common positive integer factor is 1. If this condition doesn't hold, then the hash function can produce a cycle of indices that doesn't include all indices. The easiest way to ensure that $h_2(k)$ and $m$ are co-prime is to choose a table size $m$ to be prime and choose $h_1(k) = k\;\%\; m$ and  $h_2(k) = 1 + (k\; \%\; (m - 1))$.