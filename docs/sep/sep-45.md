# SEP-45: Web Authentication for Contract Accounts

Authenticate Soroban smart contract accounts (C... addresses) with anchor services.

## Overview

SEP-45 lets contracts prove they control an account by signing authorization entries. Use it when you need to:

- Authenticate a Soroban contract with an anchor service
- Access SEP-24 deposits/withdrawals from a contract account
- Use SEP-12 KYC or SEP-38 quotes with contract accounts

**SEP-45 vs SEP-10:**
- SEP-45: For contract accounts (C... addresses)
- SEP-10: For traditional accounts (G... and M... addresses)

The flow works like this:
1. Request a challenge from the server
2. Sign authorization entries with your contract's registered signers
3. Server simulates the transaction to verify your contract's `__check_auth`
4. Server returns a JWT token for accessing protected services

## Quick Example

```php
<?php

use Soneso\StellarSDK\SEP\WebAuthForContracts\WebAuthForContracts;
use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Network;

// Your contract account (must implement __check_auth)
$contractId = "CCIBUCGPOHWMMMFPFTDWBSVHQRT4DIBJ7AD6BZJYDITBK2LCVBYW7HUQ";

// Signer registered in your contract
$signer = KeyPair::fromSeed("SXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX");

// Authenticate
$webAuth = WebAuthForContracts::fromDomain("anchor.example.com", Network::testnet());
$jwtToken = $webAuth->jwtToken($contractId, [$signer]);

echo "Authenticated! Token: " . substr($jwtToken, 0, 50) . "...\n";
```

## Detailed Usage

### Prerequisites

Your contract must:
1. Be deployed on the Stellar network
2. Implement `__check_auth` to define authorization rules
3. Have signer public keys stored in its contract storage

### Creating the Service

From stellar.toml (recommended):

```php
<?php

use Soneso\StellarSDK\SEP\WebAuthForContracts\WebAuthForContracts;
use Soneso\StellarSDK\Network;

$webAuth = WebAuthForContracts::fromDomain("anchor.example.com", Network::testnet());
```

Or with manual configuration:

```php
<?php

use Soneso\StellarSDK\SEP\WebAuthForContracts\WebAuthForContracts;
use Soneso\StellarSDK\Network;

$webAuth = new WebAuthForContracts(
    authEndpoint: "https://anchor.example.com/auth/sep45",
    webAuthContractId: "CCALHRGH5RXIDJDRLPPG4ZX2S563TB2QKKJR4STWKVQCYB6JVPYQXHRG",
    serverSigningKey: "GBWMCCC3NHSKLAOJDBKKYW7SSH2PFTTNVFKWSGLWGDLEBKLOVP5JLBBP",
    serverHomeDomain: "anchor.example.com",
    network: Network::testnet()
);
```

### Basic Authentication

```php
<?php

use Soneso\StellarSDK\SEP\WebAuthForContracts\WebAuthForContracts;
use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Network;

$contractId = "CCIBUCGPOHWMMMFPFTDWBSVHQRT4DIBJ7AD6BZJYDITBK2LCVBYW7HUQ";
$signer = KeyPair::fromSeed("SXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX");

$webAuth = WebAuthForContracts::fromDomain("anchor.example.com", Network::testnet());

// The SDK handles the full flow:
// - Requests challenge
// - Validates authorization entries
// - Signs with your keypair
// - Submits for JWT
$jwtToken = $webAuth->jwtToken($contractId, [$signer]);
```

### Signature Expiration

Signatures include an expiration ledger for replay protection. By default, the SDK sets this to current ledger + 10 (~50-60 seconds).

```php
<?php

use Soneso\StellarSDK\SEP\WebAuthForContracts\WebAuthForContracts;
use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Network;

$webAuth = WebAuthForContracts::fromDomain("anchor.example.com", Network::testnet());

// Custom expiration ledger
$jwtToken = $webAuth->jwtToken(
    $contractId,
    [$signer],
    homeDomain: null,
    clientDomain: null,
    clientDomainKeyPair: null,
    clientDomainSigningCallback: null,
    signatureExpirationLedger: 1500000
);
```

### Contracts Without Signature Requirements

Some contracts implement `__check_auth` without requiring signatures. Pass an empty signers array:

```php
// No signers needed if contract allows it
$jwtToken = $webAuth->jwtToken($contractId, []);
```

### Client Domain Verification

Non-custodial wallets can prove their domain to the anchor. Your domain needs a stellar.toml with a `SIGNING_KEY`.

```php
<?php

use Soneso\StellarSDK\SEP\WebAuthForContracts\WebAuthForContracts;
use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Network;

$contractId = "CCIBUCGPOHWMMMFPFTDWBSVHQRT4DIBJ7AD6BZJYDITBK2LCVBYW7HUQ";
$signer = KeyPair::fromSeed("SXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX");

// Your wallet's SIGNING_KEY from stellar.toml
$clientDomainKeyPair = KeyPair::fromSeed("SYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYY");

$webAuth = WebAuthForContracts::fromDomain("anchor.example.com", Network::testnet());

$jwtToken = $webAuth->jwtToken(
    $contractId,
    [$signer],
    homeDomain: "anchor.example.com",
    clientDomain: "wallet.example.com",
    clientDomainKeyPair: $clientDomainKeyPair
);
```

### Remote Client Domain Signing

For remote signing, use the `clientDomainSigningCallback` parameter. Your callback receives a `SorobanAuthorizationEntry` and must return the signed entry. See the [SDK examples](https://github.com/Soneso/stellar-php-sdk/blob/main/examples/sep-0045-webauth-contracts.md) for a complete implementation.

## Error Handling

```php
<?php

use Soneso\StellarSDK\SEP\WebAuthForContracts\WebAuthForContracts;
use Soneso\StellarSDK\SEP\WebAuthForContracts\ContractChallengeValidationErrorInvalidContractAddress;
use Soneso\StellarSDK\SEP\WebAuthForContracts\ContractChallengeValidationErrorSubInvocationsFound;
use Soneso\StellarSDK\SEP\WebAuthForContracts\ContractChallengeValidationErrorInvalidServerSignature;
use Soneso\StellarSDK\SEP\WebAuthForContracts\ContractChallengeValidationErrorMissingServerEntry;
use Soneso\StellarSDK\SEP\WebAuthForContracts\ContractChallengeValidationErrorMissingClientEntry;
use Soneso\StellarSDK\SEP\WebAuthForContracts\ContractChallengeRequestErrorResponse;
use Soneso\StellarSDK\SEP\WebAuthForContracts\SubmitContractChallengeErrorResponseException;
use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Network;

$webAuth = WebAuthForContracts::fromDomain("anchor.example.com", Network::testnet());

try {
    $jwtToken = $webAuth->jwtToken($contractId, [$signer]);
    
} catch (ContractChallengeValidationErrorInvalidContractAddress $e) {
    // Server's contract doesn't match stellar.toml
    echo "Security error: contract mismatch\n";
    
} catch (ContractChallengeValidationErrorSubInvocationsFound $e) {
    // Challenge contains unauthorized sub-invocations
    echo "Security error: sub-invocations detected\n";
    
} catch (ContractChallengeValidationErrorInvalidServerSignature $e) {
    // Server signature is invalid
    echo "Security error: bad server signature\n";
    
} catch (ContractChallengeValidationErrorMissingServerEntry $e) {
    echo "Invalid challenge: no server entry\n";
    
} catch (ContractChallengeValidationErrorMissingClientEntry $e) {
    echo "Invalid challenge: no client entry\n";
    
} catch (ContractChallengeRequestErrorResponse $e) {
    echo "Challenge request failed: " . $e->getMessage() . "\n";
    
} catch (SubmitContractChallengeErrorResponseException $e) {
    // Often means signer isn't registered in contract's __check_auth
    echo "Auth failed: " . $e->getMessage() . "\n";
}
```

### Common Issues

| Error | Cause | Solution |
|-------|-------|----------|
| `SubmitContractChallengeErrorResponseException` | Signer not in contract's `__check_auth` | Verify signer is registered in contract storage |
| `ContractChallengeValidationErrorInvalidContractAddress` | Contract address mismatch | Check stellar.toml WEB_AUTH_CONTRACT_ID |
| `ContractChallengeValidationErrorSubInvocationsFound` | Malicious challenge | Don't sign; report to anchor |

## Using the JWT Token

```php
<?php

use GuzzleHttp\Client;

$httpClient = new Client();

// Use token with SEP-24
$response = $httpClient->post('https://anchor.example.com/sep24/transactions/deposit/interactive', [
    'headers' => ['Authorization' => 'Bearer ' . $jwtToken],
    'json' => ['asset_code' => 'USDC']
]);

// Use token with SEP-12 KYC
$response = $httpClient->get('https://anchor.example.com/kyc/customer', [
    'headers' => ['Authorization' => 'Bearer ' . $jwtToken]
]);
```

## Related SEPs

- [SEP-10](sep-10.md) - Authentication for traditional accounts (G... addresses)
- [SEP-24](sep-24.md) - Interactive deposit/withdrawal
- [SEP-12](sep-12.md) - KYC API
- [SEP-38](sep-38.md) - Quotes API

## Reference

- [SEP-45 Specification](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0045.md)
- [Account Contract Example](https://github.com/stellar/anchor-platform/tree/main/soroban/contracts/account)
