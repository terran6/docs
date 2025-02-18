# Slashing

:::{Important}
Terra's slashing module inherits from the Cosmos SDK's [`slashing`](https://docs.cosmos.network/master/modules/slashing/) module. This document is a stub and covers mainly important Terra-specific notes about how it is used.
:::

The slashing module enables Terra to disincentivize any attributable action by a protocol-recognized actor with value at stake by penalizing them. The penalty is called [slashing](../../learn/glossary.md#slashing). Terra mainly uses the [`Staking`](spec-staking.md) module to slash when violating validator responsibilities. This module manages lower-level penalties at the Tendermint consensus level, such as double-signing.

## Message Types

### MsgUnjail

```go
type MsgUnjail struct {
    ValidatorAddr sdk.ValAddress `json:"address" yaml:"address"` // address of the validator operator
}
```

## Transitions

### Begin-Block

> This section was taken from the official Cosmos SDK docs, and placed here for your convenience to understand the slashing module's parameters.

At the beginning of each block, the slashing module checks for evidence of infractions or downtime of validators, double-signing, and other low-level consensus penalties.

#### Evidence handling

Tendermint blocks can include evidence, which indicates that a validator committed malicious
behavior. The relevant information is forwarded to the application as ABCI Evidence
in `abci.RequestBeginBlock` so that the validator an be punished.

For some `Evidence` submitted in `block` to be valid, it must satisfy:

`Evidence.Timestamp >= block.Timestamp - MaxEvidenceAge`

where `Evidence.Timestamp` is the timestamp in the block at height
`Evidence.Height`, and `block.Timestamp` is the current block timestamp.

If valid evidence is included in a block, the validator's stake is reduced by
some penalty (`SlashFractionDoubleSign` for equivocation) of what their stake was
when the infraction occurred instead of when the evidence was discovered. We
want to follow the stake, i.e. the stake which contributed to the infraction
should be slashed, even if it has since been redelegated or has started unbonding.

The unbondings and redelegations from the slashed validator are looped through, and the amount of stake that has moved is tracked:

```go
slashAmountUnbondings := 0
slashAmountRedelegations := 0

unbondings := getUnbondings(validator.Address)
for unbond in unbondings {

    if was not bonded before evidence.Height or started unbonding before unbonding period ago {
        continue
    }

    burn := unbond.InitialTokens * SLASH_PROPORTION
    slashAmountUnbondings += burn

    unbond.Tokens = max(0, unbond.Tokens - burn)
}

// only care if source gets slashed because we're already bonded to destination
// so if destination validator gets slashed the delegation just has same shares
// of smaller pool.
redels := getRedelegationsBySource(validator.Address)
for redel in redels {

    if was not bonded before evidence.Height or started redelegating before unbonding period ago {
        continue
    }

    burn := redel.InitialTokens * SLASH_PROPORTION
    slashAmountRedelegations += burn

    amount := unbondFromValidator(redel.Destination, burn)
    destroy(amount)
}
```

The validator is slashed and [tombstoned](../../learn/glossary.md#tombstone):

```
curVal := validator
oldVal := loadValidator(evidence.Height, evidence.Address)

slashAmount := SLASH_PROPORTION * oldVal.Shares
slashAmount -= slashAmountUnbondings
slashAmount -= slashAmountRedelegations

curVal.Shares = max(0, curVal.Shares - slashAmount)

signInfo = SigningInfo.Get(val.Address)
signInfo.JailedUntil = MAX_TIME
signInfo.Tombstoned = true
SigningInfo.Set(val.Address, signInfo)
```

This process ensures that offending validators are punished with the same amount whether they act as a single validator with X stake or as N validators with a collective X stake. The amount slashed for all double-signature infractions committed within a single slashing period is capped. For more information, see [tombstone caps](https://docs.cosmos.network/master/modules/slashing/01_concepts.html#tombstone-caps).

#### Liveness tracking

At the beginning of each block, the `ValidatorSigningInfo` for each
validator is updated and whether they've crossed below the liveness threshold over a
sliding window is checked. This sliding window is defined by `SignedBlocksWindow`, and the
index in this window is determined by `IndexOffset` found in the validator's
`ValidatorSigningInfo`. For each block processed, the `IndexOffset` is incremented
regardless of whether the validator signed. After the index is determined, the
`MissedBlocksBitArray` and `MissedBlocksCounter` are updated accordingly.

Finally, to determine whether a validator crosses below the liveness threshold,
the maximum number of blocks missed, `maxMissed`, which is
`SignedBlocksWindow - (MinSignedPerWindow * SignedBlocksWindow)`, and the minimum
height at which liveness can be determined, `minHeight`, are fetched. If the current block is
greater than `minHeight` and the validator's `MissedBlocksCounter` is greater than
`maxMissed`, they are slashed by `SlashFractionDowntime`, jailed
for `DowntimeJailDuration`, and have the following values reset:
`MissedBlocksBitArray`, `MissedBlocksCounter`, and `IndexOffset`.

:::{Important}
Liveness slashes do not lead to tombstombing.
:::

```go
height := block.Height

for vote in block.LastCommitInfo.Votes {
  signInfo := GetValidatorSigningInfo(vote.Validator.Address)

  // This is a relative index, so it counts blocks the validator SHOULD have
  // signed. The 0-value default signing info is used if no signed block is present, except for
  // start height.
  index := signInfo.IndexOffset % SignedBlocksWindow()
  signInfo.IndexOffset++

  // Update MissedBlocksBitArray and MissedBlocksCounter. The MissedBlocksCounter
  // just tracks the sum of MissedBlocksBitArray to avoid needing to
  // read/write the whole array each time.
  missedPrevious := GetValidatorMissedBlockBitArray(vote.Validator.Address, index)
  missed := !signed

  switch {
  case !missedPrevious && missed:
    // array index has changed from not missed to missed, increment counter
    SetValidatorMissedBlockBitArray(vote.Validator.Address, index, true)
    signInfo.MissedBlocksCounter++

  case missedPrevious && !missed:
    // array index has changed from missed to not missed, decrement counter
    SetValidatorMissedBlockBitArray(vote.Validator.Address, index, false)
    signInfo.MissedBlocksCounter--

  default:
    // array index at this index has not changed; no need to update counter
  }

  if missed {
    // emit events...
  }

  minHeight := signInfo.StartHeight + SignedBlocksWindow()
  maxMissed := SignedBlocksWindow() - MinSignedPerWindow()

  // If the minimum height has been reached and the validator has missed too many
  // jail and slash them.
  if height > minHeight && signInfo.MissedBlocksCounter > maxMissed {
    validator := ValidatorByConsAddr(vote.Validator.Address)

    // emit events...

    // To retrieve the stake distribution which signed the block,
    // subtract ValidatorUpdateDelay from the block height, and subtract an
    // additional 1 since this is the LastCommit.
    //
    // Note, that this CAN result in a negative "distributionHeight" up to
    // -ValidatorUpdateDelay-1, i.e. at the end of the pre-genesis block (none) = at the beginning of the genesis block.
    // That's fine since this is just used to filter unbonding delegations & redelegations.
    distributionHeight := height - sdk.ValidatorUpdateDelay - 1

    Slash(vote.Validator.Address, distributionHeight, vote.Validator.Power, SlashFractionDowntime())
    Jail(vote.Validator.Address)

    signInfo.JailedUntil = block.Time.Add(DowntimeJailDuration())

    // Reset the counter & array so that the validator won't be
    // immediately slashed for downtime upon rebonding.
    signInfo.MissedBlocksCounter = 0
    signInfo.IndexOffset = 0
    ClearValidatorMissedBlockBitArray(vote.Validator.Address)
  }

  SetValidatorSigningInfo(vote.Validator.Address, signInfo)
}
```

## Parameters

The subspace for the slashing module is `slashing`.

```go
type Params struct {
	MaxEvidenceAge          time.Duration `json:"max_evidence_age" yaml:"max_evidence_age"`
	SignedBlocksWindow      int64         `json:"signed_blocks_window" yaml:"signed_blocks_window"`
	MinSignedPerWindow      sdk.Dec       `json:"min_signed_per_window" yaml:"min_signed_per_window"`
	DowntimeJailDuration    time.Duration `json:"downtime_jail_duration" yaml:"downtime_jail_duration"`
	SlashFractionDoubleSign sdk.Dec       `json:"slash_fraction_double_sign" yaml:"slash_fraction_double_sign"`
	SlashFractionDowntime   sdk.Dec       `json:"slash_fraction_downtime" yaml:"slash_fraction_downtime"`
}
```

## Genesis parameters

The genesis parameters for the crisis module outlined in the [Genesis Builder Script](https://github.com/terra-money/genesis-tools/blob/main/src/genesis_builder.py#L168) are as follows:

``` py
    # Slashing: slash window setup
    genesis['app_state']['slashing']['params'] = {
        'downtime_jail_duration': '600s',
        'min_signed_per_window': '0.05',
        'signed_blocks_window': '10000',
        'slash_fraction_double_sign': '0.05',  # 5%
        'slash_fraction_downtime': '0.0001'    # 0.01%
```
```py
    # Consensus Params: Evidence
    genesis['consensus_params']['evidence'] = {
        'max_age_num_blocks': '100000',
        'max_age_duration': '172800000000000',
        'max_bytes': '1000000'
    }

```

### SignedBlocksWindow

- type: `int64`
- default: `100`

### MinSignedPerWindow

- type: `Dec`
- default: `.05`

### DowntimeJailDuration

- type: `time.Duration` (seconds)
- default:  600s

### SlashFractionDoubleSign

- type: `Dec`
- default: 5%

### SlashFractionDowntime

- type: `Dec`
- default: .01%
