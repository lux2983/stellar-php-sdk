# SEP-23: StrKey Encoding and Decoding

SEP-23 defines how Stellar encodes addresses between raw binary data and human-readable strings. Each address type starts with a specific letter — account IDs start with "G", secret seeds with "S", contracts with "C", and so on.

**When to use:** Validating user-entered addresses, converting between raw bytes and string representations, working with different key types.

See the [SEP-23 specification](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0023.md) for protocol details.

## Quick Example

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Crypto\StrKey;

// Generate a keypair
$keyPair = KeyPair::random();
$accountId = $keyPair->getAccountId();  // G...

// Validate an address
if (StrKey::isValidAccountId($accountId)) {
    echo "Valid account ID" . PHP_EOL;
}

// Decode to raw bytes and encode back
$rawPublicKey = StrKey::decodeAccountId($accountId);
$encoded = StrKey::encodeAccountId($rawPublicKey);
```

## Account IDs and Secret Seeds

Account IDs (G...) are public keys. Secret seeds (S...) are private keys:

```php
<?php

use Soneso\StellarSDK\Crypto\StrKey;

$accountId = 'GCEZWKCA5VLDNRLN3RPRJMRZOX3Z6G5CHCGSNFHEYVXM3XOJMDS674JZ';
$secretSeed = 'SCZANGBA5YHTNYVVV3C7CAZMTQDBJHJG6C34RFLWOEIA5MPI7YPQAAXX';

// Validate
StrKey::isValidAccountId($accountId);  // true
StrKey::isValidSeed($secretSeed);      // true

// Decode to raw 32-byte keys
$rawPublicKey = StrKey::decodeAccountId($accountId);
$rawPrivateKey = StrKey::decodeSeed($secretSeed);

// Encode raw bytes back to string
$encoded = StrKey::encodeAccountId($rawPublicKey);
$encodedSeed = StrKey::encodeSeed($rawPrivateKey);

// Derive account ID from seed
$accountId = StrKey::accountIdFromSeed($secretSeed);
```

## Muxed Accounts (M...)

Muxed accounts combine an account ID with a 64-bit integer ID:

```php
<?php

use Soneso\StellarSDK\Crypto\StrKey;
use Soneso\StellarSDK\MuxedAccount;

$accountId = 'GAQAA5L65LSYBER7AEES5KJEK32VGMFQ7NQQCC3OHSNNLXK7774VSSRL';
$userId = 1234567890;

$muxedAccount = MuxedAccount::fromAccountId($accountId, $userId);
$muxedAccountId = $muxedAccount->getAccountId(); // M...

// Validate
StrKey::isValidMuxedAccountId($muxedAccountId); // true

// Decode/encode
$rawData = StrKey::decodeMuxedAccountId($muxedAccountId);
$encoded = StrKey::encodeMuxedAccountId($rawData);
```

## Pre-Auth TX and SHA-256 Hashes

Pre-auth transaction hashes (T...) authorize specific transactions in advance. SHA-256 hashes (X...) are for hash-locked transactions:

```php
<?php

use Soneso\StellarSDK\Crypto\StrKey;

// Pre-auth TX (T...)
$transactionHash = random_bytes(32);
$preAuthTx = StrKey::encodePreAuthTx($transactionHash);
StrKey::isValidPreAuthTx($preAuthTx); // true
$decoded = StrKey::decodePreAuthTx($preAuthTx);

// SHA-256 hash signer (X...)
$hash = hash('sha256', 'secret preimage', true);
$hashSigner = StrKey::encodeSha256Hash($hash);
StrKey::isValidSha256Hash($hashSigner); // true
$decoded = StrKey::decodeSha256Hash($hashSigner);
```

## Contract IDs (C...)

Soroban contract IDs:

```php
<?php

use Soneso\StellarSDK\Crypto\StrKey;

$contractId = 'CDCGEWX4TENKBQFSG5ISXR5QNKECBCHSOHNLNPXZXOTXRELAJRAMVHTR';

// Validate
StrKey::isValidContractId($contractId); // true

// Decode to raw bytes or hex
$raw = StrKey::decodeContractId($contractId);
$hex = StrKey::decodeContractIdHex($contractId);

// Encode from raw bytes or hex
$encoded = StrKey::encodeContractId($raw);
$encodedFromHex = StrKey::encodeContractIdHex($hex);
```

## Signed Payloads (P...)

Signed payloads combine a public key with arbitrary payload data:

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Crypto\StrKey;
use Soneso\StellarSDK\SignedPayloadSigner;

$keyPair = KeyPair::random();
$payload = random_bytes(32); // 4-64 bytes

$signer = SignedPayloadSigner::fromAccountId($keyPair->getAccountId(), $payload);
$signedPayload = StrKey::encodeSignedPayload($signer); // P...

$decoded = StrKey::decodeSignedPayload($signedPayload);
echo $decoded->getSignerAccountId()->getAccountId() . PHP_EOL;
```

## Liquidity Pool and Claimable Balance IDs

Pool IDs (L...) and claimable balance IDs (B...) both support hex encoding:

```php
<?php

use Soneso\StellarSDK\Crypto\StrKey;

// Liquidity pool ID (L...)
$poolHex = 'dd7b1ab831c273310ddbec6f97870aa83c2fbd78ce22aded37ecbf4f3380fac7';
$poolId = StrKey::encodeLiquidityPoolIdHex($poolHex);
StrKey::isValidLiquidityPoolId($poolId); // true
$decoded = StrKey::decodeLiquidityPoolIdHex($poolId);

// Claimable balance ID (B...)
$balanceHex = '00000000929b20b72e5890ab51c24f1cc46fa01c4f318d8d33367d24dd614cfd';
$balanceId = StrKey::encodeClaimableBalanceIdHex($balanceHex);
StrKey::isValidClaimableBalanceId($balanceId); // true
$decoded = StrKey::decodeClaimableBalanceIdHex($balanceId);
```

## Version Bytes Reference

| Prefix | Type | Description |
|--------|------|-------------|
| G | Account ID | Ed25519 public key |
| S | Secret Seed | Ed25519 private key |
| M | Muxed Account | Account ID + 64-bit ID |
| T | Pre-Auth TX | Pre-authorized transaction hash |
| X | SHA-256 Hash | Hash signer |
| P | Signed Payload | Public key + payload |
| C | Contract ID | Soroban smart contract |
| L | Liquidity Pool ID | AMM liquidity pool |
| B | Claimable Balance | Claimable balance |

## Error Handling

Invalid addresses throw `InvalidArgumentException`:

```php
<?php

use InvalidArgumentException;
use Soneso\StellarSDK\Crypto\StrKey;

// Invalid checksum or wrong version byte throws
try {
    StrKey::decodeAccountId('GINVALIDADDRESS...');
} catch (InvalidArgumentException $e) {
    echo "Invalid: " . $e->getMessage() . PHP_EOL;
}

// Use validation to avoid exceptions
$input = 'user-provided-address';
if (StrKey::isValidAccountId($input)) {
    $raw = StrKey::decodeAccountId($input);
}
```

## Related SEPs

- [SEP-05 Key Derivation](sep-05.md) — Deriving keypairs from mnemonic phrases
- [SEP-10 Web Authentication](sep-10.md) — Uses account IDs for authentication challenges
