# On Trees

```
                O -> top tree layer
       ________/|\_________ 
      /         |          \
     O          O           O -> sub-tree layer
  / / \ \    / / \ \     / / \ \
 O O  O  O  O O  O  O   O O  O  O -> base layer trees
```

Each tree layer has its own arity `BaseTreeArity`, `SubTreeArity`, `TopTreeArity`.

The tree above for example is a tree constructued from 3 subtrees with a `BaseTreeArity` of 4.

Its also important to understand that trees are stored in reality as flattend trees basically a array of elements.

You can learn more about these trees here https://github.com/filecoin-project/merkletree.
