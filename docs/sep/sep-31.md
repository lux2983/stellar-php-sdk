# SEP-31: Cross-Border Payments

SEP-31 defines how to send payments between financial accounts that exist outside the Stellar network. A Sending Anchor initiates a payment on behalf of a Sending Client, which is delivered by a Receiving Anchor to a Receiving Client. The actual value transfer happens on the Stellar network, but the endpoints are traditional accounts (bank accounts, mobile wallets, etc.).

Use SEP-31 when:
- Building a remittance service or payment corridor
- Sending money to a recipient who needs funds delivered off-chain
- You're a Sending Anchor integrating with Receiving Anchors
- Processing B2B cross-border payments

The PHP SDK supports the Sending Anchor side of SEP-31.

See the [SEP-31 specification](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0031.md) for protocol details.

## How Cross-Border Payments Work

1. **KYC**: Register sender and receiver via SEP-12 if required
2. **Quote** (optional): Get exchange rate via SEP-38
3. **Initiate**: POST transaction to Receiving Anchor
4. **Pay**: Send Stellar payment with the provided memo
5. **Deliver**: Receiving Anchor delivers funds to recipient

## Quick Example

```php
<?php

use Soneso\StellarSDK\SEP\CrossBorderPayments\CrossBorderPaymentsService;
use Soneso\StellarSDK\SEP\CrossBorderPayments\SEP31PostTransactionsRequest;

// Connect to receiving anchor
$service = CrossBorderPaymentsService::fromDomain("receivinganchor.com");

// Create payment request
$request = new SEP31PostTransactionsRequest(
    amount: 100.00,
    assetCode: "USDC",
    assetIssuer: "GA5ZSEJYB37JRC5AVCIA5MOP4RHTM335X2KGX3IHOJAPP5RE34K4KZVN",
    senderId: "sender-customer-id",   // From SEP-12
    receiverId: "receiver-customer-id" // From SEP-12
);

$response = $service->postTransactions($request, $jwtToken);

// Get payment instructions
echo "Transaction ID: " . $response->id . "\n";
echo "Send to: " . $response->stellarAccountId . "\n";
echo "Memo: " . $response->stellarMemo . "\n";
```

## Creating the Service

**From the receiving anchor's domain** (recommended):

```php
<?php

use Soneso\StellarSDK\SEP\CrossBorderPayments\CrossBorderPaymentsService;

// Loads DIRECT_PAYMENT_SERVER from stellar.toml
$service = CrossBorderPaymentsService::fromDomain("receivinganchor.com");
```

**From a direct URL**:

```php
<?php

use Soneso\StellarSDK\SEP\CrossBorderPayments\CrossBorderPaymentsService;

$service = new CrossBorderPaymentsService("https://api.receivinganchor.com/sep31");
```

## Getting Anchor Information

Check what currencies the receiving anchor supports:

```php
<?php

use Soneso\StellarSDK\SEP\CrossBorderPayments\CrossBorderPaymentsService;

$service = CrossBorderPaymentsService::fromDomain("receivinganchor.com");

$info = $service->info($jwtToken);

// Check supported assets
foreach ($info->receiveAssets as $assetCode => $assetInfo) {
    echo "Asset: $assetCode\n";
    echo "  Enabled: " . ($assetInfo->enabled ? "Yes" : "No") . "\n";
    echo "  Min: " . $assetInfo->minAmount . "\n";
    echo "  Max: " . $assetInfo->maxAmount . "\n";
    
    // Check what KYC is needed
    if ($assetInfo->sep12 !== null) {
        echo "  Sender types: " . implode(", ", $assetInfo->sep12->senderTypes ?? []) . "\n";
        echo "  Receiver types: " . implode(", ", $assetInfo->sep12->receiverTypes ?? []) . "\n";
    }
}
```

## Complete Payment Flow

### Step 1: Register Sender and Receiver (SEP-12)

```php
<?php

use Soneso\StellarSDK\SEP\KYCService\KYCService;
use Soneso\StellarSDK\SEP\KYCService\PutCustomerInfoRequest;
use Soneso\StellarSDK\SEP\StandardKYCFields\StandardKYCFields;
use Soneso\StellarSDK\SEP\StandardKYCFields\NaturalPersonKYCFields;

$kycService = KYCService::fromDomain("receivinganchor.com");

// Register sender
$senderFields = new NaturalPersonKYCFields();
$senderFields->firstName = "Jane";
$senderFields->lastName = "Sender";
$senderFields->emailAddress = "jane@sender.com";

$senderKyc = new StandardKYCFields();
$senderKyc->naturalPersonKYCFields = $senderFields;

$senderRequest = new PutCustomerInfoRequest();
$senderRequest->jwt = $jwtToken;
$senderRequest->KYCFields = $senderKyc;
$senderRequest->type = "sep31-sender";

$senderResponse = $kycService->putCustomerInfo($senderRequest);
$senderId = $senderResponse->getId();

// Register receiver
$receiverFields = new NaturalPersonKYCFields();
$receiverFields->firstName = "Bob";
$receiverFields->lastName = "Receiver";

// Receiver's bank account (using custom fields)
$receiverRequest = new PutCustomerInfoRequest();
$receiverRequest->jwt = $jwtToken;
$receiverRequest->KYCFields = new StandardKYCFields();
$receiverRequest->KYCFields->naturalPersonKYCFields = $receiverFields;
$receiverRequest->type = "sep31-receiver";
$receiverRequest->customFields = [
    "bank_account_number" => "1234567890",
    "bank_routing_number" => "021000021",
];

$receiverResponse = $kycService->putCustomerInfo($receiverRequest);
$receiverId = $receiverResponse->getId();
```

### Step 2: Get a Quote (Optional, SEP-38)

For guaranteed exchange rates, get a quote first:

```php
<?php

use Soneso\StellarSDK\SEP\Quote\QuoteService;
use Soneso\StellarSDK\SEP\Quote\SEP38PostQuoteRequest;

$quoteService = QuoteService::fromDomain("receivinganchor.com");

// Get a firm quote
$quoteRequest = new SEP38PostQuoteRequest(
    sellAsset: "stellar:USDC:GA5ZSEJYB37JRC5AVCIA5MOP4RHTM335X2KGX3IHOJAPP5RE34K4KZVN",
    buyAsset: "iso4217:BRL",
    sellAmount: "100"
);

$quote = $quoteService->postQuote($quoteRequest, $jwtToken);
$quoteId = $quote->id;
$expiresAt = $quote->expiresAt;

echo "Quote: $quoteId (expires: $expiresAt)\n";
echo "Rate: " . $quote->price . "\n";
echo "You'll receive: " . $quote->buyAmount . " BRL\n";
```

### Step 3: Initiate the Transaction

```php
<?php

use Soneso\StellarSDK\SEP\CrossBorderPayments\CrossBorderPaymentsService;
use Soneso\StellarSDK\SEP\CrossBorderPayments\SEP31PostTransactionsRequest;

$service = CrossBorderPaymentsService::fromDomain("receivinganchor.com");

$request = new SEP31PostTransactionsRequest(
    amount: 100.00,
    assetCode: "USDC",
    assetIssuer: "GA5ZSEJYB37JRC5AVCIA5MOP4RHTM335X2KGX3IHOJAPP5RE34K4KZVN",
    destinationAsset: "iso4217:BRL",
    senderId: $senderId,
    receiverId: $receiverId,
    quoteId: $quoteId // Optional: lock in the exchange rate
);

$response = $service->postTransactions($request, $jwtToken);

$transactionId = $response->id;
$stellarAccount = $response->stellarAccountId;
$memo = $response->stellarMemo;
$memoType = $response->stellarMemoType;

echo "Send 100 USDC to: $stellarAccount\n";
echo "With memo ($memoType): $memo\n";
```

### Step 4: Send the Stellar Payment

```php
<?php

use Soneso\StellarSDK\StellarSDK;
use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Asset;
use Soneso\StellarSDK\PaymentOperationBuilder;
use Soneso\StellarSDK\TransactionBuilder;
use Soneso\StellarSDK\Memo;
use Soneso\StellarSDK\Network;

$sdk = StellarSDK::getTestNetInstance();

$senderKeyPair = KeyPair::fromSeed("SXXXXX...");
$account = $sdk->requestAccount($senderKeyPair->getAccountId());

$asset = Asset::createFromCanonicalForm(
    "USDC:GA5ZSEJYB37JRC5AVCIA5MOP4RHTM335X2KGX3IHOJAPP5RE34K4KZVN"
);

$payment = (new PaymentOperationBuilder($stellarAccount, $asset, "100"))
    ->build();

$transaction = (new TransactionBuilder($account))
    ->addOperation($payment)
    ->addMemo(Memo::id((int)$memo)) // Use memo from SEP-31 response
    ->build();

$transaction->sign($senderKeyPair, Network::testnet());
$submitResponse = $sdk->submitTransaction($transaction);

echo "Payment submitted: " . $submitResponse->getHash() . "\n";
```

## Tracking Transaction Status

Poll for updates on your transaction:

```php
<?php

use Soneso\StellarSDK\SEP\CrossBorderPayments\CrossBorderPaymentsService;

$service = CrossBorderPaymentsService::fromDomain("receivinganchor.com");

$response = $service->getTransaction($transactionId, $jwtToken);

echo "Status: " . $response->status . "\n";
echo "Amount in: " . $response->amountIn . "\n";
echo "Amount out: " . $response->amountOut . "\n";

// Check for completion or issues
switch ($response->status) {
    case "pending_sender":
        echo "Waiting for Stellar payment\n";
        break;
    case "pending_stellar":
        echo "Stellar payment received, processing\n";
        break;
    case "pending_receiver":
        echo "Delivering funds to recipient\n";
        break;
    case "completed":
        echo "Payment delivered!\n";
        break;
    case "error":
        echo "Error: " . $response->message . "\n";
        break;
}
```

## Transaction Status Callbacks

Register a URL to receive status updates:

```php
<?php

use Soneso\StellarSDK\SEP\CrossBorderPayments\CrossBorderPaymentsService;
use Soneso\StellarSDK\SEP\CrossBorderPayments\SEP31TransactionCallbackNotSupportedException;

$service = CrossBorderPaymentsService::fromDomain("receivinganchor.com");

try {
    $service->putTransactionCallback(
        $transactionId,
        "https://myanchor.com/callbacks/sep31",
        $jwtToken
    );
    echo "Callback registered\n";
} catch (SEP31TransactionCallbackNotSupportedException $e) {
    echo "Anchor doesn't support callbacks\n";
}
```

Your callback endpoint will receive POST requests with the transaction object when status changes.

## Error Handling

```php
<?php

use Soneso\StellarSDK\SEP\CrossBorderPayments\CrossBorderPaymentsService;
use Soneso\StellarSDK\SEP\CrossBorderPayments\SEP31PostTransactionsRequest;
use Soneso\StellarSDK\SEP\CrossBorderPayments\SEP31CustomerInfoNeededException;
use Soneso\StellarSDK\SEP\CrossBorderPayments\SEP31TransactionInfoNeededException;
use Soneso\StellarSDK\SEP\CrossBorderPayments\SEP31TransactionNotFoundException;
use Soneso\StellarSDK\SEP\CrossBorderPayments\SEP31BadRequestException;
use GuzzleHttp\Exception\GuzzleException;

$service = CrossBorderPaymentsService::fromDomain("receivinganchor.com");

try {
    $request = new SEP31PostTransactionsRequest(
        amount: 100.00,
        assetCode: "USDC"
    );
    
    $response = $service->postTransactions($request, $jwtToken);
    
} catch (SEP31CustomerInfoNeededException $e) {
    // Need to register sender/receiver via SEP-12
    echo "Customer info needed. Type: " . $e->type . "\n";
    // Register the customer, then retry
    
} catch (SEP31TransactionInfoNeededException $e) {
    // Additional transaction fields required
    echo "Transaction fields needed:\n";
    foreach ($e->fields as $field => $info) {
        echo "  $field: " . $info['description'] . "\n";
    }
    
} catch (SEP31BadRequestException $e) {
    // Invalid request data
    echo "Bad request: " . $e->getMessage() . "\n";
    
} catch (SEP31TransactionNotFoundException $e) {
    // Transaction not found (for GET requests)
    echo "Transaction not found\n";
    
} catch (GuzzleException $e) {
    // Network or HTTP error
    echo "Request failed: " . $e->getMessage() . "\n";
}
```

## Refund Configuration

Specify how refunds should be handled if the payment fails:

```php
<?php

use Soneso\StellarSDK\SEP\CrossBorderPayments\CrossBorderPaymentsService;
use Soneso\StellarSDK\SEP\CrossBorderPayments\SEP31PostTransactionsRequest;

$service = CrossBorderPaymentsService::fromDomain("receivinganchor.com");

$request = new SEP31PostTransactionsRequest(
    amount: 100.00,
    assetCode: "USDC",
    assetIssuer: "GA5ZSEJYB37JRC5AVCIA5MOP4RHTM335X2KGX3IHOJAPP5RE34K4KZVN",
    senderId: $senderId,
    receiverId: $receiverId,
    refundMemo: "refund-12345",
    refundMemoType: "text"
);

$response = $service->postTransactions($request, $jwtToken);
```

## Funding Methods

Some anchors support different payment rails:

```php
<?php

use Soneso\StellarSDK\SEP\CrossBorderPayments\CrossBorderPaymentsService;
use Soneso\StellarSDK\SEP\CrossBorderPayments\SEP31PostTransactionsRequest;

$service = CrossBorderPaymentsService::fromDomain("receivinganchor.com");

// Check available funding methods in /info response first
$info = $service->info($jwtToken);

$request = new SEP31PostTransactionsRequest(
    amount: 100.00,
    assetCode: "USDC",
    assetIssuer: "GA5ZSEJYB37JRC5AVCIA5MOP4RHTM335X2KGX3IHOJAPP5RE34K4KZVN",
    senderId: $senderId,
    receiverId: $receiverId,
    fundingMethod: "WIRE" // or "ACH", etc.
);

$response = $service->postTransactions($request, $jwtToken);
```

## Transaction Statuses

| Status | Description |
|--------|-------------|
| `pending_sender` | Waiting for Stellar payment from sending anchor |
| `pending_stellar` | Stellar payment received, being processed |
| `pending_receiver` | Delivering funds to receiving client |
| `pending_external` | Waiting for external system |
| `completed` | Payment delivered successfully |
| `refunded` | Payment was refunded |
| `error` | An error occurred |

## Important Notes

- **Memo is critical**: Always use the exact memo provided by the receiving anchor. The memo is how the anchor identifies your payment.
- **Source account flexibility**: Your Stellar payment can come from any account, not just the SEP-10 authenticated one.
- **Quote expiration**: If using SEP-38 quotes, submit the Stellar payment before the quote expires.
- **KYC first**: Most anchors require SEP-12 KYC for both sender and receiver before accepting transactions.

## Related SEPs

- [SEP-10](sep-10.md) - Web Authentication (required for SEP-31)
- [SEP-12](sep-12.md) - KYC API (register sender/receiver)
- [SEP-38](sep-38.md) - Anchor RFQ API (exchange rate quotes)
- [SEP-1](sep-01.md) - stellar.toml (where DIRECT_PAYMENT_SERVER is published)
