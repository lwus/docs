# Fireball

Fireball is a recipe program that combines NFTs through a burn-and-mint
mechanism. Recipe creators specify a list of ingredients that can be burned by
users to mint a new NFT.

## Recipe specification

Conceptually, a recipe is a list of ingredients. Each ingredient is a list of
specific NFTs that correspond to that ingredient.

```
recipe = [
  {
    "ingredient": "A",
    "mints": [
      <A_NFT_0_mint>,
      <A_NFT_1_mint>,
      ...
      <A_NFT_n_mint>,
    ],
  },
  {
    "ingredient": "B",
    "mints": [
      <B_NFT_0_mint>,
      ...
    ],
  },
  ...
]
```

On-chain, this is stored as a merkle tree for space efficiency of recipe
specifications. The fully specified list should be stored on e.g arweave.

Note that this methodology has some trade-offs:
- Existing NFTs can be used without any specific properties in the NFT metadata
  (e.g a specific creator or name or description) or reliance on any external
  groupings (e.g collections)
- Yet-to-be-minted NFTs cannot be specified as an ingredient!
    - A special case of this, for [limited editions](#limited-editions), is
      handled explicitly by Fireball.

### Recipe results

Currently, all Fireball recipe results are _limited edition_ prints.


### Limited editions

Creators can (optionally) allow _limited editions_ to be used 'in place' of the
_master edition_ ingredient. Importantly, this allows us to specify a recipe
without fully qualifying all the limited edition NFTs that can be used for an
ingredient!

One use case is the composition of recipes: recipe X's output can be used as an
ingredient in recipe Y by specifying one of the ingredient mints as the output
_master edition_.


## CLI

The
[`metaplex-foundation/metaplex`](https://github.com/metaplex-foundation/metaplex/)
repo contains a CLI at `/packages/cli/src/fireball-cli.ts` that can be used to
create and interact with recipes with the fireball program on chain.

### Creating a recipe

`create_recipe` expects a JSON file that specifies all NFTs that and the
ingredients they correspond to.

```
[
  {
    "mint": "5He5p6uUw7KRRzd4i7jPqcyhCc99G81wMdkz1Ub4WZyi",
    "ingredient": "mighty knighty duck recipe",
    "allowLimitedEditions": true
  },
  {
    "mint": "72a6uGvesBs6F7S6jL6N7cBFMeq4Q8GwBDAjQKNnFnZo",
    "ingredient": "duck with doughnut"
  },
  ...
}
```

- `mint` - Pubkey of the NFTs SPL token mint
- `ingredient` - String labelling each NFT as a particular ingredient
- `allowLimitedEditions` - Whether a limited edition can be used in place of
  the master edition. All NFTs for a certain ingredient should have the same
  `allowLimitedEditions` value (TODO).

#### Example list generation

To generate a list of mints from a candy machine where the NFTs are already
separated into ingredients by name, you can use

```
// getAccountsByCreatorAddress in /js/packages/cli/src/commands/signAll.ts
const accountsByCreatorAddress =
    await getAccountsByCreatorAddress(candyMachine, connection);

const inputs = accountsByCreatorAddress.map(
  it => {
    return {
      'mint': new PublicKey(it[0].mint).toBase58(),
      'ingredient': it[0].data.name,
    };
  }
);
```

For convenience, the input list is expected to be 'flat' and `create_recipe`
structures it. The restructuring can be skipped by modifying the code.

Once this file has been prepared, we can run:

```
$ ts-node src/fireball-cli.ts create_recipe --file <filepath> --keypair <filepath>
```

Which will create the on-chain recipe and initialize the merkle roots
corresponding to each ingredient. The recipe will be created at a randomly
generated Keypair location but the authority will be set to the CLI
`--keypair`.

Creation will also generate a `.log/<recipe-pubkey>/manifest.json` file that
contains the ingredients and NFTs in the order they were used to generate the
merkle trees. That is, the information that the client code in
`/packages/fireball/src/components/Redeem.tsx` requires. Roughly, that looks like

```
[
  {
    "allowLimitedEditions": [ true ],
    "ingredient": "mighty knighty duck recipe",
    "mints": [ "5He5p6uUw7KRRzd4i7jPqcyhCc99G81wMdkz1Ub4WZyi" ],
    "root": [ 201, ...  ]
  },
  {
    "ingredient": "duck with doughnut",
    "mints": [ "72a6uGvesBs6F7S6jL6N7cBFMeq4Q8GwBDAjQKNnFnZo", ...  ],
    "root": [ ... ]
  },
  ...
]
```

### Adding and removing outputs

The outputs of a recipe are simply any associated token account that the recipe PDA

```
await PublicKey.findProgramAddress(
  [FIREBALL_PREFIX, recipeKey.toBuffer()], FIREBALL_PROGRAM_ID
);
```

can sign for. Adding an output is simply creating the right ATA and
transferring the output master edition to it. This is wrapped in
`add_master_edition` by specifying the `--recipe` to add to and the `--mint` of
the master edition.

Removing an output requires one to be an `authority` on the recipe and can
similarly be done through the `reclaim_master_edition` by specifying the
`--recipe` and the `--mint`.


### Burning

A sample crank for burning ingredients after a recipe has been completed is
provided in `burn_crank`. Anyone can burn ingredients after recipe completion
and they are rewarded with the rent of the burned token account (~0.002 SOL).


## Architecture Notes

Fireball uses an intermediary `Dish` to store ingredients and to track how many
ingredients have been added so far. Note that this means we cannot begin
burning the ingredients until after all ingredients have been accumulated, as
some recipes have limited outputs and there can be a race condition to mint one
of these results.

Alternatively, we could directly in one instruction expect all the ingredients
and burn them all-or-none. However, even if we could build a more succint proof
for all these ingredients (e.g some RSA accumulator), we would still quickly
run into the limits of transaction sizing since each burn requires 1 token
account + 1 mint. This also means that we cannot expect users to burn all the
ingredients on the final mint and the ingredients should be burned through a
crank afterwards.

