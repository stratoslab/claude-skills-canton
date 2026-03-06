# DeFi / Tokenization on Canton

## Project Overview
Tokenization application using DAML Finance library on Canton Network.

## Tech Stack
- DAML + DAML Finance library
- Canton Network (production)
- TypeScript (frontend) + Java (backend automation)
- Postgres (persistent storage)

## Skills Reference
Reference: ~/claude-skills-canton/skills/

## Key Skills
- daml-finance-patterns — Holdings, accounts, instruments, settlement
- daml-interfaces — Interface hierarchy (IHolding, IAccount, IInstrument)
- multi-party-workflows — DvP, multi-party settlement
- canton-deployment — Production Canton setup
- canton-privacy — Sub-transaction privacy for financial data
- security-review — Audit contracts for vulnerabilities
- typescript-bindings — Frontend integration
- java-bindings — Backend automation bots

## DAML Finance Conventions
- Use `type I = InterfaceName` and `type V = View` aliases
- Use qualified imports: `import Daml.Finance.Interface.Holding.V4 qualified as Holding`
- Use `pure` instead of `return`
- Use flexible controllers: `actor : Party` as first choice argument
- Use `;` for short-hand constructors

## Domain Model
- **Instruments**: Tokens, bonds, equities (implement IInstrument)
- **Holdings**: Ownership records (implement IHolding, IFungible)
- **Accounts**: Custody relationships (implement IAccount)
- **Settlement**: Atomic DvP via settlement instructions

## Security Requirements
- All financial data protected by sub-transaction privacy
- Minimal observer sets on holdings
- Locked holdings during settlement
- Audit trail for all transactions
