# Deployment & Setup Checklist

This is the full bring-up sequence for a Pythia deployment. Contracts must be **deployed in dependency order**, then wired together, then have permissions set. Every step maps to a real action on the contracts described in the [Contract Reference](contract-reference/README.md).

{% hint style="info" %}
Angle-bracket names (`<oracle>`, `<finder>`, …) are the accounts you deploy to. On testnet you can use short liveness/phase values (down to the 30 s / 30 s minimums) to iterate quickly; production values are noted inline.
{% endhint %}

## 1. Deploy contracts (in order)

| # | Account | Contract |
|---|---|---|
| 1 | `<token>` | `oovp.token` |
| 2 | `<finder>` | `oovp.finder` |
| 3 | `<store>` | `oovp.store` |
| 4 | `<stake>` | `oovp.stake` |
| 5 | `<voting>` | `oovp.voting` |
| 6 | `<oracle>` | `oovp.oracle` |
| 7 | `<govern>` | `oovp.govern` |
| 8 | `<reward>` | `oovp.reward` (optional; for app reward integrations) |

## 2. Create and issue the token

```bash
cleos push action <token> create '["<issuer>","100000000.0000 OOVP"]' -p <token>@active
cleos push action <token> issue  '["<issuer>","100000000.0000 OOVP","genesis"]' -p <issuer>@active
```

## 3. Register interfaces in the finder

```bash
cleos push action <finder> setaddress '["oracle","<oracle>"]'          -p <finder>@active
cleos push action <finder> setaddress '["votingtoken","<token>"]'      -p <finder>@active
cleos push action <finder> setaddress '["store","<store>"]'            -p <finder>@active
cleos push action <finder> setaddress '["staking","<stake>"]'          -p <finder>@active
cleos push action <finder> setaddress '["governor","<govern>"]'        -p <finder>@active
```

Then whitelist collateral, add identifiers, and register any integrating contracts:

```bash
cleos push action <finder> addcollat '[{"sym":"4,OOVP","contract":"<token>"}]' -p <finder>@active
cleos push action <finder> addident  '[<identifier>,"BTC/USD"]'                 -p <finder>@active
cleos push action <finder> regcontract '["<integrator>","<integrator>"]'        -p <integrator>@active
```

## 4. Configure the store

```bash
cleos push action <store> setconfig    '["<treasury>",5000]'                          -p <store>@active   # 50% burn
cleos push action <store> setfinalfee  '[{"sym":"4,OOVP","contract":"<token>"},"1.0000 OOVP"]' -p <store>@active
# If a collateral implements a holder burn(name,asset,string):
cleos push action <store> setburnable  '[{"sym":"4,OOVP","contract":"<token>"},true]' -p <store>@active
```

## 5. Configure staking

```bash
cleos push action <stake> setvotingtkn '["<token>","4,OOVP"]'  -p <stake>@active
cleos push action <stake> setvotingctr '["<voting>"]'          -p <stake>@active
cleos push action <stake> setemitvault '["<emissions_vault>"]' -p <stake>@active
cleos push action <stake> setemission  '[500]'                 -p <stake>@active   # 0.05 OOVP/sec
cleos push action <stake> setcooldown  '[604800]'              -p <stake>@active   # 7 days (prod)
cleos push action <stake> setnovote    '[5]'                   -p <stake>@active   # 0.05% passive slash
cleos push action <stake> setlockparms '[63072000,604800,30000]' -p <stake>@active # 2y max, 1w min, 3x
```

The **emissions vault** must grant `<stake>@eosio.code` on its active permission so rewards can be paid from it.

## 6. Configure voting (DVM)

```bash
cleos push action <voting> setconfig    '["<finder>","<stake>","<token>"]' -p <voting>@active
cleos push action <voting> setphaselen  '[86400]'   -p <voting>@active   # 24h phases (prod)
cleos push action <voting> setthreshold '[5000000,5000]' -p <voting>@active # GAT 5%, SPAT 50%
cleos push action <voting> setmaxrolls  '[3]'       -p <voting>@active
```

## 7. Configure the oracle

```bash
cleos push action <oracle> setconfig     '["<finder>","<voting>","<store>"]' -p <oracle>@active
cleos push action <oracle> setdefault    '[7200]'   -p <oracle>@active   # 2h liveness (prod)
cleos push action <oracle> setlimit      '[8192]'   -p <oracle>@active
cleos push action <oracle> setburnedbps  '[5000]'   -p <oracle>@active   # 50%
cleos push action <oracle> setdefcur     '[{"sym":"8,WAX","contract":"eosio.token"}]' -p <oracle>@active
cleos push action <oracle> setfeevault   '["<oracle_fee_vault>"]' -p <oracle>@active
# Refresh the parameter cache for each currency + identifier:
cleos push action <oracle> syncparams    '[<identifier>,{"sym":"4,OOVP","contract":"<token>"}]' -p anyone@active
```

## 8. Configure governance

```bash
cleos push action <govern> setconfig     '["<finder>"]'    -p <govern>@active
cleos push action <govern> setproposer   '["<proposer>"]'  -p <govern>@active
cleos push action <govern> setemergency  '["<emergency_msig>"]' -p <govern>@active
cleos push action <govern> setminbond    '["500.0000 OOVP"]' -p <govern>@active
cleos push action <govern> setvoteperiod '[172800]' -p <govern>@active  # 48h (prod)
cleos push action <govern> setexecdelay  '[86400]'  -p <govern>@active  # 24h timelock
cleos push action <govern> setemrgdelay  '[86400]'  -p <govern>@active
cleos push action <govern> setemergtgt   '["<target>",true]' -p <govern>@active  # allowlist emergency targets
```

## 9. (Optional) Configure the reward vault

```bash
cleos push action <reward> setconfig '["<token>","4,OOVP","<admin>"]' -p <reward>@active
cleos push action <reward> addsource '["<source_contract>",0]'        -p <admin>@active
cleos push action <token>  transfer  '["<funder>","<reward>","1000000.0000 OOVP","rewardpool"]' -p <funder>@active
```

## 10. Permissions

* **`eosio.code`.** Every contract that sends inline actions needs its own `active` permission to include `<self>@eosio.code`: at minimum `<oracle>`, `<voting>`, `<stake>`, `<govern>`, `<store>`, and `<reward>`.
* **`exec` permission for governance.** For any action you want governance to control, create an `exec` permission on the target account whose authority is `<govern>@eosio.code`, and `linkauth` the action to it. Governance transactions must be authorized `target@exec` (see [On-Chain Proposals](../community/governance/dao-proposals.md)).
* **Keeper keys.** Give the keeper a low-privilege permission `linkauth`-scoped to only the [crank actions](../developers/keeper-and-crank-actions.md); it should not be able to move funds.

## 11. Post-deploy verification

* Resolve every interface from the finder and confirm it points at the right account.
* Run one full OOv3 assert → settle cycle and one OOv2 request → propose → settle cycle.
* Run one dispute → DVM vote → resolve → settle cycle on testnet ([walkthrough](../developers/resolving-disputes.md)).
* Confirm emissions accrue and `claimreward` pays from the emissions vault.
* Confirm `setburnedbps`, final fee, and `getminbond` produce the expected bond sizes.

{% hint style="warning" %}
Before mainnet, migrate all admin (`owner`/`active`) keys of the contract and vault accounts to a **multisig**, and hand parameter control to `oovp.govern`. A single-key deployer holding admin over the oracle is a launch blocker.
{% endhint %}
