# Pay-To-Token-Script-Hash (P2TSH) Specification

Authors: Vin Armani, Tobias Ruck

This specification describes the Pay-To-Token-Script-Hash format as a replacement of P2PKH for SLP token transfers. See https://github.com/jcramer/slp-specification/blob/master/slp-token-type-1.md for a specification of SLP tokens.

Instead of storing tokens in P2PKH outputs, under this specification, token wallets store tokens in P2SH outputs with the following unlock script and redeemScript:

Unlock script (scriptSig without redeemScript):
```
<signature>
```

redeemScript:
```
<token_type>     // token_type (from the SLP spec), minimally encoded (as required by consensus)
OP_DROP          // drop the version
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

## Alternatives

### Using `OP_2DROP` instead of `OP_DROP`

A previous proposal suggested to use `OP_2DROP` instead of `OP_DROP`, allowing to place arbitrary metadata in the scriptSig.

The reasoning is the following:

SLP uses the (one) OP_RETURN output to store its metadata. Additional OP_RETURN outputs are allowed but non-standard, so they will not be broadcast on the network. This makes adding additional metadata, like receipts, messages, memos and trade offers difficult.

By using OP_2DROP instead of OP_DROP, token wallets will be able to add metadata in the scriptSig of transaction inputs, after the signature, and they will simply be dropped directly in the script along with the token version using OP_2DROP.

#### Drawbacks

The drawbacks of this approach is that it makes the transactions malleable by a third party, especially if they’re not broadcast yet, and could therefore mess up a chain of valid transactions. Since currently P2PKH transactions are non-malleable, we would rather have P2TSH outputs behave exactly the same as P2PKH outputs to avoid unnecessary complexity. 

### Using a public key hash instead of a public key in the redeem script

Instead of the above script, it was suggested to use the following redeemScript:

```
<token_type>     // token_type (from the SLP spec), minimally encoded (as required by consensus)
OP_DROP          // drop the version
OP_DUP           // duplicate the public key
OP_HASH160       // calculate the input public key hash
<pubkey hash>    // public key of the 
OP_EQUALVERIFY   // verify input public key hash is the wallet's public key hash 
OP_CHECKSIG      // verify the signature
```

This would allow wallets to compute the P2TSH from the P2PKH address.

However, it has two drawbacks which the authors deemed sufficient to disqualify it from the standard:
- It increases the input size by 23 bytes, which makes token transactions significantly more expensive especially for transactions with many inputs, making it less likely to be implemented by wallet developers.
- Going the reverse, from P2TSH to P2PKH is not possible. This is quite confusing for users; so we might just as well not have any conversion at all.

It also somewhat defeats the purpose of this proposal, which is to keep the two addresses as separate as possible. Also, token wallets will watch ordinary P2PKH addresses anyway, so if one were to send tokens to an ordinary P2PKH address, the recipient would still be able to receive them.
