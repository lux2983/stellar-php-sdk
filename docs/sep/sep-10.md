# SEP-10: Stellar Web Authentication

SEP-10 defines how wallets prove account ownership to anchors and other services. When a service needs to verify you control a Stellar account, SEP-10 handles the challenge-response flow and returns a JWT token for authenticated requests.

**Use SEP-10 when:**
- Authenticating with anchors before deposits/withdrawals (SEP-6, SEP-24)
- Submitting KYC information (SEP-12)
- Accessing any service that requires proof of account ownership

**Spec:** [SEP-0010](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0010.md)

## Quick Example

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\SEP\WebAuth\WebAuth;

// Create WebAuth from the anchor's domain
$webAuth = WebAuth::fromDomain("testanchor.stellar.org", Network::testnet());

// Get JWT token - handles challenge request, signing, and submission
$userKeyPair = KeyPair::fromSeed("SCZANGBA5YHTNYVVV3C7CAZMTQDBJHJG6C34CJDQ66EQ7DZTPBRJFN4A");
$jwtToken = $webAuth->jwtToken($userKeyPair->getAccountId(), [$userKeyPair]);

// Use the token for authenticated requests
echo "Authenticated! Token: " . substr($jwtToken, 0, 50) . "...";
```

## Detailed Usage

### Creating WebAuth

**From domain (recommended):**

```php
<?php

use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\SEP\WebAuth\WebAuth;

// Loads stellar.toml and extracts WEB_AUTH_ENDPOINT and SIGNING_KEY
$webAuth = WebAuth::fromDomain("testanchor.stellar.org", Network::testnet());
```

**Manual construction:**

```php
<?php

use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\SEP\WebAuth\WebAuth;

$webAuth = new WebAuth(
    authEndpoint: "https://testanchor.stellar.org/auth",
    serverSigningKey: "GCUZ6YLL...",
    serverHomeDomain: "testanchor.stellar.org",
    network: Network::testnet()
);
```

### Standard Authentication

For most cases, `jwtToken()` handles the entire flow:

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\SEP\WebAuth\WebAuth;

$webAuth = WebAuth::fromDomain("testanchor.stellar.org", Network::testnet());
$userKeyPair = KeyPair::fromSeed("SCZANGBA5YHTNYVVV3C7CAZMTQDBJHJG6C34CJDQ66EQ7DZTPBRJFN4A");

$jwtToken = $webAuth->jwtToken(
    clientAccountId: $userKeyPair->getAccountId(),
    signers: [$userKeyPair]
);
```

The method:
1. Requests a challenge transaction from the server
2. Validates the challenge (sequence number, signatures, time bounds)
3. Signs with your keypair(s)
4. Submits back to the server
5. Returns the JWT token

### Multi-Signature Accounts

For accounts requiring multiple signatures:

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\SEP\WebAuth\WebAuth;

$webAuth = WebAuth::fromDomain("testanchor.stellar.org", Network::testnet());

// All signers needed to meet the account's threshold
$signer1 = KeyPair::fromSeed("SCZANGBA5YHTNYVVV3C7CAZMTQDBJHJG...");
$signer2 = KeyPair::fromSeed("SBGWSG6BTNCKCOB3DIFBGCVMUPQFYPA...");

$jwtToken = $webAuth->jwtToken(
    clientAccountId: $signer1->getAccountId(),
    signers: [$signer1, $signer2]
);
```

### Muxed Accounts

Muxed accounts (M... addresses) work directly:

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\MuxedAccount;
use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\SEP\WebAuth\WebAuth;

$webAuth = WebAuth::fromDomain("testanchor.stellar.org", Network::testnet());
$userKeyPair = KeyPair::fromSeed("SCZANGBA5YHTNYVVV3C7CAZMTQDBJHJG...");

// Create muxed account with user ID
$muxedAccount = new MuxedAccount($userKeyPair->getAccountId(), 1234567890);

$jwtToken = $webAuth->jwtToken(
    clientAccountId: $muxedAccount->getAccountId(), // M... address
    signers: [$userKeyPair]
);
```

For G... accounts with memo-based user separation:

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\SEP\WebAuth\WebAuth;

$webAuth = WebAuth::fromDomain("testanchor.stellar.org", Network::testnet());
$userKeyPair = KeyPair::fromSeed("SCZANGBA5YHTNYVVV3C7CAZMTQDBJHJG...");

$jwtToken = $webAuth->jwtToken(
    clientAccountId: $userKeyPair->getAccountId(),
    signers: [$userKeyPair],
    memo: 1234567890  // User ID memo
);
```

### Client Attribution (Non-Custodial Wallets)

Wallets can prove their identity to anchors using client domain verification:

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\SEP\WebAuth\WebAuth;

$webAuth = WebAuth::fromDomain("testanchor.stellar.org", Network::testnet());

$userKeyPair = KeyPair::fromSeed("SCZANGBA5YHTNYVVV3C7CAZMTQDBJHJG...");
$clientDomainKeyPair = KeyPair::fromSeed("SBGWSG6BTNCKCOB3DIFBGCVMUPQFYPA...");

$jwtToken = $webAuth->jwtToken(
    clientAccountId: $userKeyPair->getAccountId(),
    signers: [$userKeyPair],
    clientDomain: "mywallet.com",
    clientDomainKeyPair: $clientDomainKeyPair
);
```

The wallet's stellar.toml must have the `SIGNING_KEY` that matches `clientDomainKeyPair`.

**Remote signing callback** (when the client domain key isn't available locally):

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\SEP\WebAuth\WebAuth;

$webAuth = WebAuth::fromDomain("testanchor.stellar.org", Network::testnet());
$userKeyPair = KeyPair::fromSeed("SCZANGBA5YHTNYVVV3C7CAZMTQDBJHJG...");

$jwtToken = $webAuth->jwtToken(
    clientAccountId: $userKeyPair->getAccountId(),
    signers: [$userKeyPair],
    clientDomain: "mywallet.com",
    clientDomainSigningCallback: function(string $transactionXdr): string {
        // Send to your signing server and return signed XDR
        return $yourSigningService->sign($transactionXdr);
    }
);
```

### Multiple Home Domains

When an anchor serves multiple domains:

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\SEP\WebAuth\WebAuth;

$webAuth = WebAuth::fromDomain("testanchor.stellar.org", Network::testnet());
$userKeyPair = KeyPair::fromSeed("SCZANGBA5YHTNYVVV3C7CAZMTQDBJHJG...");

$jwtToken = $webAuth->jwtToken(
    clientAccountId: $userKeyPair->getAccountId(),
    signers: [$userKeyPair],
    homeDomain: "other-domain.com"  // Request challenge for specific domain
);
```

## Error Handling

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\SEP\WebAuth\WebAuth;
use Soneso\StellarSDK\SEP\WebAuth\ChallengeRequestErrorResponse;
use Soneso\StellarSDK\SEP\WebAuth\ChallengeValidationError;
use Soneso\StellarSDK\SEP\WebAuth\ChallengeValidationErrorInvalidSeqNr;
use Soneso\StellarSDK\SEP\WebAuth\ChallengeValidationErrorInvalidSignature;
use Soneso\StellarSDK\SEP\WebAuth\ChallengeValidationErrorInvalidTimeBounds;
use Soneso\StellarSDK\SEP\WebAuth\SubmitCompletedChallengeErrorResponseException;

try {
    $webAuth = WebAuth::fromDomain("testanchor.stellar.org", Network::testnet());
    $userKeyPair = KeyPair::fromSeed("SCZANGBA5YHTNYVVV3C7CAZMTQDBJHJG...");
    
    $jwtToken = $webAuth->jwtToken($userKeyPair->getAccountId(), [$userKeyPair]);
    
} catch (ChallengeRequestErrorResponse $e) {
    // Server rejected the challenge request
    echo "Challenge request failed: " . $e->getMessage();
    
} catch (ChallengeValidationErrorInvalidSeqNr $e) {
    // SECURITY: Challenge has non-zero sequence number
    // This could indicate a malicious server trying to get you to sign a real transaction
    echo "Security error: Invalid sequence number";
    
} catch (ChallengeValidationErrorInvalidSignature $e) {
    // Challenge wasn't properly signed by the server
    echo "Invalid server signature";
    
} catch (ChallengeValidationErrorInvalidTimeBounds $e) {
    // Challenge expired or time bounds invalid
    echo "Challenge expired or invalid time bounds";
    
} catch (ChallengeValidationError $e) {
    // Other validation errors (home domain, web auth domain, etc.)
    echo "Challenge validation failed: " . $e->getMessage();
    
} catch (SubmitCompletedChallengeErrorResponseException $e) {
    // Server rejected the signed challenge
    echo "Authentication failed: " . $e->getMessage();
}
```

### Common Errors

| Exception | Cause | Solution |
|-----------|-------|----------|
| `ChallengeValidationErrorInvalidSeqNr` | Sequence number ≠ 0 | **Security risk** - don't proceed |
| `ChallengeValidationErrorInvalidSignature` | Bad server signature | Check stellar.toml SIGNING_KEY |
| `ChallengeValidationErrorInvalidTimeBounds` | Challenge expired | Request a new challenge |
| `SubmitCompletedChallengeErrorResponseException` | Insufficient signers | Provide more signers |

## Security Notes

1. **Sequence number must be 0** - Non-zero means the transaction could execute on-chain. Never sign challenges with non-zero sequence numbers.

2. **Verify server signature** - The challenge must be signed by the server's key from stellar.toml.

3. **Check time bounds** - Expired challenges may be replay attacks.

4. **Store tokens securely** - JWT tokens grant access to protected services. Don't log them or expose in URLs.

5. **Use HTTPS** - All SEP-10 communication should use HTTPS.

## Related SEPs

- [SEP-01](sep-01.md) - stellar.toml discovery (provides auth endpoint)
- [SEP-06](sep-06.md) - Deposit/withdrawal (uses SEP-10 auth)
- [SEP-12](sep-12.md) - KYC API (uses SEP-10 auth)
- [SEP-24](sep-24.md) - Interactive deposit/withdrawal (uses SEP-10 auth)
