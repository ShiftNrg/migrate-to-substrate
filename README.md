# Substrate token migration tool

A proposal by Simon Warta developed within the Shift project on November 2019.

## Motivation

If you have an existing group of token holders that you want to migrate to a Substrate based blockchain, you'll most likely run into the situation that the old and the new address system are incompatible.

Since Substrate addresses are encoded pubkeys, this happens when

- **Case 1**: Some of the token holders are identified by an `address = derive(pubkey)` only, but the corresponding `pubkey` is unknown.
- **Case 2**: The derivation algorithm from secret to keypair is different for the old and the new system, such that users' pubkeys change.

To make a migration possible, we look for a way to transfer the old tokens to the new system in a secure and convenient way.

## Alternative use cases

The same problem exists when you want to perform an airdrop over a different crypto ecosystem or when switching from one signature algorithm to another. We intend to solve the primary problem in a sufficiently general fashion to cover those use cases as well.

## Vocabulary

* _target system_: the newly created Substrate based chain running the migration module designed in this document.
* _external system_: the system from which token holders migrate to the new Substrate based system. This can be a legacy system (migration case) or a parallel system (airdrop case).
* _target pubkey/address_: the public key and address on the target system.
* _external pubkey/address_: the public key and address on the external system.

## High level description

In case 2 from above, we assume the token holder uses an old keypair on the external system which is potentially different from the new keypair. For each token holder, the amount of tokens to receive on the target system is associated with the external address. Now the owner of the external address needs to prove to the target system that they own the external address. This is done by signing a message with the keypair, which belongs to that address (_proof of ownership_). The external address owner now defines where the tokens should be transferred to, usually their target address (_transfer order_). The proof of ownership as well as the transfer order are wrapped into a so called _claim transaction_, a new transaction type specific to this module.

### The chicken-or-egg problem

For spam protection reasons, the target system will usually charge some kind of fee in order to include and execute a transaction. During the onboarding process however, the token holder does not have access to their funds before the claim transaction is executed. However, executing a claim transactions requires an existing balance.

We'll see that the claim transaction can be submitted by any account on the target system, allowing community provided onboarding services.

## Implementation

The migration module is specific to Substrate. The target module for the tokens is [balances](https://substrate.dev/docs/en/next/conceptual/runtime/srml#balances).

The balance for each token holder must be known to the module and is immutable once initialized. In case of a migration, this typically comes from a state dump of the old system. For an airdrop, any logic may apply. A single balance can support multiple currencies.

### The migration module

The migration module we're designing contains the following things:

1. A set of configurations, including e.g. the address from which the tokens are distributed and a migration timeout
2. A queryable data store holding entries with external address, external pubkey and remaining claimable token balance.
3. A custom claim transaction that when executed verifies the right to claim, transfers tokens to a target address and updates the value of claimable token balance for this user to 0.

### Configuration

- Distribution address: ...
- End time: After this point in time, the shutdown transaction can be executed

### Data store

...

### The claim transaction

The claim transaction is a new transaction type specific to this module. It contains two things

1. A _proof of ownership_ of the external address
2. The _transfer order_, instructing to transfer the tokens to an arbitrary target address

The proof of ownership is a digital signature using the external system's signing algorithm. To avoid replaying existing signatures, a migration specific content needs to be signed. This is the transfer order string "claim on {chain ID}; transfer to {target address}".

Partial claims should not be supported since this feature is considered unnecessary.

Since the ownership of the target address does not need to be proven (token holder is free to transfer the tokens to wherever they want), the target system's transaction signer is not relevant. Anybody willing to pay the transaction fee can submit the claim transaction.

### The shutdown transaction

This transaction can be sent by any account and fails if the current time is not after the end time.

When executed, the remaining tokens of the distribution address are burned and the data store is cleared to an empty collection. Claim transaction logic remains unchanged by this event.

### The chicken service

The chicken service is a service to solve the chicken-or-egg problem in the onboarding process by paying and submitting claim transactions for the user. Since the transaction signer is irrelevant for the claim execution, any number of those services can exist.

This service is entirely optional and to be considered a convenience feature. The user can always execute a claim transaction after getting some initial tokens to pay the transaction fee on their own.

The basic flow is:

1. User created the content of a claim by signing the claim string
2. User sends the signed claim string via an API to the chicken service
3. Chicken services takes the claim string, wraps it into a claim transaction and submits this transaction to the target system

Since the chicken service does a charity service for the community, it needs to protect itself from being abused. Otherwise an attacker could send many invalid claims and the chicken service pays a lot of transaction fees. The goal is to pay for no more than one claim transaction per token holder. This can be achieved with the following off-chain checks:

1. Check format and validity of the claim string
2. Validate signature of the claim string
3. Derive the external address of the signer
4. Ensure the external address has tokens to be claimed (requires a query to the target system)
5. Limit claim transactions to one per external address

## Open questions

1. What kind of chain IDs exist in Substrate that can be used for the claim string?
