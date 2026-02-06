# [Stellar SDK for PHP](https://github.com/Soneso/stellar-php-sdk)

![v1.9.2](https://img.shields.io/badge/v1.9.2-green.svg) [![codecov](https://codecov.io/gh/Soneso/stellar-php-sdk/branch/main/graph/badge.svg)](https://codecov.io/gh/Soneso/stellar-php-sdk) [![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/Soneso/stellar-php-sdk)

The Soneso open source Stellar SDK for PHP provides APIs to build and sign transactions, connect and query [Horizon](https://github.com/stellar/horizon), and interact with [Soroban](https://soroban.stellar.org/) smart contracts.

## Installation

```bash
composer require soneso/stellar-php-sdk:1.9.2
```

**Requirements:** PHP 8.0+

## Quick Example

```php
<?php
require 'vendor/autoload.php';

use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\StellarSDK;
use Soneso\StellarSDK\Util\FriendBot;

// Generate a new keypair
$keyPair = KeyPair::random();
echo "Public Key: " . $keyPair->getAccountId() . "\n";
echo "Secret Key: " . $keyPair->getSecretSeed() . "\n";

// Fund on testnet
FriendBot::fundTestAccount($keyPair->getAccountId());

// Check balance
$sdk = StellarSDK::getTestNetInstance();
$account = $sdk->requestAccount($keyPair->getAccountId());

foreach ($account->getBalances() as $balance) {
    echo "Balance: " . $balance->getBalance() . " " . $balance->getAssetType() . "\n";
}
```

## Documentation

| Guide | Description |
|-------|-------------|
| **[Quick Start](docs/quick-start.md)** | Your first transaction in 15 minutes |
| **[Getting Started](docs/getting-started.md)** | Installation, keys, accounts, and fundamentals |
| **[SDK Usage](docs/sdk-usage.md)** | Transactions, operations, Horizon queries, streaming |
| **[Soroban](docs/soroban.md)** | Smart contract deployment and interaction |
| **[SEP Protocols](docs/sep/README.md)** | Anchors, authentication, KYC, cross-border payments |
| **[Advanced Topics](docs/advanced.md)** | Multi-sig, error handling, debugging, performance |

## Supported Features

### Stellar Operations
All 26 Stellar operations including payments, account management, DEX trading, claimable balances, liquidity pools, and sponsorship.

### SEP Protocols
| SEP | Description |
|-----|-------------|
| [SEP-01](docs/sep/sep-01.md) | stellar.toml configuration |
| [SEP-02](docs/sep/sep-02.md) | Federation protocol |
| [SEP-05](docs/sep/sep-05.md) | Key derivation (BIP-39 mnemonics) |
| [SEP-06](docs/sep/sep-06.md) | Programmatic deposit/withdrawal |
| [SEP-07](docs/sep/sep-07.md) | URI scheme for signing |
| [SEP-08](docs/sep/sep-08.md) | Regulated assets |
| [SEP-10](docs/sep/sep-10.md) | Web authentication |
| [SEP-11](docs/sep/sep-11.md) | Txrep (transaction representation) |
| [SEP-12](docs/sep/sep-12.md) | KYC API |
| [SEP-24](docs/sep/sep-24.md) | Interactive deposit/withdrawal |
| [SEP-30](docs/sep/sep-30.md) | Account recovery |
| [SEP-31](docs/sep/sep-31.md) | Cross-border payments |
| [SEP-38](docs/sep/sep-38.md) | Anchor RFQ API |
| [SEP-45](docs/sep/sep-45.md) | Contract account authentication |
| [SEP-53](docs/sep/sep-53.md) | Message signing |

### Soroban Smart Contracts
- Contract deployment and invocation
- High-level `SorobanClient` API
- Authorization and multi-party signing
- Type conversions and contract bindings
- Event streaming

## Example Files

The [examples](examples) folder contains standalone examples for individual operations and SEP protocols. These are useful as quick references, though the [documentation](docs/README.md) provides more context and complete workflows.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines on contributing to the SDK.

## License

The Stellar SDK for PHP is licensed under the Apache License 2.0. See [LICENSE](LICENSE) for details.
