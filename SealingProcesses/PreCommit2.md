## Precommit2

[seal_pre_commit_phase2](https://github.com/filecoin-project/rust-fil-proofs/blob/bdb7c4261e770e70dc2c93cef9f73109f06354b6/filecoin-proofs/src/api/seal.rs#L214) generates three trees `tree_c`, `tree_d`, `tree_r_last`. 


`seal_pre_commit_phase2` recives [SealPreCommitPhase1Output](https://github.com/filecoin-project/rust-fil-proofs/blob/415f9d2ea7967a0ca20f49361cc045469b454921/filecoin-proofs/src/types/mod.rs#L81) as input this includes:
```
let SealPreCommitPhase1Output {
        // the original data split into 11 layers x 32GiB
        mut labels,
        // StoreConfig relating to a binary tree of the original data
        mut config,
        // root of merkel tree above
        comm_d,
        ..
    } = phase1_output;
```

TODO: write about vanilla proofs


> Here you can find a visual explination of the tree constuction process: [figma](https://www.figma.com/file/VsodxV0DxhCevtmJIPoe57/Filecoin-tree-details?node-id=0-1&t=kA1ATuksTEKT4cwr-0)

## tree_c

`Tree_c` is generated [here](https://github.com/filecoin-project/rust-fil-proofs/blob/128f7209ec583e023f04630102ef1dd17fbe2370/storage-proofs-porep/src/stacked/vanilla/proof.rs#L1353) and depends on the [layer arity](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/filecoin-proofs/src/constants.rs#L108) for choosing the `ColumnArity` and `TreeArity`.

`tree_c` can be generated using [GPU or CPU](https://github.com/filecoin-project/rust-fil-proofs/blob/128f7209ec583e023f04630102ef1dd17fbe2370/storage-proofs-porep/src/stacked/vanilla/proof.rs#L455) this is detemened by the flags you use while compiling your client. For the sake of simplicity we will cover the CPU version. The GPU version is identical but abit more complex.

[generate_tree_c_cpu](https://github.com/filecoin-project/rust-fil-proofs/blob/128f7209ec583e023f04630102ef1dd17fbe2370/storage-proofs-porep/src/stacked/vanilla/proof.rs#L768) recives the following inputs:

```
// number of layers, this is determind by the sector size.
layers: usize
// number of nodes per BaseTree (total_nodes / num_of_base_trees)
nodes_count: usize
// number of BaseTree
tree_count: usize
// number of congifs (one config per BaseTree)
configs: Vec<StoreConfig>
// the acctual data we are going to build the tree from
labels: &LabelsCache<Tree>
```

To understand how trees are constructed in filecoin please read [this](https://github.com/mylink).

- First we [allocate](https://github.com/filecoin-project/rust-fil-proofs/blob/128f7209ec583e023f04630102ef1dd17fbe2370/storage-proofs-porep/src/stacked/vanilla/proof.rs#L785) a Vec to store the tree in.
- Depending on the `nodes_count` and `num_cpus::get()` we generate [chunck sizes](https://github.com/filecoin-project/rust-fil-proofs/blob/128f7209ec583e023f04630102ef1dd17fbe2370/storage-proofs-porep/src/stacked/vanilla/proof.rs#L795) in which we shall process the data.
- We then proceed to read the data in columns numbering 11 elements. Which are used as preimages and [hashed](https://github.com/filecoin-project/rust-fil-proofs/blob/128f7209ec583e023f04630102ef1dd17fbe2370/storage-proofs-porep/src/stacked/vanilla/proof.rs#L813) with [Poseidon hash](https://github.com/filecoin-project/rust-fil-proofs/blob/bd2e158e18dcb3e2a3e96591d98deff6c25651ac/storage-proofs-porep/src/stacked/vanilla/hash.rs#L12). The resulting hash in stored in the Vec created at stage 1.


The process defined above is executed for each Config (repersenting a BaseTree). Each time a BaseTree has been hashed its passed to [DiskTree::<Tree::Hasher, Tree::Arity, U0, U0>::from_par_iter_with_config](https://github.com/filecoin-project/rust-fil-proofs/blob/128f7209ec583e023f04630102ef1dd17fbe2370/storage-proofs-porep/src/stacked/vanilla/proof.rs#L819) which persists the BaseTree to disk.

Once all BaseTrees have been constructed we [call](https://github.com/filecoin-project/rust-fil-proofs/blob/128f7209ec583e023f04630102ef1dd17fbe2370/storage-proofs-porep/src/stacked/vanilla/proof.rs#L830):
```
create_disk_tree::<DiskTree<Tree::Hasher, Tree::Arity, Tree::SubTreeArity, Tree::TopTreeArity>,>(configs[0].size.expect("config size failure"), &configs)
```

At this stage the root of the final tree is calculated and we have a complete tree persisted on disk.


## tree_d
`Tree_d` is originally construcuted in PC1 phase. However if not found in PC2 phase `tree_d` will be [consturcted](https://github.com/filecoin-project/rust-fil-proofs/blob/128f7209ec583e023f04630102ef1dd17fbe2370/storage-proofs-porep/src/stacked/vanilla/proof.rs#L1389) as part of PC2 phase.


## tree_r_last

`tree_r_last` is constructed from the very last layer found in the replica [(layer 11)](https://github.com/filecoin-project/rust-fil-proofs/blob/128f7209ec583e023f04630102ef1dd17fbe2370/storage-proofs-porep/src/stacked/vanilla/proof.rs#L1424).

`tree_r_last` can also be constructed usign eather GPU or CPU. Im more familer with the GPU implementation and will cover it.

- Layer 11 is split into chunks of [700k elements](https://github.com/filecoin-project/rust-fil-proofs/blob/64c03eb9bf6ceed1a788e51ae9669e58cba2d977/storage-proofs-core/src/settings.rs#L44).
- These chuncks are then hashed by Poseidon(8 + 1) -> 1.
- The trees are constructed using a [TreeBuilder](https://github.com/filecoin-project/rust-fil-proofs/blob/128f7209ec583e023f04630102ef1dd17fbe2370/storage-proofs-porep/src/stacked/vanilla/proof.rs#L1086)
- Finally the complete tree is [persisted to disk](https://github.com/filecoin-project/rust-fil-proofs/blob/128f7209ec583e023f04630102ef1dd17fbe2370/storage-proofs-porep/src/stacked/vanilla/proof.rs#L1123).