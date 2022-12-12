# learning to read an audit report:

## Table of content

Skim through detailed findings on table of content to have a broad idea of found issues

## Functionality overview

Read functionality overview to get a better understanding of what have been tested exactly, when protocols have a lot of contracts, audit reports might directly concerns only a subset of those.

in our case: it seems like the whole protocol has been audited: _Stargaze marketplace is a protocol owned by the chain governance that allows users to sell and buy NFTs with both auctions and direct offers._

## Severity Descritpion

Acknowledge issues severity description

## Summary of findings

Summary of findings table gives us info on issues severity/status at the end of the audit

```
Severity: Critical | Major | Minor | Informational
Status: Pending | Acknowledged | Resolved
```

## Code quality criteria

Audits not only gives insight on contract security but also code quality evaluating code complexity, readability, documentation and test coverage.

## Detailed Findings

1. _payout_ function may panic

### Description

whenever a sell happens, buyer funds are being split between multiple entities:

- seller
- royalty_recipient
- network
- finders

audit team found that there was no control over the sum of those shares, which could cause an underflow and then panic error.

Resolved by adding:

```
sum(royalty_recipient + network + finders) < 100
```

2. Misconfig stale bid duration could causes operator to be unable to remove stale bids

stale bid duration is a _Duration_ defined on instantiate and can be update by admin via sudoMsg, the issue comes from the fact that _Duration_ is an enum that can either be a _Time_ or _Height_ and that it fails to add _Time_ and _Height_.

Resolved by adding:

```
match msg.stale_bid_duration {
        Duration::Height(_) => return Err(ContractError::InvalidDuration {}),
        Duration::Time(_) => {}
    };
```

3. Bidder can specify unchecked finders fee
