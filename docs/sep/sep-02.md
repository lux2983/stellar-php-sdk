# SEP-02: Federation Protocol

Federation allows users to send payments using human-readable addresses like `bob*example.com` instead of raw account IDs like `GCEZWKCA5VLDNRLN3RPRJMRZOX3Z6G5CHCGSNFHEYVXM3XOJMDS674JZ`. It also enables organizations to map bank accounts or other external identifiers to Stellar accounts.

**When to use:** Building a wallet that supports sending payments to Stellar addresses, or implementing a service that resolves external identifiers (bank accounts, phone numbers) to Stellar accounts.

See the [SEP-02 specification](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0002.md) for protocol details.

## Quick Example

```php
<?php

use Soneso\StellarSDK\SEP\Federation\Federation;

// Resolve a Stellar address to an account ID
$response = Federation::resolveStellarAddress('bob*soneso.com');

echo "Account: " . $response->getAccountId() . PHP_EOL;
echo "Memo: " . $response->getMemo() . PHP_EOL;
```

## Resolving Stellar Addresses

A Stellar address has the format `name*domain`. The SDK fetches the domain's stellar.toml to find the federation server, then queries it.

```php
<?php

use Soneso\StellarSDK\SEP\Federation\Federation;

$response = Federation::resolveStellarAddress('bob*soneso.com');

// The destination account for payments
$accountId = $response->getAccountId();
echo "Account ID: " . $accountId . PHP_EOL;
// GBVPKXWMAB3FIUJB6T7LF66DABKKA2ZHRHDOQZ25GBAEFZVHTBPJNOJI

// Include memo if provided (required for some destinations)
$memo = $response->getMemo();
$memoType = $response->getMemoType();

if ($memo !== null) {
    echo "Memo ({$memoType}): " . $memo . PHP_EOL;
}

// Original address for confirmation
$address = $response->getStellarAddress();
echo "Address: " . $address . PHP_EOL;
// bob*soneso.com
```

## Reverse Lookup (Account ID to Address)

Find the Stellar address associated with an account ID. You need to know which federation server to query.

```php
<?php

use Soneso\StellarSDK\SEP\Federation\Federation;

$accountId = 'GBVPKXWMAB3FIUJB6T7LF66DABKKA2ZHRHDOQZ25GBAEFZVHTBPJNOJI';
$federationServer = 'https://stellarid.io/federation';

$response = Federation::resolveStellarAccountId($accountId, $federationServer);

echo "Address: " . $response->getStellarAddress() . PHP_EOL;
// bob*soneso.com
```

## Transaction Lookup

Some federation servers can return information about who sent a transaction.

```php
<?php

use Soneso\StellarSDK\SEP\Federation\Federation;

$txId = 'c1b368c00e9852351361e07cc58c54277e7a6366580044ab152b8db9cd8ec52a';
$federationServer = 'https://stellarid.io/federation';

// Returns federation record of the sender if known
$response = Federation::resolveStellarTransactionId($txId, $federationServer);

if ($response->getStellarAddress() !== null) {
    echo "Sender: " . $response->getStellarAddress() . PHP_EOL;
}
```

## Forward Federation

Forward federation maps external identifiers (bank accounts, routing numbers, etc.) to Stellar accounts. This is used when you want to pay someone who doesn't have a Stellar address but has another type of account.

```php
<?php

use Soneso\StellarSDK\SEP\Federation\Federation;

// Pay to a bank account via an anchor
$params = [
    'forward_type' => 'bank_account',
    'swift' => 'BOPBPHMM',
    'acct' => '2382376'
];

$federationServer = 'https://stellarid.io/federation';
$response = Federation::resolveForward($params, $federationServer);

echo "Deposit to: " . $response->getAccountId() . PHP_EOL;

// Use the memo to identify the recipient
if ($response->getMemo() !== null) {
    echo "Memo ({$response->getMemoType()}): " . $response->getMemo() . PHP_EOL;
}
```

## Building a Payment with Federation

Here's how to send a payment using a Stellar address:

```php
<?php

use Exception;
use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Memo;
use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\PaymentOperationBuilder;
use Soneso\StellarSDK\SEP\Federation\Federation;
use Soneso\StellarSDK\StellarSDK;
use Soneso\StellarSDK\TransactionBuilder;

$sdk = StellarSDK::getTestNetInstance();

// Sender's keypair
$senderKeyPair = KeyPair::fromSeed('SCZANGBA5YHTNYVVV3C7CAZMTQDBJHJG6C34CPMLIHJPFV5RXN5M6CSS');
$senderAccountId = $senderKeyPair->getAccountId();

// Resolve recipient's Stellar address
$recipient = 'alice*testanchor.stellar.org';
$response = Federation::resolveStellarAddress($recipient);

$destinationId = $response->getAccountId();

// Load sender account
$senderAccount = $sdk->requestAccount($senderAccountId);

// Build payment operation
$paymentOp = (new PaymentOperationBuilder($destinationId, Asset::native(), '10'))
    ->build();

// Build transaction
$txBuilder = new TransactionBuilder($senderAccount);
$txBuilder->addOperation($paymentOp);

// Include memo if federation response requires it
if ($response->getMemo() !== null) {
    $memoType = $response->getMemoType();
    if ($memoType === 'text') {
        $txBuilder->addMemo(Memo::text($response->getMemo()));
    } elseif ($memoType === 'id') {
        $txBuilder->addMemo(Memo::id((int)$response->getMemo()));
    } elseif ($memoType === 'hash') {
        $txBuilder->addMemo(Memo::hash(base64_decode($response->getMemo())));
    }
}

$transaction = $txBuilder->build();
$transaction->sign($senderKeyPair, Network::testnet());

try {
    $sdk->submitTransaction($transaction);
    echo "Payment sent to {$recipient}" . PHP_EOL;
} catch (Exception $e) {
    echo "Payment failed: " . $e->getMessage() . PHP_EOL;
}
```

## Error Handling

```php
<?php

use Exception;
use InvalidArgumentException;
use Soneso\StellarSDK\Exceptions\HorizonRequestException;
use Soneso\StellarSDK\SEP\Federation\Federation;

// Invalid address format
try {
    Federation::resolveStellarAddress('invalid-no-asterisk');
} catch (InvalidArgumentException $e) {
    echo "Invalid format: " . $e->getMessage() . PHP_EOL;
}

// Domain without federation server
try {
    Federation::resolveStellarAddress('user*domain-without-federation.com');
} catch (Exception $e) {
    echo "No federation server: " . $e->getMessage() . PHP_EOL;
}

// User not found
try {
    $response = Federation::resolveStellarAddress('nonexistent*soneso.com');
    if ($response->getAccountId() === null) {
        echo "User not found" . PHP_EOL;
    }
} catch (HorizonRequestException $e) {
    echo "Federation error: " . $e->getMessage() . PHP_EOL;
}
```

## Finding the Federation Server

If you need to query a federation server directly without using Stellar addresses:

```php
<?php

use Soneso\StellarSDK\SEP\Toml\StellarToml;

// Get federation server URL from stellar.toml
$stellarToml = StellarToml::fromDomain('soneso.com');
$federationServer = $stellarToml->getGeneralInformation()->federationServer;

echo "Federation Server: " . $federationServer . PHP_EOL;
// https://stellarid.io/federation
```

## Related SEPs

- [SEP-01 stellar.toml](sep-01.md) - Where the `FEDERATION_SERVER` URL is published
- [SEP-10 Authentication](sep-10.md) - Some federation servers may require authentication
