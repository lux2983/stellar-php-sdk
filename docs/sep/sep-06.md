# SEP-06: Deposit and Withdrawal API

SEP-06 enables programmatic deposits and withdrawals through anchors. Users send off-chain assets (USD via bank, BTC, etc.) and receive Stellar tokens, or redeem Stellar tokens for off-chain assets.

**Use SEP-06 when:**
- Building automated deposit/withdrawal flows
- Integrating anchor services without user-facing web flows
- You need programmatic access (vs. SEP-24's interactive approach)

**Spec:** [SEP-0006](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0006.md)

## Quick Example

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\SEP\TransferServerService\DepositRequest;
use Soneso\StellarSDK\SEP\TransferServerService\TransferServerService;
use Soneso\StellarSDK\SEP\WebAuth\WebAuth;

// 1. Authenticate with the anchor
$webAuth = WebAuth::fromDomain("testanchor.stellar.org", Network::testnet());
$userKeyPair = KeyPair::fromSeed("SCZANGBA5YHTNYVVV3C7CAZMTQDBJHJG6C34CJDQ66EQ7DZTPBRJFN4A");
$jwtToken = $webAuth->jwtToken($userKeyPair->getAccountId(), [$userKeyPair]);

// 2. Create transfer service and request deposit
$transferService = TransferServerService::fromDomain("testanchor.stellar.org");

$request = new DepositRequest(
    assetCode: "USD",
    account: $userKeyPair->getAccountId(),
    jwt: $jwtToken
);

$response = $transferService->deposit($request);

echo "Deposit instructions: " . $response->how . PHP_EOL;
echo "Fee: " . $response->feeFixed . PHP_EOL;
```

## Detailed Usage

### Creating the Service

**From domain (recommended):**

```php
<?php

use Soneso\StellarSDK\SEP\TransferServerService\TransferServerService;

// Discovers TRANSFER_SERVER from stellar.toml
$transferService = TransferServerService::fromDomain("testanchor.stellar.org");
```

**Direct URL:**

```php
<?php

use Soneso\StellarSDK\SEP\TransferServerService\TransferServerService;

$transferService = new TransferServerService("https://testanchor.stellar.org/sep6");
```

### Querying Anchor Info

Check what assets and methods the anchor supports:

```php
<?php

use Soneso\StellarSDK\SEP\TransferServerService\TransferServerService;

$transferService = TransferServerService::fromDomain("testanchor.stellar.org");
$info = $transferService->info();

// Check deposit assets
foreach ($info->depositAssets as $code => $asset) {
    echo "Deposit $code: " . ($asset->enabled ? "enabled" : "disabled") . PHP_EOL;
    if ($asset->minAmount) {
        echo "  Min: $asset->minAmount" . PHP_EOL;
    }
}

// Check withdrawal assets
foreach ($info->withdrawAssets as $code => $asset) {
    echo "Withdraw $code: " . ($asset->enabled ? "enabled" : "disabled") . PHP_EOL;
}

// Feature flags
echo "Fee endpoint enabled: " . ($info->feeInfo->enabled ? "yes" : "no") . PHP_EOL;
```

### Deposits

Request deposit instructions from the anchor:

```php
<?php

use Soneso\StellarSDK\SEP\TransferServerService\DepositRequest;
use Soneso\StellarSDK\SEP\TransferServerService\TransferServerService;
use Soneso\StellarSDK\SEP\TransferServerService\CustomerInformationNeededException;
use Soneso\StellarSDK\SEP\TransferServerService\CustomerInformationStatusException;

$transferService = TransferServerService::fromDomain("testanchor.stellar.org");

$request = new DepositRequest(
    assetCode: "USD",
    account: "GCQTGZQTVZ...",  // Stellar account to receive tokens
    jwt: $jwtToken,
    type: "bank_account",      // Optional: deposit method
    amount: "100.00"           // Optional: helps anchor determine KYC needs
);

try {
    $response = $transferService->deposit($request);
    
    // Display deposit instructions to user
    echo "How to deposit: " . $response->how . PHP_EOL;
    
    if ($response->instructions) {
        foreach ($response->instructions as $key => $instruction) {
            echo "$key: " . $instruction->value . PHP_EOL;
            if ($instruction->description) {
                echo "  (" . $instruction->description . ")" . PHP_EOL;
            }
        }
    }
    
    // Fee info
    if ($response->feeFixed) {
        echo "Fixed fee: " . $response->feeFixed . PHP_EOL;
    }
    if ($response->feePercent) {
        echo "Percent fee: " . $response->feePercent . "%" . PHP_EOL;
    }
    
} catch (CustomerInformationNeededException $e) {
    // Anchor needs KYC info via SEP-12
    echo "Required fields: " . PHP_EOL;
    foreach ($e->response->fields as $field) {
        echo "  - $field" . PHP_EOL;
    }
    
} catch (CustomerInformationStatusException $e) {
    // KYC submitted but pending/denied
    echo "KYC status: " . $e->response->status . PHP_EOL;
}
```

### Withdrawals

Redeem Stellar tokens for off-chain assets:

```php
<?php

use Soneso\StellarSDK\SEP\TransferServerService\WithdrawRequest;
use Soneso\StellarSDK\SEP\TransferServerService\TransferServerService;
use Soneso\StellarSDK\SEP\TransferServerService\CustomerInformationNeededException;
use Soneso\StellarSDK\SEP\TransferServerService\CustomerInformationStatusException;

$transferService = TransferServerService::fromDomain("testanchor.stellar.org");

$request = new WithdrawRequest(
    assetCode: "NGNT",
    type: "bank_account",
    jwt: $jwtToken,
    account: "GCQTGZQTVZ...",  // Optional: source Stellar account
    amount: "500.00"
);

try {
    $response = $transferService->withdraw($request);
    
    // Where to send the Stellar payment
    echo "Send payment to: " . $response->accountId . PHP_EOL;
    
    if ($response->memoType && $response->memo) {
        echo "Memo ($response->memoType): " . $response->memo . PHP_EOL;
    }
    
    // Fee info
    if ($response->feeFixed) {
        echo "Fixed fee: " . $response->feeFixed . PHP_EOL;
    }
    
} catch (CustomerInformationNeededException $e) {
    echo "Need KYC fields: " . implode(", ", $e->response->fields) . PHP_EOL;
    
} catch (CustomerInformationStatusException $e) {
    echo "KYC status: " . $e->response->status . PHP_EOL;
}
```

### Exchange Operations (Cross-Asset)

For deposits/withdrawals with currency conversion using SEP-38 quotes:

```php
<?php

use Soneso\StellarSDK\SEP\TransferServerService\DepositExchangeRequest;
use Soneso\StellarSDK\SEP\TransferServerService\WithdrawExchangeRequest;
use Soneso\StellarSDK\SEP\TransferServerService\TransferServerService;

$transferService = TransferServerService::fromDomain("testanchor.stellar.org");

// Deposit BRL, receive USDC on Stellar
$depositExchange = new DepositExchangeRequest(
    destinationAsset: "USDC",
    sourceAsset: "iso4217:BRL",  // Off-chain BRL
    amount: "480.00",
    account: "GCQTGZQTVZ...",
    quoteId: "282837",  // From SEP-38 quote
    jwt: $jwtToken
);

$response = $transferService->depositExchange($depositExchange);

// Withdraw USDC, receive NGN to bank
$withdrawExchange = new WithdrawExchangeRequest(
    sourceAsset: "USDC",
    destinationAsset: "iso4217:NGN",
    amount: "100.00",
    type: "bank_account",
    quoteId: "282838",
    jwt: $jwtToken
);

$response = $transferService->withdrawExchange($withdrawExchange);
```

### Checking Fees

Query fees before initiating transfers:

```php
<?php

use Soneso\StellarSDK\SEP\TransferServerService\FeeRequest;
use Soneso\StellarSDK\SEP\TransferServerService\TransferServerService;

$transferService = TransferServerService::fromDomain("testanchor.stellar.org");

$feeRequest = new FeeRequest(
    operation: "deposit",
    assetCode: "USD",
    amount: 100.00,
    jwt: $jwtToken
);

$feeResponse = $transferService->fee($feeRequest);
echo "Fee for deposit: " . $feeResponse->fee . PHP_EOL;
```

### Transaction History

List all transactions for an account:

```php
<?php

use Soneso\StellarSDK\SEP\TransferServerService\AnchorTransactionsRequest;
use Soneso\StellarSDK\SEP\TransferServerService\TransferServerService;

$transferService = TransferServerService::fromDomain("testanchor.stellar.org");

$request = new AnchorTransactionsRequest(
    assetCode: "USD",
    account: "GCQTGZQTVZ...",
    jwt: $jwtToken,
    limit: 10,
    kind: "deposit"  // Optional: filter by type
);

$response = $transferService->transactions($request);

foreach ($response->transactions as $tx) {
    echo "Transaction: " . $tx->id . PHP_EOL;
    echo "  Status: " . $tx->status . PHP_EOL;
    echo "  Kind: " . $tx->kind . PHP_EOL;
    echo "  Amount: " . ($tx->amountIn ?? "pending") . PHP_EOL;
}
```

### Single Transaction Status

Check a specific transaction:

```php
<?php

use Soneso\StellarSDK\SEP\TransferServerService\AnchorTransactionRequest;
use Soneso\StellarSDK\SEP\TransferServerService\TransferServerService;

$transferService = TransferServerService::fromDomain("testanchor.stellar.org");

$request = new AnchorTransactionRequest();
$request->id = "82fhs729f63dh0v4";
$request->jwt = $jwtToken;

$response = $transferService->transaction($request);

echo "Status: " . $response->transaction->status . PHP_EOL;
echo "Kind: " . $response->transaction->kind . PHP_EOL;

// Also supports lookup by Stellar transaction hash
$request->stellarTransactionId = "b9d0b22...";
```

### Updating Pending Transactions

When anchor requests more info via `pending_transaction_info_update` status:

```php
<?php

use Soneso\StellarSDK\SEP\TransferServerService\PatchTransactionRequest;
use Soneso\StellarSDK\SEP\TransferServerService\TransferServerService;

$transferService = TransferServerService::fromDomain("testanchor.stellar.org");

$request = new PatchTransactionRequest(
    id: "82fhs729f63dh0v4",
    fields: [
        "dest" => "12345678901234",        // Bank account
        "dest_extra" => "021000021"        // Routing number
    ],
    jwt: $jwtToken
);

$response = $transferService->patchTransaction($request);
echo "Updated, status code: " . $response->getStatusCode() . PHP_EOL;
```

## Error Handling

```php
<?php

use Soneso\StellarSDK\SEP\TransferServerService\DepositRequest;
use Soneso\StellarSDK\SEP\TransferServerService\TransferServerService;
use Soneso\StellarSDK\SEP\TransferServerService\AuthenticationRequiredException;
use Soneso\StellarSDK\SEP\TransferServerService\CustomerInformationNeededException;
use Soneso\StellarSDK\SEP\TransferServerService\CustomerInformationStatusException;
use GuzzleHttp\Exception\GuzzleException;

try {
    $transferService = TransferServerService::fromDomain("testanchor.stellar.org");
    
    $request = new DepositRequest(
        assetCode: "USD",
        account: "GCQTGZQTVZ...",
        jwt: $jwtToken
    );
    
    $response = $transferService->deposit($request);
    
} catch (AuthenticationRequiredException $e) {
    // Endpoint requires SEP-10 authentication
    echo "Authentication required. Get a JWT token via SEP-10 first.";
    
} catch (CustomerInformationNeededException $e) {
    // Anchor needs KYC info - submit via SEP-12
    echo "KYC required. Fields needed:" . PHP_EOL;
    foreach ($e->response->fields as $field) {
        echo "  - $field" . PHP_EOL;
    }
    
} catch (CustomerInformationStatusException $e) {
    // KYC submitted but has issues
    $status = $e->response->status;
    if ($status === "denied") {
        echo "KYC denied. Contact anchor support.";
    } elseif ($status === "pending") {
        echo "KYC pending review. Try again later.";
    }
    
} catch (GuzzleException $e) {
    // Network errors
    echo "Request failed: " . $e->getMessage();
}
```

### Common Errors

| Exception | Cause | Solution |
|-----------|-------|----------|
| `AuthenticationRequiredException` | Missing JWT | Authenticate via SEP-10 first |
| `CustomerInformationNeededException` | KYC required | Submit info via SEP-12 |
| `CustomerInformationStatusException` | KYC pending/denied | Wait or contact anchor |

## Transaction Statuses

| Status | Meaning |
|--------|---------|
| `incomplete` | Not ready, more info needed |
| `pending_user_transfer_start` | Waiting for user to send funds |
| `pending_user_transfer_complete` | User sent funds, processing |
| `pending_external` | Waiting on external system |
| `pending_anchor` | Anchor processing |
| `pending_stellar` | Stellar transaction pending |
| `pending_transaction_info_update` | Anchor needs more transaction info |
| `pending_customer_info_update` | Anchor needs more KYC info |
| `completed` | Done |
| `refunded` | Refund sent |
| `expired` | Timed out |
| `error` | Failed |

## Related SEPs

- [SEP-10](sep-10.md) - Web authentication (required for most operations)
- [SEP-12](sep-12.md) - KYC API (for customer info submission)
- [SEP-24](sep-24.md) - Interactive deposits/withdrawals (alternative approach)
- [SEP-38](sep-38.md) - Quotes API (for exchange operations)
