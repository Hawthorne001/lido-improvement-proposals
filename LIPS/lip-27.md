---
lip: 27
title: External to the protocol ether supply for stETH
status: WIP
author: Alexey Potapkin, Eugene Mamin, Eugene Pshenichnyi
discussions-to: TBA
created: 2024-12-06
updated: 2024-12-06
---

# External to the protocol ether supply for stETH

## Simple Summary

Introduce extended accounting principles for the Lido on Ethereum protocol, enabling the protocol to manage stETH issuance and redemption backed by ether (ETH) that is external to the Lido core accounting scope.

These rules allow for the controlled minting and burning of external stETH shares, ensuring that external ether sources become part of the total ether backing stETH, while preserving core invariants and accounting integrity.

## Abstract

The Lido on Ethereum protocol currently tracks ether under its direct management — such as buffered ether, validators’ balances on the Consensus Layer (CL), and transient validator states — forming the protocol's "total pooled ether".

This proposal extends the accounting model to include an additional ether supply external to the protocol’s direct purview.

### Mint external shares

A newly introduced "external accounting contract" (trusted by the Lido core contract) is allowed to mint stETH shares to a specified recipient. The amount of external stETH shares minted is fully backed by an equivalent external ether amount (abstracted as an external balance). This increases the protocol’s stored `externalBalance` value, thus raising the total ether backing stETH.

If we denote:

- $\Delta S_{\text{ext}}$ as the number of external shares to be minted,
- $\Delta ETH_{\text{ext}}$ as the corresponding external ether amount,
- $\text{shareRate} = \frac{\text{totalPooledEther}}{\text{totalShares}}$,

then the relationship between shares and ether is given by:

$$
\Delta ETH_{\text{ext}} = \Delta S_{\text{ext}} \times \text{shareRate}.
$$

Inversely, if the external ether contribution $\Delta ETH_{\text{ext}}$ is known:

$$
\Delta S_{\text{ext}} = \frac{\Delta ETH_{\text{ext}}}{\text{shareRate}}.
$$

After minting, the external balance is updated as:

$$
\text{externalBalance}_{\text{new}} = \text{externalBalance}_{\text{old}} + \Delta ETH_{\text{ext}}.
$$

### Burn external shares

The external accounting contract can also burn a specified amount of stETH shares from its own balance. This operation represents a redemption of externally backed ether from the system’s perspective, decreasing the stored `externalBalance` value by the corresponding ether amount.

For burning:

- $\Delta S_{\text{ext}}$ is the number of external shares to burn,
- The corresponding ether amount to remove from external balance is:

$$
\Delta ETH_{\text{ext, burn}} = \Delta S_{\text{ext}} \times \text{shareRate}.
$$

Thus, the external balance becomes:

$$
\text{externalBalance}_{\text{new}} = \text{externalBalance}_{\text{old}} - \Delta ETH_{\text{ext, burn}}.
$$

### stETH token rebase

The total pooled ether (`totalPooledEther`) considering the external balance is:

$$
\text{totalPooledEther} = \text{bufferedEther} + \text{CLbalance} + \text{transientBalance} + \text{externalBalance}.
$$

The share rate is defined as:

$$
\text{shareRate} = \frac{\text{totalPooledEther}}{\text{totalShares}}.
$$

Any changes in `externalBalance` affect `totalPooledEther` and thus the `shareRate` during a rebase.

### Reward and fees distribution

> TODO

Rewards and fees are distributed proportionally to all stETH holders, including those stETH shares minted from external sources. As `externalBalance` changes the total pooled ether, all stETH holders benefit (or incur penalties) uniformly according to the updated `shareRate`.

## Motivation

- Flexibility: integrations and building primitives that manage ether outside of the Lido on Ethereum protocol may still want to represent that ether as stETH shares.
- Uniform accounting: by incorporating external ether into the regular stETH rebase mechanics, the entire system remains consistent and leverages existing logic.
- Broader options and future compatibility: the approach suggested fosters new integrations, use cases, and products that manage validator sets or external vaults while still benefiting from Lido’s liquidity and staking model.

## Design assumptions and invariants

- External ether as real backing: external shares are always intended to be fully backed by genuine external ether. The system treats this external ether similarly to protocol-held ether in terms of calculation for stETH's `shareRate`.
- No spurious rebases: minting or burning external shares does not by itself trigger a rebase. Rebases remain driven by the AccountingOracle reporting only.
- Bounded Access: Only a designated core contract for external accounting, trusted by the Lido core contract, can invoke external share minting/burning.

### Risks not concerned

TBA

### Risks addressed

TBA

### Invariants upheld

- No arbitrary inflation: external shares can only be minted if there is a corresponding external ether contribution.
- Accurate `shareRate` maintenance: external balance changes are integrated into the total pooled ether calculation, preserving the stETH's `shareRate` formula.
- Restricted burn operations: stETH burning is restricted to known burners (the Burner contract for internal operations and the external accounting contract for external shares), avoiding arbitrary destruction of shares.

## Specification

### Mint external shares

#### Function: `mintExternalShares(address receiver, uint256 sharesAmount)`

    Input:
        `receiver`: The address that will receive the newly minted external stETH.
        `sharesAmount`: The number of shares to mint.
    Actions:
        - Compute the corresponding external ether amount `deltaExternalBalance` at the current stETH's `shareRate`.
        - Increase `externalBalance` by `deltaExternalBalance`.
        - Mint `sharesAmount` of stETH to receiver.

### Burn external shares

#### Function: `burnExternalShares(uint256 sharesAmount)`

    Input:
        `sharesAmount`: The number of shares to burn from the caller’s balance.
    Actions:
        - Compute the external ether redemption ΔETHext, burnΔETHext, burn​.
        - Decrease `externalBalance` by ΔETHext, burnΔETHext, burn​.
        - Burn `sharesAmount` of stETH from the caller’s balance.

#### Function: `burnExternalShares`

## Rationale

### Abstracting external ether

The design does not require the protocol to 'understand' the nature of external ether. It only requires trusted external accounting, ensuring that the claimed external ether is credible. The external ether could represent aggregator-controlled validator balances or, potentially, other complex DeFi arrangements.

### Why track external shares instead of external ether directly

stETH is defined by its share mechanics, and all protocol logic revolves around shares and their rate. By expressing external contributions and withdrawals in terms of stETH shares, the protocol remains consistent. Ether is only an input or output measure, while shares define the in-protocol liquidity and distribution rules.

### Why extend rebase

Incorporating external ether into rebase ensures that the `shareRate` calculation remains a single cohesive formula. Without merging external ether into rebase logic, the system would have to manage multiple share rate domains, creating fragmentation and complexity.

### Rewards and fees

Because all stETH holders share the same `shareRate`, any changes to total pooled ether (including externally sourced rewards or penalties) are proportionately distributed. This ensures fairness and uniformity in how the protocol accounts for and distributes external inputs (socialization angle).

## Security considerations

### Implementation risk

A faulty implementation could allow malicious actors to mint or burn stETH at will, breaking the token’s economic model. Strong code audits, well-defined access controls, and thorough sophisticated testing are essential.

### Staking limits and pause

Pausing staking is unaffected by external shares per se. However, if the protocol is under stress or paused, external share minting/burning might also be restricted.

### One new minter and one new burner for stETH

This LIP introduces a new class of minter and burner (the external accounting contract), increasing the importance of access control, deployment configuration, and monitoring.

The burning is allowed only from the own external accounting contract balance.

## Failure modes

### External ether drained

If external ether sources fail or experience a slash, external shares remain in circulation. The `shareRate` logic naturally accounts for losses when the next rebase updates the `externalBalance`, thus stETH holders' in the shared risk of losses.

## Reference implementation

A preliminary draft of the implementation is available in the [feat: Vaults](https://github.com/lidofinance/core/pulls).

## Links and references

- [Lido rebase documentation](https://docs.lido.fi/contracts/lido#rebase)
- [stETH share mechanics](https://docs.lido.fi/guides/lido-tokens-integration-guide#steth-internals-share-mechanics)
