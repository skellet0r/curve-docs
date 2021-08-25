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

Voting Escrow Boost Delegation
==============================

The veBoost contract is `ERC721-compliant <https://eips.ethereum.org/EIPS/eip-721>`_ with added enumeration and metadata extensions, for more details on the ERC721 interface please visit https://eips.ethereum.org/EIPS/eip-721.

Querying Balances and Boost Details
===================================

.. py:function:: BoostDelegation.adjusted_balance_of(_addr: address) -> uint256: view

    Adjusted veCRV balance after accounting for delegations and boosts. If an account has no delegations or received boosts, this will return the same value as the veCRV balance of the account.

    * ``_addr``: Account of interest

        .. code-block:: python

            >>> veboost = Contract('0xc620aaFD6Caa3Cb7566e54176dD2ED1A81d05655')
            >>> veboost.adjusted_balance_of('0xF89501B77b2FA6329F94F5A05FE84cEbb5c8b1a0')
            5464191329389144503333564

.. py:function:: BoostDelegation.delegated_boost(_addr: address) -> uint256: view

    Query the total effective (absolute value) delegated boost value of an account. 

    * ``_addr``: Account of interest

        .. code-block:: python

            >>> veboost.delegated_boost('0xB9fC157394Af804a3578134A6585C0dc9cc990d4')
            7302246215171192832
            
.. py:function:: BoostDelegation.received_boost(_addr: address) -> uint256: view

    Query the total effective (maximum of either the value or 0) received boost value of an account. 

    * ``_addr``: Account of interest

        .. code-block:: python

            >>> veboost.received_boost('0xB9fC157394Af804a3578134A6585C0dc9cc990d4')
            0

.. py:function:: BoostDelegation.token_boost(_token_id: uint256) -> int256: view

    Query the effective value of a boost, this will be negative if the token is past it's expiration. 

    * ``_token_id``: The token id to query

        .. code-block:: python

            >>> veboost.token_boost(84123270500954000169498590239642917020204751406968249562230920786212529635331)
            8461023254675706880

.. py:function:: BoostDelegation.token_expiry(_token_id: uint256) -> uint256: view

    Query the timestamp of a boost token's expiry. If the token has been cancelled this will equal 0.

    * ``_token_id``: The token id to query

        .. code-block:: python

            >>> veboost.token_expiry(84123270500954000169498590239642917020204751406968249562230920786212529635331)
            1635722896

.. py:function:: BoostDelegation.token_cancel_time(_token_id: uint256) -> uint256: view

    Query the timestamp of a boost token's cancel time. This is the point at which the delegator can nullify the boost. A receiver can cancel a token at any point. Anyone can nullify a token's boost after it's expiration.

    * ``_token_id``: The token id to query

        .. code-block:: python

            >>> veboost.token_cancel_time(84123270500954000169498590239642917020204751406968249562230920786212529635331)
            1632526172

.. py:function:: BoostDelegation.get_token_id(_delegator: address, _id: uint256) -> uint256: pure

    Simple method to get the token id's mintable by a delegator. This is equivalent to left shifting the delegator address 96 bits and adding `_id`. 

    * ``_delegator``: The delegator account address
    * ``_id``: The id, must be less than 2 ** 96

        .. code-block:: python

            >>> veboost.get_token_id("0xF89501B77b2FA6329F94F5A05FE84cEbb5c8b1a0", 2)
            112436858509691644084087600949642065935449759732829008227238981820833689763842

.. py:function:: BoostDelegation.calc_boost_bias_slope(_delegator: address, _percentage: int256, _expire_time: int256[, _extend_token_id: uint256 = 2 ** 96]) -> (int256, int256): view

    Calculate the bias and slope for a boost token, taking into account the delegators current veCRV balance.

    This is most useful calculate the slope and bias of the equation representing a boost token. If given the optional _extend_token_id parameter, the token with that id, will be effectively nullfiied before calculating the bias and slope.

    * ``_delegator``: The account to delegate boost from
    * ``_percentage``: The percentage of the _delegator's delegable veCRV to delegate
    * ``_expire_time``: The time at which the boost value of the token will reach 0, and subsequently become negative
    * ``_extend_token_id``: OPTIONAL id of delegators tokens, which if set will first nullify the boost of the token, before calculating the bias and slope. Useful for calculating the new bias and slope when extending a token, or determining the bias and slope of a subsequent token after cancelling an existing one. Will have no effect if _delegator is not the delegator of the token.

        .. code-block:: python

            >>> veboost.calc_boost_bias_slope("0xF89501B77b2FA6329F94F5A05FE84cEbb5c8b1a0", 10_000, 1632526995)
            (945874996724549775572359708672, -9559339358802321408)

