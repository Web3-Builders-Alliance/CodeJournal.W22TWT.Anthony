# State:

### Single Items:

##### Config

```
pub struct Config {
    pub threshold: Threshold,
    pub total_weight: u64,
    pub max_voting_period: Duration,
}
```

where
`total_weight` would be the sum of weight, combined by all voters `Threshold` & `Duration` are enums imported from cw-utils and a respectively indication on how should the vote be counted and the full duration for a proposal

Threshold can either be :

- AbsoluteCount { weight: u64 }
- AbsolutePercentage { percentage: Decimal }
- ThresholdQuorum { threshold: Decimal, quorum: Decimal }

Duration can either be:

- Height(u64),
- Time(u64),

##### Proposal_count (u64)

### Multiple Items: Maps

##### BALLOTS [responsible for storing the votes]

associate a tuples (u64,addr) -> Ballot

where the tuple represent a pair (proposal_id, user_address) to a Ballot Struct which holds:

- pub weight: u64,
- pub vote: Vote,

##### PROPOSALS

associate a u64 -> Proposal

```
pub struct Proposal {
   pub title: String,
   pub description: String,
   pub start_height: u64,
   pub expires: Expiration,
   pub msgs: Vec<CosmosMsg<Empty>>,
   pub status: Status,
   /// pass requirements
   pub threshold: Threshold,
   // the total weight when the proposal started (used to calculate percentages)
   pub total_weight: u64,
   // summary of existing votes
   pub votes: Votes,
   /// The address that created the proposal.
   pub proposer: Addr,
   /// The deposit that was paid along with this proposal. This may
   /// be refunded upon proposal completion.
   pub deposit: Option<DepositInfo>,
}
```

Proposal Struct also comes with implemented method such as:

- update_status: sets the status of the proposal to current_status
- current_status: returns what the status should be
- is_passed: true if this proposal is sure to pass
- is_rejected: true if this proposal is sure to be rejected

##### VOTERS

associate an address -> proposal_id

##### More

also contains a helper method that gives next_proposal_id

# Msg

### Instantiate:

- voters: vec<Voter>
- threshold: Threshold
- max_voting_period: Duration

where voters is the list of voters and their respective weights for the multisig and threshold and max_voting_period will be used to setup config state.

### Execute:

- Propose -> create a proposal
- Vote -> vote on a proposal
- Execute -> execute a passed proposal
- Close -> close a proposal that didn't pass

### Query:

- Threshold -> Config.threshold
- Proposal -> Proposal
- ListProposals -> list of proposals (oldest first ?)
- ReverseProposals -> list of proposals (recent first?)
- Vote -> vote for a given address and proposal_id
- ListVotes -> list of votes for a given proposal_id
- Voter -> weight associated to a voter
- ListVoters -> list multisig voters

NB: all queries that returns list should implement pagination, by accepting
limit and start_after parameters, little twist with ReverseProposals that expects start_before instead of start_after.

NB²: the macro `#[returns(<ResponseType>)]` gives info on the response type for the query

in that specific contract almost every ResponseType has been extracted in the packages to be reuse easily elsewhere

### Errors

- Std(#[from] StdError),
- Threshold(#[from] ThresholdError),
- ZeroWeight {},
- UnreachableWeight {},
- NoVoters {},
- Unauthorized {},
- NotOpen {},
- Expired {},
- NotExpired {},
- WrongExpiration {},
- AlreadyVoted {},
- WrongExecuteStatus {},
- WrongCloseStatus {},

NB: `#[error(<ErrorMessage>)]` macro indicate which message will be returned when each errors occur

### Contract

Let's have a look at the implementation now:

#### Instantiate

1. if Voters is empty throw a NoVoters error
2. compute total weight -> sum(Voter.weight)
3. validate threshold
4. set contract version in case of future migration
5. set CONFIG
6. validate each Voter address and set VOTERS
   - iter() returns an iterable
7. Ok()

#### Execute implementations:

##### execute_propose

1. check if sender is registered as voter
   - use `may_load` to load VOTERS state
   - use `ok_or()` to throw specific error
2. load config
3. Set proposal expiration date
   - using `.after(<BlockHeight>)`
   - use `cmp` crate to compare values
   - throws if expiration is invalid
4. instantiate proposal
   - update prop.status to current.status [ why ?!]
   - use `next_id` method to generate proposal_id
   - save state
5. add first yes vote from 'proposer' in BALLOTS
   - step could be passed when proposer weight is 0 [i guess]

##### execute_vote

1. make sure sender is registered as Voter and that their voting_weight is >= 1
   - use `may_load` to load VOTERS
2. load Proposal and check its current status
   - voter allowed to vote on Open, Passed & Rejected proposals throw `NotOpen` otherwise
   - check expiration using `is_expired()` helper throws `Expired`
3. cast vote
   - if user already voted throw `AlreadyVoted`
   - else create a BALLOT
   - update proposal.votes
   - save proposal

##### execute_execute

1. load proposal
2. update_status
   - then check if status == Passed else throws `WrongExecuteStatus`
3. set proposal status to executed & save
4. add prop.msgs as messages in Response

##### execute_close

1. load proposal
2. check if status is 'closable' ie not Executed, Rejected nor Passed else throws `WrongCloseStatus`

   - something i'm missing here, not sure what is going on here ?!

   ```
    if [Status::Executed, Status::Rejected, Status::Passed].contains(&prop.status) {
       return Err(ContractError::WrongCloseStatus {});
   }
   // Avoid closing of Passed due to expiration proposals
   if prop.current_status(&env.block) == Status::Passed {
       return Err(ContractError::WrongCloseStatus {});
   }
   ```

3. check if proposal has expired else throws `NotExpired`
4. set proposal status to Rejected

#### Query implementations:

NB: pagination params:

```
MAX_LIMIT: u32 = 30;
DEFAULT_LIMIT: u32 = 10;
```

##### query_threshold

1. load CONFIG
2. returns total_weight

##### query_proposal(proposal_id)

1. load Proposal
2. use `prop.current_status(block)`
3. init a `ThresholdResponse`
4. returns `ProposalResponse`

##### query_vote(proposal_id, voter)

1. validate voter address
2. load Ballot
3. returns `VoteResponse`

##### list_proposals(start_after, limit)

1. initialize limit value
   - use `.unwrap_or(DEFAULT_LIMIT).min(MAX_LIMIT)`
2. initialize start value
   - using `start_after.map(Bound::exclusive);` start_after value which represent a proposal_id will be excluded from Response
3. Load Proposals range using pagination parameters

   ```
   let proposals = PROPOSALS
       .range(deps.storage, start, None, Order::Ascending)
       .take(limit)
       .map(|p| map_proposal(&env.block, p))
       .collect::<StdResult<_>>()?;
   ```

   where:

   - `range` gives an iterator
   - `take(n)` yields n elements
   - yielded elements are mapped to `ProposalResponse`
   - `collect()` transform iterator into collection

##### reverse_proposals(start_before, limit)

same logic as `list_proposals` query but with pagination reversed, ie:
init end value
min range is None, max is end & order is Descending

##### list_votes(proposal_id, start_after, limit)

1. init pagination values
2. Load Ballots
3. map each Ballot to VoteInfo
4. make a collection out of it
5. return `VoteListresponse`

##### query_voter(voter)

1. `addr_validate(vote)`
2. load Voter to get associated weight
3. return `VoterResponse`

##### list_voters(start_after, limit)

1. init pagination values
2. load Voters
3. map each Voter to `VoterDetail`
4. make a collection out of it
5. return `VoterListResponse`

#### Things i learned:

- how a cw3 multisig works
  - how to instantiate one
  - how to create | vote | execute | close a proposal
- env param contains info on blockHeight and can be used for Expiration
- how to use pagination in queries that return lists

#### Things i'm not sure i got right:

- proposal status pending doesn't seem to be used anywhere
- what's the use of `prop.current_status()` if it update status value to prop.current_status ??
- in execute_close why are there two tests that check if proposal.status == Passed
- what does .prefix() do [contracts.rs, l.368]
- difference between:
  - `.collect::<StdResult<_>>()?;`
  - `.collect();`
