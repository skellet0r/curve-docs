.. _dao-veboost:

=============================================
Curve DAO: Vote-Escrowed CRV Boost Delegation
=============================================

Previously, the only way to receive boosted rewards on provided liquidity was to acquire CRV and then vote-lock it, exclusively from an EOA. However, with veCRV Boost Delegation (veBoost), veCRV holders can now create veBoost tokens and effectively delegate their boost (on eligible gauges) to any account (even contracts), while still retaining their governance abilities and access to their entitled portion of platform wide trading fees.

Implementation Details
======================

The Voting Escrow Boost Delegation contract is an `ERC721-compliant <https://eips.ethereum.org/EIPS/eip-721>`_ smart contract, which maintains the equations representing each accounts delegated and received boosts. Each veBoost tokens is effectively a linear equation represented as an NFT, which is freely tradeable, but only mintable by an account with veCRV (or their approved operator).

Internally the veBoost contract calculates the adjusted balance of an account by taking the accounts vanilla veCRV balance, subtracting the absolute value of an account's delegated boost, and adding the maximum of either their received boost or 0.

:math:`\text{adjusted_balance}(account) = \text{vecrv_balance}(account) - abs(\text{delegated_boost}(account)) + max(\text{received_boost}(account), 0)`

It's important to note that, we are always dealing with constantly decreasing linear equations for `vecrv_balance`, `delegated_boost`, and `received_boost`. We use these linear equations to eliminate the need for iteration, specifically the need to iterate over all the veBoost tokens an account either possesses or has delegated, in order to calculate their adjusted balance. However, since we use linear equations, there is a point in time at which the effective value of a veBoost token becomes negative, and this is at it's expiry time.

When minting or extending a veBoost token, delegators (or their operators) have the ability to set the expiry time of a token, with a minimum expiry time of 1 week, and the maximum being the end of the delegator's veCRV lock. What this means for delegators is, after a token has expired, it will become negatively valued and unless it is cancelled/extended delegators will not be able to mint new veBoost tokens or extend existing ones. Delegators will also be negatively impacted since we calculate adjusted boost with the absolute value of `delegated_boost`, negative outstanding veboost tokens will eat away at a delegators received boost as well as their veCRV balance until dealt with.

For receivers of veBoost tokens, since we use linear equations, there is no limitation on the number of veboost tokens one can be in possession of. However, negatively valued veboost tokens in ones possession will reduce a receiver's recevied boost balance. If a receiver's sum of veboost tokens equates to a negative number, it is worth 0 when calculating their adjusted balance, and they will effectively default to their veCRV balance (unless they have delegated boost as well).

For delegators, the amount of boost available to delegate can be calculated by subtracting their default veCRV balance by the absolute value of their outstanding delegated boosts. This prevents a delegator from delegating boost they have received, and by using the absolute value of their outstanding delegated boosts, their balance is not inflated by negatively valued outstanding boosts.
