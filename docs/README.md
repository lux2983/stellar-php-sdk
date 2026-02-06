# Stellar PHP SDK Documentation

Welcome to the documentation for the Stellar SDK for PHP. Whether you're building a wallet, integrating with anchors, or deploying smart contracts, you'll find what you need here.

## Documentation Structure

### Getting Started
- **[Quick Start Guide](quick-start.md)** - Your first transaction in 15 minutes
- **[Getting Started Guide](getting-started.md)** - Installation, keys, and account basics

### Core Usage
- **[SDK Usage](sdk-usage.md)** - Transactions, operations, queries, and common patterns
- **[Soroban Guide](soroban.md)** - Smart contract interaction

### Protocols & Standards
- **[SEP Implementations](sep/README.md)** - Stellar Ecosystem Proposal support

### Advanced Topics
- **[Advanced Guide](advanced.md)** - Multi-sig, fee bumps, optimization, debugging

## Key Features

The SDK provides a PHP implementation of the Stellar protocol:

- **Full Horizon API** - Accounts, transactions, payments, assets, and streaming
- **Soroban support** - Deploy and interact with smart contracts
- **18 SEP protocols** - Authentication, deposits, KYC, cross-border payments
- **Transaction building** - All 27 Stellar operations with type-safe builders
- **Multi-signature** - Threshold-based signing workflows

## SDK Capabilities

### Core Operations
- Ed25519 keypair generation and management
- Mnemonic phrase support (12/24 words, BIP-39)
- Transaction building, signing, and submission
- Fee bump transactions
- Muxed account support

### Horizon Queries
- Accounts, assets, ledgers
- Transactions, operations, effects
- Offers, order books, trades
- Liquidity pools, claimable balances
- Server-Sent Events (SSE) streaming

### Soroban (Smart Contracts)
- Contract deployment and invocation
- Transaction simulation
- State preflight and restoration
- Event queries
- Authorization handling

### SEP Protocol Support

The SDK implements these Stellar Ecosystem Proposals:

| Protocol | Description |
|----------|-------------|
| SEP-1 | stellar.toml discovery |
| SEP-2 | Federation protocol |
| SEP-5 | Key derivation (mnemonics) |
| SEP-6 | Programmatic deposit/withdrawal |
| SEP-7 | URI scheme for signing |
| SEP-8 | Regulated assets |
| SEP-9 | Standard KYC fields |
| SEP-10 | Web authentication |
| SEP-11 | Txrep (human-readable transactions) |
| SEP-12 | KYC API |
| SEP-23 | StrKey encoding |
| SEP-24 | Interactive deposit/withdrawal |
| SEP-29 | Account memo requirements |
| SEP-30 | Account recovery |
| SEP-31 | Cross-border payments |
| SEP-38 | Anchor quotes |
| SEP-45 | Contract account authentication |
| SEP-53 | Message signing and verification |

## Learning Paths

**Building a wallet?**
Quick Start → Accounts & Keys → Transactions → SEP-10 → SEP-24

**Building an anchor?**
Quick Start → SEP-1 → SEP-10 → SEP-12 → SEP-24/SEP-31

**Working with smart contracts?**
Quick Start → Soroban Guide → Advanced

## Requirements

- PHP 8.0 or higher
- Composer

## Getting Help

- **[GitHub Issues](https://github.com/Soneso/stellar-php-sdk/issues)** - Bug reports and feature requests
- **[GitHub Discussions](https://github.com/Soneso/stellar-php-sdk/discussions)** - Questions and ideas
- **[Stellar Discord](https://discord.gg/stellardev)** - Community support
- **[API Reference](https://soneso.github.io/stellar-php-sdk/packages/Soneso-StellarSDK.html)** - PHPDoc documentation

## Additional Resources

- **[Stellar Documentation](https://developers.stellar.org/)** - Official Stellar docs
- **[Horizon API Reference](https://developers.stellar.org/docs/data/apis/horizon/)** - Horizon endpoints
- **[Soroban Documentation](https://developers.stellar.org/docs/build/smart-contracts/)** - Smart contract guides

## Contributing

See [CONTRIBUTING.md](../CONTRIBUTING.md) for contribution guidelines.

## License

Apache License 2.0. See [LICENSE](../LICENSE) for details.

---

**Navigation**: [Quick Start →](quick-start.md)
