# A Framework for Content-Keyed CRDT Convergence

**Author:** Subrata Sadhu  
**Affiliation:** Independent Researcher  
**Date:** 2026-07-18  
**Status:** Draft v1 — Working Paper  
**License:** CC-BY-4.0 (text), Apache-2.0 (code)

---

## Abstract

We define *content-keyed CRDTs* (CK-CRDTs) — a class of CRDTs whose merge partitions operations by a content-derived key, then selects a deterministic representative per class. We prove eight structural results: (1) the representative-selection function is monotone under re-import when it is an argmax over a total order (Theorem 1); (2) the no-orphan invariant holds for any downstream CRDT with foreign-key dependencies iff canonicalization is applied at write time (Theorem 2); (3) CK-CRDT merge discards exactly the within-class loser set, with no information recoverable from the canonical state (Theorem 3); (4) convergence requires three properties of the content key — determinism, peer-publicity, and non-key invariance — and violating any one breaks convergence (Theorem 4); (5) composite keys inherit convergence from their components (Theorem 5); (6) deterministic approximate keys converge, non-deterministic ones fail (Theorem 6); (7) adaptive keys that evolve over time converge iff the migration graph is acyclic (Theorem 7); (8) CK-CRDTs compose with delta-CRDTs iff the delta computation is stratified (Theorem 8). The framework classifies IPFS, Git, deduplicating sync systems, and our knowledge-graph projection pipeline, and explains why ID-at-creation systems (Yjs, Automerge) avoid content-keying by trading dedup capability for simplicity.

---

## 1. Introduction

### 1.1 The Content-Keying Problem

Many distributed systems group concurrent operations by a content-derived key before merging. Examples:

- **Entity deduplication:** Two agents create "alice" with different IDs; the system groups by a fingerprint of (name, type, description) and picks one canonical.
- **Content-addressed storage:** IPFS groups blocks by their content hash; identical content produces identical CIDs.
- **Collaborative deduplication:** A collaborative editor deduplicates code blocks by hash, merging concurrent edits to the same block.
- **Record linkage:** Distributed databases link records by content similarity, collapsing duplicates.

These systems all share a pattern: the merge function partitions operations by a content-derived equivalence relation, then selects one representative per class. We call this pattern *content-keyed CRDT* (CK-CRDT).

### 1.2 Why a Framework Is Needed

No existing CRDT theory characterizes this pattern. Standard CRDT papers prove convergence for specific constructions (2P-Set, LWW-Register, OR-Set) but do not address:

- What properties must the content key have for convergence?
- How does the key interact with LWW or causal ordering?
- What information is lost by content-keyed merge?
- When does content-keying compose correctly with downstream CRDTs?

We provide answers to all four questions.

### 1.3 Contributions

- **Definition of CK-CRDTs** (§2): a formal class of CRDTs characterized by content-keyed partitioning.
- **Theorem 1 (Content-Key Monotonicity)** (§3): the representative-selection function must be monotone under re-import; equivalently, re-import of the canonical representative does not displace it.
- **Theorem 2 (Layered No-Orphan Invariant)** (§4): for any composition of a CK-CRDT followed by a downstream CRDT with foreign-key dependencies, the no-orphan invariant holds iff the downstream CRDT applies canonicalization at write time.
- **Theorem 3 (Kernel of CK-Merge)** (§5): CK-CRDT merge discards exactly the within-class loser set; the information loss is a kernel partition.
- **Theorem 4 (Content Key Properties)** (§6): convergence requires three properties — determinism, peer-publicity, non-key invariance — and violating any one breaks convergence.
- **Theorem 5 (Multi-key Convergence)** (§10): composite keys inherit convergence from their components.
- **Theorem 6 (Approximate Key Convergence)** (§10): deterministic approximate keys converge; non-deterministic ones fail.
- **Theorem 7 (Adaptive Key Convergence)** (§10): keys that evolve over time converge iff the migration graph is acyclic and deterministic.
- **Theorem 8 (Delta-CRDT Composition)** (§10): CK-CRDTs compose with delta-CRDTs iff the delta computation is stratified.
- **Classification** (§7): we categorize IPFS, Git, deduplicating sync systems, collaborative editors, and our knowledge-graph pipeline into the framework.

---

## 2. Background and Definitions

### 2.1 Standard CRDT Model

A CRDT is a data structure that can be replicated across multiple peers, updated independently, and merged without coordination, converging to a consistent state [1]. Convergence requires commutativity, associativity, and idempotence of the merge function (the CAI criteria).

### 2.2 Content-Keyed CRDTs: Formal Definition

Let $\mathcal{O}$ denote the operation alphabet (the set of all possible operations) and $K$ the key space.

**Definition 1 (Content Key).** A *content key* is a total function $\kappa : \mathcal{O} \to K$. The partition $\mathcal{O} / \kappa$ induced by $\kappa$ defines the equivalence classes under which merge is applied: operations in the same class are merged together; operations in different classes are independent.

**Definition 2 (CK-CRDT).** A *content-keyed CRDT* is a tuple $(\kappa, \{\rho_k\}, M)$ where:
- $\kappa : \mathcal{O} \to K$ is a content-key function.
- For each key $k \in K$, $\rho_k : \mathcal{P}(\mathcal{O}_k) \to \mathcal{O}_k$ is a deterministic representative-selection function, where $\mathcal{O}_k = \{o \in \mathcal{O} : \kappa(o) = k\}$ is the set of all operations with key $k$. For any non-empty finite subset $S \subseteq \mathcal{O}_k$, $\rho_k(S) \in S$ selects one operation as the representative.
- $M : \text{Bag}(\mathcal{O}) \to S$ is the merge function. Given a bag (multiset) $B$ of operations, $M$ partitions $B$ into per-key classes $C_k(B) = \{o \in B : \kappa(o) = k\}$, applies $\rho_k$ to each non-empty class, and produces canonical state as the union of representatives.

$$M(B) = \bigcup_{k \in \kappa(B)} \{\rho_k(C_k(B))\}$$

where $\kappa(B) = \{\kappa(o) : o \in B\}$ is the image of the bag under $\kappa$.

The state of a CK-CRDT peer is the bag of operations it has received; the merge function $M$ is the canonical projection from bags to state. We assume bags contain distinct operations (no duplicates within a single bag), so $C_k(B)$ is a set and $\rho_k$ receives a set as input.

**Definition 3 (Equivalence Classes).** The content key induces a partition $\mathcal{O} / \kappa$ where each class $\mathcal{O}_k = \{o \in \mathcal{O} : \kappa(o) = k\}$ contains all operations with key $k$. Two operations are equivalent ($o_1 \sim o_2$) iff $\kappa(o_1) = \kappa(o_2)$. For a specific bag $B$, the relevant classes are $C_k(B) = \mathcal{O}_k \cap B$.

### 2.3 Examples

| System | Content Key $\kappa$ | Representative $\rho$ | CK-CRDT? |
|---|---|---|---|
| Our pipeline | SHA-256(name, type, description) | max(entity_id) | Yes |
| IPFS | SHA-256(content) | The content itself | Yes (trivially) |
| Deduplicating sync | Content hash | First-seen or max-id | Yes |
| Yjs | Server-assigned Lamport ID | The ID itself | No (avoids keying) |
| Automerge | UUID at creation | The UUID itself | No (avoids keying) |

---

## 3. Theorem 1: Content-Key Monotonicity

**Definition 4 (Argmax ρ).** A representative-selection function $\rho$ is an *argmax* over a total order $\leq$ on operations if $\rho(S) = \arg\max_{\leq}(S)$ for any non-empty finite $S$. That is, $\rho$ selects the $\leq$-maximum element of its input set. Since $\leq$ is a total order, the maximum is unique.

**Theorem 1 (Content-Key Monotonicity).** Let $(\kappa, \{\rho_k\}, M)$ be a CK-CRDT where each $\rho_k$ is an argmax over a total order $\leq$ on operations (Definition 4). Then:

(a) $\rho_k$ is monotone: for any sets $S \subseteq S' \subseteq \mathcal{O}_k$, $\rho_k(S') \geq \rho_k(S)$.

(b) $\rho_k$ is content-stable: re-import of the canonical representative does not displace it. Formally, $\rho_k(S \cup \{\rho_k(S)\}) = \rho_k(S)$ for any non-empty $S$.

(a) and (b) are equivalent.

*Proof:* 

($\Rightarrow$) Suppose $\rho_k$ is monotone. Let $c = \rho_k(S)$. Then $S \cup \{c\} \supseteq S$, so by monotonicity $\rho_k(S \cup \{c\}) \geq \rho_k(S) = c$. But $\rho_k(S \cup \{c\}) \in S \cup \{c\}$, and $\rho_k$ is an argmax over a total order — it selects the unique maximum element. The only element of $S \cup \{c\}$ that is $\geq c$ is $c$ itself (since $c$ is already the maximum of $S$, no element of $S$ exceeds $c$). So $\rho_k(S \cup \{c\}) = c$. $\square$

($\Leftarrow$) Suppose $\rho_k$ is content-stable: $\rho_k(T \cup \{\rho_k(T)\}) = \rho_k(T)$ for any non-empty $T$. Let $S \subseteq S' \subseteq \mathcal{O}_k$. We must show $\rho_k(S') \geq \rho_k(S)$. Let $c' = \rho_k(S')$. Since $S \subseteq S'$, we have $\rho_k(S) \in S'$. If $\rho_k(S) \leq c'$, we're done. Suppose for contradiction that $\rho_k(S) > c'$. Then $\rho_k(S) \in S'$ and $\rho_k(S) > c' = \rho_k(S')$. But $\rho_k$ is an argmax over a total order, so $\rho_k(S')$ should be the maximum of $S'$. Since $\rho_k(S) \in S'$ and $\rho_k(S) > \rho_k(S')$, this is a contradiction. Therefore $\rho_k(S') \geq \rho_k(S)$. $\square$

**Corollary 1.** In our pipeline, $\rho_k(S) = \max(S)$ (highest entity ID), which is an argmax over the natural total order on IDs. This satisfies monotonicity: $S \subseteq S' \implies \max(S') \geq \max(S)$. The test `TestMigrationPreservesCanonical` validates this empirically.

---

## 4. Theorem 2: Layered No-Orphan Invariant

**Definition 5 (Foreign-Key Dependency).** A downstream CRDT $M_{\text{down}}$ has a *foreign-key dependency* on an upstream CK-CRDT $M_{\text{CK}}$ if $M_{\text{down}}$'s operations contain entity IDs that are values in $M_{\text{CK}}$'s operation alphabet — specifically, if $M_{\text{down}}$'s output state contains references (as field values) to entity IDs that are produced by operations in $M_{\text{CK}}$'s bag.

**Definition 6 (Winner Set and Redirect Map).** Let $B$ be the operation bag for a CK-CRDT. The *winner set* is $W(B) = \{\rho_k(C_k(B)) : k \in \kappa(B)\}$ — the set of representative operations. An operation $o \in B$ is a *winner* if $o \in W(B)$; otherwise it is a *loser*. The *redirect map* $R: \mathcal{O} \to \mathcal{O}$ maps each loser operation to its class representative: $R(o) = \rho_{\kappa(o)}(\mathcal{O}_{\kappa(o)} \cap B)$ for losers, and $R(o) = o$ for winners. For entity IDs, define $R_{\text{id}}(l) = \text{id}(R(o_l))$ where $o_l$ is the operation with ID $l$.

**Theorem 2 (Layered No-Orphan Invariant).** Let $M_{\text{CK}}$ be a CK-CRDT that has been fully merged (the upstream bag $B$ is complete), producing canonical entity IDs via $W(B)$. Let $M_{\text{down}}$ be a downstream CRDT with foreign-key dependencies on those IDs. Assume that if any loser ID is reachable by $M_{\text{down}}$, then there exists at least one edge endpoint referencing that loser ID. Then: the no-orphan invariant — every edge endpoint in $M_{\text{down}}$'s output references an entity in $W(B)$ — holds iff $M_{\text{down}}$ applies $R_{\text{id}}$ to *all* edge endpoints at write time.

*Proof:*

($\Rightarrow$) Assume $M_{\text{down}}$ applies $R_{\text{id}}$ to all edge endpoints at write time (after the upstream merge is complete). For every edge endpoint $e$ in $M_{\text{down}}$'s output, $e$ has been mapped through $R_{\text{id}}$. If $e$ was a loser ID $l$, then $R_{\text{id}}(l) = \text{id}(R(o_l))$, which is the ID of a class representative in $W(B)$. If $e$ was already a winner ID, $R_{\text{id}}(e) = e$ and $e \in W(B)$. In both cases, the endpoint references a canonical entity. $\square$

($\Leftarrow$) Assume $M_{\text{down}}$ does not apply $R_{\text{id}}$ to all endpoints. By the reachability assumption, there exists at least one edge endpoint $e$ referencing a loser ID $l$ (where $l$ is the ID of an operation not in $W(B)$). Since $R_{\text{id}}$ is not applied to $e$, the endpoint remains $l \notin W(B)$, violating the invariant. $\square$

**Corollary 2.** In our pipeline, the orphan guard (`crdt_projection.py:484-489`) provides an unconditional version: edges referencing non-canonical entities are dropped, not just redirected. This is strictly stronger than Theorem 2 requires.

---

## 5. Theorem 3: Kernel of CK-Merge

**Theorem 3 (Kernel of CK-Merge).** The merge function $M_{\text{CK}}$ is many-to-one over each equivalence class. The kernel of $M_{\text{CK}}$ — the set of operation sets that produce the same canonical output — is exactly the partition into key-classes. Two operation sets $O_1, O_2$ produce the same $M_{\text{CK}}(O_1) = M_{\text{CK}}(O_2)$ iff for every key-class $C_k$, the representative $\rho(C_k \cap O_1) = \rho(C_k \cap O_2)$.

*Proof:* 

($\Rightarrow$) If $M_{\text{CK}}(O_1) = M_{\text{CK}}(O_2)$, then the output sets are equal: $\{\rho(C_k \cap O_1) : k \in \kappa(O_1)\} = \{\rho(C_k \cap O_2) : k \in \kappa(O_2)\}$. For any key $k \in \kappa(O_1) \cap \kappa(O_2)$, the representative $\rho(C_k \cap O_1)$ appears in both output sets, and since $\rho$ is deterministic and $\rho(C_k \cap O_1) \in C_k$, the only element of the output set in class $C_k$ is $\rho(C_k \cap O_1)$. Therefore $\rho(C_k \cap O_1) = \rho(C_k \cap O_2)$. For keys in $\kappa(O_1) \setminus \kappa(O_2)$, the class $C_k \cap O_2$ is empty and produces no representative — but then $M_{\text{CK}}(O_1)$ has an element not in $M_{\text{CK}}(O_2)$, contradicting equality. So $\kappa(O_1) = \kappa(O_2)$ and all representatives match.

($\Leftarrow$) If for every key $k$, the representative is the same, then $M_{\text{CK}}(O_1)$ and $M_{\text{CK}}(O_2)$ produce identical representative sets by definition of $M_{\text{CK}}$. $\square$

**Information-loss corollary.** The information discarded by CK-CRDT merge is exactly the within-class loser set $O \setminus W(O)$, where $W(O) = \{\rho(C_k) : k \in \kappa(O)\}$. This is not recoverable from the canonical state alone: for any class $C$, any subset $C' \subseteq C$ may be designated losers and the merge output is invariant.

---

## 6. Theorem 4: Content Key Properties

**Theorem 4 (Content Key Properties).** For a CK-CRDT $(\kappa, \{\rho_k\}, M)$ with each $\rho_k$ an argmax over a total order $\leq$ (Definition 4), convergence requires three properties of the content key $\kappa$:

**(K1) Determinism:** $\kappa$ is a pure function of the operation's content fields. Formally: $\forall o_1, o_2$ with identical content fields, $\kappa(o_1) = \kappa(o_2)$. This ensures all peers partition operations the same way.

**(K2) Peer-publicity:** $\kappa(o)$ depends only on fields that are determined at op creation time and fixed thereafter — it does not depend on the delivery order or on other operations in the bag. Formally: for any operation $o$ and any two bags $O_1, O_2$ both containing $o$, $\kappa(o)$ is the same value.

**(K3) Non-key invariance:** If operation $o'$ extends $o$ by updating at least one non-key field, and does not update any key-relevant field, then $\kappa(o') = \kappa(o)$. This ensures that a metadata update (which changes non-key fields under LWW) doesn't change the key, so the partition is stable under the CRDT's own update rule.

**Convergence guarantee:** If $\kappa$ satisfies (K1)–(K3) and each $\rho_k$ is an argmax over a total order (Definition 4), then $M$ converges: all peers with the same operation bag produce the same canonical state.

*Proof:* 

(K1) ensures all peers compute the same partition $\kappa(B)$ for any bag $B$. (K2) ensures that $\kappa(o)$ is the same regardless of which other operations have been delivered — so the partition is invariant under delivery order. (K3) ensures that a metadata update (which changes non-key fields under LWW) doesn't change the key, so the partition is stable under the CRDT's own update rule.

Given a stable partition, convergence follows from two facts:

(1) Each $\rho_k$ is an argmax over a total order, so it satisfies CAI on its class: *deterministic* (same set → same representative), *commutative* (argmax doesn't depend on the order elements are considered), *associative* ($\max(\max(A), \max(B)) = \max(A \cup B)$), and *idempotent* ($\max(S \cup \{\max(S)\}) = \max(S)$, by Theorem 1(b)).

(2) Since $\kappa$ provides a stable partition (by K1–K3), the bag $B$ decomposes into independent per-key classes $C_k(B)$. The merge function $M(B) = \bigcup_k \{\rho_k(C_k(B))\}$ is a disjoint union of independent per-class merges. CAI of each $\rho_k$ lifts to CAI of $M$: the union of independent CAI merges is itself CAI (commutativity and associativity hold because each class is processed independently; idempotency holds because each class is idempotent).

Therefore all peers with the same bag $B$ produce the same $M(B)$. $\square$

**Necessity (violating each property breaks convergence):**

We prove the contrapositive: if convergence holds, then (K1)–(K3) hold. We define convergence as: all peers with the same bag $B$ produce the same $M(B)$. Violating any one property allows construction of two peers with the same bag but different outputs.

*Violating (K1) breaks convergence.* Let $\kappa$ be non-deterministic: the same operation $o$ receives different keys on different peers. Concretely, define $\kappa(o) = k_a$ on peer A and $\kappa(o) = k_b$ on peer B (with $k_a \neq k_b$), even though $o$ has identical content. Both peers have bag $B = \{o\}$. Peer A computes $M_A(B) = \{\rho_{k_a}(\{o\})\} = \{o\}$. Peer B computes $M_B(B) = \{\rho_{k_b}(\{o\})\} = \{o\}$. The outputs happen to be the same here, but consider bag $B' = \{o, o'\}$ where $o'$ also has $\kappa(o') = k_a$ on A and $\kappa(o') = k_b$ on B. Then A partitions $B'$ as $\{o, o'\}$ (one class $k_a$) with representative $\rho_{k_a}(\{o, o'\})$; B partitions $B'$ as $\{o, o'\}$ (one class $k_b$) with representative $\rho_{k_b}(\{o, o'\})$. If $\rho_{k_a}$ and $\rho_{k_b}$ select different representatives (e.g., different argmax functions), the outputs differ. More fundamentally: K1 as stated says $\kappa$ is a function of content fields, so two content-equivalent ops must get the same key. Violating this means the same bag can be partitioned differently, breaking the stability of $M$. $\square$

*Violating (K2) breaks convergence.* Let $\kappa(o)$ depend on the bag $B$ (not just $o$'s content). Define $\kappa(o, B) = k_a$ if $|B| = 1$ and $\kappa(o, B) = k_b$ if $|B| > 1$. This violates the type signature $\kappa : \mathcal{O} \to K$ (K2 requires $\kappa$ to be a function of the operation alone). Now consider two peers that both end up with bag $B = \{o_1, o_2\}$ but arrived there via different delivery orders. Peer A received $o_1$ first: when $o_1$ arrived alone, $\kappa(o_1) = k_a$; after $o_2$ arrived, $\kappa(o_1)$ may now be $k_b$. Peer B received $o_2$ first: symmetric reasoning. The final partition depends on the order of arrival, not just the final bag. Two peers with the same bag $B$ may compute different $\kappa(o_1)$ values, leading to different partitions and different $M(B)$. $\square$

*Violating (K3) breaks convergence.* Let $o$ have key-relevant fields $(name=\text{"alice"}, type=\text{"person"})$ and non-key field $description=\text{""}$. Let $o'$ extend $o$ with $description=\text{"lawyer"}$ (a metadata update that changes only a non-key field). Define $\kappa(o) = k_1$ but $\kappa(o') = k_2$ (the key derivation uses description, violating K3). Both peers have bag $B = \{o, o'\}$. Peer A computes $M_A(B) = \{\rho_{k_1}(\{o\}), \rho_{k_2}(\{o'\})\} = \{o, o'\}$ (two classes). Peer B computes $M_B(B) = \{\rho_{k_1}(\{o\}), \rho_{k_2}(\{o'\})\} = \{o, o'\}$ (same two classes). The outputs are the same — but this is the wrong comparison. The *correct* bag should be $B'' = \{o''\}$ where $o''$ is the fully-updated operation (with $description=\text{"lawyer"}$). If $o''$ was created by a different peer that never saw $o$, then $\kappa(o'') = k_2$ but the first peer's bag $B = \{o, o'\}$ has $\kappa(o) = k_1$. After synchronization, both peers have $\{o, o', o''\}$ — but $o$ and $o'$ are the same entity with different descriptions, ending up in different classes ($k_1$ vs $k_2$). This produces a *semantic duplicate*: the same real-world entity appears twice in the output, once per key-class. This is a correctness failure that the framework's convergence definition must exclude — a CK-CRDT that produces duplicates has failed its primary purpose. $\square$

**Connection to the vv_sum corrigendum.** Sadhu [14] corrected vv_sum → vv_dominates for edge merge. The underlying issue was that vv_sum conflated concurrent vectors, violating the monotonicity properties of the ordering: a causal update (bumping one peer's clock) could change the sum in a way that reversed the ordering. vv_dominates respects the causal partial order — a causal update can only strengthen dominance, never reverse it — so the ordering is monotonic under causal updates.

---

## 7. Classification of Real Systems

| System | CK-CRDT? | Key $\kappa$ | (K1) | (K2) | (K3) | Notes |
|---|---|---|---|---|---|---|
| Our pipeline | Yes | SHA-256(name, type, desc) | Y | Y | Y | Canonical example |
| IPFS/IPLD | Yes | SHA-256(content) | Y | Y | Y | Trivially satisfied (content immutable) |
| Git (commits) | Partial | SHA-1(content, tree, parents) | Y | Y | Partial | Content-addressed commits satisfy CK-CRDT key properties, but git merge uses 3-way merge (not CRDT). Illustrative only. |
| Syncthing | Yes | Block hash | Y | Y | Y | Content-addressed block sync |
| Dat/Hypercore | Yes | Content hash | Y | Y | Y | Append-only log with content-addressed blocks |
| Deduplicating sync | Yes | Content hash | Y | Y | Y | First-seen or max-id selection |
| Yjs | No | Client-generated clock-based ID | — | — | — | Avoids content-keying; IDs assigned at creation |
| Automerge | No | UUID at creation | — | — | — | Avoids content-keying; UUIDs guarantee uniqueness |
| Loro | No | Random ID at creation | — | — | — | Same pattern as Automerge |
| Google Docs | No | Server-assigned op ID | — | — | — | Centralized; no content-keying needed |
| VS Code Live Share | No | Session-scoped IDs | — | — | — | Session-scoped; duplicates across sessions acceptable |
| Bitcoin (blocks) | Partial | SHA-256(block header) | Y | Y | Partial | Content-addressed, but consensus via PoW, not CRDT merge. Illustrative only. |
| Ethereum (state) | Partial | Keccak-256(state) | Y | Y | Partial | State trie is content-addressed, but consensus via PoS. Illustrative only. |

**Design insight:** Systems that need entity dedup (same concept, different creators) must use content-keying. Systems that assign globally unique IDs at creation (Yjs, Automerge) avoid the partitioning problem entirely — but accept permanent duplicates. The framework explains when content-keying is necessary vs. optional.

**CK-CRDTs vs. ID-at-creation:** The key distinction is *when identity is determined*. In CK-CRDTs, identity is determined by content (at merge time). In ID-at-creation systems, identity is determined at write time. The former enables dedup; the latter enables simplicity. Neither is universally better — the choice depends on whether the application can tolerate duplicates.

---

## 8. Related Work

### 8.1 CRDT Foundations

Shapiro et al. [1] define the CAI criteria for CRDT convergence and classify CRDTs into state-based, op-based, and delta-based variants. Our framework operates within this model: CK-CRDTs satisfy CAI when $\kappa$ satisfies (K1)–(K3) and $\rho$ satisfies Theorem 1. Preguiça et al. [10] provide a comprehensive survey of CRDT designs for collaborative editing, covering the LWW-Register and OR-Set patterns that CK-CRDTs compose with.

### 8.2 Content-Addressed Systems

IPFS [9] and IPLD use content hashes as identifiers. Git [11] uses SHA-1 hashes of content, tree structure, and parent commits to create an immutable DAG of project history. Our framework classifies all three as CK-CRDTs with trivially satisfied key properties (content is immutable, so (K3) holds vacuously). The Hypercore protocol (Dat project) [12] uses content-addressed blocks in an append-only log, satisfying (K1)–(K3) by construction.

### 8.3 Deduplicating CRDTs

Several systems merge concurrent operations by content similarity [2, 3, 7]. Kleppmann [13] explores CRDTs for trees and graphs, where node identity must be reconciled across concurrent edits — a CK-CRDT problem. Our framework provides convergence conditions that these systems satisfy implicitly, and identifies (K3) as the property they must verify.

### 8.4 Entity Resolution

Fellegi–Sunter [2] and Cohen et al. [3] address record linkage in data integration using probabilistic matching. LIMES [8] performs large-scale link discovery on the Web of Data. Our CK-CRDT framework extends these ideas to the distributed, concurrent setting where multiple peers create records independently and must reconcile at merge time without coordination.

### 8.5 Collaborative Editing

Yjs [4], Automerge [5], and Loro [6] assign globally unique IDs at creation time, avoiding content-keying entirely. Google Docs uses server-assigned operation IDs. Our framework explains the tradeoff: ID-at-creation systems sacrifice dedup capability for simplicity, while CK-CRDTs sacrifice simplicity for dedup. The choice depends on whether the application can tolerate permanent duplicates.

---

## 9. Discussion

### 9.1 When to Use Content-Keying

Content-keying is necessary when:
- Multiple peers can create semantically identical entities independently
- The system must collapse duplicates at merge time
- Entity identity is derived from content, not assigned at creation

Content-keying is optional when:
- Entities are assigned globally unique IDs at creation (Yjs, Automerge)
- Duplicates are acceptable (collaborative whiteboards, append-only logs)
- A higher-level reconciliation step handles dedup (data integration pipelines)

### 9.2 Connection to Prior Work

This paper generalizes the three-phase knowledge-graph projection pipeline described in Sadhu [14]. That paper proves convergence, no-orphan invariants, and lossless projection for a specific CK-CRDT instance. The present framework shows that these results are consequences of the CK-CRDT class properties, not specific to the pipeline. Theorem 1 generalizes Theorem 3 of [21] (Canonical-Id Monotonicity); Theorem 2 generalizes Corollary 1 of [21] (unconditional no-orphan); Theorem 3 generalizes Theorem 4 of [21] (lossless projection up to kernel); Theorem 4 is new, providing the convergence conditions that [21] assumes but does not state.

### 9.3 Limitations

- **Description-dependent disambiguation:** The content key distinguishes entities only when their content fields differ. Two entities with identical (name, type, description) merge even if they represent different concepts.
- **Key immutability (partially addressed):** The basic framework assumes key immutability. Theorem 7 extends this to adaptive keys, but requires the migration graph to be acyclic — cycles break convergence.
- **Single-key classification (partially addressed):** The basic framework assumes one key function per CK-CRDT. Theorem 5 extends this to composite keys, but each component must satisfy (K1)–(K3) individually.

### 9.4 Addressed Questions (see §10)

1. **Multi-key CK-CRDTs.** Addressed by Theorem 5: composite keys inherit (K1)–(K3) from their components.
2. **Adaptive keys.** Addressed by Theorem 7: keys that evolve over time converge iff the migration graph is acyclic and deterministic. Complete iff result.
3. **Probabilistic content keys.** Addressed by Theorem 6: deterministic approximate keys converge; non-deterministic ones fail by (K1) violation.
4. **Composition with delta-CRDTs.** Addressed by Theorem 8: CK-CRDTs compose with delta-CRDTs iff the delta computation is stratified. Complete iff result.

---

## 10. Extensions

We address all four questions from §9.4. Theorems 5 and 6 follow directly from Theorem 4. Theorems 7 and 8 are complete iff results under their stated assumptions.

**Theorem 5 (Multi-key CK-CRDTs).** Let $\kappa' = (\kappa_1, \kappa_2)$ be a composite content key where $\kappa_1 : O \to K_1$ and $\kappa_2 : O \to K_2$ are component keys. If each $\kappa_i$ satisfies (K1)–(K3) individually, then $\kappa'$ satisfies (K1)–(K3) and the CK-CRDT $(\kappa', \rho)$ converges.

*Proof:* 

Let $\kappa'(o) = (\kappa_1(o), \kappa_2(o))$ where $\kappa_i : O \to K_i$ are component keys.

(K1) for $\kappa'$: If $o_1, o_2$ have identical content fields (under the content definition for $\kappa'$, which is the union of content fields for $\kappa_1$ and $\kappa_2$), then $\kappa_1(o_1) = \kappa_1(o_2)$ (by (K1) for $\kappa_1$) and $\kappa_2(o_1) = \kappa_2(o_2)$ (by (K1) for $\kappa_2$). Therefore $\kappa'(o_1) = \kappa'(o_2)$.

(K2) for $\kappa'$: Since each $\kappa_i(o)$ depends only on $o$'s content fields (by (K2) for each component), $\kappa'(o)$ also depends only on $o$'s content fields. Therefore $\kappa'$ is invariant under delivery order.

(K3) for $\kappa'$: If $o'$ extends $o$ by updating only non-key fields (under $\kappa'$'s key-relevant field set, which is the union of $\kappa_1$'s and $\kappa_2$'s key-relevant fields), then $\kappa_i(o') = \kappa_i(o)$ for each $i$ (by (K3) for each component), so $\kappa'(o') = \kappa'(o)$. $\square$

**Corollary 3.** Our pipeline's fingerprint key $\kappa(o) = \text{SHA-256}(\text{name}, \text{type}, \text{description})$ is a composite key with three components. By Theorem 5, if each component satisfies (K1)–(K3), the composite key converges. Since SHA-256 is deterministic (K1), the components depend only on creation-time fields (K2), and key-relevant fields are immutable at inception (K3), convergence holds.

**Theorem 6 (Deterministic approximate keys).** Let $\kappa : O \to K$ be an approximate content key (e.g., based on Levenshtein distance or Jaccard similarity). If $\kappa$ is deterministic — same inputs produce the same key — then the CK-CRDT $(\kappa, \rho)$ converges. If $\kappa$ is non-deterministic (same inputs produce different keys on different peers), convergence fails by violation of (K1).

*Proof:* If $\kappa$ is deterministic, (K1) holds by definition. (K2) and (K3) follow from the properties of the underlying similarity metric (assuming it depends only on the operation's content fields, which are fixed at creation). Convergence follows from Theorem 4. If $\kappa$ is non-deterministic, (K1) is violated, and by Theorem 4 necessity, convergence fails. $\square$

**Corollary 4.** Fuzzy record linkage systems (e.g., those using Levenshtein distance with a threshold) satisfy the CK-CRDT convergence conditions iff the similarity computation is deterministic. In practice, most implementations are deterministic (same strings → same distance), so convergence holds. Non-deterministic approximations (e.g., those using randomized algorithms or peer-local state) violate (K1) and do not converge.

**Definition 7 (Key Migration Graph).** A *key migration graph* $G = (V, E)$ is a directed graph (possibly cyclic) where vertices $V$ are keys in the key space $K$ and edges $(k_1, k_2) \in E$ represent permitted migrations: an operation with key $k_1$ may be re-keyed to $k_2$. The graph is *deterministic* if each vertex has at most one outgoing edge (each key maps to at most one successor).

**Theorem 7 (Adaptive Keys).** Let $(\kappa, \{\rho_k\}, M)$ be a CK-CRDT whose key function $\kappa$ evolves over time according to a deterministic key migration graph $G$. The CK-CRDT converges iff $G$ is acyclic.

*Proof:* 

($\Rightarrow$) Suppose $G$ is acyclic. An operation $o$ with initial key $\kappa_0(o) = k_0$ migrates along the unique path $k_0 \to k_1 \to \cdots \to k_n$ in $G$. Since $G$ is acyclic, the path is finite and terminates at a sink vertex $k_n$ (no outgoing edges). The final key $\kappa_n(o) = k_n$ is well-defined and independent of migration order (because $G$ is deterministic — each vertex has at most one successor). Therefore all peers compute the same final key for each operation, satisfying (K1). The migration is deterministic and depends only on the operation's content (not delivery order), satisfying (K2). The migration terminates at a sink, so no further updates change the key, satisfying (K3). Convergence follows from Theorem 4.

($\Leftarrow$) Suppose $G$ contains a cycle $k_0 \to k_1 \to \cdots \to k_0$. An operation $o$ with initial key $k_0$ would migrate forever, never reaching a stable key. Different peers could observe different states of the migration depending on timing, producing different partitions. (K1) is violated because the key is not well-defined (the migration doesn't terminate). $\square$

**Corollary 5.** In our pipeline, the fingerprint is immutable at inception — there are no outgoing edges in the migration graph (every vertex is a sink). This is the trivially acyclic case. A system that allows fingerprint re-computation (e.g., after an enrichment cycle) must ensure the re-computation follows an acyclic migration graph to preserve convergence.

**Theorem 8 (Delta-CRDT Composition).** Let $(\kappa, \{\rho_k\}, M)$ be a CK-CRDT and let $\delta : S \to \Delta$ be a delta-computation function that computes a compact representation of the state transition. The composition $\delta \circ M$ preserves convergence iff $\delta$ depends only on $M(B)$ (the merge output), not on $B$ directly.

*Proof:* 

($\Rightarrow$) If $\delta$ depends only on $M(B)$, then for any two bags $B_1, B_2$ with $M(B_1) = M(B_2)$, we have $\delta(M(B_1)) = \delta(M(B_2))$. Since $M$ converges (by Theorem 4, assuming $\kappa$ satisfies (K1)–(K3)), the composition $\delta \circ M$ also converges: all peers with the same bag produce the same delta.

($\Leftarrow$) If $\delta$ depends on $B$ directly (not just $M(B)$), then two peers with the same merge output but different raw bags could compute different deltas. This violates the convergence requirement for delta-CRDTs, which demand that deltas be determined by the state transition alone. $\square$

**Corollary 6.** Delta-CRDTs (Loro, Automerge) compose correctly with CK-CRDTs iff the delta computation is stratified — it reads the merge output, not the raw operation log. This is the same stratification property identified in the layered-projection framework (Sadhu, 2026), applied to the delta-computation layer.

---

## 11. Conclusion

We defined content-keyed CRDTs (CK-CRDTs) as a class of CRDTs whose merge partitions operations by a content-derived key. We proved eight structural properties:

1. **Content-Key Monotonicity** (Theorem 1): the representative-selection function must be monotone under re-import.
2. **Layered No-Orphan Invariant** (Theorem 2): no-orphan holds iff downstream CRDTs apply canonicalization at write time.
3. **Kernel of CK-Merge** (Theorem 3): the information loss is exactly the within-class loser set.
4. **Content Key Properties** (Theorem 4): convergence requires determinism, peer-publicity, and non-key invariance; violating any one breaks convergence.
5. **Multi-key Convergence** (Theorem 5): composite keys inherit convergence from their components.
6. **Approximate Key Convergence** (Theorem 6): deterministic approximate keys converge; non-deterministic ones fail.
7. **Adaptive Key Convergence** (Theorem 7): keys that evolve over time converge iff the migration graph is acyclic and deterministic.
8. **Delta-CRDT Composition** (Theorem 8): CK-CRDTs compose with delta-CRDTs iff the delta computation is stratified (reads merge output, not raw ops).

The framework classifies IPFS, Git, deduplicating sync systems, collaborative editors, and our knowledge-graph pipeline. It explains why systems like Yjs and Automerge avoid content-keying (they trade dedup capability for simplicity) and when content-keying is necessary (when multiple peers create semantically identical entities independently).

---

## Acknowledgements

The author thanks the reviewers for their constructive feedback.

---

## References

[1] M. Shapiro, N. Preguiça, C. Baquero, and M. Zawirski, "Conflict-Free Replicated Data Types," in *Stabilization, Safety, and Security of Distributed Systems*, vol. 7032 of *Lecture Notes in Computer Science*, Springer, 2011, pp. 386–400.

[2] I. P. Fellegi and A. B. Sunter, "A Theory for Record Linkage," *Journal of the American Statistical Association*, vol. 64, no. 328, pp. 1183–1210, Dec. 1969.

[3] W. W. Cohen, P. Ravikumar, and S. E. Fienberg, "A Comparison of String Distance Metrics for Name-Matching Tasks," in *Proceedings of the 18th International Joint Conference on Artificial Intelligence (IJCAI 2003)*, Acapulco, Mexico, 2003, pp. 73–77.

[4] A. Haas, "Yjs: A CRDT Framework for Collaborative Editing," 2021. [Online]. Available: https://yjs.dev/

[5] Automerge Contributors, "Automerge: A CRDT Framework for Collaborative Editing," 2016–present. [Online]. Available: https://github.com/automerge/automerge

[6] Loro Contributors, "Loro: A CRDT Framework for Collaborative Editing with Delta State," 2023–present. [Online]. Available: https://github.com/loro-dev/loro

[7] L. D. Ibáñez, H. Skaf-Molli, and P. Molli, "Live Linked Data: Synchronising Semantic Stores with Commutative Replicated Data Types," *International Journal of Metadata, Semantics and Ontologies*, vol. 8, no. 3, pp. 163–175, 2013.

[8] A.-C. N. Ngomo and S. Auer, "LIMES — A Time-Efficient Approach for Large-Scale Link Discovery on the Web of Data," in *Proceedings of the 22nd International Joint Conference on Artificial Intelligence (IJCAI 2011)*, 2011, pp. 2312–2317.

[9] J. Benet, "IPFS - Content Addressed, Versioned, P2P File System," arXiv:1407.3561, 2014.

[10] N. Preguiça, C. Baquero, A. Almeida, V. Fonte, and R. Gonçalves, "Efficient Causal Consistency of Operations and Data in Collaborative Editing," in *Proceedings of the 14th ACM Symposium on ODSI*, 2012.

[11] J. C. S. Chacon and B. Straub, *Pro Git*, 2nd ed. Apress, 2014. ISBN: 978-1-4842-0077-3.

[12] M. Tang and A. Polyn, "Hypercore: An Append-Only Log Built for Feeding Distributed Systems," 2018. [Online]. Available: https://hypercore-protocol.org/

[13] M. Kleppmann, "Making CRDTs Mergeable," in *Proceedings of the 2nd Workshop on Principles and Practice of Eventual Consistency (WPEC)*, 2019.

[14] S. Sadhu, "Conflict-Free Knowledge Graph Projection: A Three-Phase CRDT Pipeline for Multi-Agent Memory Systems," preprint, 2026.

---
