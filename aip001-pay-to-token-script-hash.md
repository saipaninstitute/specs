# Pay-To-Token-Script-Hash (P2TSH) Specification

Authors: Vin Armani, Tobias Ruck

This specification describes the Pay-To-Token-Script-Hash format as a replacement of P2PKH for SLP token transfers. See https://github.com/jcramer/slp-specification/blob/master/slp-token-type-1.md for a specification of SLP tokens.

Instead of storing tokens in P2PKH outputs, under this specification, token wallets store tokens in P2SH outputs with the following unlock script and redeemScript:

Unlock script (scriptSig without redeemScript):
```
<signature>
<arbitrary metadata>
```

redeemScript:
```
<token_type>     // token_type (from the SLP spec), minimally encoded (as required by consensus)
OP_2DROP         // drop the version and one additional item from the input script, the metadata
<pubkey>         // public key of this address
OP_CHECKSIG      // verify the signature
```

## Motivation

### Accidental burning of tokens

With the introduction of SLP tokens, wallet developers were faced with the challenge of how to deal with SLP tokens. SLP tokens have the peculiar property that, when used improperly, or by a wallet that doesn't support the specification, the tokens would be burned instead of being part of a transaction that is rejected by the network.

HD wallets have dealt with this by using a different derivation path for SLP tokens than for satoshis, however, not all wallets follow this recommendation, and people are still able to send money to a non-SLP wallet. Different addresses for satoshis and tokens provide part of the solution, but the main benefits are lost if people are still able to import a seed phrase from a token wallet into a non-token wallet.

Under this specification, wallets will encode the token_type in a P2SH address as described above, and traditional non-token wallets will be completely oblivious to these outputs, even if they are from the same derivation path, and therefore become un-burnable.

Only a token wallet that implements this spec will even be aware of the outputs in the wallet, and these wallets will do their due dilligence to prevent burning of tokens.

If a user tries to send money to an SLP address which is P2PKH and not P2SH (i.e., not following this spec), they can be prompted with "Are you absolutely sure the recipient wallet supports SLP tokens?".

Wallet developers are encouraged to only use P2TSH outputs for token outputs, but continue to use P2PKH outputs for non-token outputs so traditional non-token wallets can still interface the satoshis of the wallet as if it was a normal wallet.

### SLP Metadata

SLP uses the (one) OP_RETURN output to store its metadata. Additional OP_RETURN outputs are allowed but non-standard, so they will not be broadcast on the network. This makes adding additional metadata, like receipts, messages, memos and trade offers difficult.

Under this specification, token wallets will be able to add metadata in the scriptSig of transaction inputs, after the signature, and they will simply be dropped directly in the script along with the token version using OP_2DROP.

## Drawbacks

The drawbacks of this proposal is that it makes the transactions maleable by a third party, e.g. a miner, and could therefore mess up a chain of valid transactions. However, for most use-cases, this can be avoided by using pre-consensus on transactions, and for most other use-cases where non-maleability is important, specific covenants can be used which don't suffer maleability issues.

If this does turn out to be an issue, the metadata could be removed by replacing OP_2DROP with OP_DROP, however, the benefits of metadata seem to outweigh the drawbacks.
