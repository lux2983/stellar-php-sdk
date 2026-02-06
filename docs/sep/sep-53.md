# SEP-53: Sign and Verify Messages

Prove ownership of a Stellar private key by signing arbitrary messages.

## Overview

SEP-53 defines how to sign and verify messages with Stellar keypairs. Use it when you need to:

- Authenticate users by proving key ownership
- Sign attestations or consent agreements
- Verify signatures from other Stellar SDKs
- Create provable off-chain statements

The protocol adds a prefix (`"Stellar Signed Message:\n"`) before hashing, which prevents signed messages from being confused with transaction signatures.

## Quick Example

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;

// Sign a message
$keyPair = KeyPair::fromSeed("SXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX");
$signature = $keyPair->signMessage("I agree to the terms of service");

// Verify the signature
$isValid = $keyPair->verifyMessage("I agree to the terms of service", $signature);
echo $isValid ? "Valid\n" : "Invalid\n";
```

## Detailed Usage

### Signing Messages

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;

$keyPair = KeyPair::fromSeed("SXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX");

$message = "User consent granted at 2025-01-15T12:00:00Z";
$signature = $keyPair->signMessage($message);

// Returns raw 64 bytes - encode for transmission
$base64Signature = base64_encode($signature);
echo "Signature: " . $base64Signature . "\n";

// Or as hex
$hexSignature = bin2hex($signature);
echo "Signature (hex): " . $hexSignature . "\n";
```

### Verifying Messages

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;

// Verify with just the public key
$publicKey = KeyPair::fromAccountId("GABC...");

$message = "User consent granted at 2025-01-15T12:00:00Z";
$base64Signature = "..."; // Received from client

$signature = base64_decode($base64Signature);
$isValid = $publicKey->verifyMessage($message, $signature);

if ($isValid) {
    echo "Signature verified\n";
} else {
    echo "Invalid signature\n";
}
```

### Verifying Hex-Encoded Signatures

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;

$publicKey = KeyPair::fromAccountId("GABC...");

$hexSignature = "a1b2c3d4..."; // Received as hex
$signature = hex2bin($hexSignature);

$isValid = $publicKey->verifyMessage($message, $signature);
```

### Signing Binary Data

The message doesn't have to be text:

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;

$keyPair = KeyPair::fromSeed("SXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX");

// Sign file contents
$fileContents = file_get_contents("document.pdf");
$signature = $keyPair->signMessage($fileContents);

$base64Signature = base64_encode($signature);
```

### Authentication Flow Example

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;

// Server generates a challenge
$challenge = "authenticate:" . bin2hex(random_bytes(16)) . ":" . time();

// Client signs the challenge
$clientKeyPair = KeyPair::fromSeed("SXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX");
$signature = $clientKeyPair->signMessage($challenge);
$response = [
    'account_id' => $clientKeyPair->getAccountId(),
    'signature' => base64_encode($signature),
    'challenge' => $challenge
];

// Server verifies
$publicKey = KeyPair::fromAccountId($response['account_id']);
$signature = base64_decode($response['signature']);

if ($publicKey->verifyMessage($response['challenge'], $signature)) {
    echo "User authenticated as " . $response['account_id'] . "\n";
} else {
    echo "Authentication failed\n";
}
```

## Error Handling

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;

// Signing requires a private key
$publicKeyOnly = KeyPair::fromAccountId("GABC...");

try {
    // This throws TypeError - no private key available
    $signature = $publicKeyOnly->signMessage("test");
} catch (\TypeError $e) {
    echo "Cannot sign: keypair has no private key\n";
}
```

### Common Verification Failures

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;

$publicKey = KeyPair::fromAccountId("GABC...");
$signature = base64_decode($receivedSignature);

if (!$publicKey->verifyMessage($message, $signature)) {
    // Possible causes:
    // 1. Message was modified
    // 2. Signature was modified/corrupted
    // 3. Wrong public key used
    // 4. Signature was for a different message
    
    http_response_code(401);
    echo json_encode(["error" => "Invalid signature"]);
    exit;
}
```

## Protocol Details

SEP-53 signing works like this:

```
signature = Ed25519Sign(privateKey, SHA256("Stellar Signed Message:\n" + message))
```

Verification reverses it:

```
valid = Ed25519Verify(publicKey, SHA256("Stellar Signed Message:\n" + message), signature)
```

The `"Stellar Signed Message:\n"` prefix provides domain separation. A signed message can never be confused with a Stellar transaction signature.

## Security Notes

### Display Messages Before Signing

Always show users the full message before signing. Never auto-sign without user review. This prevents phishing where users sign malicious content.

### Key Ownership vs Account Control

A valid signature proves the signer has the private key. It doesn't prove they control the account:

- **Multi-sig accounts**: One signature doesn't mean transaction authority
- **Revoked signers**: A key may have been removed from the account
- **Weight thresholds**: The key may lack sufficient weight

For critical operations, check the account's current state on-chain.

### Signature Encoding

SEP-53 doesn't specify an encoding format. Common choices:

| Encoding | Pros | Cons |
|----------|------|------|
| Base64 | Compact, URL-safe variant available | Needs decode |
| Hex | Human-readable, simple | 2x larger |

Pick one and document it. The raw signature is always 64 bytes.

## Cross-SDK Compatibility

SEP-53 signatures work across all Stellar SDKs. A signature created in Java, Python, or Flutter can be verified in PHP and vice versa.

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;

// Signature from Java/Python/Flutter SDK
$base64Signature = "...";
$message = "Cross-platform message";

$publicKey = KeyPair::fromAccountId("GABC...");
$signature = base64_decode($base64Signature);

if ($publicKey->verifyMessage($message, $signature)) {
    echo "Verified across SDKs\n";
}
```

## Related SEPs

- [SEP-10](sep-10.md) - Web authentication for accounts
- [SEP-45](sep-45.md) - Web authentication for contract accounts

## Reference

- [SEP-53 Specification](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0053.md)
- [KeyPair Source Code](https://github.com/Soneso/stellar-php-sdk/blob/main/Soneso/StellarSDK/Crypto/KeyPair.php)
