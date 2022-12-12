Keeps a record of minters, their associated parameters and status,
minters are here responsible for minting sg721.

# State:

```
pub type Extension = Option<Empty>;

pub type BaseMinterParams = MinterParams<Extension>;

pub const SUDO_PARAMS: Item<BaseMinterParams> = Item::new("sudo-params");
```

where MinterParams comes from sg2 packages

# Msg

introduce dependencies from sg2

```
#[cw_serde]
pub struct CreateMinterMsg<T> {
    pub init_msg: T,
    pub collection_params: CollectionParams,
}

#[cw_serde]
pub struct CollectionParams {
    /// The collection code id
    pub code_id: u64,
    pub name: String,
    pub symbol: String,
    pub info: CollectionInfo<RoyaltyInfoResponse>,
}

#[cw_serde]
pub struct UpdateMinterParamsMsg<T> {
    /// The minter code id
    pub code_id: Option<u64>,
    pub creation_fee: Option<Coin>,
    pub min_mint_price: Option<Coin>,
    pub mint_fee_bps: Option<u64>,
    pub max_trading_offset_secs: Option<u64>,
    pub extension: T,
}

#[cw_serde]
pub enum Sg2ExecuteMsg<T> {
    CreateMinter(CreateMinterMsg<T>),
}
```

which suggests that all those params can be updated by governance

### Instantiate:

```
pub struct InstantiateMsg {
    pub params: BaseMinterParams,
}
```

Provide minter parameters for this factory instance

### Execute:

#### CreateMinter {BaseMinterCreateMsg}

instantiate a base-minter contract

### Query:

#### Params {}

returns ParamsResponse

### Errors:

```
    #[error("{0}")]
    Std(#[from] StdError),

    #[error("{0}")]
    Fee(#[from] FeeError),

    #[error("{0}")]
    Payment(#[from] PaymentError),

    #[error("Unauthorized")]
    Unauthorized {},

    #[error("InvalidDenom")]
    InvalidDenom {},
```

# Contract

#### Instantiate

1. Set contract version for future migration
2. Save sudo params in state

#### Execute implementations:

##### Execute create_minter

1. use must_pay helper => Returns the amount if only one denom and non-zero amount. Errors otherwise.

2. use check_fair_burn() => related to stargaze transaction fee management (fees are divided, a part is burned and the other is distributed)

3. create instantiate message for minter contract

4. send Ok and add instantiate message into response

#### Query implementations:

returns sudo params loaded from state

#### Sudo:

This seems to be a new entry_point
according to https://docs.cosmwasm.com/docs/1.0/smart-contracts/sudo/
this entry point can only be called by trusted/allowed sources

in our case, sudo entry point is used to update factory parameters

##### Update Params:

update sudo params based on updateMessage:

- if contains value then update it in sudo params using unwrap_or()

```
 params.code_id = param_msg.code_id.unwrap_or(params.code_id);
```

- enforce denom for creation_fee and mint_price

```
if let Some(creation_fee) = param_msg.creation_fee {
        ensure_eq!(
            &creation_fee.denom,
            &NATIVE_DENOM,
            ContractError::InvalidDenom {}
        );
        params.creation_fee = creation_fee;
    }

    if let Some(min_mint_price) = param_msg.min_mint_price {
        ensure_eq!(
            &min_mint_price.denom,
            &NATIVE_DENOM,
            ContractError::InvalidDenom {}
        );
        params.min_mint_price = min_mint_price;
    }
```

#### Things i learned:

- some entry point are not directly callable by wallet or contracts, sudo can not be called by external users or contracts, but only trusted (native/Go) code in the blockchain
- how governance execute proposal works

- this contract is central to stargaze activity as it's the one instantiating minter which will be able to mint sg721 for their collections.

#### Things i'm not sure i got right:

- at first msg.rs file was quite confusing as almost all it contains is related to sudo params update which is not in the execute entry point
