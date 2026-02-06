### SEP-0053 - Sign and Verify Messages

Stellar message signing is described in [SEP-0053](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0053.md). The SEP defines the standard way for Stellar account holders to sign and verify arbitrary messages, proving ownership of a private key without exposing it. This is useful for authentication, attestations, and verifying key ownership in contexts outside of transaction signing.

## Quick Start

Generate a keypair, sign a message, and verify the signature:

```php
<?php declare(strict_types=1);

use Soneso\StellarSDK\Crypto\KeyPair;

// Generate a keypair
$keyPair = KeyPair::random();

// Sign a message
$message = "Hello, Stellar!";
$signature = $keyPair->signMessage($message);

// Verify the signature
$isValid = $keyPair->verifyMessage($message, $signature);
echo $isValid ? "Valid signature" : "Invalid signature";
```

## API Reference

### signMessage

Signs a message according to SEP-53 specification and returns the raw signature bytes:

```php
public function signMessage(string $message): ?string
```

**Parameters:**
- `$message` - The message to sign (arbitrary text or binary data)

**Returns:**
- Raw signature bytes (64 bytes), or `null` if signing fails

**Throws:**
- `TypeError` - If the keypair has no private key (under strict_types=1)

**Encoding for transmission:**
SEP-53 does not mandate a specific string encoding format. Use `base64_encode()` or `bin2hex()` to encode the signature for transmission or storage:

```php
<?php declare(strict_types=1);

use Soneso\StellarSDK\Crypto\KeyPair;

$keyPair = KeyPair::fromSeed("SBXXX...");
$signature = $keyPair->signMessage($message);

if ($signature !== null) {
    $base64Signature = base64_encode($signature);
    $hexSignature = bin2hex($signature);
}
```

### verifyMessage

Verifies a SEP-53 message signature using this keypair's public key:

```php
public function verifyMessage(string $message, string $signature): bool
```

**Parameters:**
- `$message` - The original message that was signed
- `$signature` - The raw signature bytes to verify

**Returns:**
- `true` if the signature is valid for this message and public key, `false` otherwise

**Decoding received signatures:**
If you receive a signature as a base64 or hex string, decode it before passing to `verifyMessage()`:

```php
<?php declare(strict_types=1);

use Soneso\StellarSDK\Crypto\KeyPair;

$publicKeyPair = KeyPair::fromAccountId("GXXX...");
$signature = base64_decode($base64Signature);
$isValid = $publicKeyPair->verifyMessage($message, $signature);
```

## Usage Examples

### Signing and encoding a message

Sign a message and encode the result for transmission over HTTP or storage:

```php
<?php declare(strict_types=1);

use Soneso\StellarSDK\Crypto\KeyPair;

$keyPair = KeyPair::fromSeed("SBXXX..."); // Your secret seed

$message = "I authorize payment #12345";
$signature = $keyPair->signMessage($message);

if ($signature === null) {
    throw new RuntimeException("Failed to sign message");
}

// Encode for transmission as base64
$base64Signature = base64_encode($signature);
echo "Signature (base64): " . $base64Signature . PHP_EOL;

// Or encode as hex
$hexSignature = bin2hex($signature);
echo "Signature (hex): " . $hexSignature . PHP_EOL;
```

### Signing binary data

Sign arbitrary binary data such as file contents or serialized objects:

```php
<?php declare(strict_types=1);

use Soneso\StellarSDK\Crypto\KeyPair;

$keyPair = KeyPair::fromSeed("SBXXX...");

// Sign binary data (e.g., file contents or serialized data)
$binaryData = file_get_contents("document.pdf");
$signature = $keyPair->signMessage($binaryData);

if ($signature !== null) {
    $base64Signature = base64_encode($signature);
    echo "Document signed: " . $base64Signature . PHP_EOL;
}
```

### Verifying a message with base64 signature

Verify a signature received from a client as a base64-encoded string:

```php
<?php declare(strict_types=1);

use Soneso\StellarSDK\Crypto\KeyPair;

// Public key of the signer
$publicKeyPair = KeyPair::fromAccountId("GXXX...");

$message = "I authorize payment #12345";
$base64Signature = "YWJjZGVm..."; // Received from client

// Decode and verify
$signature = base64_decode($base64Signature);
$isValid = $publicKeyPair->verifyMessage($message, $signature);

if ($isValid) {
    echo "Signature verified: message was signed by this account" . PHP_EOL;
} else {
    echo "Invalid signature: message was not signed by this account" . PHP_EOL;
}
```

### Verifying a message with hex signature

Verify a signature received as a hexadecimal string:

```php
<?php declare(strict_types=1);

use Soneso\StellarSDK\Crypto\KeyPair;

$publicKeyPair = KeyPair::fromAccountId("GXXX...");

$message = "User consent granted at 2025-10-05T12:34:56Z";
$hexSignature = "a1b2c3d4..."; // Received as hex string

// Decode from hex and verify
$signature = hex2bin($hexSignature);
$isValid = $publicKeyPair->verifyMessage($message, $signature);
```

### Handling verification failure

Handle verification failures and provide appropriate feedback:

```php
<?php declare(strict_types=1);

use Soneso\StellarSDK\Crypto\KeyPair;

$publicKeyPair = KeyPair::fromAccountId("GXXX...");

$message = "Expected message";
$signature = base64_decode($receivedSignature);

if (!$publicKeyPair->verifyMessage($message, $signature)) {
    // Verification failed - signature does not match
    // Possible causes:
    // - Message was modified
    // - Signature was modified or corrupted
    // - Wrong public key used for verification
    // - Signature was created for a different message

    http_response_code(401);
    echo json_encode(["error" => "Invalid signature"]);
    exit;
}

// Signature is valid, proceed with request
echo "Authenticated successfully" . PHP_EOL;
```

## Test Vectors

Official test vectors from the SEP-53 specification. Use these to validate cross-SDK compatibility:

### ASCII Message Test Vector

```php
<?php declare(strict_types=1);

use Soneso\StellarSDK\Crypto\KeyPair;

$seed = "SAKICEVQLYWGSOJS4WW7HZJWAHZVEEBS527LHK5V4MLJALYKICQCJXMW";
$expectedAccountId = "GBXFXNDLV4LSWA4VB7YIL5GBD7BVNR22SGBTDKMO2SBZZHDXSKZYCP7L";
$message = "Hello, World!";

$keyPair = KeyPair::fromSeed($seed);
assert($keyPair->getAccountId() === $expectedAccountId, "Account ID mismatch");

$signature = $keyPair->signMessage($message);
$base64Signature = base64_encode($signature);
$hexSignature = bin2hex($signature);

// Expected signatures from SEP-53 specification
$expectedBase64 = "fO5dbYhXUhBMhe6kId/cuVq/AfEnHRHEvsP8vXh03M1uLpi5e46yO2Q8rEBzu3feXQewcQE5GArp88u6ePK6BA==";
$expectedHex = "7cee5d6d885752104c85eea421dfdcb95abf01f1271d11c4bec3fcbd7874dccd6e2e98b97b8eb23b643cac4073bb77de5d07b0710139180ae9f3cbba78f2ba04";

assert($base64Signature === $expectedBase64, "Base64 signature mismatch");
assert($hexSignature === $expectedHex, "Hex signature mismatch");

// Also verify the signature works
assert($keyPair->verifyMessage($message, $signature), "Verification failed");

echo "ASCII test vector: PASSED" . PHP_EOL;
```

### Japanese (UTF-8) Message Test Vector

```php
<?php declare(strict_types=1);

use Soneso\StellarSDK\Crypto\KeyPair;

$seed = "SAKICEVQLYWGSOJS4WW7HZJWAHZVEEBS527LHK5V4MLJALYKICQCJXMW";
$message = "こんにちは、世界！"; // "Hello, World!" in Japanese

$keyPair = KeyPair::fromSeed($seed);
$signature = $keyPair->signMessage($message);

$expectedBase64 = "CDU265Xs8y3OWbB/56H9jPgUss5G9A0qFuTqH2zs2YDgTm+++dIfmAEceFqB7bhfN3am59lCtDXrCtwH2k1GBA==";
$expectedHex = "083536eb95ecf32dce59b07fe7a1fd8cf814b2ce46f40d2a16e4ea1f6cecd980e04e6fbef9d21f98011c785a81edb85f3776a6e7d942b435eb0adc07da4d4604";

assert(base64_encode($signature) === $expectedBase64, "Base64 signature mismatch");
assert(bin2hex($signature) === $expectedHex, "Hex signature mismatch");
assert($keyPair->verifyMessage($message, $signature), "Verification failed");

echo "Japanese (UTF-8) test vector: PASSED" . PHP_EOL;
```

### Binary Data Test Vector

```php
<?php declare(strict_types=1);

use Soneso\StellarSDK\Crypto\KeyPair;

$seed = "SAKICEVQLYWGSOJS4WW7HZJWAHZVEEBS527LHK5V4MLJALYKICQCJXMW";

// Binary message (provided as base64 in the spec)
$messageBase64 = "2zZDP1sa1BVBfLP7TeeMk3sUbaxAkUhBhDiNdrksaFo=";
$message = base64_decode($messageBase64);

$keyPair = KeyPair::fromSeed($seed);
$signature = $keyPair->signMessage($message);

$expectedBase64 = "VA1+7hefNwv2NKScH6n+Sljj15kLAge+M2wE7fzFOf+L0MMbssA1mwfJZRyyrhBORQRle10X1Dxpx+UOI4EbDQ==";
$expectedHex = "540d7eee179f370bf634a49c1fa9fe4a58e3d7990b0207be336c04edfcc539ff8bd0c31bb2c0359b07c9651cb2ae104e4504657b5d17d43c69c7e50e23811b0d";

assert(base64_encode($signature) === $expectedBase64, "Base64 signature mismatch");
assert(bin2hex($signature) === $expectedHex, "Hex signature mismatch");
assert($keyPair->verifyMessage($message, $signature), "Verification failed");

echo "Binary data test vector: PASSED" . PHP_EOL;
```

## Protocol Details

SEP-53 message signing follows this process:

1. **Prefix the message**: Prepend `"Stellar Signed Message:\n"` to the message
2. **Hash**: Calculate SHA-256 hash of the prefixed message
3. **Sign**: Sign the hash using Ed25519 with the private key

The formula is:
```
signature = Ed25519Sign(privkey, SHA256("Stellar Signed Message:\n" || message))
```

Verification follows the inverse process:
```
valid = Ed25519Verify(pubkey, SHA256("Stellar Signed Message:\n" || message), signature)
```

The prefix provides domain separation, ensuring that a signed message cannot be interpreted as a valid Stellar transaction.

## Security Notes

### Domain Separation

The `"Stellar Signed Message:\n"` prefix prevents signed messages from being confused with transaction signatures. Without this prefix, a malicious party could potentially trick a user into signing what appears to be a message but is actually a transaction hash.

### Key Ownership vs Account Control

SEP-53 signatures prove ownership of a private key, not necessarily control of a Stellar account. Consider these scenarios:

- **Multi-signature accounts**: A signature from one key proves ownership of that key, but the account may require multiple signatures for transactions
- **Revoked signers**: A key may have been removed from an account's signer list after the signature was created
- **Weight thresholds**: The signing key may not have sufficient weight to authorize account operations

For critical operations, verify the current account state on the Stellar ledger, not just the message signature.

### User Confirmation

Applications MUST display the full message content to users before signing. This prevents phishing attacks where users unknowingly sign malicious content. Never auto-sign messages without explicit user review.

### Handling Signing Failures

The `signMessage()` method can return `null` on cryptographic errors, or throw a `TypeError` if the keypair has no private key. Always check for both:

```php
<?php declare(strict_types=1);

use Soneso\StellarSDK\Crypto\KeyPair;

$keyPair = KeyPair::fromSeed("SBXXX...");

try {
    $signature = $keyPair->signMessage("test message");
    
    if ($signature === null) {
        // Cryptographic error occurred
        throw new RuntimeException("Signing failed due to crypto error");
    }
    
    // Success - use the signature
    $base64Signature = base64_encode($signature);
    
} catch (\TypeError $e) {
    // Keypair has no private key
    throw new RuntimeException("Cannot sign: no private key available");
}
```

## Interoperability

SEP-53 implementations are compatible across Stellar SDKs. A signature created with one SDK can be verified by another.

### Verifying a signature from another SDK

Verify signatures created by Java, Python, Flutter, or any other SEP-53 compliant SDK:

```php
<?php declare(strict_types=1);

use Soneso\StellarSDK\Crypto\KeyPair;

// Signature created by Java SDK, Python SDK, or Flutter SDK
$base64Signature = "hZ3+..."; // Received from another SDK
$message = "Cross-platform message";

$publicKeyPair = KeyPair::fromAccountId("GXXX...");
$signature = base64_decode($base64Signature);

if ($publicKeyPair->verifyMessage($message, $signature)) {
    echo "Signature verified across SDKs" . PHP_EOL;
}
```

**Compatible SDKs:**
- Java Stellar SDK
- Python Stellar SDK
- Flutter Stellar SDK
- JavaScript Stellar SDK

All implementations follow the same SEP-53 specification with identical prefix, hashing, and signing algorithms.

## References

- [SEP-0053 Specification](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0053.md)
- [Stellar Developer Documentation](https://developers.stellar.org/)
- [KeyPair Source Code](https://github.com/Soneso/stellar-php-sdk/blob/main/Soneso/StellarSDK/Crypto/KeyPair.php)
