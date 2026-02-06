# SEP-24: Interactive Deposit and Withdrawal

SEP-24 defines the standard way to move money between traditional financial systems and the Stellar network. The anchor hosts a web interface where users complete the deposit or withdrawal process, handling KYC, payment methods, and compliance requirements.

Use SEP-24 when:
- You want to deposit fiat currency (USD, EUR, etc.) to receive Stellar tokens
- You want to withdraw Stellar tokens back to a bank account or other payment method
- The anchor needs to collect information interactively from the user
- You're building a wallet that connects to regulated on/off ramps

See the [SEP-24 specification](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0024.md) for protocol details.

## Quick Example

```php
<?php

use Soneso\StellarSDK\SEP\Interactive\InteractiveService;
use Soneso\StellarSDK\SEP\Interactive\SEP24DepositRequest;

// Create service from anchor's domain
$service = InteractiveService::fromDomain("testanchor.stellar.org");

// Start a deposit flow (requires SEP-10 JWT token)
$request = new SEP24DepositRequest();
$request->jwt = $jwtToken;
$request->assetCode = "USD";

$response = $service->deposit($request);

// Open this URL in a browser or webview for the user
$interactiveUrl = $response->url;
$transactionId = $response->id;

echo "Open: $interactiveUrl\n";
echo "Transaction ID: $transactionId\n";
```

## Creating the Interactive Service

**From an anchor's domain** (recommended):

```php
<?php

use Soneso\StellarSDK\SEP\Interactive\InteractiveService;

// Loads the TRANSFER_SERVER_SEP0024 URL from stellar.toml
$service = InteractiveService::fromDomain("testanchor.stellar.org");
```

**From a direct URL**:

```php
<?php

use Soneso\StellarSDK\SEP\Interactive\InteractiveService;

$service = new InteractiveService("https://api.anchor.com/sep24");
```

## Getting Anchor Information

Before starting a deposit or withdrawal, check what the anchor supports:

```php
<?php

use Soneso\StellarSDK\SEP\Interactive\InteractiveService;

$service = InteractiveService::fromDomain("testanchor.stellar.org");

$info = $service->info(); // Optional: pass "de" for German, etc.

// Check supported deposit assets
$depositAssets = $info->getDepositAssets();
foreach ($depositAssets as $code => $asset) {
    echo "Deposit: $code\n";
    echo "  Enabled: " . ($asset->enabled ? "Yes" : "No") . "\n";
    echo "  Min: " . $asset->minAmount . "\n";
    echo "  Max: " . $asset->maxAmount . "\n";
    echo "  Fee: " . $asset->feeFixed . " + " . $asset->feePercent . "%\n";
}

// Check supported withdrawal assets
$withdrawAssets = $info->getWithdrawAssets();

// Check feature support
$features = $info->featureFlags;
echo "Claimable balances: " . ($features?->claimableBalances ? "Yes" : "No") . "\n";
```

## Deposit Flow

A deposit converts external funds (bank transfer, card, etc.) into Stellar tokens sent to your account.

### Basic Deposit

```php
<?php

use Soneso\StellarSDK\SEP\Interactive\InteractiveService;
use Soneso\StellarSDK\SEP\Interactive\SEP24DepositRequest;

$service = InteractiveService::fromDomain("testanchor.stellar.org");

$request = new SEP24DepositRequest();
$request->jwt = $jwtToken; // From SEP-10 authentication
$request->assetCode = "USD";

$response = $service->deposit($request);

// Show the interactive URL to your user
$url = $response->url;
$transactionId = $response->id;

// The user completes the deposit in their browser
// Then poll for status updates (see "Tracking Transactions" below)
```

### Deposit with Amount and Account Options

```php
<?php

use Soneso\StellarSDK\SEP\Interactive\InteractiveService;
use Soneso\StellarSDK\SEP\Interactive\SEP24DepositRequest;

$service = InteractiveService::fromDomain("testanchor.stellar.org");

$request = new SEP24DepositRequest();
$request->jwt = $jwtToken;
$request->assetCode = "USD";
$request->amount = "100.00";

// Receive tokens on a different account than the one used for SEP-10 auth
$request->account = "GXXXXXXX...";
$request->memo = "12345";
$request->memoType = "id";

// Wallet metadata (helpful for anchors to track integrations)
$request->walletName = "MyWallet";
$request->walletUrl = "https://mywallet.com";

$response = $service->deposit($request);
```

### Pre-filling KYC Data

Speed up the interactive flow by providing KYC data upfront:

```php
<?php

use Soneso\StellarSDK\SEP\Interactive\InteractiveService;
use Soneso\StellarSDK\SEP\Interactive\SEP24DepositRequest;
use Soneso\StellarSDK\SEP\StandardKYCFields\StandardKYCFields;
use Soneso\StellarSDK\SEP\StandardKYCFields\NaturalPersonKYCFields;

$service = InteractiveService::fromDomain("testanchor.stellar.org");

$personFields = new NaturalPersonKYCFields();
$personFields->firstName = "Jane";
$personFields->lastName = "Doe";
$personFields->emailAddress = "jane@example.com";

$kycFields = new StandardKYCFields();
$kycFields->naturalPersonKYCFields = $personFields;

$request = new SEP24DepositRequest();
$request->jwt = $jwtToken;
$request->assetCode = "USD";
$request->kycFields = $kycFields;

$response = $service->deposit($request);
// The anchor will pre-fill these fields in the interactive form
```

## Withdrawal Flow

A withdrawal converts Stellar tokens into external funds sent to a bank, card, or other destination.

### Basic Withdrawal

```php
<?php

use Soneso\StellarSDK\SEP\Interactive\InteractiveService;
use Soneso\StellarSDK\SEP\Interactive\SEP24WithdrawRequest;

$service = InteractiveService::fromDomain("testanchor.stellar.org");

$request = new SEP24WithdrawRequest();
$request->jwt = $jwtToken;
$request->assetCode = "USD";

$response = $service->withdraw($request);

// Show the interactive URL to your user
$url = $response->url;
$transactionId = $response->id;

// After completing the form, the anchor will tell the user where to send tokens
```

### Withdrawal with Options

```php
<?php

use Soneso\StellarSDK\SEP\Interactive\InteractiveService;
use Soneso\StellarSDK\SEP\Interactive\SEP24WithdrawRequest;

$service = InteractiveService::fromDomain("testanchor.stellar.org");

$request = new SEP24WithdrawRequest();
$request->jwt = $jwtToken;
$request->assetCode = "USD";
$request->amount = "500.00";

// For refunds if the withdrawal fails
$request->refundMemo = "refund-123";
$request->refundMemoType = "text";

$response = $service->withdraw($request);
```

## Tracking Transactions

After starting a deposit or withdrawal, poll for status updates:

### Get a Single Transaction

```php
<?php

use Soneso\StellarSDK\SEP\Interactive\InteractiveService;
use Soneso\StellarSDK\SEP\Interactive\SEP24TransactionRequest;

$service = InteractiveService::fromDomain("testanchor.stellar.org");

$request = new SEP24TransactionRequest();
$request->jwt = $jwtToken;
$request->id = $transactionId; // From deposit/withdraw response

$response = $service->transaction($request);
$tx = $response->transaction;

echo "Status: " . $tx->status . "\n";
echo "Amount: " . $tx->amountIn . " " . $tx->amountInAsset . "\n";

// Check if there's an action needed from the user
if ($tx->status === "pending_user_transfer_start") {
    // User needs to send the Stellar payment
    echo "Send to: " . $tx->withdrawAnchorAccount . "\n";
    echo "Memo: " . $tx->withdrawMemo . "\n";
}
```

### Get Transaction by Stellar Transaction ID

```php
<?php

use Soneso\StellarSDK\SEP\Interactive\InteractiveService;
use Soneso\StellarSDK\SEP\Interactive\SEP24TransactionRequest;

$service = InteractiveService::fromDomain("testanchor.stellar.org");

$request = new SEP24TransactionRequest();
$request->jwt = $jwtToken;
$request->stellarTransactionId = "abc123..."; // Stellar transaction hash

$response = $service->transaction($request);
```

### Get All Transactions

```php
<?php

use Soneso\StellarSDK\SEP\Interactive\InteractiveService;
use Soneso\StellarSDK\SEP\Interactive\SEP24TransactionsRequest;

$service = InteractiveService::fromDomain("testanchor.stellar.org");

$request = new SEP24TransactionsRequest();
$request->jwt = $jwtToken;
$request->assetCode = "USD";
$request->limit = 10;
$request->kind = "deposit"; // or "withdrawal"

// Only transactions after this date
$request->noOlderThan = new DateTime("2024-01-01");

$response = $service->transactions($request);

foreach ($response->transactions as $tx) {
    echo $tx->id . ": " . $tx->status . " - " . $tx->amountIn . "\n";
}
```

## Transaction Statuses

Common statuses you'll encounter:

| Status | Description |
|--------|-------------|
| `incomplete` | User hasn't completed the interactive flow |
| `pending_user_transfer_start` | Waiting for user to send/receive funds |
| `pending_anchor` | Anchor is processing |
| `pending_stellar` | Waiting for Stellar transaction |
| `completed` | Transaction finished successfully |
| `error` | Something went wrong |
| `expired` | Transaction timed out |

## Error Handling

```php
<?php

use Soneso\StellarSDK\SEP\Interactive\InteractiveService;
use Soneso\StellarSDK\SEP\Interactive\SEP24DepositRequest;
use Soneso\StellarSDK\SEP\Interactive\SEP24TransactionRequest;
use Soneso\StellarSDK\SEP\Interactive\SEP24AuthenticationRequiredException;
use Soneso\StellarSDK\SEP\Interactive\SEP24TransactionNotFoundException;
use Soneso\StellarSDK\SEP\Interactive\RequestErrorException;
use GuzzleHttp\Exception\GuzzleException;

$service = InteractiveService::fromDomain("testanchor.stellar.org");

try {
    $request = new SEP24DepositRequest();
    $request->jwt = $jwtToken;
    $request->assetCode = "USD";
    
    $response = $service->deposit($request);
    
} catch (SEP24AuthenticationRequiredException $e) {
    // JWT token is invalid or expired
    // Re-authenticate with SEP-10
    echo "Need to re-authenticate\n";
    
} catch (RequestErrorException $e) {
    // Anchor returned an error (invalid asset, amount out of range, etc.)
    echo "Error: " . $e->getMessage() . "\n";
    
} catch (GuzzleException $e) {
    // Network or HTTP error
    echo "Request failed: " . $e->getMessage() . "\n";
}

// For transaction queries
try {
    $txRequest = new SEP24TransactionRequest();
    $txRequest->jwt = $jwtToken;
    $txRequest->id = "invalid-id";
    
    $response = $service->transaction($txRequest);
    
} catch (SEP24TransactionNotFoundException $e) {
    echo "Transaction not found\n";
}
```

## Using SEP-38 Quotes

For exchange rate quotes before deposit/withdrawal:

```php
<?php

use Soneso\StellarSDK\SEP\Interactive\InteractiveService;
use Soneso\StellarSDK\SEP\Interactive\SEP24DepositRequest;

$service = InteractiveService::fromDomain("testanchor.stellar.org");

// First, get a quote from SEP-38 (see SEP-38 documentation)
$quoteId = "quote-abc-123";

$request = new SEP24DepositRequest();
$request->jwt = $jwtToken;
$request->assetCode = "USD";
$request->quoteId = $quoteId;
$request->sourceAsset = "iso4217:EUR"; // Depositing EUR, receiving USD tokens

$response = $service->deposit($request);
```

## Fee Information

The anchor's `/info` response includes fee data. For complex fee schedules:

```php
<?php

use Soneso\StellarSDK\SEP\Interactive\InteractiveService;
use Soneso\StellarSDK\SEP\Interactive\SEP24FeeRequest;

$service = InteractiveService::fromDomain("testanchor.stellar.org");

$info = $service->info();

// Check if fee endpoint is available
if ($info->feeEndpointInfo?->enabled) {
    $feeRequest = new SEP24FeeRequest(
        operation: "deposit",
        assetCode: "USD",
        amount: 1000.00,
        jwt: $jwtToken
    );
    
    $feeResponse = $service->fee($feeRequest);
    echo "Fee for $1000 deposit: $" . $feeResponse->fee . "\n";
}
```

Note: The fee endpoint is deprecated in favor of SEP-38. New anchors should use SEP-38 `/price` for fee information.

## Claimable Balances

If your account can't receive assets directly (no trustline), request claimable balances:

```php
<?php

use Soneso\StellarSDK\SEP\Interactive\InteractiveService;
use Soneso\StellarSDK\SEP\Interactive\SEP24DepositRequest;

$service = InteractiveService::fromDomain("testanchor.stellar.org");

$request = new SEP24DepositRequest();
$request->jwt = $jwtToken;
$request->assetCode = "USD";
$request->claimableBalanceSupported = true;

$response = $service->deposit($request);
// The anchor may create a claimable balance instead of a direct payment
```

## Related SEPs

- [SEP-1](sep-01.md) - stellar.toml (where TRANSFER_SERVER_SEP0024 is published)
- [SEP-10](sep-10.md) - Web Authentication (required for SEP-24)
- [SEP-12](sep-12.md) - KYC API (often used alongside SEP-24)
- [SEP-38](sep-38.md) - Anchor RFQ API (quotes for exchange rates)
- [SEP-6](sep-06.md) - Programmatic Deposit/Withdrawal (non-interactive alternative)
