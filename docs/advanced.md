# Advanced Topics

This guide covers multi-signature setups, error handling, fee optimization, debugging, and production deployment patterns.

**Prerequisites**: Familiarity with [SDK Usage](sdk-usage.md) and basic transaction building.

---

## Multi-Signature Accounts

Multi-signature (multisig) accounts require multiple parties to authorize transactions. This is useful for shared custody, escrow, and organizational accounts.

### Setting Up a 2-of-3 Multisig Account

A 2-of-3 setup requires any 2 of 3 signers to approve transactions:

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\SetOptionsOperationBuilder;
use Soneso\StellarSDK\Signer;
use Soneso\StellarSDK\StellarSDK;
use Soneso\StellarSDK\TransactionBuilder;

$sdk = StellarSDK::getTestNetInstance();

// The account that will become multisig
$mainKeyPair = KeyPair::fromSeed('SCZANGBA5YHTNYVVV3C7CAZMTQDBJHJG6C34JFD6XVEAEPTBED53FETV');
$mainAccount = $sdk->requestAccount($mainKeyPair->getAccountId());

// Two additional signers
$signer1 = KeyPair::random();
$signer2 = KeyPair::random();

// Build transaction: add signers and set thresholds
$transaction = (new TransactionBuilder($mainAccount))
    ->addOperation(
        (new SetOptionsOperationBuilder())
            ->setSigner(Signer::ed25519PublicKey($signer1), 1) // weight 1
            ->build()
    )
    ->addOperation(
        (new SetOptionsOperationBuilder())
            ->setSigner(Signer::ed25519PublicKey($signer2), 1) // weight 1
            ->build()
    )
    ->addOperation(
        (new SetOptionsOperationBuilder())
            ->setMasterKeyWeight(1)  // master key weight 1
            ->setLowThreshold(2)     // 2 signatures for low-security ops
            ->setMediumThreshold(2)  // 2 signatures for medium-security ops
            ->setHighThreshold(2)    // 2 signatures for high-security ops
            ->build()
    )
    ->build();

// Only master key signs the setup transaction
$transaction->sign($mainKeyPair, Network::testnet());
$response = $sdk->submitTransaction($transaction);
```

### Signing a Multisig Transaction

After setup, transactions need signatures from multiple parties:

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\PaymentOperationBuilder;
use Soneso\StellarSDK\StellarSDK;
use Soneso\StellarSDK\TransactionBuilder;
use Soneso\StellarSDK\AbstractTransaction;

$sdk = StellarSDK::getTestNetInstance();

// Load the multisig account
$mainKeyPair = KeyPair::fromSeed('SCZANGBA5YHTNYVVV3C7CAZMTQDBJHJG6C34JFD6XVEAEPTBED53FETV');
$signer1 = KeyPair::fromSeed('SBBBBBBB...');  // First additional signer
$account = $sdk->requestAccount($mainKeyPair->getAccountId());

// Build a payment transaction
$payment = (new PaymentOperationBuilder(
    'GDESTINATION...',
    \Soneso\StellarSDK\Asset::native(),
    '100'
))->build();

$transaction = (new TransactionBuilder($account))
    ->addOperation($payment)
    ->build();

// First signer signs and serializes for transport
$transaction->sign($mainKeyPair, Network::testnet());
$xdrBase64 = $transaction->toEnvelopeXdrBase64();

// --- Transport XDR to second signer (email, API, etc.) ---

// Second signer deserializes, signs, and submits
$transaction = AbstractTransaction::fromEnvelopeBase64XdrString($xdrBase64);
$transaction->sign($signer1, Network::testnet());

$response = $sdk->submitTransaction($transaction);
```

### Pre-Authorized Transactions

Authorize a specific transaction in advance. The pre-auth signer is consumed when the transaction submits:

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\PaymentOperationBuilder;
use Soneso\StellarSDK\SetOptionsOperationBuilder;
use Soneso\StellarSDK\Signer;
use Soneso\StellarSDK\StellarSDK;
use Soneso\StellarSDK\TransactionBuilder;

$sdk = StellarSDK::getTestNetInstance();
$sourceKeyPair = KeyPair::fromSeed('SCZANGBA...');
$sourceAccount = $sdk->requestAccount($sourceKeyPair->getAccountId());

// Build transaction to pre-authorize (with NEXT sequence number)
$sourceAccount->incrementSequenceNumber();
$preAuthPayment = (new PaymentOperationBuilder('GDEST...', \Soneso\StellarSDK\Asset::native(), '50'))->build();
$preAuthTx = (new TransactionBuilder($sourceAccount))->addOperation($preAuthPayment)->build();

// Add transaction hash as signer
$preAuthSignerKey = Signer::preAuthTx($preAuthTx, Network::testnet());
$sourceAccount = $sdk->requestAccount($sourceKeyPair->getAccountId()); // Reload

$setupTx = (new TransactionBuilder($sourceAccount))
    ->addOperation((new SetOptionsOperationBuilder())->setSigner($preAuthSignerKey, 1)->build())
    ->build();
$setupTx->sign($sourceKeyPair, Network::testnet());
$sdk->submitTransaction($setupTx);

// Later: submit pre-authorized transaction (no signature needed)
$sdk->submitTransaction($preAuthTx);
```

---

## Error Handling

### Catching Horizon Errors

All SDK requests can throw `HorizonRequestException`. Handle these to build reliable applications:

```php
<?php

use Soneso\StellarSDK\Exceptions\HorizonRequestException;
use Soneso\StellarSDK\StellarSDK;

$sdk = StellarSDK::getTestNetInstance();

try {
    $account = $sdk->requestAccount('GINVALIDACCOUNT');
} catch (HorizonRequestException $e) {
    $statusCode = $e->getStatusCode();
    
    if ($statusCode === 404) {
        echo "Account not found\n";
    } elseif ($statusCode === 429) {
        // Rate limited - wait before retrying
        $retryAfter = $e->getRetryAfter();
        echo "Rate limited. Retry after {$retryAfter} seconds\n";
    } else {
        // Get detailed error info
        $errorResponse = $e->getHorizonErrorResponse();
        if ($errorResponse !== null) {
            echo "Error: " . $errorResponse->getDetail() . "\n";
        }
    }
}
```

### Transaction Failure Analysis

When transactions fail, examine the result codes to understand why:

```php
<?php

use Soneso\StellarSDK\Exceptions\HorizonRequestException;
use Soneso\StellarSDK\StellarSDK;
use Soneso\StellarSDK\AbstractTransaction;

$sdk = StellarSDK::getTestNetInstance();

try {
    $response = $sdk->submitTransaction($transaction);
    
    if ($response->isSuccessful()) {
        echo "Transaction succeeded: " . $response->getHash() . "\n";
    }
} catch (HorizonRequestException $e) {
    $errorResponse = $e->getHorizonErrorResponse();
    
    if ($errorResponse !== null && $errorResponse->getExtras() !== null) {
        $extras = $errorResponse->getExtras();
        $resultCodes = $extras->getResultCodes();
        
        // Transaction-level error
        $txCode = $resultCodes->getTransactionResultCode();
        echo "Transaction failed: {$txCode}\n";
        
        // Operation-level errors (if tx_failed)
        foreach ($resultCodes->getOperationsResultCodes() as $i => $opCode) {
            if ($opCode !== 'op_success') {
                echo "Operation {$i} failed: {$opCode}\n";
            }
        }
        
        // Raw XDR for detailed debugging
        echo "Result XDR: " . $extras->getResultXdr() . "\n";
    }
}
```

### Common Result Codes

**Transaction codes:**
- `tx_success` - All operations succeeded
- `tx_failed` - One operation failed (check operation codes)
- `tx_bad_seq` - Wrong sequence number
- `tx_bad_auth` - Missing or invalid signatures
- `tx_insufficient_balance` - Can't pay fee
- `tx_insufficient_fee` - Fee too low
- `tx_too_early` / `tx_too_late` - Outside time bounds

**Operation codes:**
- `op_underfunded` - Source doesn't have enough balance
- `op_no_trust` - Destination doesn't trust the asset
- `op_low_reserve` - Would drop below minimum reserve
- `op_line_full` - Trustline balance limit reached

---

## Fee Strategies

### Reading Network Fee Stats

Check current network conditions before setting fees:

```php
<?php

use Soneso\StellarSDK\StellarSDK;

$sdk = StellarSDK::getTestNetInstance();
$feeStats = $sdk->feeStats()->execute();

// Last ledger info
echo "Last ledger: " . $feeStats->getLastLedger() . "\n";
echo "Base fee: " . $feeStats->getLastLedgerBaseFee() . " stroops\n";
echo "Capacity used: " . $feeStats->getLedgerCapacityUsage() . "\n";

// Fee percentiles (what other transactions are paying)
$maxFee = $feeStats->getMaxFee();
echo "P50 (median): " . $maxFee->getP50() . " stroops\n";
echo "P90: " . $maxFee->getP90() . " stroops\n";
echo "P99: " . $maxFee->getP99() . " stroops\n";
```

### Dynamic Fee Selection

Set fees based on urgency and network conditions:

```php
<?php

use Soneso\StellarSDK\StellarSDK;
use Soneso\StellarSDK\TransactionBuilder;

$sdk = StellarSDK::getTestNetInstance();

// Get current fee stats
$feeStats = $sdk->feeStats()->execute();
$maxFee = $feeStats->getMaxFee();

// Choose fee based on priority
$urgency = 'normal'; // 'low', 'normal', 'high'
$baseFee = match($urgency) {
    'low' => (int)$maxFee->getP10(),      // Might wait a few ledgers
    'normal' => (int)$maxFee->getP50(),   // Usually included quickly
    'high' => (int)$maxFee->getP99(),     // Almost guaranteed next ledger
};

// Minimum is 100 stroops
$baseFee = max(100, $baseFee);

$account = $sdk->requestAccount('GSOURCE...');
$transaction = (new TransactionBuilder($account))
    ->setMaxOperationFee($baseFee)
    ->addOperation($operation)
    ->build();
```

### Fee Bump Transactions

Bump the fee on a stuck transaction without rebuilding it:

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\FeeBumpTransactionBuilder;
use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\StellarSDK;
use Soneso\StellarSDK\AbstractTransaction;

$sdk = StellarSDK::getTestNetInstance();

// Original transaction that's stuck (already signed)
$originalTxXdr = 'AAAA...'; // base64 XDR from earlier
$innerTx = AbstractTransaction::fromEnvelopeBase64XdrString($originalTxXdr);

// Fee payer (can be different from original source)
$feePayerKeyPair = KeyPair::fromSeed('SFEE...');

// Create fee bump with higher fee
// New base fee must be higher than inner transaction's fee
$feeBump = (new FeeBumpTransactionBuilder($innerTx))
    ->setFeeAccount($feePayerKeyPair->getAccountId())
    ->setBaseFee(500) // 500 stroops per operation
    ->build();

// Only the fee payer signs the bump
$feeBump->sign($feePayerKeyPair, Network::testnet());

$response = $sdk->submitTransaction($feeBump);
```

---

## Debugging Transactions

### Inspecting Transaction Contents

Parse any transaction from XDR to examine its structure:

```php
<?php

use Soneso\StellarSDK\AbstractTransaction;
use Soneso\StellarSDK\Transaction;
use Soneso\StellarSDK\FeeBumpTransaction;

$xdrBase64 = 'AAAA...'; // Transaction envelope XDR
$tx = AbstractTransaction::fromEnvelopeBase64XdrString($xdrBase64);

if ($tx instanceof Transaction) {
    echo "Source: " . $tx->getSourceAccount()->getAccountId() . "\n";
    echo "Sequence: " . $tx->getSequenceNumber() . "\n";
    echo "Fee: " . $tx->getFee() . " stroops\n";
    echo "Operations: " . count($tx->getOperations()) . "\n";
    echo "Signatures: " . count($tx->getSignatures()) . "\n";
    
    // Check time bounds
    $timeBounds = $tx->getTimeBounds();
    if ($timeBounds !== null) {
        echo "Valid from: " . $timeBounds->getMinTime()->format('c') . "\n";
        echo "Valid until: " . $timeBounds->getMaxTime()->format('c') . "\n";
    }
    
    // Examine each operation
    foreach ($tx->getOperations() as $i => $op) {
        echo "Op {$i}: " . get_class($op) . "\n";
    }
}

if ($tx instanceof FeeBumpTransaction) {
    echo "Fee bump from: " . $tx->getFeeAccount()->getAccountId() . "\n";
    echo "Inner tx hash: " . bin2hex($tx->getInnerTx()->hash(\Soneso\StellarSDK\Network::testnet())) . "\n";
}
```

### Human-Readable Transaction Format (SEP-11)

Convert transactions to TxRep format for easier debugging:

```php
<?php

use Soneso\StellarSDK\SEP\TxRep\TxRep;

$xdrBase64 = 'AAAA...'; // Transaction envelope

// Convert to human-readable format
$txRep = TxRep::fromTransactionEnvelopeXdrBase64($xdrBase64);
echo $txRep;

// Output shows all fields in readable format:
// type: ENVELOPE_TYPE_TX
// tx.sourceAccount: GABC...
// tx.fee: 100
// tx.seqNum: 123456789
// tx.operations[0].type: PAYMENT
// ...
```

### Verifying Transaction Hashes

Confirm a transaction hash matches what you expect:

```php
<?php

use Soneso\StellarSDK\AbstractTransaction;
use Soneso\StellarSDK\Network;

$xdrBase64 = 'AAAA...';
$tx = AbstractTransaction::fromEnvelopeBase64XdrString($xdrBase64);

// Hash depends on network
$testnetHash = bin2hex($tx->hash(Network::testnet()));
$mainnetHash = bin2hex($tx->hash(Network::public()));

echo "Testnet hash: {$testnetHash}\n";
echo "Mainnet hash: {$mainnetHash}\n";

// Compare with expected
$expectedHash = 'abc123...';
if ($testnetHash === $expectedHash) {
    echo "Hash verified!\n";
}
```

---

## Performance Tips

### Reusing the HTTP Client

Configure a custom Guzzle client for connection pooling and timeouts:

```php
<?php

use GuzzleHttp\Client;
use GuzzleHttp\HandlerStack;
use Soneso\StellarSDK\StellarSDK;

// Custom client with persistent connections
$httpClient = new Client([
    'base_uri' => 'https://horizon.stellar.org',
    'timeout' => 30,
    'connect_timeout' => 5,
    'http_errors' => false,
    // Guzzle reuses connections by default
]);

$sdk = StellarSDK::getPublicNetInstance();
$sdk->setHttpClient($httpClient);

// All subsequent requests use the same connection pool
$account1 = $sdk->requestAccount('GACCOUNT1...');
$account2 = $sdk->requestAccount('GACCOUNT2...');
```

### Caching Account Sequences

Cache sequence numbers to avoid repeated Horizon calls when building multiple transactions:

```php
<?php

use Soneso\StellarSDK\Account;
use Soneso\StellarSDK\StellarSDK;

$sdk = StellarSDK::getTestNetInstance();

// Fetch once, use for multiple transactions
$response = $sdk->requestAccount('GSOURCE...');
$account = new Account($response->getAccountId(), $response->getSequenceNumber());

// Build first transaction (sequence increments automatically)
$tx1 = (new \Soneso\StellarSDK\TransactionBuilder($account))->addOperation($op1)->build();

// Build second transaction (uses incremented sequence)
$tx2 = (new \Soneso\StellarSDK\TransactionBuilder($account))->addOperation($op2)->build();

// After submission, refresh if needed
$response = $sdk->requestAccount('GSOURCE...');
```

---

## Testnet vs Mainnet

### Network Configuration

```php
<?php

use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\StellarSDK;

// Testnet - for development
$testnetSdk = StellarSDK::getTestNetInstance();
$testnetNetwork = Network::testnet();

// Mainnet - for production
$mainnetSdk = StellarSDK::getPublicNetInstance();
$mainnetNetwork = Network::public();

// Custom network (private network or specific Horizon)
$customSdk = new StellarSDK('https://horizon.custom.example.com');
$customNetwork = new Network('Custom Network Passphrase');

// Sign for the correct network
$transaction->sign($keyPair, $testnetNetwork);  // testnet
$transaction->sign($keyPair, $mainnetNetwork);  // mainnet
```

### Funding Testnet Accounts

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Util\FriendBot;

$keyPair = KeyPair::random();
FriendBot::fundTestAccount($keyPair->getAccountId());
echo "Funded: " . $keyPair->getAccountId() . "\n";
```

### Test Cleanup

Merge test accounts back to recover XLM:

```php
<?php

use Soneso\StellarSDK\AccountMergeOperationBuilder;
use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\StellarSDK;
use Soneso\StellarSDK\TransactionBuilder;

$sdk = StellarSDK::getTestNetInstance();
$testKeyPair = KeyPair::fromSeed('STEST...');
$account = $sdk->requestAccount($testKeyPair->getAccountId());

// Merge back to permanent account (recovers all XLM)
$mergeOp = (new AccountMergeOperationBuilder('GPERMANENT...'))->build();
$tx = (new TransactionBuilder($account))->addOperation($mergeOp)->build();
$tx->sign($testKeyPair, Network::testnet());
$sdk->submitTransaction($tx);
```

---

## Related Documentation

- [SDK Usage Guide](sdk-usage.md) - Transaction building, operations, Horizon queries
- [Soroban Guide](soroban.md) - Smart contract interactions and Soroban RPC
- [SEP Protocols](sep/README.md) - Interoperability standards
