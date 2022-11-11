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
7. Ok()
