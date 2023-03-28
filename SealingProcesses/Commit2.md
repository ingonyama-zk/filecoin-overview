## Commit2

In Commit1 phase the user generates a vanilla proof per partition. The proof is constructed by choosing a random leaf and showing that there ia a valid path from leaf_i to the root (merkle proof).

These proofs are then passed to Commit2 where we make them succinct (short proofs that are easy to verify).

// Returns [`SealCommitPhase2Output`] struct containing vector of zk-SNARK proofs.

`seal_commit_phase2` is the entry point for C2.

TODO: cover inputs 

Each `vanilla proof` is [converted](https://github.com/filecoin-project/rust-fil-proofs/blob/76de0be6eb009a28c764b45383979bc62475b54c/storage-proofs-core/src/compound_proof.rs#L245) into a Circuit.

The proofs can then be [batch generated](https://github.com/filecoin-project/rust-fil-proofs/blob/76de0be6eb009a28c764b45383979bc62475b54c/storage-proofs-core/src/compound_proof.rs#L259) according to `priority` (true == use GPU, false === dont use GPU).

`create_proof_batch_priority_inner` is then called with randomization.

TODO: explain proof synthesize and generation.

Finally the 10 proofs are [stored](https://github.com/filecoin-project/rust-fil-proofs/blob/bdb7c4261e770e70dc2c93cef9f73109f06354b6/filecoin-proofs/src/api/seal.rs#L534) in a Vec<u8> , [verified](https://github.com/filecoin-project/rust-fil-proofs/blob/bdb7c4261e770e70dc2c93cef9f73109f06354b6/filecoin-proofs/src/api/seal.rs#L541) before being retuned:

```
let out = SealCommitOutput { proof: buf };
```

These proofs can now be broadcasted to prove that a sector has been sealed and stored correctly. Its important to also understand that The trusted setup places a limit on circuit size. This limits us from generating a single proof, but rather 10 smaller proofs.