# Vibe Coding with Reflector

Build Stellar apps with an AI coding assistant and the Reflector skill.

This guide is for developers who can run a project locally and want help integrating oracle prices. Start with a small, read-only feature, verify its behavior, then build on it. Each prompt below describes a concrete result and the evidence needed to trust it.

## 1. Give your assistant the skill and the context

Make the Reflector skill available to your coding assistant, including its `references/` directory. A pasted `SKILL.md` alone does not include the linked implementation guidance. If your assistant cannot find a reference, provide that file or direct it to the official source.

Use the following opening prompt. Replace the bracketed values before sending it:

```text
Use the Reflector skill for this task. Read SKILL.md and only the references
needed for the implementation.

I am building [what the application does].
Stack: [existing framework, language, and package manager].
Network: [testnet or mainnet].
Assets: [symbols or exact Stellar asset identities].
Feature: [display prices / use prices in a contract / receive alerts].
Existing project: [relevant files or repository location].
Done means: [observable behavior].

Inspect the project first. Identify the appropriate Reflector product and
feed. Verify addresses and API signatures against official sources and the
installed dependency version. Do not invent missing configuration.
Implement the smallest working feature and report how you verified it.
```

The skill supplies integration guidance. Your prompt supplies product decisions: what to build, which network to target, and what successful behavior looks like.

## 2. Choose the product and reference

| Your goal | Starting point | Skill reference |
|---|---|---|
| Display public oracle prices | Pulse | `references/js-client.md` |
| Read prices inside a Soroban contract | Pulse or Beam, depending on requirements | `references/rust-consumer.md` |
| Evaluate faster or customized feeds | Beam | `references/architecture.md` |
| Receive off-chain price notifications | Flare | `references/flare-dao.md` |
| Work on governance | DAO | `references/flare-dao.md` |
| Select oracle addresses and asset keys | Deployment discovery | `references/deployments.md` |

These paths are relative to the Reflector skill directory. Read deployment guidance alongside the reference for your implementation.

Pulse publishes at a five-minute interval. Consumer contracts read previously published oracle state; a read does not fetch a fresh exchange quote. Use the feed interval when deciding whether a product meets your application's needs. See the [official contract overview](https://github.com/reflector-network/reflector-contract#how-it-works).

**Version check:** the supplied skill describes Beam fees per read, while the current [official JS client README](https://github.com/reflector-network/contract-client-js#price-oracles---pulseclient-and-beamclient) documents prepaid access through `track` and `trackedUntil`. Have the assistant verify the target deployment's fee and authorization behavior. Do not combine examples from different contract versions.

## 3. First project: a BTC price reader

Use a terminal script to establish that configuration and asset selection work before adding a UI.

```text
Use the Reflector skill to create a small TypeScript script that reads BTC
from a Pulse CEX/DEX feed on Stellar testnet.

Read references/js-client.md and references/deployments.md. Use the official
@reflector/contract-client package and check compatible dependencies.

Create an .env.example with RPC URL, public account key, oracle contract ID,
and explicit network passphrase. Validate configuration at startup.
The script must not require a secret key or wallet signature for price reads.

Verify the oracle address from an official source, then query its asset
registry and metadata. Confirm BTC is supported before reading its price.
Print the network, oracle address, asset identity, base denomination,
formatted price, quote timestamp, and quote age.

Keep price arithmetic in bigint or another exact representation.
Handle missing quotes and RPC errors explicitly. Use a configurable maximum
age; for this tutorial, start with 600 seconds and label it a demo policy.
Reject future timestamps as invalid for this demo.

Include a run command. Run a live smoke check if configuration and network
access are available. Otherwise report the exact missing input and clearly
separate mocked results from live results.
```

The official JS client simulates read calls; state-changing calls require signing. Its public timestamps use seconds. These details matter when connecting the script and calculating age. See [client initialization and timestamp documentation](https://github.com/reflector-network/contract-client-js#initialization).

**Checkpoint:** you should have a reproducible command and output identifying the actual feed. A plausible price alone is insufficient evidence that the correct network and asset were selected.

## 4. Turn the reader into an interface

Once the reader works, extend the existing project:

```text
Use the Reflector skill and reuse the verified price reader to add a price
card to this app. Follow the existing framework and component conventions.

Show price, base denomination, last update, quote age, and network.
Implement loading, available, stale, unavailable, and connection-error states.
Keep any last-known price visibly labeled with its timestamp when stale.
Never replace a missing quote with zero or generated sample data.

Choose a refresh and caching policy appropriate to the feed interval.
Recalculate age as time passes so a cached quote cannot remain marked fresh.
Keep private RPC credentials on the server. Do not add a wallet connection
solely to display a public Pulse quote.

Verify the state transitions using controlled fixtures and report whether
the live data path was also exercised.
```

Pair Reflector with the `dapp` skill when the feature also needs wallets or transaction submission. Use the `data` skill when the task requires additional Stellar RPC queries. Ask for related skills only when the work needs them.

## 5. Use prices inside a smart contract

For a contract, give the assistant the operation that depends on the quote and the required failure behavior. A dashboard's displayed price must not become an unverified input to financial contract logic.

```text
Use the Reflector and smart-contracts skills to add a Pulse oracle adapter
to this Soroban project. Read references/rust-consumer.md and verify the
consumer interface against the target contract version.

The adapter should return a validated quote or a typed failure.
Use configurable oracle and asset identities. Validate the expected base
denomination. Handle both unavailable quotes and failed cross-contract calls.
Do not unwrap an optional quote in a production path.

Compare the quote timestamp with ledger time. Reject future timestamps,
quotes beyond the configured maximum age, and nonpositive prices for this
valuation use case. Make the age boundary explicit and test it.

Read oracle decimals. Document token units separately from price units.
Use checked scaling, multiplication, and division, and specify rounding.
Return explicit errors for unsupported scales or arithmetic overflow.

Test fresh, missing, stale, future-dated, zero, and negative quotes;
unexpected base denomination; cross-contract failures; and overflow.
Prove the dependent operation makes no state change when validation fails.
Run the project's relevant tests and build, then report the results.
```

Reflector prices use integer encoding with precision obtained from `decimals()`. Freshness should be evaluated against ledger time inside a contract. See the [official contract interface and usage](https://github.com/reflector-network/reflector-contract#usage).

The 600-second threshold from the reader is a tutorial choice. Set a production threshold based on the operation, feed behavior, and acceptable delay. Also define who can change oracle configuration and protect that authority.

## 6. Debug with evidence

Give the assistant the exact error and enough configuration to reproduce it. Redact secrets and credential-bearing RPC URLs.

```text
Use the Reflector skill to diagnose this failure before changing code.

Expected behavior: [what should happen].
Actual behavior: [exact error or unexpected result].
Network and feed family: [values].
Oracle contract ID and asset key: [public values].
Installed package versions: [versions].
Relevant code and reproduction command: [details].

Check network configuration, deployment identity, assets(), asset encoding,
base denomination, quote timestamp, decimals, and RPC diagnostics.
Explain the evidence for the cause, implement a focused fix, and rerun the
reproduction. Do not silently switch feeds or substitute mock prices.
```

| Symptom | Investigation |
|---|---|
| Missing quote or `AssetMissing` | Match the asset variant and exact value against `assets()` |
| DEX asset cannot be found by ticker | Check whether the feed expects a SAC `C…` address |
| Price differs by orders of magnitude | Inspect oracle decimals and token amount units separately |
| Quote appears extremely old or future-dated | Check seconds versus milliseconds and the clock source |
| Works on one network only | Verify RPC, passphrase, deployment, and asset keys together |
| Beam access fails | Verify caller, access expiry, authorization, and deployed fee model |

The supplied deployment reference distinguishes the network hosting the oracle from the market it reports. A testnet oracle reporting pubnet DEX prices can require pubnet asset keys. Verify the selected feed rather than mechanically replacing every address with a testnet equivalent.

## 7. Prompts for the next feature

**Portfolio valuation**

```text
Use the Reflector skill to add portfolio valuation. Discover supported
assets and verify each quote's base denomination. Specify token decimals
and rounding. Show unsupported or stale positions explicitly, and mark the
total incomplete if any required position cannot be valued. Do not add
values in different base denominations without an explicit conversion.
```

**Historical prices**

```text
Use the Reflector skill to add a historical price chart. Verify the API's
record limits and available retention. Label gaps and timestamps accurately.
If calculating an average, name the method. For a time-weighted average,
define the window, duration weights, ordering, and missing-data policy.
Do not label an ordinary sample mean TWAP without verifying its assumptions.
```

**Flare alerts**

```text
Use the Reflector skill and references/flare-dao.md to implement a Flare
alert integration. Verify the current subscription API, fee requirements,
payload format, and supported webhook verification mechanism.
Define the asset, threshold, heartbeat, expiry, and callback endpoint.
Handle duplicate events and retries. Test locally before activating a paid
subscription. Report any missing configuration instead of inventing it.
```

## 8. Review before shipping

```text
Review this implementation using the Reflector skill and the relevant
Stellar skills. Inspect the code and tests, not just the README.

Check feed identity and source, exact asset encoding, base denomination,
missing-price behavior, timestamp units, freshness boundaries, precision,
rounding, overflow, and RPC failures. Check authorization for configuration
changes and any paid or state-changing operation.

For financial operations, verify that invalid oracle data blocks the
operation. For a UI, verify that unavailable and stale states remain clear.

Return actionable findings with file locations and fixes. Distinguish
tests actually run, live checks actually performed, and unverified claims.
```

Before switching networks, verify the deployment, RPC, passphrase, supported asset keys, and base denomination again. Record the selected contract, dependency versions, verification date, and smoke-check results with the project. Assign responsibility for monitoring feed availability and retention or access expiry where applicable.

## Sources and scope

Prepared October 5, 2026, using the supplied Reflector skill and its JavaScript, Rust, and deployment references, with checks against the official repositories below.

- [Reflector contract repository](https://github.com/reflector-network/reflector-contract)
- [Reflector JavaScript client](https://github.com/reflector-network/contract-client-js)
- [Stellar oracle providers](https://developers.stellar.org/docs/data/oracles/oracle-providers)
- [Reflector Pulse deployment page](https://reflector.network/pulse)
- [Reflector interface documentation](https://reflector.network/docs/interface)

The Reflector website pages required JavaScript in the research environment, so their deployment contents were not independently verified. This guide deliberately leaves address discovery to each implementation. No live oracle calls, generated application builds, or contract tests were performed while writing this guide.
