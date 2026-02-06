# SEP-08: Regulated Assets

SEP-08 handles assets that require issuer approval for every transaction. These "regulated assets" enable compliance with securities laws, KYC/AML requirements, and jurisdiction restrictions.

**Use SEP-08 when:**
- Transacting with assets marked as regulated in stellar.toml
- Working with securities tokens or compliance-controlled assets
- Building wallets that support regulated asset transfers

**Spec:** [SEP-0008](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0008.md)

## Quick Example

```php
<?php

use Soneso\StellarSDK\SEP\RegulatedAssets\RegulatedAssetsService;
use Soneso\StellarSDK\SEP\RegulatedAssets\SEP08PostTransactionSuccess;
use Soneso\StellarSDK\SEP\RegulatedAssets\SEP08PostTransactionRejected;

// Create service from anchor domain
$service = RegulatedAssetsService::fromDomain("regulated-asset-issuer.com");

// Get regulated assets defined in stellar.toml
$regulatedAssets = $service->regulatedAssets;
echo "Found " . count($regulatedAssets) . " regulated asset(s)" . PHP_EOL;

// Submit a transaction for approval
$signedTxXdr = "AAAAAgAAAA..."; // Your signed transaction as base64 XDR
$response = $service->postTransaction(
    tx: $signedTxXdr,
    approvalServer: $regulatedAssets[0]->approvalServer
);

if ($response instanceof SEP08PostTransactionSuccess) {
    echo "Approved! Submit this transaction: " . $response->tx . PHP_EOL;
} elseif ($response instanceof SEP08PostTransactionRejected) {
    echo "Rejected: " . $response->error . PHP_EOL;
}
```

## How Regulated Assets Work

1. **Issuer flags**: Asset issuer account has `AUTH_REQUIRED` and `AUTH_REVOCABLE` flags set
2. **Approval server**: Each transaction must be submitted to the issuer's approval server
3. **Compliance check**: Server evaluates the transaction against its rules
4. **Signing**: If approved, server signs and returns the transaction
5. **Submission**: Wallet submits the approved transaction to Stellar

The approval server controls who can transact and may require additional KYC information.

## Detailed Usage

### Creating the Service

**From domain (recommended):**

```php
<?php

use Soneso\StellarSDK\SEP\RegulatedAssets\RegulatedAssetsService;

// Loads stellar.toml and extracts regulated assets
$service = RegulatedAssetsService::fromDomain("regulated-asset-issuer.com");
```

**From StellarToml data:**

```php
<?php

use Soneso\StellarSDK\SEP\RegulatedAssets\RegulatedAssetsService;
use Soneso\StellarSDK\SEP\Toml\StellarToml;
use Soneso\StellarSDK\Network;

$toml = StellarToml::fromDomain("regulated-asset-issuer.com");
$service = new RegulatedAssetsService(
    tomlData: $toml,
    network: Network::testnet()
);
```

### Discovering Regulated Assets

The service extracts regulated assets from stellar.toml automatically:

```php
<?php

use Soneso\StellarSDK\SEP\RegulatedAssets\RegulatedAssetsService;

$service = RegulatedAssetsService::fromDomain("regulated-asset-issuer.com");

foreach ($service->regulatedAssets as $asset) {
    echo "Asset: " . $asset->code . PHP_EOL;
    echo "Issuer: " . $asset->issuer . PHP_EOL;
    echo "Approval server: " . $asset->approvalServer . PHP_EOL;
    
    if ($asset->approvalCriteria) {
        echo "Criteria: " . $asset->approvalCriteria . PHP_EOL;
    }
}
```

### Checking Authorization Requirements

Verify an asset has proper issuer flags before transacting:

```php
<?php

use Soneso\StellarSDK\SEP\RegulatedAssets\RegulatedAssetsService;

$service = RegulatedAssetsService::fromDomain("regulated-asset-issuer.com");
$asset = $service->regulatedAssets[0];

// Checks that issuer has AUTH_REQUIRED and AUTH_REVOCABLE flags
$needsApproval = $service->authorizationRequired($asset);

if ($needsApproval) {
    echo "This asset requires approval server for all transactions";
} else {
    echo "Warning: Asset flags not properly configured";
}
```

### Building a Transaction for Approval

Create and sign your transaction, then submit for approval:

```php
<?php

use Soneso\StellarSDK\Asset;
use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\PaymentOperationBuilder;
use Soneso\StellarSDK\SEP\RegulatedAssets\RegulatedAssetsService;
use Soneso\StellarSDK\StellarSDK;
use Soneso\StellarSDK\TransactionBuilder;

$sdk = StellarSDK::getTestNetInstance();
$service = RegulatedAssetsService::fromDomain("regulated-asset-issuer.com");
$regulatedAsset = $service->regulatedAssets[0];

// Sender's keypair
$senderKeyPair = KeyPair::fromSeed("SCZANGBA5YHTNYVVV3C7CAZMTQDBJHJG...");
$senderAccount = $sdk->requestAccount($senderKeyPair->getAccountId());

// Build the payment transaction
$asset = Asset::createNonNativeAsset($regulatedAsset->code, $regulatedAsset->issuer);

$payment = (new PaymentOperationBuilder(
    destination: "GDEST...",
    asset: $asset,
    amount: "100"
))->build();

$transaction = (new TransactionBuilder($senderAccount))
    ->addOperation($payment)
    ->build();

// Sign with sender's key
$transaction->sign($senderKeyPair, Network::testnet());

// Submit to approval server
$txXdr = $transaction->toEnvelopeXdrBase64();
$response = $service->postTransaction(
    tx: $txXdr,
    approvalServer: $regulatedAsset->approvalServer
);
```

### Handling Approval Responses

The approval server returns one of five response types:

```php
<?php

use Soneso\StellarSDK\SEP\RegulatedAssets\RegulatedAssetsService;
use Soneso\StellarSDK\SEP\RegulatedAssets\SEP08PostTransactionSuccess;
use Soneso\StellarSDK\SEP\RegulatedAssets\SEP08PostTransactionRevised;
use Soneso\StellarSDK\SEP\RegulatedAssets\SEP08PostTransactionPending;
use Soneso\StellarSDK\SEP\RegulatedAssets\SEP08PostTransactionActionRequired;
use Soneso\StellarSDK\SEP\RegulatedAssets\SEP08PostTransactionRejected;
use Soneso\StellarSDK\StellarSDK;

$service = RegulatedAssetsService::fromDomain("regulated-asset-issuer.com");
$response = $service->postTransaction($txXdr, $approvalServer);

if ($response instanceof SEP08PostTransactionSuccess) {
    // Transaction approved and signed - submit to network
    echo "Approved! Message: " . $response->message . PHP_EOL;
    $sdk = StellarSDK::getTestNetInstance();
    $result = $sdk->submitTransactionEnvelopeXdrBase64($response->tx);
    
} elseif ($response instanceof SEP08PostTransactionRevised) {
    // Transaction was modified to be compliant - review and submit
    echo "Revised for compliance: " . $response->message . PHP_EOL;
    // Review $response->tx to see what changed before submitting
    
} elseif ($response instanceof SEP08PostTransactionPending) {
    // Approval pending - check back later
    echo "Pending. Check again in " . $response->timeout . " seconds" . PHP_EOL;
    echo "Message: " . $response->message . PHP_EOL;
    
} elseif ($response instanceof SEP08PostTransactionActionRequired) {
    // User action needed - see next section
    echo "Action required: " . $response->message . PHP_EOL;
    echo "Action URL: " . $response->actionUrl . PHP_EOL;
    
} elseif ($response instanceof SEP08PostTransactionRejected) {
    // Transaction rejected
    echo "Rejected: " . $response->error . PHP_EOL;
}
```

### Handling Action Required

When the server needs additional information:

```php
<?php

use Soneso\StellarSDK\SEP\RegulatedAssets\RegulatedAssetsService;
use Soneso\StellarSDK\SEP\RegulatedAssets\SEP08PostTransactionActionRequired;
use Soneso\StellarSDK\SEP\RegulatedAssets\SEP08PostActionDone;
use Soneso\StellarSDK\SEP\RegulatedAssets\SEP08PostActionNextUrl;

$service = RegulatedAssetsService::fromDomain("regulated-asset-issuer.com");
$response = $service->postTransaction($txXdr, $approvalServer);

if ($response instanceof SEP08PostTransactionActionRequired) {
    echo "Action needed: " . $response->message . PHP_EOL;
    
    // Check what fields are required
    if ($response->actionFields) {
        echo "Required fields:" . PHP_EOL;
        foreach ($response->actionFields as $field) {
            echo "  - $field" . PHP_EOL;
        }
    }
    
    // If action_method is POST, submit fields programmatically
    if ($response->actionMethod === "POST") {
        $actionResponse = $service->postAction(
            url: $response->actionUrl,
            actionFields: [
                "email_address" => "user@example.com",
                "mobile_number" => "+1234567890"
            ]
        );
        
        if ($actionResponse instanceof SEP08PostActionDone) {
            // Resubmit the original transaction
            echo "Action complete. Resubmitting transaction..." . PHP_EOL;
            $response = $service->postTransaction($txXdr, $approvalServer);
            
        } elseif ($actionResponse instanceof SEP08PostActionNextUrl) {
            // More steps needed - open URL in browser
            echo "Open this URL: " . $actionResponse->nextUrl . PHP_EOL;
        }
    } else {
        // action_method is GET - open URL in browser for user
        echo "Open browser: " . $response->actionUrl . PHP_EOL;
    }
}
```

## Complete Workflow Example

Full example showing the approval flow for a regulated asset transfer:

```php
<?php

use Soneso\StellarSDK\Asset;
use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\PaymentOperationBuilder;
use Soneso\StellarSDK\SEP\RegulatedAssets\RegulatedAssetsService;
use Soneso\StellarSDK\SEP\RegulatedAssets\SEP08PostTransactionSuccess;
use Soneso\StellarSDK\SEP\RegulatedAssets\SEP08PostTransactionRevised;
use Soneso\StellarSDK\SEP\RegulatedAssets\SEP08PostTransactionPending;
use Soneso\StellarSDK\SEP\RegulatedAssets\SEP08PostTransactionActionRequired;
use Soneso\StellarSDK\SEP\RegulatedAssets\SEP08PostTransactionRejected;
use Soneso\StellarSDK\StellarSDK;
use Soneso\StellarSDK\TransactionBuilder;

// Setup
$sdk = StellarSDK::getTestNetInstance();
$service = RegulatedAssetsService::fromDomain("regulated-asset-issuer.com");
$regulatedAsset = $service->regulatedAssets[0];

$senderKeyPair = KeyPair::fromSeed("SCZANGBA5YHTNYVVV3C7CAZMTQDBJHJG...");
$recipientId = "GDESTINATION...";

// Verify asset requires approval
if (!$service->authorizationRequired($regulatedAsset)) {
    throw new Exception("Asset not properly configured for regulation");
}

// Build transaction
$senderAccount = $sdk->requestAccount($senderKeyPair->getAccountId());
$asset = Asset::createNonNativeAsset($regulatedAsset->code, $regulatedAsset->issuer);

$transaction = (new TransactionBuilder($senderAccount))
    ->addOperation(
        (new PaymentOperationBuilder($recipientId, $asset, "100"))->build()
    )
    ->build();

$transaction->sign($senderKeyPair, Network::testnet());
$txXdr = $transaction->toEnvelopeXdrBase64();

// Submit for approval
$response = $service->postTransaction($txXdr, $regulatedAsset->approvalServer);

// Handle response
$approvedTx = null;

if ($response instanceof SEP08PostTransactionSuccess) {
    $approvedTx = $response->tx;
} elseif ($response instanceof SEP08PostTransactionRevised) {
    // Review what changed before accepting
    echo "Transaction revised: " . $response->message . PHP_EOL;
    $approvedTx = $response->tx;
} elseif ($response instanceof SEP08PostTransactionPending) {
    echo "Try again in " . $response->timeout . " seconds" . PHP_EOL;
} elseif ($response instanceof SEP08PostTransactionActionRequired) {
    echo "User action needed at: " . $response->actionUrl . PHP_EOL;
} elseif ($response instanceof SEP08PostTransactionRejected) {
    throw new Exception("Transaction rejected: " . $response->error);
}

// Submit approved transaction to network
if ($approvedTx) {
    $result = $sdk->submitTransactionEnvelopeXdrBase64($approvedTx);
    echo "Transaction submitted: " . $result->getHash() . PHP_EOL;
}
```

## Error Handling

```php
<?php

use Soneso\StellarSDK\SEP\RegulatedAssets\RegulatedAssetsService;
use Soneso\StellarSDK\SEP\RegulatedAssets\SEP08IncompleteInitData;
use Soneso\StellarSDK\SEP\RegulatedAssets\SEP08InvalidPostTransactionResponse;
use Soneso\StellarSDK\SEP\RegulatedAssets\SEP08InvalidPostActionResponse;
use Soneso\StellarSDK\Exceptions\HorizonRequestException;
use GuzzleHttp\Exception\GuzzleException;

try {
    $service = RegulatedAssetsService::fromDomain("regulated-asset-issuer.com");
    $response = $service->postTransaction($txXdr, $approvalServer);
    
} catch (SEP08IncompleteInitData $e) {
    // Missing network passphrase or horizon URL in stellar.toml
    echo "stellar.toml incomplete: " . $e->getMessage();
    
} catch (SEP08InvalidPostTransactionResponse $e) {
    // Approval server returned unexpected response
    echo "Invalid response from approval server: " . $e->getMessage();
    echo "HTTP status: " . $e->getCode();
    
} catch (SEP08InvalidPostActionResponse $e) {
    // Action endpoint returned unexpected response
    echo "Invalid action response: " . $e->getMessage();
    
} catch (HorizonRequestException $e) {
    // Failed to check issuer flags
    echo "Horizon error: " . $e->getMessage();
    
} catch (GuzzleException $e) {
    // Network error
    echo "Request failed: " . $e->getMessage();
}
```

### Response Types

| Response Class | Status | Meaning |
|---------------|--------|---------|
| `SEP08PostTransactionSuccess` | `success` | Approved and signed - submit it |
| `SEP08PostTransactionRevised` | `revised` | Modified for compliance - review before submitting |
| `SEP08PostTransactionPending` | `pending` | Check back after `timeout` seconds |
| `SEP08PostTransactionActionRequired` | `action_required` | User must complete action at URL |
| `SEP08PostTransactionRejected` | `rejected` | Denied - see error message |

## Related SEPs

- [SEP-01](sep-01.md) - stellar.toml (defines regulated assets)
- [SEP-09](sep-09.md) - Standard KYC fields (used in action_required flows)
- [SEP-10](sep-10.md) - Web authentication (may be required by approval server)
