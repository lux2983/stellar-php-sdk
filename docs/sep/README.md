# SEP Implementations

The SDK supports Stellar Ecosystem Proposals (SEPs) for interoperability with the Stellar ecosystem.

## What are SEPs?

Stellar Ecosystem Proposals (SEPs) define standards for how services, applications, and organizations interact with the Stellar network. They ensure consistent implementation of common patterns like domain verification, authentication, and asset transfers.

Think of SEPs as the "rules of the road" that let different Stellar applications talk to each other. When you use SEP-10 for authentication, any anchor that implements SEP-10 will understand your auth requests.

## Implemented SEPs

| SEP | Title | Documentation |
|-----|-------|---------------|
| SEP-01 | Stellar TOML | [sep-01-stellar-toml.md](sep-01-stellar-toml.md) |
| SEP-02 | Federation Protocol | [sep-02-federation.md](sep-02-federation.md) |
| SEP-05 | Key Derivation Methods | [sep-05-key-derivation.md](sep-05-key-derivation.md) |
| SEP-06 | Programmatic Deposit and Withdrawal | [sep-06-transfer.md](sep-06-transfer.md) |
| SEP-07 | URI Scheme | [sep-07-uri-scheme.md](sep-07-uri-scheme.md) |
| SEP-08 | Regulated Assets | [sep-08-regulated.md](sep-08-regulated.md) |
| SEP-10 | Web Authentication | [sep-10-auth.md](sep-10-auth.md) |
| SEP-11 | Transaction Representation (Txrep) | [sep-11-txrep.md](sep-11-txrep.md) |
| SEP-12 | KYC API | [sep-12-kyc.md](sep-12-kyc.md) |
| SEP-24 | Interactive Deposit and Withdrawal | [sep-24-interactive.md](sep-24-interactive.md) |
| SEP-30 | Account Recovery | [sep-30-recovery.md](sep-30-recovery.md) |
| SEP-31 | Cross-Border Payments | [sep-31-cross-border.md](sep-31-cross-border.md) |
| SEP-38 | Anchor RFQ API | [sep-38-quotes.md](sep-38-quotes.md) |
| SEP-45 | Contract Account Authentication | [sep-45-contract-auth.md](sep-45-contract-auth.md) |
| SEP-53 | Memos for Custodial Accounts | [sep-53-memo.md](sep-53-memo.md) |

## Which SEP Do I Need?

### Building a Wallet

Start with authentication, then add deposit/withdrawal support:

1. **SEP-10** — Authenticate users with anchors
2. **SEP-24** — Interactive deposit/withdrawal (recommended for most wallets)
3. **SEP-06** — Programmatic deposit/withdrawal (for automated flows)

SEP-24 shows the user a web interface hosted by the anchor. SEP-06 handles everything via API calls. Most wallets use SEP-24 because it offloads UI complexity to the anchor.

### Building an Anchor

Anchors need to publish their capabilities and handle user verification:

1. **SEP-01** — Publish your stellar.toml so wallets can discover your services
2. **SEP-10** — Authenticate incoming requests
3. **SEP-12** — Collect and verify KYC information
4. **SEP-24** — Interactive deposit/withdrawal for retail users
5. **SEP-31** — Cross-border payments for B2B flows

If you're building a remittance service, add **SEP-38** for exchange rate quotes.

### Working with Regulated Assets

Some assets require issuer approval for every transaction:

1. **SEP-08** — Get approval before submitting transactions with regulated assets

The issuer's approval server reviews each transaction and either approves, rejects, or requests modifications.

### Other Common Use Cases

| Use Case | SEPs |
|----------|------|
| Human-readable addresses (email-style) | SEP-02 |
| Deterministic key generation from mnemonics | SEP-05 |
| Payment requests via URI | SEP-07 |
| Human-readable transaction format | SEP-11 |
| Account recovery via custodians | SEP-30 |
| Custodial account identification | SEP-53 |
| Soroban contract authentication | SEP-45 |

## Learning More

Each SEP documentation page includes:
- Overview of what the protocol does
- Working code examples
- Error handling patterns
- Links to related SEPs

For the official specifications, see the [Stellar SEP repository](https://github.com/stellar/stellar-protocol/tree/master/ecosystem).

---

[← Back to Documentation](../README.md)
