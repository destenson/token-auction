# Continuous Splitting Token Auction

This Ethereum contract provides a set of auctions for use with
standard tokens.

The auction was designed for the [Maker] DAO, providing a simple and
passive liquidation mechanism to sell off collateral and debt.
However, the contract is generic and agnostic about its user
provided they adhere to the [token standard][ERC20].

[Maker]: https://makerdao.com
[ERC20]: https://github.com/EIPS/issues/20

## Description

The auction provides three auction types - regular *forward* and
*reverse* auctions and a composite *two-way* auction that switches
from forward to reverse once a threshold bid is reached.

In a **forward** auction, bidders compete on the amount they are
willing to pay for the lot.

In a **reverse** auction, bidders compete on the amount of lot they
are willing to receive for a given payment.

A **two-way** auction initially behaves as a forward auction. Once a
given quantity of the buy token has been bid the auction reverses
and becomes a reverse auction.

The auctions are **splitting**. Bidders can bid on a fraction of the
full lot, provided their bid is an increase in valuation. Doing so
*splits* the auction: the previous bidder has their bid quantity
reduced at the same valuation; subsequent bidders can bid on this
reduced quantity or on the new split quantity. There is no limit to
the number of times an auction can be split.

The splittable unit of an auction is an **auctionlet**. Each auction
initially has a single auctionlet. Auctionlets are the object on
which bidders place bids. Splitting an auctionlet produces two new
auctionlets and deletes the old one.

The auctions are **continuous**. The auction beneficiaries are
continually rewarded as new bids are made: by increasing amounts of
the buy token in a forward auction, and by increasing amounts of
forgone sell token in a reverse auction. Highest bidders are
continually rewarded as well: bids are locked once they have existed
for a given duration with no higher bids, at which point the highest
bidder can claim their dues.

The auctions are **managed**. An auction manager can manage an
unlimited number of auctions of multiple types.


## Usage

There are three ways of interacting with the auction contract:
[management](#management), [auction creation](#auction-creation),
and [bidding](#bidding). Auction users should subscribe to the
[auction events](#events) to be kept aware of auction creation, new
bids, splits and reversals.

### Management

Create a new splitting auction manager:

```
manager = new SplittingAuctionManager();
```

Create a new non-splitting auction manager:

```
manager = new AuctionManager();
```

In a non-splitting auction bidders must bid on the full lot.

The creator of the manager has no special permissions - they
cannot shutdown or withdraw funds from the manager.


### Auction creation

All created auctions return a pair of `uint (id , base)` that
uniquely identify the new auction and its base auctionlet. These
identifiers are used when bidding and claiming.

Note: you will not be able to use named arguments on auction
creation due to an upstream [solidity issue][name-args-issue].

[named-args-issue]: TODO

Create a new **forward** auction:

```
var (id, base) = manager.newAuction( beneficiary
                                   , sell_token
                                   , buy_token
                                   , sell_amount
                                   , start_bid
                                   , min_increase
                                   , duration
                                   )
```

- `address beneficiary` is an address to send auction proceeds to,
  i.e. it will receive `buy_token`.
- `ERC20 buy_token` is the standard token that bidders use to pay with.
- `ERC20 sell_token` is the standard token that the bidders are
  bidding to receive.
- `uint sell_amount` is the amount of `sell_token` to be taken from
  the creator into escrow. The creator must `approve` this amount
  before creating the auction.
- `uint start_bid` is the minimum bid on the auction. The first bid
  must be at least this much plus the minimum increase.
- `uint min_increase` is the integer percentage amount that each bid
  must increase on the last by (in terms of `buy_token`).
- `uint duration` is the time after which a bid will be locked and
  claimable by its highest bidder.


Create a new **reverse** auction:

```
var (id, base) = manager.newReverseAuction( beneficiary
                                          , sell_token
                                          , buy_token
                                          , max_sell_amount
                                          , buy_amount
                                          , min_decrease
                                          , duration
                                          )
```

Arguments are as for the forward auction with the following extras:

- `uint max_sell_amount` is the maximum amount of `sell_token` that
  the creator is willing to sell, from which bids work downwards.
  This will be taken from the creator on initialisation.
- `uint buy_amount` is the amount of `buy_token` to be paid for the
  full lot, whatever that ends up being.
- `uint min_decrease` is the integer percentage amount that each bid
  must decrease on the last by (in terms of `sell_token`).
- The `beneficiary` receives at most `buy_amount` of the
  `buy_token` and also receives `sell_token` as it is forgone by
  bidders.


Create a new **two-way** auction:

```
var (id, base) = manager.newTwoWayAuction( beneficiary
                                         , sell_token
                                         , buy_token
                                         , sell_amount
                                         , start_bid
                                         , min_increase
                                         , min_decrease
                                         , duration
                                         , collection_limit
                                         )
```

Arguments are as for the forward and reverse auctions with the
following extras:

- `uint collection_limit` is the total bid quantity of `buy_token`
  at which the auction will reverse.



### Bidding

**Bid** on an auctionlet:

```
manager.bid(base, bid_amount)
```

- `uint base` is the bid identifier described above.
- `uint bid_amount`

This will throw if the `bid_amount` is not greater than the last bid
by `min_increase`. The `bid_amount` is transferred from the bidder,
so they must `approve` it first. The excess `buy_token` given by the
bid (over the last) is sent directly to the `beneficiary`.


**Split** an auctionlet:

```
var (new_id, split_id) = manager.bid(base, bid_amount, split_amount)
```

- `uint split_amount` is the reduced quantity of `sell_token` on
  which the new bid is being made.

Splitting returns a pair of identifiers:

- `uint split_id` refers to the auctionlet on which the caller is
  now the highest bidder.
- `uint new_id` refers to the auctionlet on which the previous
  bidder remains as the highest bidder (bidding for a reduced amount
  of `sell_token`).


**Claim** an elapsed auctionlet:

```
manager.claim(base)
```

This will send the `sell_token` associated with `base` to the
highest bidder. `claim` will throw if `duration` has not elapsed
since the last high bid on `base`.


### Events

**Creation** of a new auction:

```
NewAuction(uint indexed id, uint base_id)
```

- `uint id` is the auction id.
- `uint base_id` is the base auctionlet_id (on which bidding should
  start)

**Reversal** of a two-way auction:

```
AuctionReversal(uint indexed auction_id)
```

- `uint auction_id` is the id of the auction that has been reversed.


Successful new **bid** on an auctionlet:

```
Bid(uint indexed auctionlet_id)
```

- `uint auctionlet_id` is the id of the auctionlet that has been bid
  on.

Successful **split** of an auctionlet:

```
Split(uint base_id, uint new_id, uint split_id)
```

- `uint base_id` is the id of the auctionlet that has been split
  (and deleted).
- `uint new_id` is the id of the new auctionlet that the previous
  bidder retains.
- `uint split_id` is the id of the new auctionlet that the splitter
  is now the high bidder on.


## Advanced Usage

### Multiple beneficiaries

Forward and two-way auctions can be configured to have multiple
beneficiary addresses, each of which will receive, in turn, a given
payout of auction rewards as higher bids come in.

Create a *multiple beneficiary* auction:

```
var (id, base) = manager.newAuction( beneficiaries
                                   , payouts
                                   , sell_token
                                   , buy_token
                                   , sell_amount
                                   , start_bid
                                   , min_increase
                                   , duration);
```

- `address[] beneficiaries` is the array of beneficiary addresses
- `uint[] payouts` is the array of corresponding payouts

Note these constraints:

- `beneficiaries` and `payouts` must be equal in length
- `payouts[0]` must be greater than the `start_bid`

The two-way auction is created similarly, with the sum of the
payouts being taken to be the `collection_limit`:

```
var (id, base) = manager.newTwoWayAuction( beneficiaries
                                         , payouts
                                         , sell_token
                                         , buy_token
                                         , sell_amount
                                         , start_bid
                                         , min_increase
                                         , min_decrease
                                         , duration
                                         );
```

The beneficiaries array is only used in the forward auction or in
the forward part of the two-way auction.


### Reverse auction refund address

In the reverse auction (or the reverse part of the two-way auction),
bidders compete to receive diminishing amounts of the `sell_token`
in return for given `buy_token`. As bids come in, increasing
quantities of the `sell_token` are forgone by bidders.

The default behaviour is to refund this excess `sell_token` to the
(first) beneficiary. It is possible for the auction *creator* to set
the refund address arbitrarily.

After creating an auction:

```
manager.setRefundAddress(auction_id, some_arbitrary_address);
```

Note that this address can be set any number of times to any number
of addresses during the course of the auction. Accordingly a
beneficiary expecting funds needs to trust, or be controlled by, the
auction creator.
