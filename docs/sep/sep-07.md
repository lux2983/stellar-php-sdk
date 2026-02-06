# SEP-07: URI Scheme for Delegated Signing

SEP-07 defines a URI scheme (`web+stellar:`) that enables applications to request transaction signing from external wallets. Instead of handling private keys directly, your application generates a URI that a wallet can open, sign, and submit.

**When to use:** Building web applications that need users to sign transactions, creating payment request links, or integrating with hardware wallets or other signing services.

See the [SEP-07 specification](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0007.md) for protocol details.

## Quick Example

```php
<?php

use Soneso\StellarSDK\SEP\URIScheme\URIScheme;

$uriScheme = new URIScheme();

// Generate a payment request URI
$uri = $uriScheme->generatePayOperationURI(
    'GDGUF4SCNINRDCRUIVOMDYGIMXOWVP3ZLMTL2OGQIWMFDDSECZSFQMQV',
    amount: '100',
    assetCode: 'USDC',
    assetIssuer: 'GA5ZSEJYB37JRC5AVCIA5MOP4RHTM335X2KGX3IHOJAPP5RE34K4KZVN'
);

echo $uri . PHP_EOL;
// web+stellar:pay?destination=GDGUF4SC...&amount=100&asset_code=USDC&asset_issuer=GA5ZSEJY...
```

## Generating URIs

### Transaction Signing (tx operation)

Request a wallet to sign a specific transaction:

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\SEP\URIScheme\URIScheme;
use Soneso\StellarSDK\SetOptionsOperationBuilder;
use Soneso\StellarSDK\StellarSDK;
use Soneso\StellarSDK\TransactionBuilder;

$sdk = StellarSDK::getTestNetInstance();

// Source account (the account that will sign)
$sourceKeyPair = KeyPair::fromSeed('SCZANGBA5YHTNYVVV3C7CAZMTQDBJHJG6C34CPMLIHJPFV5RXN5M6CSS');
$accountId = $sourceKeyPair->getAccountId();
$sourceAccount = $sdk->requestAccount($accountId);

// Build a transaction
$setOptionsOp = (new SetOptionsOperationBuilder())
    ->setSourceAccount($accountId)
    ->setHomeDomain('www.example.com')
    ->build();

$transaction = (new TransactionBuilder($sourceAccount))
    ->addOperation($setOptionsOp)
    ->build();

// Generate URI from unsigned transaction
$uriScheme = new URIScheme();
$uri = $uriScheme->generateSignTransactionURI($transaction->toEnvelopeXdrBase64());

echo $uri . PHP_EOL;
// web+stellar:tx?xdr=AAAAAgAAAAD...
```

### Transaction URI with Options

```php
<?php

use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\SEP\URIScheme\URIScheme;

$uriScheme = new URIScheme();
$xdrBase64 = 'AAAAAgAAAAD...'; // Your transaction XDR

$uri = $uriScheme->generateSignTransactionURI(
    $xdrBase64,
    callback: 'url:https://example.com/callback',  // Where to POST signed tx
    publicKey: 'GDGUF4SCNINRDCRUIVOMDYGIMXOWVP3ZLMTL2OGQIWMFDDSECZSFQMQV', // Required signer
    message: 'Please sign to update your settings',  // User-facing message
    networkPassphrase: Network::testnet()->getNetworkPassphrase(),
    originDomain: 'example.com'  // Your domain (requires signature)
);

echo $uri . PHP_EOL;
```

### Payment Request (pay operation)

Request a payment without pre-building a transaction. The wallet decides how to fulfill it:

```php
<?php

use Soneso\StellarSDK\SEP\URIScheme\URIScheme;

$uriScheme = new URIScheme();

// Simple XLM payment
$uri = $uriScheme->generatePayOperationURI(
    'GDGUF4SCNINRDCRUIVOMDYGIMXOWVP3ZLMTL2OGQIWMFDDSECZSFQMQV',
    amount: '50.5'
);
echo $uri . PHP_EOL;
// web+stellar:pay?destination=GDGUF4SC...&amount=50.5

// Payment with specific asset
$uri = $uriScheme->generatePayOperationURI(
    'GDGUF4SCNINRDCRUIVOMDYGIMXOWVP3ZLMTL2OGQIWMFDDSECZSFQMQV',
    amount: '100',
    assetCode: 'USDC',
    assetIssuer: 'GA5ZSEJYB37JRC5AVCIA5MOP4RHTM335X2KGX3IHOJAPP5RE34K4KZVN',
    memo: 'order-12345',
    memoType: 'MEMO_TEXT'
);
echo $uri . PHP_EOL;
```

### Donation Request (no amount)

Let the user decide how much to send:

```php
<?php

use Soneso\StellarSDK\SEP\URIScheme\URIScheme;

$uriScheme = new URIScheme();

// Omit amount for donations
$uri = $uriScheme->generatePayOperationURI(
    'GDGUF4SCNINRDCRUIVOMDYGIMXOWVP3ZLMTL2OGQIWMFDDSECZSFQMQV',
    message: 'Support our project!'
);

echo $uri . PHP_EOL;
```

## Signing URIs

If your application issues payment request URIs and wants to prove authenticity, sign them with a URI_REQUEST_SIGNING_KEY (which you publish in your stellar.toml):

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\SEP\URIScheme\URIScheme;

$uriScheme = new URIScheme();

// Your signing keypair (matches URI_REQUEST_SIGNING_KEY in stellar.toml)
$signerKeyPair = KeyPair::fromSeed('SCZANGBA5YHTNYVVV3C7CAZMTQDBJHJG6C34CPMLIHJPFV5RXN5M6CSS');

// URI must include origin_domain before signing
$uri = 'web+stellar:tx?xdr=AAAAAgAAAAD...&origin_domain=example.com';

// Sign the URI
$signedUri = $uriScheme->signURI($uri, $signerKeyPair);

echo $signedUri . PHP_EOL;
// web+stellar:tx?xdr=...&origin_domain=example.com&signature=bIZ53bPK...
```

## Validating URIs

Before processing a URI from an untrusted source, validate it:

```php
<?php

use Soneso\StellarSDK\SEP\URIScheme\URIScheme;
use Soneso\StellarSDK\SEP\URIScheme\URISchemeError;

$uriScheme = new URIScheme();
$uri = 'web+stellar:tx?xdr=...&origin_domain=example.com&signature=...';

try {
    // Validates signature against stellar.toml URI_REQUEST_SIGNING_KEY
    $isValid = $uriScheme->checkUIRSchemeIsValid($uri);
    echo "URI is valid and signed by " . $uriScheme->getParameterValue('origin_domain', $uri) . PHP_EOL;
} catch (URISchemeError $e) {
    switch ($e->getCode()) {
        case URISchemeError::missingOriginDomain:
            echo "URI has no origin_domain parameter" . PHP_EOL;
            break;
        case URISchemeError::invalidOriginDomain:
            echo "Invalid origin domain format" . PHP_EOL;
            break;
        case URISchemeError::missingSignature:
            echo "URI has no signature" . PHP_EOL;
            break;
        case URISchemeError::tomlNotFoundOrInvalid:
            echo "Could not fetch stellar.toml from origin domain" . PHP_EOL;
            break;
        case URISchemeError::tomlSignatureMissing:
            echo "stellar.toml has no URI_REQUEST_SIGNING_KEY" . PHP_EOL;
            break;
        case URISchemeError::invalidSignature:
            echo "Signature verification failed" . PHP_EOL;
            break;
    }
}
```

## Signing and Submitting Transactions

Sign a transaction from a URI and submit it:

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\SEP\URIScheme\URIScheme;

$uriScheme = new URIScheme();

// The URI containing the transaction
$uri = 'web+stellar:tx?xdr=AAAAAgAAAAD...';

// User's signing keypair
$signerKeyPair = KeyPair::fromSeed('SCZANGBA5YHTNYVVV3C7CAZMTQDBJHJG6C34CPMLIHJPFV5RXN5M6CSS');

// Sign and submit
$response = $uriScheme->signAndSubmitTransaction($uri, $signerKeyPair, Network::testnet());

// Check result
if ($response->submitTransactionResponse !== null) {
    // Submitted directly to Stellar network
    $txResponse = $response->submitTransactionResponse;
    if ($txResponse->isSuccessful()) {
        echo "Transaction successful!" . PHP_EOL;
        echo "Hash: " . $txResponse->getHash() . PHP_EOL;
    } else {
        echo "Transaction failed" . PHP_EOL;
    }
} elseif ($response->callBackResponse !== null) {
    // Sent to callback URL
    $httpResponse = $response->callBackResponse;
    echo "Callback response: " . $httpResponse->getStatusCode() . PHP_EOL;
}
```

## Extracting URI Parameters

Get specific values from a URI:

```php
<?php

use Soneso\StellarSDK\SEP\URIScheme\URIScheme;

$uriScheme = new URIScheme();
$uri = 'web+stellar:pay?destination=GDGUF4SC...&amount=100&memo=order-123';

$destination = $uriScheme->getParameterValue('destination', $uri);
$amount = $uriScheme->getParameterValue('amount', $uri);
$memo = $uriScheme->getParameterValue('memo', $uri);
$callback = $uriScheme->getParameterValue('callback', $uri); // null if not present

echo "Pay {$amount} to {$destination}" . PHP_EOL;
if ($memo !== null) {
    echo "Memo: {$memo}" . PHP_EOL;
}
```

## Error Handling

```php
<?php

use Exception;
use InvalidArgumentException;
use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\SEP\URIScheme\URIScheme;
use Soneso\StellarSDK\SEP\URIScheme\URISchemeError;

$uriScheme = new URIScheme();

// Invalid XDR in URI
try {
    $uri = 'web+stellar:tx?xdr=invalid-base64';
    $keyPair = KeyPair::fromSeed('SCZANGBA5YHTNYVVV3C7CAZMTQDBJHJG6C34CPMLIHJPFV5RXN5M6CSS');
    $uriScheme->signAndSubmitTransaction($uri, $keyPair, Network::testnet());
} catch (InvalidArgumentException $e) {
    echo "Invalid URI: " . $e->getMessage() . PHP_EOL;
}

// Validation errors
try {
    $uri = 'web+stellar:tx?xdr=...&origin_domain=example.com';
    $uriScheme->checkUIRSchemeIsValid($uri);
} catch (URISchemeError $e) {
    echo $e->toString() . PHP_EOL;
}

// Network errors during submission
try {
    $uri = 'web+stellar:tx?xdr=...';
    $keyPair = KeyPair::fromSeed('SCZANGBA5YHTNYVVV3C7CAZMTQDBJHJG6C34CPMLIHJPFV5RXN5M6CSS');
    $response = $uriScheme->signAndSubmitTransaction($uri, $keyPair, Network::testnet());
    
    if ($response->submitTransactionResponse !== null) {
        if (!$response->submitTransactionResponse->isSuccessful()) {
            echo "Transaction failed: " . $response->submitTransactionResponse->getExtras()->getResultCodesTransaction() . PHP_EOL;
        }
    }
} catch (Exception $e) {
    echo "Submission error: " . $e->getMessage() . PHP_EOL;
}
```

## Security Notes

- **Always validate signed URIs** before showing origin_domain to users
- **Get user consent** before signing any transaction from a URI
- **Check transaction details** - display what the user is signing
- **Be careful with callbacks** - they receive your signed transaction
- **Messages can be spoofed** - only trust message content if signature is valid

## Related SEPs

- [SEP-01 stellar.toml](sep-01.md) - Where `URI_REQUEST_SIGNING_KEY` is published
- [SEP-11 Txrep](sep-11.md) - Human-readable transaction format used in `replace` parameter
