# Architecture

This document explains how the pieces of this repo fit together: which package
owns what, where a transaction travels, and how failures reach the user. Read
it before changing a boundary. For setup and conventions, see
[CONTRIBUTING.md](CONTRIBUTING.md).

## Layers

```
┌──────────────────────────────┐   ┌──────────────────────────────┐
│ apps/web  (Next.js app)      │   │ apps/docs  (Astro Starlight) │
│ pages, components, wallet,   │   │ documentation site; embeds   │
│ network checks, position list│   │ SDK examples                 │
└──────────────┬───────────────┘   └──────────────────────────────┘
               │ imports
┌──────────────▼───────────────┐
│ packages/sdk  (@sororail/sdk)│   Typed clients, signers, errors,
│ no UI, no secret keys        │   amount helpers. Published package.
└──────────────┬───────────────┘
               │ JSON-RPC over HTTPS (Soroban RPC)
┌──────────────▼───────────────┐
│ Soroban network (testnet)    │   Contracts live in the separate
│ Sororail contracts           │   Sororail/sororail-contracts repo.
└──────────────────────────────┘
```

| Layer | Location | Owns | Must not |
|---|---|---|---|
| Contracts | [Sororail/sororail-contracts](https://github.com/Sororail/sororail-contracts) | Rules, balances, the error code ABI | Be duplicated here, beyond the mirrored error table |
| SDK | `packages/sdk/src` | Building, simulating and submitting calls; decoding errors; amount conversion | Hold or request a secret key; import React or any UI code |
| Web app | `apps/web/src` | Screens, wallet connection, network checks, the local position list | Talk to the contracts except through the SDK |
| Docs | `apps/docs` | Guides and concept pages | Contain code the SDK examples do not compile |

### SDK (`packages/sdk/src`)

- `clients/` has one client per contract: `escrow`, `stream`, `vesting`,
  `recurring` and `batchPayout`. Each extends `base.ts`, which holds the shared
  logic for building, simulating, signing and polling a transaction.
- `signers/` defines the `Signer` interface and two implementations:
  `FreighterSigner` for browsers and `KeypairSigner` for scripts and tests.
- `errors/` defines the error classes. `codes.ts` mirrors
  `sororail_common::errors` in the contracts repo, and `decodeError` turns a
  numeric code into a `ContractError`.
- `utils/` holds amount conversion (`amounts.ts`), batching (`batch.ts`) and
  `ScVal` helpers (`scval.ts`).
- `index.ts` is the public surface. `scripts/api-surface.mjs` checks it against
  a committed snapshot, so an export change is always a visible diff.

### Web app (`apps/web/src`)

- `app/` holds the routes: overview, payroll, streams, vesting and escrow.
- `lib/wallet.tsx` provides `WalletProvider`. It connects Freighter, polls for
  account and network changes, and blocks signing on a network mismatch.
- `lib/network.ts` reads the RPC URL and network passphrase from environment
  variables and checks the RPC's network at startup.
- `lib/positions.ts` keeps the list of contract addresses the user has added.
  The list is stored in the browser. Every figure shown for a position is read
  from the chain, so losing the list loses no funds or state.
- `components/` holds the shared UI, including `Confirm` and `Feedback`.

## Data flow: a state-changing action

1. A user submits a form, for example "withdraw" on a stream.
2. The page calls the matching SDK client, which builds the transaction. The
   client needs a `publicKey` and does not sign.
3. The client simulates the transaction against Soroban RPC. The UI can show
   the result, such as the amount that would be received, before anyone signs.
4. The client passes the transaction to the `Signer`. With `FreighterSigner`,
   the extension shows the transaction to the user and returns signed XDR. The
   signer re-checks the account and network first.
5. The client submits the signed transaction and polls RPC until it is final.
6. The page refreshes the position from the chain.

Steps 2 to 5 are separate calls (`prepare`, `simulate`, `signAndSend`) so that
a confirmation screen can show the effect before a signature is requested.

## Error handling

Every failure the SDK throws extends `SororailError`:

| Error | Raised when | Where it comes from |
|---|---|---|
| `ValidationError` | Arguments are wrong, before anything is sent | Building a call |
| `ContractError` | The contract refused the call | `simulate()` or `signAndSend()` |
| `SigningError` | The wallet could not sign | `signAndSend()`; includes `NetworkMismatchError` |
| `NetworkError` | RPC could not be reached or rejected the request | Any stage; the original failure is `cause` |

The web app never shows a raw contract code. `ContractError.message` is the
sentence shown to the user, and `Feedback.tsx` pairs it with a recovery step.

Amounts are `bigint` in every layer. Conversion to a display value happens only
at the edge, in `amounts.ts` and `Money.tsx`.

## Where to change what

- **A new contract function:** add it to the matching client in
  `packages/sdk/src/clients`, then update the API snapshot and add a test.
- **A new contract error:** add it inside its owning range in
  `packages/sdk/src/errors/codes.ts`. Do not renumber existing entries.
- **A new screen:** add a route under `apps/web/src/app` and call the SDK from
  it. Do not import the contract ABI directly.
- **A new signing method:** implement `Signer` in `packages/sdk/src/signers`.
  Do not add any path that accepts a secret key in the browser.
