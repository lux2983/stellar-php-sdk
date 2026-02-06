# SEP-01: Stellar Info File (stellar.toml)

The stellar.toml file is a standardized configuration file that organizations host at their domain to publish information about their Stellar integration. It tells wallets and other services how to interact with your organization's accounts, assets, and services.

**When to use:** Anchors, asset issuers, and organizations need to publish a stellar.toml. Wallets and services need to read it to discover endpoints for SEP-6, SEP-10, SEP-24, federation, and other protocols.

See the [SEP-01 specification](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0001.md) for protocol details.

## Quick Example

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

The SDK automatically fetches from `https://DOMAIN/.well-known/stellar.toml`:

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

If you already have the TOML content:

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

## Accessing Data

### General Information

Service endpoints and account information:

```php
<?php

use Soneso\StellarSDK\SEP\Toml\StellarToml;

$stellarToml = StellarToml::fromDomain('testanchor.stellar.org');
$info = $stellarToml->getGeneralInformation();

// Service endpoints
$federationServer = $info->federationServer;      // SEP-02 Federation
$authServer = $info->authServer;                  // SEP-10 Authentication  
$transferServer = $info->transferServer;          // SEP-06 Deposit/Withdrawal
$transferServerSep24 = $info->transferServerSep24; // SEP-24 Interactive
$kycServer = $info->kYCServer;                    // SEP-12 KYC
$webAuthEndpoint = $info->webAuthEndpoint;        // SEP-10 Web Auth
$directPaymentServer = $info->directPaymentServer; // SEP-31 Direct Payments
$anchorQuoteServer = $info->anchorQuoteServer;    // SEP-38 Quotes

// Signing keys
$signingKey = $info->signingKey;                  // For SEP-10 challenges
$uriSigningKey = $info->uriRequestSigningKey;    // For SEP-07 URIs

// Network info
$networkPassphrase = $info->networkPassphrase;
$horizonUrl = $info->horizonUrl;

// Organization accounts
$accounts = $info->accounts; // Array of account IDs
```

### Organization Documentation

Contact and compliance information:

```php
<?php

use Soneso\StellarSDK\SEP\Toml\StellarToml;

$stellarToml = StellarToml::fromDomain('testanchor.stellar.org');
$docs = $stellarToml->getDocumentation();

if ($docs !== null) {
    echo "Name: " . $docs->orgName . PHP_EOL;
    echo "DBA: " . $docs->orgDBA . PHP_EOL;
    echo "URL: " . $docs->orgUrl . PHP_EOL;
    echo "Logo: " . $docs->orgLogo . PHP_EOL;
    echo "Description: " . $docs->orgDescription . PHP_EOL;
    echo "Email: " . $docs->orgOfficialEmail . PHP_EOL;
    echo "Support: " . $docs->orgSupportEmail . PHP_EOL;
    echo "Twitter: " . $docs->orgTwitter . PHP_EOL;
    echo "GitHub: " . $docs->orgGithub . PHP_EOL;
}
```

### Currencies (Assets)

Information about assets issued by the organization:

```php
<?php

use Soneso\StellarSDK\SEP\Toml\StellarToml;

$stellarToml = StellarToml::fromDomain('testanchor.stellar.org');
$currencies = $stellarToml->getCurrencies();

if ($currencies !== null) {
    foreach ($currencies as $currency) {
        echo "Code: " . $currency->code . PHP_EOL;
        echo "Issuer: " . $currency->issuer . PHP_EOL;
        echo "Name: " . $currency->name . PHP_EOL;
        echo "Description: " . $currency->desc . PHP_EOL;
        echo "Decimals: " . $currency->displayDecimals . PHP_EOL;
        echo "Anchored: " . ($currency->isAssetAnchored ? 'Yes' : 'No') . PHP_EOL;
        echo "---" . PHP_EOL;
    }
}
```

### Linked Currencies

Some stellar.toml files link to separate TOML files for currency details:

```php
<?php

use Soneso\StellarSDK\SEP\Toml\StellarToml;

$stellarToml = StellarToml::fromDomain('example.com');
$currencies = $stellarToml->getCurrencies();

foreach ($currencies as $currency) {
    // Check if currency details are in a separate file
    if ($currency->toml !== null) {
        $linkedCurrency = StellarToml::currencyFromUrl($currency->toml);
        echo "Code: " . $linkedCurrency->code . PHP_EOL;
        echo "Issuer: " . $linkedCurrency->issuer . PHP_EOL;
    }
}
```

### Validators

For organizations running Stellar validators:

```php
<?php

use Soneso\StellarSDK\SEP\Toml\StellarToml;

$stellarToml = StellarToml::fromDomain('stellar.org');
$validators = $stellarToml->getValidators();

if ($validators !== null) {
    foreach ($validators as $validator) {
        echo "Alias: " . $validator->alias . PHP_EOL;
        echo "Name: " . $validator->displayName . PHP_EOL;
        echo "Public Key: " . $validator->publicKey . PHP_EOL;
        echo "Host: " . $validator->host . PHP_EOL;
    }
}
```

## Error Handling

```php
<?php

use Exception;
use Soneso\StellarSDK\SEP\Toml\StellarToml;

try {
    $stellarToml = StellarToml::fromDomain('nonexistent-domain.invalid');
} catch (Exception $e) {
    // Domain unreachable or stellar.toml not found
    echo "Failed to load stellar.toml: " . $e->getMessage() . PHP_EOL;
}

// Check for missing data
$stellarToml = StellarToml::fromDomain('example.com');
$info = $stellarToml->getGeneralInformation();

if ($info->webAuthEndpoint === null) {
    echo "This anchor doesn't support SEP-10 authentication" . PHP_EOL;
}

if ($info->transferServerSep24 === null) {
    echo "This anchor doesn't support SEP-24 interactive deposits" . PHP_EOL;
}
```

## Related SEPs

- [SEP-02 Federation](sep-02.md) - Uses `FEDERATION_SERVER` from stellar.toml
- [SEP-10 Authentication](sep-10.md) - Uses `WEB_AUTH_ENDPOINT` and `SIGNING_KEY`
- [SEP-24 Interactive](sep-24.md) - Uses `TRANSFER_SERVER_SEP0024`
- [SEP-07 URI Scheme](sep-07.md) - Uses `URI_REQUEST_SIGNING_KEY`
