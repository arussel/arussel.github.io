---
layout: post
title: "Building a Solana Lottery: Oracles, Keepers, and Production Infrastructure"
date: 2024-12-03
categories: [solana, blockchain, rust]
description: "A deep dive into building a decentralized lottery on Solana — oracle integration, keeper patterns, and the realities of deploying on-chain infrastructure."
image: /assets/images/open-lotto-architecture.svg
---

I recently built Open Lotto, a decentralized lottery on Solana. What started as a weekend project turned into a deep dive into oracle integration, keeper patterns, and the realities of deploying on-chain infrastructure. Here's what I learned.

## Why Build a Lottery?

Lotteries are deceptively simple. Buy ticket, pick winner, pay out. But that simplicity hides real challenges:

- **Randomness**: How do you pick a winner that no one can manipulate?
- **Timing**: Who triggers the draw when the pot ends?
- **Trust**: How do users know the game is fair?

These problems exist in traditional lotteries too — they're just hidden behind closed doors. On-chain, everything is transparent, which means you need cryptographic guarantees instead of legal ones.

## The Architecture

Three components work together:

1. **On-chain program** (Anchor/Rust) — Holds funds, validates tickets, enforces rules
2. **Keeper** (Rust CLI) — Triggers draws and settlements when pots end
3. **Oracle** (Switchboard) — Provides verifiable randomness

Users buy tickets directly on-chain. When a pot ends, the keeper requests randomness from Switchboard, calls the draw function to lock in that randomness, waits for the oracle to reveal, then settles the lottery. The winner claims their prize.

![Open Lotto Architecture](/assets/images/open-lotto-architecture.svg)

## The Randomness Problem

This is where it gets interesting. You can't generate randomness on-chain — everything is deterministic and public. Common approaches:

**Block hash** — Use a future block's hash. Problem: validators can manipulate it if the prize is worth more than the block reward.

**Commit-reveal** — Users commit to a secret, then reveal. Problem: Last revealer can choose not to reveal if they'd lose.

**VRF (Verifiable Random Function)** — An oracle generates randomness with a cryptographic proof. This is what I chose.

## Why Switchboard?

Switchboard uses Intel SGX enclaves (TEE — Trusted Execution Environment). The randomness is generated inside tamper-proof hardware that even the oracle operator can't manipulate.

The flow:

1. My keeper commits to a slot number
2. The oracle's TEE uses that slot's blockhash as a seed
3. The TEE generates randomness and signs it
4. Anyone can verify the signature matches the TEE's attestation

The slot commitment is key — it locks in which blockhash will be used before anyone (including the oracle) knows the result.

## The Pain of Integrating Switchboard from Rust

Switchboard provides a TypeScript SDK. I wanted Rust.

This meant reverse engineering everything:

**Account layouts**: I had to fetch the IDL from devnet and manually calculate byte offsets for parsing queue data and randomness accounts.

**Instruction discriminators**: Anchor uses the first 8 bytes of SHA256("global:instruction_name"). No Rust helper, so I built my own.

**Account ordering**: The randomnessInit instruction requires 13 accounts in a specific order. One wrong position and you get cryptic errors like "AccountOwnedByWrongProgram".

**PDA derivation**: Finding the right seeds for reward escrow, LUT signer, program state — all undocumented for Rust.

The on-chain program side was easy — the switchboard_on_demand crate handles parsing the randomness account. But calling Switchboard from a Rust client? I spent more time reading their TypeScript SDK source code than writing my own.

If you're building a Rust client for Switchboard, budget significant time for this. Or just use TypeScript.

## The Keeper Problem

Who calls draw_lottery when a pot ends? The blockchain doesn't have cron jobs.

Options I considered:

**Clockwork** — Solana's native automation. Clean, but another dependency.

**Gelato/Chainlink Keepers** — More Ethereum-focused.

**Self-hosted bot** — Full control, but you're responsible for uptime.

I went with a self-hosted Rust CLI that can run as a bot. For a production lottery, you'd want:

- Multiple keeper instances for redundancy
- Monitoring and alerting
- Automatic retry with exponential backoff
- Health checks

The keeper is the single point of failure. If it doesn't call draw, the pot just sits there. Users can still buy tickets, but no one wins.

## Production Considerations

### Costs

Reality check time:

| Operation | Cost |
|-----------|------|
| Deploy program | ~2 SOL (~$300) |
| Init pot manager | ~0.02 SOL |
| Per draw cycle | ~0.01 SOL |
| Oracle fees | ~0.001–0.01 SOL |

With a 10% fee and 0.01 SOL tickets, I need 10 tickets per pot just to break even on keeper costs. This pushes toward either higher ticket prices, longer pot durations, or minimum pot sizes before drawing.

### The Devnet Reality

Here's something no tutorial mentions: **devnet infrastructure is unreliable**.

I integrated Switchboard On-Demand, got everything working locally with mocked randomness, deployed to devnet, and… all 18 oracles in the queue had expired TEE attestations. The randomnessCommit call fails on every single oracle.

Devnet is for testing, not for the oracle operators' SLAs. Mainnet oracles are maintained because real money is involved.

Lesson: Your local tests passing means nothing until you've tested on the actual network you'll deploy to.

### Account Bloat

Every ticket is a PDA. A popular lottery could create thousands of accounts. Considerations:

- Account rent adds up (~0.002 SOL per ticket)
- Closing accounts after claiming requires careful design
- Historical data vs on-chain storage tradeoffs

### Security Surface

Things that could go wrong:

- **Keeper key compromise** — They can't steal funds, but they can grief by not drawing
- **Oracle collusion** — Mitigated by TEE, but not impossible
- **Smart contract bugs** — The usual suspects (reentrancy, overflow, access control)
- **Front-running** — Less relevant for lotteries, but worth considering for the claim phase

## What I'd Do Differently

**Start with mainnet cost estimation.** I designed for devnet's free transactions, then realized mainnet economics change everything.

**Abstract the oracle earlier.** Switchboard's API is complex (13 accounts for randomnessInit!). I'd wrap it in a cleaner interface from day one.

**Build monitoring first.** Keeper uptime is critical. I should have built observability before the happy path.

**Consider a factory pattern.** Right now, only I can create lottery pots. A factory would let anyone deploy their own lottery — just call create_lottery(name, ticket_price, duration, fee) and get a new independent instance. Permissionless lottery creation, each with its own pot manager and treasury.

## The Code

Open source on GitHub: [github.com/arussel/open-lotto](https://github.com/arussel/open-lotto)

- `programs/open-lotto/` — Anchor program
- `cli/` — Rust keeper CLI
- Tests use LiteSVM with mocked Switchboard data

## Takeaways

Building on Solana is genuinely different from EVM chains. The account model takes getting used to, but PDAs are elegant once they click. The ecosystem is younger — expect rougher edges and less documentation.

For anyone considering oracle integration: budget more time than you think. The happy path is documented; the edge cases aren't.

And if you're building something that needs randomness: test on mainnet with real oracles before you commit to an architecture. Devnet will lie to you.
