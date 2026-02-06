# SEP-30: Account Recovery

SEP-30 defines a protocol for recovering access to Stellar accounts when the owner loses their private key. Recovery servers act as additional signers on an account, allowing the user to regain control by proving their identity through alternate methods like email, phone, or another Stellar address.

Use SEP-30 when:
- Building a wallet with account recovery features
- You want to protect users from permanent key loss
- Implementing shared account access between multiple parties
- Setting up multi-device account access with recovery options

See the [SEP-30 specification](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0030.md) for protocol details.

## How Recovery Works

1. **Registration**: Register your account with a recovery server, providing identity information
2. **Add Signer**: Add the server's signer key to your Stellar account
3. **Recovery**: If you lose your key, authenticate with the recovery server via alternate methods
4. **Sign Transaction**: The server signs a transaction that adds your new key to the account

## Quick Example

```php
<?php

use Soneso\StellarSDK\SEP\Recovery\RecoveryService;
use Soneso\StellarSDK\SEP\Recovery\SEP30Request;
use Soneso\StellarSDK\SEP\Recovery\SEP30RequestIdentity;
use Soneso\StellarSDK\SEP\Recovery\SEP30AuthMethod;

// Connect to recovery server
$service = new RecoveryService("https://recovery.example.com");

// Set up identity with authentication methods
$authMethods = [
    new SEP30AuthMethod("email", "user@example.com"),
    new SEP30AuthMethod("phone_number", "+14155551234"),
];
$identity = new SEP30RequestIdentity("owner", $authMethods);

// Register account with recovery server (requires SEP-10 JWT)
$request = new SEP30Request([$identity]);
$response = $service->registerAccount($accountId, $request, $jwtToken);

// Get the signer key to add to your account
$signerKey = $response->signers[0]->key;
echo "Add this signer to your account: $signerKey\n";
```

## Creating the Recovery Service

```php
<?php

use Soneso\StellarSDK\SEP\Recovery\RecoveryService;

// Create from recovery server URL
$service = new RecoveryService("https://recovery.example.com");
```

## Registering an Account

Register your Stellar account with the recovery server:

```php
<?php

use Soneso\StellarSDK\SEP\Recovery\RecoveryService;
use Soneso\StellarSDK\SEP\Recovery\SEP30Request;
use Soneso\StellarSDK\SEP\Recovery\SEP30RequestIdentity;
use Soneso\StellarSDK\SEP\Recovery\SEP30AuthMethod;

$service = new RecoveryService("https://recovery.example.com");

// Define how the user can prove their identity
$authMethods = [
    new SEP30AuthMethod("stellar_address", "GXXXX..."), // SEP-10 auth
    new SEP30AuthMethod("email", "user@example.com"),
    new SEP30AuthMethod("phone_number", "+14155551234"),
];

// Create identity with role "owner"
$identity = new SEP30RequestIdentity("owner", $authMethods);

// Register with the recovery server
$request = new SEP30Request([$identity]);
$response = $service->registerAccount($accountId, $request, $jwtToken);

// The response includes signer keys to add to your Stellar account
foreach ($response->signers as $signer) {
    echo "Signer key: " . $signer->key . "\n";
}
```

### Adding the Recovery Signer to Your Account

After registration, add the recovery server's signer to your Stellar account:

```php
<?php

use Soneso\StellarSDK\StellarSDK;
use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\SetOptionsOperationBuilder;
use Soneso\StellarSDK\TransactionBuilder;
use Soneso\StellarSDK\Network;

$sdk = StellarSDK::getTestNetInstance();

$accountKeyPair = KeyPair::fromSeed("SXXXXXX...");
$accountId = $accountKeyPair->getAccountId();
$account = $sdk->requestAccount($accountId);

// Add recovery server as a signer with weight 1
$signerKey = $response->signers[0]->key; // From registration response
$operation = (new SetOptionsOperationBuilder())
    ->setSigner($signerKey, 1) // Weight 1
    ->build();

// Set thresholds so recovery requires multiple signers
$thresholdOp = (new SetOptionsOperationBuilder())
    ->setHighThreshold(2)
    ->setMediumThreshold(2)
    ->setLowThreshold(2)
    ->build();

$transaction = (new TransactionBuilder($account))
    ->addOperation($operation)
    ->addOperation($thresholdOp)
    ->build();

$transaction->sign($accountKeyPair, Network::testnet());
$sdk->submitTransaction($transaction);
```

## Multi-Server Recovery

For better security, register with multiple recovery servers:

```php
<?php

use Soneso\StellarSDK\SEP\Recovery\RecoveryService;
use Soneso\StellarSDK\SEP\Recovery\SEP30Request;
use Soneso\StellarSDK\SEP\Recovery\SEP30RequestIdentity;
use Soneso\StellarSDK\SEP\Recovery\SEP30AuthMethod;

// Register with first recovery server
$service1 = new RecoveryService("https://recovery1.example.com");
$authMethods = [new SEP30AuthMethod("email", "user@example.com")];
$identity = new SEP30RequestIdentity("owner", $authMethods);
$request = new SEP30Request([$identity]);

$response1 = $service1->registerAccount($accountId, $request, $jwtToken1);
$signerKey1 = $response1->signers[0]->key;

// Register with second recovery server
$service2 = new RecoveryService("https://recovery2.example.com");
$response2 = $service2->registerAccount($accountId, $request, $jwtToken2);
$signerKey2 = $response2->signers[0]->key;

// Add both signers, each with weight 1
// Set threshold to 2, so both servers must sign for recovery
```

## Recovering an Account

When you lose your private key, recover by authenticating with the recovery server:

```php
<?php

use Soneso\StellarSDK\StellarSDK;
use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\SetOptionsOperationBuilder;
use Soneso\StellarSDK\TransactionBuilder;
use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\SEP\Recovery\RecoveryService;

$service = new RecoveryService("https://recovery.example.com");

// Get account details to find signing address
$accountDetails = $service->accountDetails($accountId, $recoveryJwt);
$signingAddress = $accountDetails->signers[0]->key;

// Generate a new keypair for the recovered account
$newKeyPair = KeyPair::random();
$newPublicKey = $newKeyPair->getAccountId();

// Build a transaction to add the new key
$sdk = StellarSDK::getTestNetInstance();
$account = $sdk->requestAccount($accountId);

$operation = (new SetOptionsOperationBuilder())
    ->setSigner($newPublicKey, 10) // High weight to regain control
    ->build();

$transaction = (new TransactionBuilder($account))
    ->addOperation($operation)
    ->build();

// Get the recovery server to sign
$txBase64 = $transaction->toEnvelopeXdrBase64();
$signatureResponse = $service->signTransaction(
    $accountId,
    $signingAddress,
    $txBase64,
    $recoveryJwt // JWT proving identity via alternate auth
);

// Add the server's signature and submit
$signature = $signatureResponse->signature;
$transaction->addSignature($signingAddress, base64_decode($signature));

$sdk->submitTransaction($transaction);

echo "Account recovered! New key: " . $newKeyPair->getSecretSeed() . "\n";
```

## Updating Identity Information

Update authentication methods for a registered account:

```php
<?php

use Soneso\StellarSDK\SEP\Recovery\RecoveryService;
use Soneso\StellarSDK\SEP\Recovery\SEP30Request;
use Soneso\StellarSDK\SEP\Recovery\SEP30RequestIdentity;
use Soneso\StellarSDK\SEP\Recovery\SEP30AuthMethod;

$service = new RecoveryService("https://recovery.example.com");

// New auth methods (replaces all existing ones)
$newAuthMethods = [
    new SEP30AuthMethod("email", "newemail@example.com"),
    new SEP30AuthMethod("phone_number", "+14155559999"),
];
$identity = new SEP30RequestIdentity("owner", $newAuthMethods);

$request = new SEP30Request([$identity]);
$response = $service->updateIdentitiesForAccount($accountId, $request, $jwtToken);
```

## Shared Account Access

SEP-30 supports multiple parties sharing access to an account:

```php
<?php

use Soneso\StellarSDK\SEP\Recovery\RecoveryService;
use Soneso\StellarSDK\SEP\Recovery\SEP30Request;
use Soneso\StellarSDK\SEP\Recovery\SEP30RequestIdentity;
use Soneso\StellarSDK\SEP\Recovery\SEP30AuthMethod;

$service = new RecoveryService("https://recovery.example.com");

// Primary owner
$ownerAuth = [new SEP30AuthMethod("email", "owner@example.com")];
$ownerIdentity = new SEP30RequestIdentity("owner", $ownerAuth);

// Shared user
$sharedAuth = [new SEP30AuthMethod("email", "partner@example.com")];
$sharedIdentity = new SEP30RequestIdentity("shared", $sharedAuth);

// Register both identities
$request = new SEP30Request([$ownerIdentity, $sharedIdentity]);
$response = $service->registerAccount($accountId, $request, $jwtToken);

// Both users can now recover the account
```

## Getting Account Details

Check registration status and current signers:

```php
<?php

use Soneso\StellarSDK\SEP\Recovery\RecoveryService;

$service = new RecoveryService("https://recovery.example.com");

$response = $service->accountDetails($accountId, $jwtToken);

echo "Address: " . $response->address . "\n";

foreach ($response->identities as $identity) {
    echo "Identity role: " . $identity->role . "\n";
}

foreach ($response->signers as $signer) {
    echo "Signer key: " . $signer->key . "\n";
}
```

## Listing Registered Accounts

List all accounts registered under your identity:

```php
<?php

use Soneso\StellarSDK\SEP\Recovery\RecoveryService;

$service = new RecoveryService("https://recovery.example.com");

// Get first page
$response = $service->accounts($jwtToken);

foreach ($response->accounts as $account) {
    echo "Account: " . $account->address . "\n";
}

// Get next page (pagination)
if (count($response->accounts) > 0) {
    $lastAddress = end($response->accounts)->address;
    $nextPage = $service->accounts($jwtToken, after: $lastAddress);
}
```

## Deleting Registration

Remove your account from the recovery server:

```php
<?php

use Soneso\StellarSDK\SEP\Recovery\RecoveryService;

$service = new RecoveryService("https://recovery.example.com");

$response = $service->deleteAccount($accountId, $jwtToken);

// After deletion, also remove the server's signer from your Stellar account
```

## Error Handling

```php
<?php

use Soneso\StellarSDK\SEP\Recovery\RecoveryService;
use Soneso\StellarSDK\SEP\Recovery\SEP30Request;
use Soneso\StellarSDK\SEP\Recovery\SEP30RequestIdentity;
use Soneso\StellarSDK\SEP\Recovery\SEP30AuthMethod;
use Soneso\StellarSDK\SEP\Recovery\SEP30BadRequestResponseException;
use Soneso\StellarSDK\SEP\Recovery\SEP30UnauthorizedResponseException;
use Soneso\StellarSDK\SEP\Recovery\SEP30NotFoundResponseException;
use Soneso\StellarSDK\SEP\Recovery\SEP30ConflictResponseException;
use GuzzleHttp\Exception\GuzzleException;

$service = new RecoveryService("https://recovery.example.com");

try {
    $authMethods = [new SEP30AuthMethod("email", "user@example.com")];
    $identity = new SEP30RequestIdentity("owner", $authMethods);
    $request = new SEP30Request([$identity]);
    
    $response = $service->registerAccount($accountId, $request, $jwtToken);
    
} catch (SEP30BadRequestResponseException $e) {
    // Invalid request data
    echo "Bad request: " . $e->getMessage() . "\n";
    
} catch (SEP30UnauthorizedResponseException $e) {
    // JWT token invalid or insufficient
    echo "Unauthorized: " . $e->getMessage() . "\n";
    
} catch (SEP30NotFoundResponseException $e) {
    // Account not found (for queries/updates)
    echo "Not found: " . $e->getMessage() . "\n";
    
} catch (SEP30ConflictResponseException $e) {
    // Account already registered
    echo "Conflict: " . $e->getMessage() . "\n";
    
} catch (GuzzleException $e) {
    // Network or HTTP error
    echo "Request failed: " . $e->getMessage() . "\n";
}
```

## Authentication Methods

SEP-30 supports these standard authentication types:

| Type | Format | Example |
|------|--------|---------|
| `stellar_address` | G... public key | `GDUAB...` |
| `phone_number` | E.164 format | `+14155551234` |
| `email` | Standard email | `user@example.com` |

The `stellar_address` type uses SEP-10 authentication and provides the strongest security. Phone and email authentication depend on the recovery server's verification process.

## Security Considerations

- **Multi-server setup**: Use 2+ recovery servers with account threshold set to require multiple signatures
- **Signer weights**: Give each recovery server weight=1 and set thresholds to 2+
- **Phone auth risks**: Phone numbers are vulnerable to SIM swapping attacks
- **Regular checks**: Periodically verify your recovery setup with `accountDetails()`
- **Key rotation**: Update authentication methods if phone numbers or emails change
- **HTTPS only**: All recovery server communication should use HTTPS

## Related SEPs

- [SEP-10](sep-10.md) - Web Authentication (used for stellar_address auth method)
- [Stellar Accounts](../getting-started.md#accounts) - How to manage account signers and thresholds
