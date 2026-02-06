# SEP-07: URI Scheme for Delegated Signing

SEP-07 defines a URI scheme (`web+stellar:`) that enables applications to request transaction signing from external wallets. Instead of handling private keys directly, your application generates a URI that a wallet can open, sign, and submit.

**When to use:** Building web applications that need users to sign transactions, creating payment request links, or integrating with hardware wallets or other signing services.

See the [SEP-07 specification](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0007.md) for protocol details.

## Quick Example

Generate a simple payment request URI that can be opened by any SEP-07 compatible wallet:

```php
<?php

use Soneso\StellarSDK\SEP\URIScheme\URIScheme;

$uriScheme = new URIScheme();

// Generate a payment request URI
$uri = $uriScheme->generatePayOperationURI(
    destinationAccountId: 'GDGUF4SCNINRDCRUIVOMDYGIMXOWVP3ZLMTL2OGQIWMFDDSECZSFQMQV',
    amount: '100',
    assetCode: 'USDC',
    assetIssuer: 'GA5ZSEJYB37JRC5AVCIA5MOP4RHTM335X2KGX3IHOJAPP5RE34K4KZVN'
);

echo $uri . PHP_EOL;
// web+stellar:pay?destination=GDGUF4SC...&amount=100&asset_code=USDC&asset_issuer=GA5ZSEJY...
```

## Generating URIs

### Transaction Signing (tx operation)

Request a wallet to sign a specific transaction. The `tx` operation is useful when you need precise control over the transaction details:

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
$uri = $uriScheme->generateSignTransactionURI(
    transactionEnvelopeXdrBase64: $transaction->toEnvelopeXdrBase64()
);

echo $uri . PHP_EOL;
// web+stellar:tx?xdr=AAAAAgAAAAD...
```

### Transaction URI with Options

Create a more detailed transaction URI with callback URL, required signer, user message, and origin verification. These options give wallets more context about the request:

```php
<?php

use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\SEP\URIScheme\URIScheme;

$uriScheme = new URIScheme();
$xdrBase64 = 'AAAAAgAAAAD...'; // Your transaction XDR

$uri = $uriScheme->generateSignTransactionURI(
    transactionEnvelopeXdrBase64: $xdrBase64,
    callback: 'url:https://example.com/callback',  // Where to POST signed tx
    publicKey: 'GDGUF4SCNINRDCRUIVOMDYGIMXOWVP3ZLMTL2OGQIWMFDDSECZSFQMQV', // Required signer
    message: 'Please sign to update your settings',  // User-facing message (max 300 chars)
    networkPassphrase: Network::testnet()->getNetworkPassphrase(),
    originDomain: 'example.com'  // Your domain (requires signature)
);

echo $uri . PHP_EOL;
```

### Transaction Chaining

The `chain` parameter allows you to request multiple signatures in sequence. After signing the first transaction, the wallet processes the next URI. This is useful for multi-step operations:

```php
<?php

use Soneso\StellarSDK\SEP\URIScheme\URIScheme;

$uriScheme = new URIScheme();

// Second transaction URI (to be executed after the first)
$secondTxUri = 'web+stellar:tx?xdr=AAAAAgBBBBBB...';

// First transaction URI that chains to the second
$uri = $uriScheme->generateSignTransactionURI(
    transactionEnvelopeXdrBase64: 'AAAAAgAAAAD...',
    chain: $secondTxUri  // Next URI to process after signing
);

echo $uri . PHP_EOL;
// Note: Maximum 7 levels of chaining allowed per specification
```

### Transaction Field Replacement

The `replace` parameter allows wallets to substitute transaction fields before signing. This uses SEP-11 (Txrep) format to specify which fields can be replaced:

```php
<?php

use Soneso\StellarSDK\SEP\URIScheme\URIScheme;

$uriScheme = new URIScheme();

// Allow wallet to replace source account and sequence number
$replace = 'sourceAccount:TX_SOURCE_ACCOUNT,seqNum:TX_SEQ_NUM';

$uri = $uriScheme->generateSignTransactionURI(
    transactionEnvelopeXdrBase64: 'AAAAAgAAAAD...',
    replace: $replace  // SEP-11 field replacement specification
);

echo $uri . PHP_EOL;
```

### Payment Request (pay operation)

Request a payment without pre-building a transaction. The wallet decides how to fulfill the payment (direct payment or path payment), providing maximum flexibility:

```php
<?php

use Soneso\StellarSDK\SEP\URIScheme\URIScheme;

$uriScheme = new URIScheme();

// Simple XLM payment
$uri = $uriScheme->generatePayOperationURI(
    destinationAccountId: 'GDGUF4SCNINRDCRUIVOMDYGIMXOWVP3ZLMTL2OGQIWMFDDSECZSFQMQV',
    amount: '50.5'
);
echo $uri . PHP_EOL;
// web+stellar:pay?destination=GDGUF4SC...&amount=50.5
```

Request a payment with a specific asset and memo. Memos are useful for exchange deposits or identifying payments:

```php
<?php

use Soneso\StellarSDK\SEP\URIScheme\URIScheme;

$uriScheme = new URIScheme();

// Payment with specific asset and memo
$uri = $uriScheme->generatePayOperationURI(
    destinationAccountId: 'GDGUF4SCNINRDCRUIVOMDYGIMXOWVP3ZLMTL2OGQIWMFDDSECZSFQMQV',
    amount: '100',
    assetCode: 'USDC',
    assetIssuer: 'GA5ZSEJYB37JRC5AVCIA5MOP4RHTM335X2KGX3IHOJAPP5RE34K4KZVN',
    memo: 'order-12345',
    memoType: 'MEMO_TEXT'
);
echo $uri . PHP_EOL;
```

### Donation Request (no amount)

Let the user decide how much to send by omitting the amount parameter. This is ideal for donations or tips:

```php
<?php

use Soneso\StellarSDK\SEP\URIScheme\URIScheme;

$uriScheme = new URIScheme();

// Omit amount for donations
$uri = $uriScheme->generatePayOperationURI(
    destinationAccountId: 'GDGUF4SCNINRDCRUIVOMDYGIMXOWVP3ZLMTL2OGQIWMFDDSECZSFQMQV',
    message: 'Support our project!'
);

echo $uri . PHP_EOL;
```

## Signing URIs

If your application issues payment request URIs and wants to prove authenticity, sign them with a `URI_REQUEST_SIGNING_KEY`. The signing key must be published in your domain's `stellar.toml` file:

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

Before processing a URI from an untrusted source, validate it. The validation fetches the `stellar.toml` from the origin domain and verifies the cryptographic signature:

```php
<?php

use Soneso\StellarSDK\SEP\URIScheme\URIScheme;
use Soneso\StellarSDK\SEP\URIScheme\URISchemeError;

$uriScheme = new URIScheme();
$uri = 'web+stellar:tx?xdr=...&origin_domain=example.com&signature=...';

try {
    // Validates signature against stellar.toml URI_REQUEST_SIGNING_KEY
    $isValid = $uriScheme->checkUIRSchemeIsValid($uri);
    
    $originDomain = $uriScheme->getParameterValue(
        URIScheme::originDomainParameterName, 
        $uri
    );
    echo "URI is valid and signed by " . $originDomain . PHP_EOL;
} catch (URISchemeError $e) {
    echo $e->toString() . PHP_EOL;
}
```

Handle specific validation errors to provide better user feedback. Each error code indicates a specific validation failure:

```php
<?php

use Soneso\StellarSDK\SEP\URIScheme\URIScheme;
use Soneso\StellarSDK\SEP\URIScheme\URISchemeError;

$uriScheme = new URIScheme();
$uri = 'web+stellar:tx?xdr=...&origin_domain=example.com&signature=...';

try {
    $uriScheme->checkUIRSchemeIsValid($uri);
    echo "URI validation successful" . PHP_EOL;
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

Sign a transaction from a URI and submit it to the network or callback URL. The method automatically determines where to submit based on the URI's `callback` parameter:

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\SEP\URIScheme\URIScheme;
use Soneso\StellarSDK\SEP\URIScheme\SubmitUriSchemeTransactionResponse;

$uriScheme = new URIScheme();

// The URI containing the transaction
$uri = 'web+stellar:tx?xdr=AAAAAgAAAAD...';

// User's signing keypair
$signerKeyPair = KeyPair::fromSeed('SCZANGBA5YHTNYVVV3C7CAZMTQDBJHJG6C34CPMLIHJPFV5RXN5M6CSS');

// Sign and submit
$response = $uriScheme->signAndSubmitTransaction(
    url: $uri, 
    signerKeyPair: $signerKeyPair, 
    network: Network::testnet()
);

// Check result - response contains either network or callback response
if ($response->getSubmitTransactionResponse() !== null) {
    // Submitted directly to Stellar network
    $txResponse = $response->getSubmitTransactionResponse();
    if ($txResponse->isSuccessful()) {
        echo "Transaction successful!" . PHP_EOL;
        echo "Hash: " . $txResponse->getHash() . PHP_EOL;
    } else {
        echo "Transaction failed" . PHP_EOL;
    }
} elseif ($response->getCallBackResponse() !== null) {
    // Sent to callback URL
    $httpResponse = $response->getCallBackResponse();
    echo "Callback response: " . $httpResponse->getStatusCode() . PHP_EOL;
}
```

## Extracting URI Parameters

Get specific values from a URI using the `getParameterValue` method. Use the class constants for parameter names to avoid typos:

```php
<?php

use Soneso\StellarSDK\SEP\URIScheme\URIScheme;

$uriScheme = new URIScheme();
$uri = 'web+stellar:pay?destination=GDGUF4SC...&amount=100&memo=order-123';

// Extract parameters using constant names
$destination = $uriScheme->getParameterValue(URIScheme::destinationParameterName, $uri);
$amount = $uriScheme->getParameterValue(URIScheme::amountParameterName, $uri);
$memo = $uriScheme->getParameterValue(URIScheme::memoParameterName, $uri);
$callback = $uriScheme->getParameterValue(URIScheme::callbackParameterName, $uri);

echo "Pay {$amount} to {$destination}" . PHP_EOL;
if ($memo !== null) {
    echo "Memo: {$memo}" . PHP_EOL;
}
if ($callback !== null) {
    echo "Callback URL: {$callback}" . PHP_EOL;
}
```

Available parameter constants:

| Constant | Parameter |
|----------|-----------|
| `URIScheme::xdrParameterName` | `xdr` |
| `URIScheme::replaceParameterName` | `replace` |
| `URIScheme::callbackParameterName` | `callback` |
| `URIScheme::publicKeyParameterName` | `pubkey` |
| `URIScheme::chainParameterName` | `chain` |
| `URIScheme::messageParameterName` | `msg` |
| `URIScheme::networkPassphraseParameterName` | `network_passphrase` |
| `URIScheme::originDomainParameterName` | `origin_domain` |
| `URIScheme::signatureParameterName` | `signature` |
| `URIScheme::destinationParameterName` | `destination` |
| `URIScheme::amountParameterName` | `amount` |
| `URIScheme::assetCodeParameterName` | `asset_code` |
| `URIScheme::assetIssuerParameterName` | `asset_issuer` |
| `URIScheme::memoParameterName` | `memo` |
| `URIScheme::memoTypeParameterName` | `memo_type` |

## Error Handling

Handle errors that may occur during URI processing. The SDK throws specific exceptions for different failure scenarios:

```php
<?php

use Exception;
use GuzzleHttp\Exception\GuzzleException;
use InvalidArgumentException;
use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Exceptions\HorizonRequestException;
use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\SEP\URIScheme\URIScheme;
use Soneso\StellarSDK\SEP\URIScheme\URISchemeError;

$uriScheme = new URIScheme();

// Handle invalid XDR in URI
try {
    $uri = 'web+stellar:tx?xdr=invalid-base64';
    $keyPair = KeyPair::fromSeed('SCZANGBA5YHTNYVVV3C7CAZMTQDBJHJG6C34CPMLIHJPFV5RXN5M6CSS');
    $uriScheme->signAndSubmitTransaction($uri, $keyPair, Network::testnet());
} catch (InvalidArgumentException $e) {
    echo "Invalid URI: " . $e->getMessage() . PHP_EOL;
}
```

Handle validation errors with detailed error codes:

```php
<?php

use Soneso\StellarSDK\SEP\URIScheme\URIScheme;
use Soneso\StellarSDK\SEP\URIScheme\URISchemeError;

$uriScheme = new URIScheme();

try {
    $uri = 'web+stellar:tx?xdr=...&origin_domain=example.com';
    $uriScheme->checkUIRSchemeIsValid($uri);
} catch (URISchemeError $e) {
    // Use toString() for human-readable error message
    echo $e->toString() . PHP_EOL;
    // Or use getCode() for programmatic handling
    echo "Error code: " . $e->getCode() . PHP_EOL;
}
```

Handle network and callback submission errors:

```php
<?php

use Exception;
use GuzzleHttp\Exception\GuzzleException;
use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Exceptions\HorizonRequestException;
use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\SEP\URIScheme\URIScheme;

$uriScheme = new URIScheme();
$uri = 'web+stellar:tx?xdr=...';
$keyPair = KeyPair::fromSeed('SCZANGBA5YHTNYVVV3C7CAZMTQDBJHJG6C34CPMLIHJPFV5RXN5M6CSS');

try {
    $response = $uriScheme->signAndSubmitTransaction($uri, $keyPair, Network::testnet());
    
    if ($response->getSubmitTransactionResponse() !== null) {
        $txResponse = $response->getSubmitTransactionResponse();
        if (!$txResponse->isSuccessful()) {
            // Transaction was submitted but failed on-chain
            $resultCodes = $txResponse->getExtras()->getResultCodesTransaction();
            echo "Transaction failed: " . $resultCodes . PHP_EOL;
        }
    }
} catch (HorizonRequestException $e) {
    // Network error communicating with Horizon
    echo "Horizon error: " . $e->getMessage() . PHP_EOL;
} catch (GuzzleException $e) {
    // HTTP error when submitting to callback URL
    echo "Callback error: " . $e->getMessage() . PHP_EOL;
} catch (Exception $e) {
    echo "Unexpected error: " . $e->getMessage() . PHP_EOL;
}
```

## Testing with Mock HTTP Handler

For unit testing, inject a mock HTTP handler to simulate stellar.toml fetching and callback responses without making real network requests:

```php
<?php

use GuzzleHttp\Handler\MockHandler;
use GuzzleHttp\HandlerStack;
use GuzzleHttp\Psr7\Response;
use Soneso\StellarSDK\SEP\URIScheme\URIScheme;

// Create mock responses
$mock = new MockHandler([
    new Response(200, [], 'URI_REQUEST_SIGNING_KEY="GDGUF4SC..."'),
]);

$handlerStack = HandlerStack::create($mock);

// Inject mock handler for testing
$uriScheme = new URIScheme();
$uriScheme->setMockHandlerStack($handlerStack);

// Now validation will use mock responses instead of real HTTP requests
```

## Security Notes

- **Always validate signed URIs** before showing `origin_domain` to users or processing transactions
- **Get user consent** before signing any transaction from a URI - display transaction details clearly
- **Check transaction details** - parse and display what the user is signing before they confirm
- **Be careful with callbacks** - callback URLs receive your signed transaction; only trust callbacks from validated origins
- **Messages can be spoofed** - only trust `message` content if the URI signature is valid
- **Verify the network** - check `network_passphrase` matches your expected network (testnet vs mainnet)

## Related SEPs

- [SEP-01 stellar.toml](sep-01.md) - Where `URI_REQUEST_SIGNING_KEY` is published for signature verification
- [SEP-11 Txrep](sep-11.md) - Human-readable transaction format used in the `replace` parameter
