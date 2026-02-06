# SEP-01: Stellar Info File (stellar.toml)

The stellar.toml file is a standardized configuration file that anchors and organizations host at their domains. It tells wallets and other services how to interact with their accounts, assets, and services. The SDK fetches and parses these files so your application can discover anchor endpoints.

**When to use:** Use this when your application needs to discover an anchor's service endpoints (SEP-6, SEP-10, SEP-24, federation, etc.) by fetching their stellar.toml file.

See the [SEP-01 specification](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0001.md) for protocol details.

## Quick Example

Load a stellar.toml file from a domain and access service endpoints:

```php
<?php

use Soneso\StellarSDK\SEP\Toml\StellarToml;

// Load stellar.toml from a domain
$stellarToml = StellarToml::fromDomain('testanchor.stellar.org');

// Get service endpoints
$info = $stellarToml->getGeneralInformation();
echo "Transfer Server: " . $info->transferServerSep24 . PHP_EOL;
echo "Web Auth: " . $info->webAuthEndpoint . PHP_EOL;
```

## Loading stellar.toml

### From a Domain

The `fromDomain()` method automatically fetches from `https://DOMAIN/.well-known/stellar.toml` and parses the content:

```php
<?php

use Soneso\StellarSDK\SEP\Toml\StellarToml;

$stellarToml = StellarToml::fromDomain('soneso.com');

// Access organization info
$docs = $stellarToml->getDocumentation();
if ($docs !== null) {
    echo "Organization: " . $docs->orgName . PHP_EOL;
    echo "Support: " . $docs->orgSupportEmail . PHP_EOL;
}
```

### From a String

If you already have the TOML content (e.g., from a cache or custom source), you can parse it directly using the constructor:

```php
<?php

use Soneso\StellarSDK\SEP\Toml\StellarToml;

$tomlContent = '
VERSION="2.0.0"
NETWORK_PASSPHRASE="Test SDF Network ; September 2015"
FEDERATION_SERVER="https://example.com/federation"
AUTH_SERVER="https://example.com/auth"
TRANSFER_SERVER_SEP0024="https://example.com/sep24"

[DOCUMENTATION]
ORG_NAME="Example Anchor"
ORG_URL="https://example.com"
';

$stellarToml = new StellarToml($tomlContent);
$info = $stellarToml->getGeneralInformation();
echo "Version: " . $info->version . PHP_EOL;
```

### With Custom HTTP Client

For testing or custom HTTP configurations (proxies, timeouts, etc.), you can pass your own Guzzle HTTP client:

```php
<?php

use GuzzleHttp\Client;
use Soneso\StellarSDK\SEP\Toml\StellarToml;

// Create a custom HTTP client with specific settings
$httpClient = new Client([
    'timeout' => 10,
    'verify' => true,
]);

$stellarToml = StellarToml::fromDomain('testanchor.stellar.org', $httpClient);
```

## Accessing Data

### General Information

The `getGeneralInformation()` method returns service endpoints, signing keys, and network configuration:

```php
<?php

use Soneso\StellarSDK\SEP\Toml\StellarToml;

$stellarToml = StellarToml::fromDomain('testanchor.stellar.org');
$info = $stellarToml->getGeneralInformation();

// Service endpoints
$federationServer = $info->federationServer;       // SEP-02 Federation
$authServer = $info->authServer;                   // SEP-03 Compliance (deprecated)
$transferServer = $info->transferServer;           // SEP-06 Deposit/Withdrawal
$transferServerSep24 = $info->transferServerSep24; // SEP-24 Interactive
$kycServer = $info->kYCServer;                     // SEP-12 KYC
$webAuthEndpoint = $info->webAuthEndpoint;         // SEP-10 Web Auth
$directPaymentServer = $info->directPaymentServer; // SEP-31 Direct Payments
$anchorQuoteServer = $info->anchorQuoteServer;     // SEP-38 Quotes

// SEP-45 Contract Auth endpoints
$webAuthForContracts = $info->webAuthForContractsEndpoint;
$webAuthContractId = $info->webAuthContractId;

// Signing keys
$signingKey = $info->signingKey;               // For SEP-10 challenges
$uriSigningKey = $info->uriRequestSigningKey;  // For SEP-07 URIs

// Network info
$networkPassphrase = $info->networkPassphrase;
$horizonUrl = $info->horizonUrl;
$version = $info->version;  // SEP-1 version

// Organization accounts - array of Stellar account IDs controlled by this domain
$accounts = $info->accounts;
```

### Organization Documentation

The `getDocumentation()` method returns organization contact info, compliance details, and attestations:

```php
<?php

use Soneso\StellarSDK\SEP\Toml\StellarToml;

$stellarToml = StellarToml::fromDomain('testanchor.stellar.org');
$docs = $stellarToml->getDocumentation();

if ($docs !== null) {
    // Basic organization info
    echo "Name: " . $docs->orgName . PHP_EOL;
    echo "DBA: " . $docs->orgDBA . PHP_EOL;
    echo "URL: " . $docs->orgUrl . PHP_EOL;
    echo "Logo: " . $docs->orgLogo . PHP_EOL;
    echo "Description: " . $docs->orgDescription . PHP_EOL;

    // Contact information
    echo "Email: " . $docs->orgOfficialEmail . PHP_EOL;
    echo "Support: " . $docs->orgSupportEmail . PHP_EOL;
    echo "Phone: " . $docs->orgPhoneNumber . PHP_EOL;

    // Social media
    echo "Twitter: " . $docs->orgTwitter . PHP_EOL;
    echo "GitHub: " . $docs->orgGithub . PHP_EOL;
    echo "Keybase: " . $docs->orgKeybase . PHP_EOL;

    // Physical address with attestation
    echo "Address: " . $docs->orgPhysicalAddress . PHP_EOL;
    echo "Address Proof: " . $docs->orgPhysicalAddressAttestation . PHP_EOL;
    echo "Phone Proof: " . $docs->orgPhoneNumberAttestation . PHP_EOL;

    // Licensing information
    echo "Licensing Authority: " . $docs->orgLicensingAuthority . PHP_EOL;
    echo "License Type: " . $docs->orgLicenseType . PHP_EOL;
    echo "License Number: " . $docs->orgLicenseNumber . PHP_EOL;
}
```

### Principals (Points of Contact)

The `getPrincipals()` method returns the organization's key personnel with their contact information and verification details:

```php
<?php

use Soneso\StellarSDK\SEP\Toml\StellarToml;

$stellarToml = StellarToml::fromDomain('testanchor.stellar.org');
$principals = $stellarToml->getPrincipals();

if ($principals !== null) {
    echo "Number of principals: " . $principals->count() . PHP_EOL;

    foreach ($principals as $person) {
        echo "Name: " . $person->name . PHP_EOL;
        echo "Email: " . $person->email . PHP_EOL;
        echo "Keybase: " . $person->keybase . PHP_EOL;
        echo "Twitter: " . $person->twitter . PHP_EOL;
        echo "Telegram: " . $person->telegram . PHP_EOL;
        echo "GitHub: " . $person->github . PHP_EOL;

        // Verification hashes (SHA-256)
        echo "ID Photo Hash: " . $person->idPhotoHash . PHP_EOL;
        echo "Verification Photo Hash: " . $person->verificationPhotoHash . PHP_EOL;
        echo "---" . PHP_EOL;
    }

    // Convert to array for further processing
    $principalsArray = $principals->toArray();
}
```

### Currencies (Assets)

The `getCurrencies()` method returns information about assets issued by the organization, including Stellar Assets and Soroban token contracts:

```php
<?php

use Soneso\StellarSDK\SEP\Toml\StellarToml;

$stellarToml = StellarToml::fromDomain('testanchor.stellar.org');
$currencies = $stellarToml->getCurrencies();

if ($currencies !== null) {
    echo "Number of currencies: " . $currencies->count() . PHP_EOL;

    foreach ($currencies as $currency) {
        // Basic asset identification
        echo "Code: " . $currency->code . PHP_EOL;
        echo "Issuer: " . $currency->issuer . PHP_EOL;
        echo "Contract: " . $currency->contract . PHP_EOL;  // Soroban contract ID
        echo "Name: " . $currency->name . PHP_EOL;
        echo "Description: " . $currency->desc . PHP_EOL;
        echo "Status: " . $currency->status . PHP_EOL;  // live, dead, test, private
        echo "Decimals: " . $currency->displayDecimals . PHP_EOL;
        echo "Image: " . $currency->image . PHP_EOL;
        echo "Conditions: " . $currency->conditions . PHP_EOL;

        // Code template for pattern matching (e.g., CORN???????? for futures)
        echo "Code Template: " . $currency->codeTemplate . PHP_EOL;

        // Supply information
        echo "Fixed Supply: " . $currency->fixedNumber . PHP_EOL;
        echo "Max Supply: " . $currency->maxNumber . PHP_EOL;
        echo "Unlimited: " . ($currency->isUnlimited ? 'Yes' : 'No') . PHP_EOL;

        // Anchored asset information
        echo "Anchored: " . ($currency->isAssetAnchored ? 'Yes' : 'No') . PHP_EOL;
        echo "Anchor Type: " . $currency->anchorAssetType . PHP_EOL;  // fiat, crypto, stock, etc.
        echo "Anchor Asset: " . $currency->anchorAsset . PHP_EOL;
        echo "Attestation: " . $currency->attestationOfReserve . PHP_EOL;
        echo "Redemption: " . $currency->redemptionInstructions . PHP_EOL;

        // Collateral addresses (for anchored crypto tokens)
        if ($currency->collateralAddresses !== null) {
            foreach ($currency->collateralAddresses as $addr) {
                echo "Collateral Address: " . $addr . PHP_EOL;
            }
        }

        // SEP-08 Regulated assets
        echo "Regulated: " . ($currency->regulated ? 'Yes' : 'No') . PHP_EOL;
        echo "Approval Server: " . $currency->approvalServer . PHP_EOL;
        echo "Approval Criteria: " . $currency->approvalCriteria . PHP_EOL;

        echo "---" . PHP_EOL;
    }

    // Convert to array for further processing
    $currenciesArray = $currencies->toArray();
}
```

### Linked Currencies

Some stellar.toml files link to separate TOML files for currency details using the `toml` field. Use `currencyFromUrl()` to load the full currency data:

```php
<?php

use Soneso\StellarSDK\SEP\Toml\StellarToml;

$stellarToml = StellarToml::fromDomain('example.com');
$currencies = $stellarToml->getCurrencies();

if ($currencies !== null) {
    foreach ($currencies as $currency) {
        // Check if currency details are in a separate file
        if ($currency->toml !== null) {
            $linkedCurrency = StellarToml::currencyFromUrl($currency->toml);
            echo "Code: " . $linkedCurrency->code . PHP_EOL;
            echo "Issuer: " . $linkedCurrency->issuer . PHP_EOL;
            echo "Name: " . $linkedCurrency->name . PHP_EOL;
        }
    }
}
```

### Validators

The `getValidators()` method returns information for organizations running Stellar validator nodes:

```php
<?php

use Soneso\StellarSDK\SEP\Toml\StellarToml;

$stellarToml = StellarToml::fromDomain('stellar.org');
$validators = $stellarToml->getValidators();

if ($validators !== null) {
    echo "Number of validators: " . $validators->count() . PHP_EOL;

    foreach ($validators as $validator) {
        echo "Alias: " . $validator->alias . PHP_EOL;
        echo "Name: " . $validator->displayName . PHP_EOL;
        echo "Public Key: " . $validator->publicKey . PHP_EOL;
        echo "Host: " . $validator->host . PHP_EOL;
        echo "History Archive: " . $validator->history . PHP_EOL;
        echo "---" . PHP_EOL;
    }

    // Convert to array for further processing
    $validatorsArray = $validators->toArray();
}
```

## Error Handling

The SDK throws exceptions when the stellar.toml file cannot be fetched or parsed. Always wrap calls in try-catch blocks and validate that optional endpoints exist before using them:

```php
<?php

use Exception;
use Soneso\StellarSDK\SEP\Toml\StellarToml;

// Handle network and parsing errors
try {
    $stellarToml = StellarToml::fromDomain('nonexistent-domain.invalid');
} catch (Exception $e) {
    // Domain unreachable, stellar.toml not found, or TOML parsing failed
    echo "Failed to load stellar.toml: " . $e->getMessage() . PHP_EOL;
}

// Handle errors when loading linked currency files
try {
    $currency = StellarToml::currencyFromUrl('https://example.com/INVALID.toml');
} catch (Exception $e) {
    echo "Failed to load currency TOML: " . $e->getMessage() . PHP_EOL;
}
```

Always check for null values before using optional service endpoints to determine which SEPs an anchor supports:

```php
<?php

use Soneso\StellarSDK\SEP\Toml\StellarToml;

$stellarToml = StellarToml::fromDomain('example.com');
$info = $stellarToml->getGeneralInformation();

// Check which services this anchor supports
if ($info->webAuthEndpoint === null) {
    echo "This anchor doesn't support SEP-10 authentication" . PHP_EOL;
}

if ($info->transferServerSep24 === null) {
    echo "This anchor doesn't support SEP-24 interactive deposits" . PHP_EOL;
}

if ($info->kYCServer === null) {
    echo "This anchor doesn't support SEP-12 KYC" . PHP_EOL;
}

if ($info->directPaymentServer === null) {
    echo "This anchor doesn't support SEP-31 cross-border payments" . PHP_EOL;
}

if ($info->anchorQuoteServer === null) {
    echo "This anchor doesn't support SEP-38 quotes" . PHP_EOL;
}

// Check for SEP-45 contract authentication support
if ($info->webAuthForContractsEndpoint !== null && $info->webAuthContractId !== null) {
    echo "This anchor supports SEP-45 contract authentication" . PHP_EOL;
}
```

## Related SEPs

- [SEP-02 Federation](sep-02.md) - Uses `FEDERATION_SERVER`
- [SEP-06 Deposit/Withdrawal](sep-06.md) - Uses `TRANSFER_SERVER`
- [SEP-07 URI Scheme](sep-07.md) - Uses `URI_REQUEST_SIGNING_KEY`
- [SEP-08 Regulated Assets](sep-08.md) - Uses currency `regulated`, `approval_server`, `approval_criteria`
- [SEP-10 Authentication](sep-10.md) - Uses `WEB_AUTH_ENDPOINT` and `SIGNING_KEY`
- [SEP-12 KYC](sep-12.md) - Uses `KYC_SERVER`
- [SEP-24 Interactive](sep-24.md) - Uses `TRANSFER_SERVER_SEP0024`
- [SEP-31 Cross-Border](sep-31.md) - Uses `DIRECT_PAYMENT_SERVER`
- [SEP-38 Quotes](sep-38.md) - Uses `ANCHOR_QUOTE_SERVER`
- [SEP-41 Token Interface](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0041.md) - Currency `contract` field for Soroban tokens
- [SEP-45 Contract Auth](sep-45.md) - Uses `WEB_AUTH_FOR_CONTRACTS_ENDPOINT` and `WEB_AUTH_CONTRACT_ID`
