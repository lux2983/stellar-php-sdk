# SEP-12: KYC API

The SEP-12 protocol defines how to submit and manage customer information for Know Your Customer (KYC) requirements. Anchors use this to collect identity documents, personal information, and verification data before processing deposits, withdrawals, or payments.

Use SEP-12 when:
- An anchor requires identity verification before deposit/withdrawal
- You need to check what KYC information an anchor requires
- You want to update previously submitted customer information
- You need to verify contact information (phone, email)

See the [SEP-12 specification](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0012.md) for protocol details.

## Quick Example

```php
<?php

use Soneso\StellarSDK\SEP\KYCService\KYCService;
use Soneso\StellarSDK\SEP\KYCService\GetCustomerInfoRequest;
use Soneso\StellarSDK\SEP\KYCService\PutCustomerInfoRequest;
use Soneso\StellarSDK\SEP\StandardKYCFields\StandardKYCFields;
use Soneso\StellarSDK\SEP\StandardKYCFields\NaturalPersonKYCFields;

// Create service from anchor's domain
$kycService = KYCService::fromDomain("testanchor.stellar.org");

// Check what info the anchor needs (using SEP-10 JWT token)
$request = new GetCustomerInfoRequest();
$request->jwt = $jwtToken;
$response = $kycService->getCustomerInfo($request);

echo "Status: " . $response->getStatus() . "\n";

// Submit customer information
$personFields = new NaturalPersonKYCFields();
$personFields->firstName = "Jane";
$personFields->lastName = "Doe";
$personFields->emailAddress = "jane@example.com";

$kycFields = new StandardKYCFields();
$kycFields->naturalPersonKYCFields = $personFields;

$putRequest = new PutCustomerInfoRequest();
$putRequest->jwt = $jwtToken;
$putRequest->KYCFields = $kycFields;

$putResponse = $kycService->putCustomerInfo($putRequest);
$customerId = $putResponse->getId();
```

## Creating the KYC Service

**From an anchor's domain** (recommended):

```php
<?php

use Soneso\StellarSDK\SEP\KYCService\KYCService;

// Loads service URL from stellar.toml automatically
$kycService = KYCService::fromDomain("testanchor.stellar.org");
```

**From a direct URL**:

```php
<?php

use Soneso\StellarSDK\SEP\KYCService\KYCService;

$kycService = new KYCService("https://api.anchor.com/kyc");
```

## Checking Customer Status

Before submitting data, check what fields the anchor requires:

```php
<?php

use Soneso\StellarSDK\SEP\KYCService\KYCService;
use Soneso\StellarSDK\SEP\KYCService\GetCustomerInfoRequest;

$kycService = KYCService::fromDomain("testanchor.stellar.org");

$request = new GetCustomerInfoRequest();
$request->jwt = $jwtToken; // From SEP-10 authentication

// For existing customers, include their ID
$request->id = $customerId;

// Specify the type of operation they're doing
$request->type = "sep31-sender"; // or "sep6-deposit", etc.

$response = $kycService->getCustomerInfo($request);

// Check status
$status = $response->getStatus();
// Possible values: ACCEPTED, PROCESSING, NEEDS_INFO, REJECTED

// See what fields are still needed
$fieldsNeeded = $response->getFields();
if ($fieldsNeeded !== null) {
    foreach ($fieldsNeeded as $fieldName => $field) {
        echo "Field: $fieldName\n";
        echo "  Description: " . $field->getDescription() . "\n";
        echo "  Required: " . ($field->isOptional() ? "No" : "Yes") . "\n";
    }
}

// See what was already provided
$fieldsProvided = $response->getProvidedFields();
```

## Submitting Customer Information

### Personal Information

```php
<?php

use Soneso\StellarSDK\SEP\KYCService\KYCService;
use Soneso\StellarSDK\SEP\KYCService\PutCustomerInfoRequest;
use Soneso\StellarSDK\SEP\StandardKYCFields\StandardKYCFields;
use Soneso\StellarSDK\SEP\StandardKYCFields\NaturalPersonKYCFields;

$kycService = KYCService::fromDomain("testanchor.stellar.org");

$personFields = new NaturalPersonKYCFields();
$personFields->firstName = "Jane";
$personFields->lastName = "Doe";
$personFields->emailAddress = "jane@example.com";
$personFields->mobileNumber = "+14155551234";
$personFields->birthDate = "1990-05-15";
$personFields->address = "123 Main St, San Francisco, CA 94102";
$personFields->addressCountryCode = "USA";

$kycFields = new StandardKYCFields();
$kycFields->naturalPersonKYCFields = $personFields;

$request = new PutCustomerInfoRequest();
$request->jwt = $jwtToken;
$request->KYCFields = $kycFields;
$request->type = "sep31-sender";

$response = $kycService->putCustomerInfo($request);
$customerId = $response->getId(); // Save this for future requests
```

### Uploading ID Documents

```php
<?php

use Soneso\StellarSDK\SEP\KYCService\KYCService;
use Soneso\StellarSDK\SEP\KYCService\PutCustomerInfoRequest;
use Soneso\StellarSDK\SEP\StandardKYCFields\StandardKYCFields;
use Soneso\StellarSDK\SEP\StandardKYCFields\NaturalPersonKYCFields;

$kycService = KYCService::fromDomain("testanchor.stellar.org");

// Load ID document images as binary data
$idFrontBytes = file_get_contents('/path/to/id_front.jpg');
$idBackBytes = file_get_contents('/path/to/id_back.jpg');

$personFields = new NaturalPersonKYCFields();
$personFields->idType = "passport";
$personFields->idNumber = "AB123456";
$personFields->idCountryCode = "USA";
$personFields->photoIdFront = $idFrontBytes;
$personFields->photoIdBack = $idBackBytes;

$kycFields = new StandardKYCFields();
$kycFields->naturalPersonKYCFields = $personFields;

$request = new PutCustomerInfoRequest();
$request->jwt = $jwtToken;
$request->id = $customerId; // Update existing customer
$request->KYCFields = $kycFields;

$response = $kycService->putCustomerInfo($request);
```

### Organization KYC

```php
<?php

use Soneso\StellarSDK\SEP\KYCService\KYCService;
use Soneso\StellarSDK\SEP\KYCService\PutCustomerInfoRequest;
use Soneso\StellarSDK\SEP\StandardKYCFields\StandardKYCFields;
use Soneso\StellarSDK\SEP\StandardKYCFields\OrganizationKYCFields;

$kycService = KYCService::fromDomain("testanchor.stellar.org");

$orgFields = new OrganizationKYCFields();
$orgFields->name = "Acme Corporation";
$orgFields->registeredAddress = "456 Business Ave, New York, NY 10001";
$orgFields->addressCountryCode = "USA";
$orgFields->registrationNumber = "12345678";

$kycFields = new StandardKYCFields();
$kycFields->organizationKYCFields = $orgFields;

$request = new PutCustomerInfoRequest();
$request->jwt = $jwtToken;
$request->KYCFields = $kycFields;
$request->type = "sep31-sender";

$response = $kycService->putCustomerInfo($request);
```

## Verifying Contact Information

Some anchors send verification codes to phone or email. Submit the code to verify:

```php
<?php

use Soneso\StellarSDK\SEP\KYCService\KYCService;
use Soneso\StellarSDK\SEP\KYCService\PutCustomerVerificationRequest;

$kycService = KYCService::fromDomain("testanchor.stellar.org");

$request = new PutCustomerVerificationRequest();
$request->jwt = $jwtToken;
$request->id = $customerId;
$request->verificationFields = [
    "mobile_number_verification" => "123456", // Code sent via SMS
];

$response = $kycService->putCustomerVerification($request);
echo "Status after verification: " . $response->getStatus() . "\n";
```

## Callback Notifications

Register a URL to receive status change notifications:

```php
<?php

use Soneso\StellarSDK\SEP\KYCService\KYCService;
use Soneso\StellarSDK\SEP\KYCService\PutCustomerCallbackRequest;

$kycService = KYCService::fromDomain("testanchor.stellar.org");

$request = new PutCustomerCallbackRequest();
$request->jwt = $jwtToken;
$request->id = $customerId;
$request->url = "https://myapp.com/kyc-callback";

$response = $kycService->putCustomerCallback($request);
// The anchor will POST to your URL when status changes
```

## Deleting Customer Data

Request deletion of all stored customer data (for GDPR compliance, etc.):

```php
<?php

use Soneso\StellarSDK\SEP\KYCService\KYCService;

$kycService = KYCService::fromDomain("testanchor.stellar.org");

$accountId = "GXXXXXX..."; // Stellar account ID
$response = $kycService->deleteCustomer($accountId, $jwtToken);

if ($response->getStatusCode() === 200) {
    echo "Customer data deleted\n";
}
```

## File Upload Endpoint

For complex data structures, upload files separately:

```php
<?php

use Soneso\StellarSDK\SEP\KYCService\KYCService;
use Soneso\StellarSDK\SEP\KYCService\PutCustomerInfoRequest;

$kycService = KYCService::fromDomain("testanchor.stellar.org");

// Upload file first
$fileBytes = file_get_contents('/path/to/document.pdf');
$fileResponse = $kycService->postCustomerFile($fileBytes, $jwtToken);
$fileId = $fileResponse->getFileId();

// Reference the file in customer data
$request = new PutCustomerInfoRequest();
$request->jwt = $jwtToken;
$request->id = $customerId;
$request->customFields = [
    "photo_id_front_file_id" => $fileId,
];

$response = $kycService->putCustomerInfo($request);
```

## Error Handling

```php
<?php

use Soneso\StellarSDK\SEP\KYCService\KYCService;
use Soneso\StellarSDK\SEP\KYCService\GetCustomerInfoRequest;
use GuzzleHttp\Exception\GuzzleException;

$kycService = KYCService::fromDomain("testanchor.stellar.org");

try {
    $request = new GetCustomerInfoRequest();
    $request->jwt = $jwtToken;
    $response = $kycService->getCustomerInfo($request);
    
    switch ($response->getStatus()) {
        case "ACCEPTED":
            // Customer is verified, proceed with transaction
            break;
        case "PROCESSING":
            // Anchor is reviewing, check back later
            break;
        case "NEEDS_INFO":
            // More information required
            $fields = $response->getFields();
            break;
        case "REJECTED":
            // Verification failed
            $message = $response->getMessage();
            break;
    }
} catch (GuzzleException $e) {
    // HTTP errors (network issues, server errors)
    echo "Request failed: " . $e->getMessage() . "\n";
}
```

## Shared/Omnibus Accounts

When multiple customers share a single Stellar account, use memos to distinguish them:

```php
<?php

use Soneso\StellarSDK\SEP\KYCService\KYCService;
use Soneso\StellarSDK\SEP\KYCService\PutCustomerInfoRequest;
use Soneso\StellarSDK\SEP\StandardKYCFields\StandardKYCFields;
use Soneso\StellarSDK\SEP\StandardKYCFields\NaturalPersonKYCFields;

$kycService = KYCService::fromDomain("testanchor.stellar.org");

$personFields = new NaturalPersonKYCFields();
$personFields->firstName = "Jane";
$personFields->lastName = "Doe";

$kycFields = new StandardKYCFields();
$kycFields->naturalPersonKYCFields = $personFields;

$request = new PutCustomerInfoRequest();
$request->jwt = $jwtToken;
$request->KYCFields = $kycFields;
$request->memo = "12345"; // Unique identifier for this customer
$request->memoType = "id";

$response = $kycService->putCustomerInfo($request);
```

## Related SEPs

- [SEP-10](sep-10.md) - Web Authentication (required for KYC requests)
- [SEP-9](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0009.md) - Standard KYC Fields specification
- [SEP-24](sep-24.md) - Interactive Deposit/Withdrawal (often requires KYC)
- [SEP-31](sep-31.md) - Cross-Border Payments (requires sender/receiver KYC)
