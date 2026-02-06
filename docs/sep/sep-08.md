# SEP-08: Regulated Assets

SEP-08 handles assets that require issuer approval for every transaction. These "regulated assets" enable compliance with securities laws, KYC/AML requirements, and jurisdiction restrictions.

**Use SEP-08 when:**
- Transacting with assets marked as regulated in stellar.toml
- Working with securities tokens or compliance-controlled assets
- Building wallets that support regulated asset transfers

**Spec:** [SEP-0008](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0008.md)

## Quick Example

This example demonstrates the complete flow: creating a service from a domain, discovering regulated assets, and submitting a transaction for approval.

```php
<?php

use Soneso\StellarSDK\SEP\RegulatedAssets\RegulatedAssetsService;
use Soneso\StellarSDK\SEP\RegulatedAssets\SEP08PostTransactionSuccess;
use Soneso\StellarSDK\SEP\RegulatedAssets\SEP08PostTransactionRejected;

// Create service from anchor domain - this loads stellar.toml and extracts regulated assets
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

// Handle the response based on its type
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

The `RegulatedAssetsService` provides two ways to initialize: from a domain name (recommended) or from existing stellar.toml data.

**From domain (recommended):**

This approach automatically fetches and parses the stellar.toml file from the specified domain using SEP-1 discovery.

```php
<?php

use Soneso\StellarSDK\SEP\RegulatedAssets\RegulatedAssetsService;

// Loads stellar.toml and extracts regulated assets automatically
$service = RegulatedAssetsService::fromDomain("regulated-asset-issuer.com");
```

**From StellarToml data:**

Use this approach when you already have the stellar.toml data or need to specify a custom network and Horizon URL.

```php
<?php

use Soneso\StellarSDK\SEP\RegulatedAssets\RegulatedAssetsService;
use Soneso\StellarSDK\SEP\Toml\StellarToml;
use Soneso\StellarSDK\Network;

// Load stellar.toml separately for more control
$toml = StellarToml::fromDomain("regulated-asset-issuer.com");

// Create service with explicit network configuration
$service = new RegulatedAssetsService(
    tomlData: $toml,
    network: Network::testnet()
);
```

**With custom Horizon URL:**

Override the Horizon URL when using a private network or custom infrastructure.

```php
<?php

use Soneso\StellarSDK\SEP\RegulatedAssets\RegulatedAssetsService;
use Soneso\StellarSDK\Network;

$service = RegulatedAssetsService::fromDomain(
    domain: "regulated-asset-issuer.com",
    horizonUrl: "https://my-horizon.example.com",
    network: new Network("Custom Network Passphrase")
);
```

### Discovering Regulated Assets

The service automatically extracts regulated assets from the stellar.toml file. Each `RegulatedAsset` contains the asset code, issuer, approval server URL, and optional approval criteria.

```php
<?php

use Soneso\StellarSDK\SEP\RegulatedAssets\RegulatedAssetsService;

$service = RegulatedAssetsService::fromDomain("regulated-asset-issuer.com");

// Iterate through all regulated assets defined in stellar.toml
foreach ($service->regulatedAssets as $asset) {
    echo "Asset: " . $asset->code . PHP_EOL;
    echo "Issuer: " . $asset->issuer . PHP_EOL;
    echo "Approval server: " . $asset->approvalServer . PHP_EOL;
    
    // approvalCriteria is optional - describes what the issuer checks
    if ($asset->approvalCriteria) {
        echo "Criteria: " . $asset->approvalCriteria . PHP_EOL;
    }
}
```

### Checking Authorization Requirements

Before transacting, verify that the asset issuer has properly configured their account with the required authorization flags (AUTH_REQUIRED and AUTH_REVOCABLE).

```php
<?php

use Soneso\StellarSDK\SEP\RegulatedAssets\RegulatedAssetsService;

$service = RegulatedAssetsService::fromDomain("regulated-asset-issuer.com");
$asset = $service->regulatedAssets[0];

// Checks that issuer has both AUTH_REQUIRED and AUTH_REVOCABLE flags set
$needsApproval = $service->authorizationRequired($asset);

if ($needsApproval) {
    echo "This asset requires approval server for all transactions" . PHP_EOL;
} else {
    echo "Warning: Asset flags not properly configured for regulation" . PHP_EOL;
}
```

### Building a Transaction for Approval

Create and sign your transaction as usual, then submit the base64 XDR to the approval server instead of directly to the Stellar network.

```php
<?php

use Soneso\StellarSDK\Asset;
use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\PaymentOperationBuilder;
use Soneso\StellarSDK\SEP\RegulatedAssets\RegulatedAssetsService;
use Soneso\StellarSDK\StellarSDK;
use Soneso\StellarSDK\TransactionBuilder;

// Initialize SDK and service
$sdk = StellarSDK::getTestNetInstance();
$service = RegulatedAssetsService::fromDomain("regulated-asset-issuer.com");
$regulatedAsset = $service->regulatedAssets[0];

// Sender's keypair - keep your secret key secure!
$senderKeyPair = KeyPair::fromSeed("SCZANGBA5YHTNYVVV3C7CAZMTQDBJHJG...");
$senderAccount = $sdk->requestAccount($senderKeyPair->getAccountId());

// Build the payment transaction using the regulated asset
$asset = Asset::createNonNativeAsset($regulatedAsset->code, $regulatedAsset->issuer);

$payment = (new PaymentOperationBuilder(
    destination: "GDESTINATION...",
    asset: $asset,
    amount: "100"
))->build();

$transaction = (new TransactionBuilder($senderAccount))
    ->addOperation($payment)
    ->build();

// Sign with sender's key before submitting to approval server
$transaction->sign($senderKeyPair, Network::testnet());

// Get the base64-encoded XDR envelope for submission
$txXdr = $transaction->toEnvelopeXdrBase64();

// Submit to approval server (not to Stellar network yet)
$response = $service->postTransaction(
    tx: $txXdr,
    approvalServer: $regulatedAsset->approvalServer
);
```

### Handling Approval Responses

The approval server returns one of five response types. Use `instanceof` checks to determine how to proceed with each response.

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
    // Transaction approved and signed by the issuer - ready to submit to network
    echo "Approved!" . PHP_EOL;
    if ($response->message) {
        echo "Message: " . $response->message . PHP_EOL;
    }
    $sdk = StellarSDK::getTestNetInstance();
    $result = $sdk->submitTransactionEnvelopeXdrBase64($response->tx);
    echo "Submitted: " . $result->getHash() . PHP_EOL;
    
} elseif ($response instanceof SEP08PostTransactionRevised) {
    // Transaction was modified to be compliant - REVIEW CHANGES before submitting
    echo "Revised for compliance: " . $response->message . PHP_EOL;
    // IMPORTANT: Inspect $response->tx to see what changed before submitting
    // The issuer may have added operations, changed amounts, etc.
    
} elseif ($response instanceof SEP08PostTransactionPending) {
    // Approval pending - check back later (timeout is in milliseconds)
    $timeoutSeconds = $response->timeout / 1000;
    echo "Pending. Check again in " . $timeoutSeconds . " seconds" . PHP_EOL;
    if ($response->message) {
        echo "Message: " . $response->message . PHP_EOL;
    }
    
} elseif ($response instanceof SEP08PostTransactionActionRequired) {
    // User action needed - see next section for handling this response
    echo "Action required: " . $response->message . PHP_EOL;
    echo "Action URL: " . $response->actionUrl . PHP_EOL;
    echo "Action method: " . $response->actionMethod . PHP_EOL; // 'GET' or 'POST'
    
} elseif ($response instanceof SEP08PostTransactionRejected) {
    // Transaction rejected - cannot be made compliant
    echo "Rejected: " . $response->error . PHP_EOL;
}
```

### Handling Action Required

When the approval server needs additional information (KYC/AML data), it returns an `action_required` response. The `actionMethod` property indicates how to provide the information.

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
    
    // Check what SEP-9 fields are requested (optional - server may not specify)
    if ($response->actionFields) {
        echo "Requested fields:" . PHP_EOL;
        foreach ($response->actionFields as $field) {
            echo "  - $field" . PHP_EOL;
        }
    }
    
    // Handle based on action method (default is 'GET')
    if ($response->actionMethod === "POST") {
        // Submit fields programmatically if wallet has them
        $actionResponse = $service->postAction(
            url: $response->actionUrl,
            actionFields: [
                "email_address" => "user@example.com",
                "mobile_number" => "+1234567890"
            ]
        );
        
        if ($actionResponse instanceof SEP08PostActionDone) {
            // Server has enough info - resubmit the original transaction
            echo "Action complete. Resubmitting transaction..." . PHP_EOL;
            $response = $service->postTransaction($txXdr, $approvalServer);
            
        } elseif ($actionResponse instanceof SEP08PostActionNextUrl) {
            // More steps needed - user must complete action in browser
            echo "Additional action needed." . PHP_EOL;
            if ($actionResponse->message) {
                echo "Instructions: " . $actionResponse->message . PHP_EOL;
            }
            echo "Open this URL: " . $actionResponse->nextUrl . PHP_EOL;
        }
    } else {
        // Action method is GET (default) - open URL in browser for user
        echo "Open browser: " . $response->actionUrl . PHP_EOL;
        // After user completes action, resubmit the original transaction
    }
}
```

## Complete Workflow Example

This full example demonstrates the entire approval flow for a regulated asset transfer, including authorization checks, transaction building, and response handling.

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

// Setup - initialize SDK and load regulated asset information
$sdk = StellarSDK::getTestNetInstance();
$service = RegulatedAssetsService::fromDomain("regulated-asset-issuer.com");
$regulatedAsset = $service->regulatedAssets[0];

$senderKeyPair = KeyPair::fromSeed("SCZANGBA5YHTNYVVV3C7CAZMTQDBJHJG...");
$recipientId = "GDESTINATION...";

// Step 1: Verify asset requires approval (issuer has proper flags)
if (!$service->authorizationRequired($regulatedAsset)) {
    throw new Exception("Asset not properly configured for regulation");
}

// Step 2: Build and sign the transaction
$senderAccount = $sdk->requestAccount($senderKeyPair->getAccountId());
$asset = Asset::createNonNativeAsset($regulatedAsset->code, $regulatedAsset->issuer);

$transaction = (new TransactionBuilder($senderAccount))
    ->addOperation(
        (new PaymentOperationBuilder($recipientId, $asset, "100"))->build()
    )
    ->build();

$transaction->sign($senderKeyPair, Network::testnet());
$txXdr = $transaction->toEnvelopeXdrBase64();

// Step 3: Submit for approval
$response = $service->postTransaction($txXdr, $regulatedAsset->approvalServer);

// Step 4: Handle response
$approvedTx = null;

if ($response instanceof SEP08PostTransactionSuccess) {
    // Approved without changes
    $approvedTx = $response->tx;
    
} elseif ($response instanceof SEP08PostTransactionRevised) {
    // Transaction was modified - review changes before accepting
    echo "Transaction revised: " . $response->message . PHP_EOL;
    // IMPORTANT: Inspect the revised transaction before submitting
    $approvedTx = $response->tx;
    
} elseif ($response instanceof SEP08PostTransactionPending) {
    // Wait and retry (timeout is in milliseconds)
    $waitSeconds = max(5, $response->timeout / 1000);
    echo "Try again in " . $waitSeconds . " seconds" . PHP_EOL;
    
} elseif ($response instanceof SEP08PostTransactionActionRequired) {
    // User must complete action at the provided URL
    echo "User action needed at: " . $response->actionUrl . PHP_EOL;
    
} elseif ($response instanceof SEP08PostTransactionRejected) {
    throw new Exception("Transaction rejected: " . $response->error);
}

// Step 5: Submit approved transaction to Stellar network
if ($approvedTx) {
    $result = $sdk->submitTransactionEnvelopeXdrBase64($approvedTx);
    echo "Transaction submitted: " . $result->getHash() . PHP_EOL;
}
```

## Error Handling

The SDK throws specific exceptions for different error conditions. Handle each type appropriately to provide meaningful feedback to users.

```php
<?php

use Soneso\StellarSDK\SEP\RegulatedAssets\RegulatedAssetsService;
use Soneso\StellarSDK\SEP\RegulatedAssets\SEP08IncompleteInitData;
use Soneso\StellarSDK\SEP\RegulatedAssets\SEP08InvalidPostTransactionResponse;
use Soneso\StellarSDK\SEP\RegulatedAssets\SEP08InvalidPostActionResponse;
use Soneso\StellarSDK\Exceptions\HorizonRequestException;
use GuzzleHttp\Exception\GuzzleException;
use Exception;

try {
    // Initialize service - may fail if stellar.toml is incomplete
    $service = RegulatedAssetsService::fromDomain("regulated-asset-issuer.com");
    
    // Check authorization flags - may fail if Horizon is unreachable
    $needsApproval = $service->authorizationRequired($service->regulatedAssets[0]);
    
    // Submit transaction - may fail if approval server returns invalid response
    $response = $service->postTransaction($txXdr, $approvalServer);
    
} catch (SEP08IncompleteInitData $e) {
    // Missing NETWORK_PASSPHRASE or HORIZON_URL in stellar.toml
    // and values could not be inferred from known networks
    echo "stellar.toml configuration incomplete: " . $e->getMessage() . PHP_EOL;
    echo "Ensure NETWORK_PASSPHRASE and HORIZON_URL are set, or provide them explicitly." . PHP_EOL;
    
} catch (SEP08InvalidPostTransactionResponse $e) {
    // Approval server returned malformed response (missing fields, unknown status, etc.)
    echo "Invalid response from approval server: " . $e->getMessage() . PHP_EOL;
    echo "HTTP status: " . $e->getCode() . PHP_EOL;
    // Consider retry with exponential backoff for transient errors
    
} catch (SEP08InvalidPostActionResponse $e) {
    // Action endpoint returned malformed response
    echo "Invalid action response: " . $e->getMessage() . PHP_EOL;
    echo "HTTP status: " . $e->getCode() . PHP_EOL;
    // Fall back to opening action_url in browser
    
} catch (HorizonRequestException $e) {
    // Failed to check issuer flags or submit transaction
    echo "Horizon error: " . $e->getMessage() . PHP_EOL;
    // Check if Horizon is reachable and issuer account exists
    
} catch (GuzzleException $e) {
    // Network error (connection timeout, DNS failure, etc.)
    echo "Network request failed: " . $e->getMessage() . PHP_EOL;
    // Implement retry logic with backoff
    
} catch (Exception $e) {
    // Catch-all for unexpected errors
    echo "Unexpected error: " . $e->getMessage() . PHP_EOL;
}
```

### Response Types

| Response Class | Status | Meaning |
|---------------|--------|---------|
| `SEP08PostTransactionSuccess` | `success` | Approved and signed - submit it |
| `SEP08PostTransactionRevised` | `revised` | Modified for compliance - review before submitting |
| `SEP08PostTransactionPending` | `pending` | Check back after `timeout` milliseconds |
| `SEP08PostTransactionActionRequired` | `action_required` | User must complete action at URL |
| `SEP08PostTransactionRejected` | `rejected` | Denied - see error message |

### Action Response Types

| Response Class | Result | Meaning |
|---------------|--------|---------|
| `SEP08PostActionDone` | `no_further_action_required` | Resubmit the original transaction |
| `SEP08PostActionNextUrl` | `follow_next_url` | Open `nextUrl` in browser for user |

### Exception Types

| Exception Class | Cause |
|----------------|-------|
| `SEP08IncompleteInitData` | Missing network passphrase or Horizon URL |
| `SEP08InvalidPostTransactionResponse` | Malformed approval server response |
| `SEP08InvalidPostActionResponse` | Malformed action endpoint response |

## Related SEPs

- [SEP-01](sep-01.md) - stellar.toml (defines regulated assets)
- [SEP-09](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0009.md) - Standard KYC fields (used in action_required flows)
- [SEP-10](sep-10.md) - Web authentication (may be required by approval server)
