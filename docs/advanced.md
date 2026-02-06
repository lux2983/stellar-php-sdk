# Advanced Topics

This guide covers multi-signature setups, error handling, fee optimization, debugging, and production deployment patterns.

**Prerequisites**: Familiarity with [SDK Usage](sdk-usage.md) and basic transaction building.

## Table of Contents

1. [Multi-Signature Accounts](#multi-signature-accounts)
2. [Error Handling](#error-handling)
3. [Fee Strategies](#fee-strategies)
4. [Debugging Transactions](#debugging-transactions)
5. [Performance Tips](#performance-tips)
6. [Security Best Practices](#security-best-practices)
7. [Testing Patterns](#testing-patterns)
8. [Advanced Network Operations](#advanced-network-operations)
9. [Testnet vs Mainnet](#testnet-vs-mainnet)

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

use Soneso\StellarSDK\Asset;
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
    Asset::native(),
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
use Soneso\StellarSDK\Asset;
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
$preAuthPayment = (new PaymentOperationBuilder('GDEST...', Asset::native(), '50'))->build();
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

Production applications need proper error handling for network issues, rate limits, and transaction failures. The SDK provides specific exception types and error details.

### Exception Types

The SDK throws specific exceptions for different error conditions:

```php
<?php

use Soneso\StellarSDK\Exceptions\HorizonRequestException;
use Soneso\StellarSDK\Exceptions\AccountNotFoundException;
use Soneso\StellarSDK\StellarSDK;

$sdk = StellarSDK::getTestNetInstance();

try {
    $account = $sdk->requestAccount('GINVALIDACCOUNT');
} catch (AccountNotFoundException $e) {
    // Account doesn't exist on network
    echo "Account not found: " . $e->getMessage() . "\n";
} catch (HorizonRequestException $e) {
    $statusCode = $e->getStatusCode();
    
    switch ($statusCode) {
        case 404:
            echo "Resource not found\n";
            break;
        case 429:
            // Rate limited - implement exponential backoff
            $retryAfter = $e->getRetryAfter() ?: 1;
            echo "Rate limited. Retry after {$retryAfter} seconds\n";
            break;
        case 500:
        case 502:
        case 503:
        case 504:
            // Server errors - retry with backoff
            echo "Server error: {$statusCode}. Retrying...\n";
            break;
        default:
            // Get detailed error info
            $errorResponse = $e->getHorizonErrorResponse();
            if ($errorResponse !== null) {
                echo "Error: " . $errorResponse->getDetail() . "\n";
            }
    }
}
```

### Retry Patterns with Exponential Backoff

Implement retry logic for transient failures:

```php
<?php

use Soneso\StellarSDK\Exceptions\HorizonRequestException;
use Soneso\StellarSDK\StellarSDK;

class RetryableOperation
{
    private StellarSDK $sdk;
    private int $maxRetries;
    private int $baseDelay;
    
    public function __construct(StellarSDK $sdk, int $maxRetries = 3, int $baseDelay = 1000)
    {
        $this->sdk = $sdk;
        $this->maxRetries = $maxRetries;
        $this->baseDelay = $baseDelay; // milliseconds
    }
    
    public function executeWithRetry(callable $operation, array $retryableStatusCodes = [429, 500, 502, 503, 504])
    {
        $lastException = null;
        
        for ($attempt = 1; $attempt <= $this->maxRetries; $attempt++) {
            try {
                return $operation();
                
            } catch (HorizonRequestException $e) {
                $lastException = $e;
                $statusCode = $e->getStatusCode();
                
                // Don't retry client errors (4xx except 429)
                if ($statusCode >= 400 && $statusCode < 500 && $statusCode !== 429) {
                    throw $e;
                }
                
                if (!in_array($statusCode, $retryableStatusCodes)) {
                    throw $e;
                }
                
                if ($attempt === $this->maxRetries) {
                    break; // Last attempt, don't delay
                }
                
                $delay = $this->calculateDelay($attempt, $e->getRetryAfter());
                error_log("Attempt {$attempt} failed (HTTP {$statusCode}). Retrying in {$delay}ms...");
                usleep($delay * 1000); // Convert to microseconds
                
            } catch (\Exception $e) {
                // Network errors, etc. - retry
                $lastException = $e;
                
                if ($attempt === $this->maxRetries) {
                    break;
                }
                
                $delay = $this->calculateDelay($attempt);
                error_log("Attempt {$attempt} failed: " . $e->getMessage() . ". Retrying in {$delay}ms...");
                usleep($delay * 1000);
            }
        }
        
        throw new \RuntimeException(
            "Operation failed after {$this->maxRetries} attempts. Last error: " . 
            ($lastException ? $lastException->getMessage() : 'Unknown error'),
            0,
            $lastException
        );
    }
    
    private function calculateDelay(int $attempt, ?int $retryAfterHeader = null): int
    {
        if ($retryAfterHeader) {
            return $retryAfterHeader * 1000; // Convert seconds to milliseconds
        }
        
        // Exponential backoff with jitter: base * (2^attempt) + random(0, 1000)
        return $this->baseDelay * (2 ** ($attempt - 1)) + random_int(0, 1000);
    }
    
    public function submitTransactionWithRetry($transaction)
    {
        return $this->executeWithRetry(function () use ($transaction) {
            return $this->sdk->submitTransaction($transaction);
        });
    }
    
    public function requestAccountWithRetry(string $accountId)
    {
        return $this->executeWithRetry(function () use ($accountId) {
            return $this->sdk->requestAccount($accountId);
        });
    }
}

// Usage
$retryable = new RetryableOperation($sdk, 3, 1000);

try {
    $account = $retryable->requestAccountWithRetry('GACCOUNT...');
    $response = $retryable->submitTransactionWithRetry($transaction);
} catch (\RuntimeException $e) {
    error_log("Operation failed permanently: " . $e->getMessage());
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
        
        // Transaction-level error
        $txCode = $extras->getResultCodesTransaction();
        echo "Transaction failed: {$txCode}\n";
        
        // Operation-level errors (if tx_failed)
        foreach ($extras->getResultCodesOperation() ?? [] as $i => $opCode) {
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
use Soneso\StellarSDK\FeeBumpTransaction;
use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\Transaction;

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
    echo "Inner tx hash: " . bin2hex($tx->getInnerTx()->hash(Network::testnet())) . "\n";
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
use Soneso\StellarSDK\TransactionBuilder;

$sdk = StellarSDK::getTestNetInstance();

// Fetch once, use for multiple transactions
$response = $sdk->requestAccount('GSOURCE...');
$account = new Account($response->getAccountId(), $response->getSequenceNumber());

// Build first transaction (sequence increments automatically)
$tx1 = (new TransactionBuilder($account))->addOperation($op1)->build();

// Build second transaction (uses incremented sequence)
$tx2 = (new TransactionBuilder($account))->addOperation($op2)->build();

// After submission, refresh if needed
$response = $sdk->requestAccount('GSOURCE...');
```

---

## Security Best Practices

Secure key management and transaction handling are important for production applications. Follow these patterns to protect funds and maintain security.

### Secure Key Management

Never hardcode private keys in your source code. Use environment variables and secure storage:

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;

class SecureKeyManager
{
    public static function getKeyPairFromEnv(string $envVar): KeyPair
    {
        $seed = getenv($envVar);
        if (!$seed) {
            throw new \InvalidArgumentException("Environment variable {$envVar} not found");
        }
        
        return KeyPair::fromSeed($seed);
    }

    public static function clearSensitiveData(string &$secretSeed): void
    {
        // Clear sensitive data from memory
        $secretSeed = str_repeat("\0", strlen($secretSeed));
        unset($secretSeed);
    }
}

// Production usage
$sourceKeyPair = SecureKeyManager::getKeyPairFromEnv('STELLAR_SECRET_KEY');
// Use keypair...

// Clear from memory when done (defensive programming)
$seed = getenv('STELLAR_SECRET_KEY');
SecureKeyManager::clearSensitiveData($seed);
```

### Multi-Signature Coordination Patterns

Coordinate multi-signature transactions safely across multiple parties:

```php
<?php

use Soneso\StellarSDK\AbstractTransaction;
use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\StellarSDK;

class MultiSigCoordinator
{
    private StellarSDK $sdk;
    private Network $network;
    
    public function __construct(StellarSDK $sdk, Network $network)
    {
        $this->sdk = $sdk;
        $this->network = $network;
    }
    
    public function createSigningRequest(string $transactionXdr, array $requiredSigners): array
    {
        return [
            'transaction_xdr' => $transactionXdr,
            'network' => $this->network->getNetworkPassphrase(),
            'required_signers' => $requiredSigners,
            'expires_at' => time() + 3600, // 1 hour expiry
            'created_at' => time()
        ];
    }
    
    public function validateAndSign(array $signingRequest, KeyPair $signer): string
    {
        // Validate request hasn't expired
        if (time() > $signingRequest['expires_at']) {
            throw new \RuntimeException('Signing request has expired');
        }
        
        // Validate network matches
        if ($signingRequest['network'] !== $this->network->getNetworkPassphrase()) {
            throw new \RuntimeException('Network mismatch');
        }
        
        // Load and sign transaction
        $transaction = AbstractTransaction::fromEnvelopeBase64XdrString(
            $signingRequest['transaction_xdr']
        );
        $transaction->sign($signer, $this->network);
        
        return $transaction->toEnvelopeXdrBase64();
    }
}
```

### Authorization Best Practices

Implement proper authorization checks for account operations:

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\StellarSDK;

class AuthorizationValidator
{
    private StellarSDK $sdk;
    
    public function __construct(StellarSDK $sdk)
    {
        $this->sdk = $sdk;
    }
    
    public function validateSignerAccess(string $accountId, string $signerPublicKey): bool
    {
        $accountResponse = $this->sdk->requestAccount($accountId);
        
        // Check if key is the master key
        if ($accountResponse->getAccountId() === $signerPublicKey) {
            return $accountResponse->getSigners()[0]->getWeight() > 0;
        }
        
        // Check if key is in signers list
        foreach ($accountResponse->getSigners() as $signer) {
            if ($signer->getKey() === $signerPublicKey && $signer->getWeight() > 0) {
                return true;
            }
        }
        
        return false;
    }
    
    public function getRequiredSignatures(string $accountId, string $operationType): int
    {
        $accountResponse = $this->sdk->requestAccount($accountId);
        
        // Map operation types to threshold levels
        return match ($operationType) {
            'payment', 'create_account' => $accountResponse->getThresholds()->getLowThreshold(),
            'change_trust', 'manage_offer' => $accountResponse->getThresholds()->getMedThreshold(),
            'account_merge', 'set_options' => $accountResponse->getThresholds()->getHighThreshold(),
            default => $accountResponse->getThresholds()->getLowThreshold()
        };
    }
}
```

### Production Security Checklist

Before deploying to production, verify these security measures:

```php
<?php

class SecurityAudit
{
    public static function performSecurityChecklist(): array
    {
        $issues = [];
        
        // Check environment configuration
        if (!getenv('STELLAR_SECRET_KEY')) {
            $issues[] = 'STELLAR_SECRET_KEY environment variable not set';
        }
        
        if (!getenv('STELLAR_NETWORK_PASSPHRASE')) {
            $issues[] = 'STELLAR_NETWORK_PASSPHRASE environment variable not set';
        }
        
        // Check file permissions (secrets should not be world-readable)
        $secretFiles = ['.env', 'stellar-keys.txt', 'config/secrets.php'];
        foreach ($secretFiles as $file) {
            if (file_exists($file)) {
                $perms = substr(sprintf('%o', fileperms($file)), -4);
                if ($perms !== '0600' && $perms !== '0400') {
                    $issues[] = "File {$file} has insecure permissions: {$perms}";
                }
            }
        }
        
        // Check for hardcoded seeds in code
        $codeFiles = glob('*.php');
        foreach ($codeFiles as $file) {
            $content = file_get_contents($file);
            if (preg_match('/S[A-Z2-7]{55}/', $content)) {
                $issues[] = "Potential hardcoded secret seed found in {$file}";
            }
        }
        
        return $issues;
    }
}

// Run security audit
$securityIssues = SecurityAudit::performSecurityChecklist();
if (!empty($securityIssues)) {
    foreach ($securityIssues as $issue) {
        error_log("SECURITY: {$issue}");
    }
}
```

---

## Testing Patterns

Testing Stellar transactions requires careful setup and mocking strategies. Use PHPUnit for test coverage.

### Unit Testing with PHPUnit

Test transaction building logic without network calls:

```php
<?php

use PHPUnit\Framework\TestCase;
use Soneso\StellarSDK\Account;
use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\Asset;
use Soneso\StellarSDK\PaymentOperation;
use Soneso\StellarSDK\PaymentOperationBuilder;
use Soneso\StellarSDK\TransactionBuilder;

class TransactionBuilderTest extends TestCase
{
    private KeyPair $sourceKeyPair;
    private Account $sourceAccount;
    
    protected function setUp(): void
    {
        $this->sourceKeyPair = KeyPair::random();
        $this->sourceAccount = new Account(
            $this->sourceKeyPair->getAccountId(),
            12345
        );
    }
    
    public function testPaymentTransactionBuilding(): void
    {
        $payment = (new PaymentOperationBuilder(
            'GDESTINATION123...',
            Asset::native(),
            '100.50'
        ))->build();
        
        $transaction = (new TransactionBuilder($this->sourceAccount))
            ->addOperation($payment)
            ->build();
        
        $this->assertEquals(1, count($transaction->getOperations()));
        $this->assertEquals(100, $transaction->getFee());
        $this->assertInstanceOf(
            PaymentOperation::class,
            $transaction->getOperations()[0]
        );
    }
    
    public function testMultiSigTransactionSigning(): void
    {
        $signer1 = KeyPair::random();
        $signer2 = KeyPair::random();
        
        $payment = (new PaymentOperationBuilder(
            'GDEST...',
            Asset::native(),
            '50'
        ))->build();
        
        $transaction = (new TransactionBuilder($this->sourceAccount))
            ->addOperation($payment)
            ->build();
        
        // Sign with multiple keys
        $transaction->sign($this->sourceKeyPair, Network::testnet());
        $transaction->sign($signer1, Network::testnet());
        
        $this->assertEquals(2, count($transaction->getSignatures()));
    }
}
```

### Integration Testing Patterns

Test against real network with proper setup and cleanup:

```php
<?php

use PHPUnit\Framework\TestCase;
use Soneso\StellarSDK\AccountMergeOperationBuilder;
use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\StellarSDK;
use Soneso\StellarSDK\TransactionBuilder;
use Soneso\StellarSDK\Util\FriendBot;

class StellarIntegrationTest extends TestCase
{
    private StellarSDK $sdk;
    private KeyPair $testKeyPair;
    
    protected function setUp(): void
    {
        $this->sdk = StellarSDK::getTestNetInstance();
        $this->testKeyPair = KeyPair::random();
        
        // Fund test account
        FriendBot::fundTestAccount($this->testKeyPair->getAccountId());
        
        // Wait for account to be available
        sleep(2);
    }
    
    protected function tearDown(): void
    {
        // Merge account back to recover XLM (cleanup)
        try {
            $account = $this->sdk->requestAccount($this->testKeyPair->getAccountId());
            $mergeOp = (new AccountMergeOperationBuilder(
                'GAIH3ULLFQ4DGSECF2AR555KZ4KNDGEKN4AFI4SU2M7B43MGK3QJZNSR' // Friendbot
            ))->build();
            
            $tx = (new TransactionBuilder($account))
                ->addOperation($mergeOp)
                ->build();
                
            $tx->sign($this->testKeyPair, Network::testnet());
            $this->sdk->submitTransaction($tx);
        } catch (\Exception $e) {
            // Cleanup failed, but don't fail test
            error_log("Cleanup failed: " . $e->getMessage());
        }
    }
    
    public function testAccountCreation(): void
    {
        $account = $this->sdk->requestAccount($this->testKeyPair->getAccountId());
        
        $this->assertEquals($this->testKeyPair->getAccountId(), $account->getAccountId());
        $this->assertGreaterThan(0, $account->getSequenceNumber());
    }
}
```

### Mocking the SDK for Tests

Mock Horizon responses to test error handling:

```php
<?php

use PHPUnit\Framework\TestCase;
use Soneso\StellarSDK\Exceptions\HorizonRequestException;
use Soneso\StellarSDK\Responses\Account\AccountResponse;
use Soneso\StellarSDK\StellarSDK;

class MockedHorizonTest extends TestCase
{
    public function testHandleAccountNotFound(): void
    {
        // Mock StellarSDK
        $mockSdk = $this->createMock(StellarSDK::class);
        
        // Configure mock to throw 404 exception
        $mockSdk->method('requestAccount')
            ->willThrowException(new HorizonRequestException(
                'Account not found',
                404
            ));
        
        $service = new AccountService($mockSdk);
        
        $result = $service->safeGetAccount('GINVALIDACCOUNT');
        
        $this->assertNull($result);
    }
    
    public function testHandleRateLimiting(): void
    {
        $mockSdk = $this->createMock(StellarSDK::class);
        
        // First call rate limited, second succeeds
        $mockSdk->method('requestAccount')
            ->willReturnOnConsecutiveCalls(
                $this->throwException(new HorizonRequestException('Rate limited', 429)),
                $this->createMockAccount()
            );
        
        $service = new AccountService($mockSdk);
        
        $result = $service->getAccountWithRetry('GVALIDACCOUNT');
        
        $this->assertNotNull($result);
    }
    
    private function createMockAccount()
    {
        $mock = $this->createMock(AccountResponse::class);
        $mock->method('getAccountId')->willReturn('GVALIDACCOUNT');
        return $mock;
    }
}

class AccountService
{
    private StellarSDK $sdk;
    
    public function __construct(StellarSDK $sdk)
    {
        $this->sdk = $sdk;
    }
    
    public function safeGetAccount(string $accountId): ?AccountResponse
    {
        try {
            return $this->sdk->requestAccount($accountId);
        } catch (HorizonRequestException $e) {
            if ($e->getStatusCode() === 404) {
                return null;
            }
            throw $e;
        }
    }
    
    public function getAccountWithRetry(string $accountId, int $maxRetries = 3): ?AccountResponse
    {
        for ($i = 0; $i < $maxRetries; $i++) {
            try {
                return $this->sdk->requestAccount($accountId);
            } catch (HorizonRequestException $e) {
                if ($e->getStatusCode() === 429) {
                    $retryAfter = $e->getRetryAfter() ?: 1;
                    sleep($retryAfter);
                    continue;
                }
                throw $e;
            }
        }
        
        return null;
    }
}
```

### Testing Error Scenarios

Test how your code handles various failure modes:

```php
<?php

use PHPUnit\Framework\TestCase;
use Soneso\StellarSDK\Exceptions\HorizonRequestException;

class ErrorHandlingTest extends TestCase
{
    public function testTransactionFailureHandling(): void
    {
        $this->expectException(\RuntimeException::class);
        $this->expectExceptionMessage('Transaction failed: tx_bad_seq');
        
        $processor = new TransactionProcessor();
        
        // Simulate transaction with wrong sequence number
        $processor->handleTransactionFailure([
            'type' => 'transaction_failed',
            'extras' => [
                'result_codes' => [
                    'transaction' => 'tx_bad_seq'
                ]
            ]
        ]);
    }
    
    public function testNetworkConnectivityIssues(): void
    {
        $processor = new NetworkProcessor();
        
        // Test connection timeout
        $result = $processor->handleNetworkError(new \Exception('Connection timeout'));
        
        $this->assertFalse($result['success']);
        $this->assertEquals('retry', $result['action']);
    }
    
    public function testInsufficientBalanceScenarios(): void
    {
        $validator = new TransactionValidator();
        
        $isValid = $validator->validateBalance('GACCOUNT', '1000');
        
        $this->assertFalse($isValid);
    }
}
```

---

## Advanced Network Operations

Advanced networking patterns for production applications including streaming, failover, and custom HTTP configuration.

### SSE Streaming with Reconnection

Stream real-time data from Horizon with automatic reconnection:

```php
<?php

use GuzzleHttp\Client;
use GuzzleHttp\Exception\ConnectException;
use Psr\Http\Message\ResponseInterface;
use Soneso\StellarSDK\StellarSDK;

class StellarStreamer
{
    private StellarSDK $sdk;
    private Client $httpClient;
    private int $reconnectDelay;
    
    public function __construct(StellarSDK $sdk, int $reconnectDelay = 5)
    {
        $this->sdk = $sdk;
        $this->reconnectDelay = $reconnectDelay;
        $this->httpClient = new Client([
            'timeout' => 0, // No timeout for streaming
            'stream' => true
        ]);
    }
    
    public function streamTransactions(string $accountId, callable $onTransaction, callable $onError = null): void
    {
        $cursor = 'now';
        
        while (true) {
            try {
                $url = $this->sdk->getServer()->getServerUrl() . '/accounts/' . $accountId . '/transactions';
                $url .= '?cursor=' . $cursor . '&order=asc';
                
                $response = $this->httpClient->get($url, [
                    'headers' => ['Accept' => 'text/event-stream'],
                    'stream' => true
                ]);
                
                $this->processEventStream($response, $onTransaction, $cursor);
                
            } catch (ConnectException $e) {
                if ($onError) {
                    $onError($e);
                }
                
                error_log("Stream disconnected, reconnecting in {$this->reconnectDelay}s: " . $e->getMessage());
                sleep($this->reconnectDelay);
                
            } catch (\Exception $e) {
                if ($onError) {
                    $onError($e);
                }
                
                error_log("Stream error: " . $e->getMessage());
                sleep($this->reconnectDelay);
            }
        }
    }
    
    private function processEventStream(ResponseInterface $response, callable $onTransaction, string &$cursor): void
    {
        $body = $response->getBody();
        
        while (!$body->eof()) {
            $line = $this->readLine($body);
            
            if (strpos($line, 'data: ') === 0) {
                $data = substr($line, 6);
                $transaction = json_decode($data, true);
                
                if ($transaction && isset($transaction['paging_token'])) {
                    $cursor = $transaction['paging_token'];
                    $onTransaction($transaction);
                }
            }
        }
    }
    
    private function readLine($stream): string
    {
        $line = '';
        while (!$stream->eof()) {
            $char = $stream->read(1);
            if ($char === "\n") {
                break;
            }
            $line .= $char;
        }
        return trim($line);
    }
}

// Usage example
$sdk = StellarSDK::getPublicNetInstance();
$streamer = new StellarStreamer($sdk);

$streamer->streamTransactions(
    'GACCOUNT...',
    function ($transaction) {
        echo "New transaction: " . $transaction['hash'] . "\n";
        // Process transaction
    },
    function ($error) {
        error_log("Stream error: " . $error->getMessage());
    }
);
```

### Server Failover Patterns

Implement automatic failover across multiple Horizon servers:

```php
<?php

use GuzzleHttp\Exception\ConnectException;
use Soneso\StellarSDK\StellarSDK;

class FailoverStellarSDK
{
    private array $servers;
    private int $currentIndex = 0;
    private int $maxRetries;
    
    public function __construct(array $servers, int $maxRetries = 3)
    {
        $this->servers = $servers;
        $this->maxRetries = $maxRetries;
    }
    
    public function executeWithFailover(callable $operation)
    {
        $exceptions = [];
        
        for ($attempt = 0; $attempt < $this->maxRetries; $attempt++) {
            $serverUrl = $this->getCurrentServer();
            $sdk = new StellarSDK($serverUrl);
            
            try {
                return $operation($sdk);
                
            } catch (ConnectException $e) {
                $exceptions[] = [
                    'server' => $serverUrl,
                    'exception' => $e
                ];
                
                $this->switchToNextServer();
                
                if ($attempt < $this->maxRetries - 1) {
                    error_log("Server {$serverUrl} failed, switching to next server");
                    sleep(1); // Brief delay before retry
                }
            }
        }
        
        $errorMessage = "All servers failed:\n";
        foreach ($exceptions as $error) {
            $errorMessage .= "- {$error['server']}: {$error['exception']->getMessage()}\n";
        }
        
        throw new \RuntimeException($errorMessage);
    }
    
    private function getCurrentServer(): string
    {
        return $this->servers[$this->currentIndex];
    }
    
    private function switchToNextServer(): void
    {
        $this->currentIndex = ($this->currentIndex + 1) % count($this->servers);
    }
    
    public function requestAccount(string $accountId)
    {
        return $this->executeWithFailover(function (StellarSDK $sdk) use ($accountId) {
            return $sdk->requestAccount($accountId);
        });
    }
    
    public function submitTransaction($transaction)
    {
        return $this->executeWithFailover(function (StellarSDK $sdk) use ($transaction) {
            return $sdk->submitTransaction($transaction);
        });
    }
}

// Usage
$failoverSdk = new FailoverStellarSDK([
    'https://horizon.stellar.org',
    'https://horizon-backup1.example.com',
    'https://horizon-backup2.example.com'
]);

try {
    $account = $failoverSdk->requestAccount('GACCOUNT...');
    echo "Account loaded from: " . $failoverSdk->getCurrentServer() . "\n";
} catch (\RuntimeException $e) {
    error_log("All servers failed: " . $e->getMessage());
}
```

### Custom HTTP Configuration

Configure HTTP client with custom timeouts, retry policies, and middleware:

```php
<?php

use GuzzleHttp\Client;
use GuzzleHttp\HandlerStack;
use GuzzleHttp\Middleware;
use GuzzleHttp\Psr7\Request;
use GuzzleHttp\Psr7\Response;
use Soneso\StellarSDK\StellarSDK;

class CustomHttpStellarSDK
{
    private StellarSDK $sdk;
    
    public function __construct(string $serverUrl)
    {
        $httpClient = $this->createCustomHttpClient();
        $this->sdk = new StellarSDK($serverUrl);
        $this->sdk->setHttpClient($httpClient);
    }
    
    private function createCustomHttpClient(): Client
    {
        $stack = HandlerStack::create();
        
        // Add retry middleware
        $stack->push(Middleware::retry(
            function ($retries, Request $request, Response $response = null, \Exception $exception = null) {
                // Retry on 5xx errors or connection issues
                if ($retries < 3) {
                    if ($exception instanceof ConnectException) {
                        return true;
                    }
                    if ($response && $response->getStatusCode() >= 500) {
                        return true;
                    }
                }
                return false;
            },
            function ($retries) {
                // Exponential backoff: 1s, 2s, 4s
                return 1000 * (2 ** $retries);
            }
        ));
        
        // Add logging middleware
        $stack->push(Middleware::mapRequest(function (Request $request) {
            error_log("HTTP Request: " . $request->getMethod() . ' ' . $request->getUri());
            return $request;
        }));
        
        $stack->push(Middleware::mapResponse(function (Response $response) {
            error_log("HTTP Response: " . $response->getStatusCode());
            return $response;
        }));
        
        return new Client([
            'handler' => $stack,
            'timeout' => 30, // 30 second timeout
            'connect_timeout' => 5, // 5 second connection timeout
            'http_errors' => false, // Don't throw on HTTP errors
            'headers' => [
                'User-Agent' => 'MyApp/1.0 stellar-php-sdk',
                'Accept' => 'application/json',
            ],
            // Connection pooling
            'curl' => [
                CURLOPT_TCP_KEEPALIVE => 1,
                CURLOPT_TCP_KEEPIDLE => 300,
                CURLOPT_TCP_KEEPINTVL => 60,
            ]
        ]);
    }
    
    public function __call(string $method, array $arguments)
    {
        return $this->sdk->{$method}(...$arguments);
    }
}

// Usage
$customSdk = new CustomHttpStellarSDK('https://horizon.stellar.org');

// All requests now use custom HTTP configuration
try {
    $account = $customSdk->requestAccount('GACCOUNT...');
    $response = $customSdk->submitTransaction($transaction);
} catch (\Exception $e) {
    error_log("Request failed after retries: " . $e->getMessage());
}
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
