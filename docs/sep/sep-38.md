# SEP-38: Anchor RFQ API

Get exchange quotes between Stellar assets and off-chain assets (like fiat currencies).

## Overview

SEP-38 lets anchors provide price quotes for asset exchanges. Use it when you need to:

- Show users estimated conversion rates before a deposit or withdrawal
- Lock in a firm exchange rate for a transaction
- Get available trading pairs from an anchor

Quotes come in two types:
- **Indicative quotes**: Estimated prices that may change
- **Firm quotes**: Locked prices valid for a limited time

SEP-38 is used alongside SEP-6, SEP-24, or SEP-31 for the actual asset transfer.

## Quick Example

```php
<?php

use Soneso\StellarSDK\SEP\Quote\QuoteService;

// Connect to anchor's quote service
$quoteService = QuoteService::fromDomain("anchor.example.com");

// Get available assets
$info = $quoteService->info();
foreach ($info->assets as $asset) {
    echo $asset->asset . "\n";
}

// Get indicative price for selling USD
$prices = $quoteService->prices(
    sellAsset: "iso4217:USD",
    sellAmount: "100"
);

foreach ($prices->buyAssets as $buyAsset) {
    echo "Buy " . $buyAsset->asset . " at price " . $buyAsset->price . "\n";
}
```

## Detailed Usage

### Creating the Service

From stellar.toml (recommended):

```php
<?php

use Soneso\StellarSDK\SEP\Quote\QuoteService;

$quoteService = QuoteService::fromDomain("anchor.example.com");
```

Or with a direct URL:

```php
<?php

use Soneso\StellarSDK\SEP\Quote\QuoteService;

$quoteService = new QuoteService("https://anchor.example.com/sep38");
```

### Asset Identification Format

SEP-38 uses a specific format for identifying assets:

| Type | Format | Example |
|------|--------|---------|
| Stellar asset | `stellar:CODE:ISSUER` | `stellar:USDC:GA5ZSEJYB37JRC5AVCIA5MOP4RHTM335X2KGX3IHOJAPP5RE34K4KZVN` |
| Fiat currency | `iso4217:CODE` | `iso4217:USD` |

### Getting Available Assets

```php
<?php

use Soneso\StellarSDK\SEP\Quote\QuoteService;

$quoteService = QuoteService::fromDomain("anchor.example.com");

// Authentication is optional (depends on anchor)
$jwtToken = null; // Or obtain via SEP-10

$info = $quoteService->info($jwtToken);

foreach ($info->assets as $asset) {
    echo "Asset: " . $asset->asset . "\n";
    
    // Check delivery methods if available
    if ($asset->sellDeliveryMethods !== null) {
        foreach ($asset->sellDeliveryMethods as $method) {
            echo "  Sell via: " . $method->name . "\n";
        }
    }
}
```

### Getting Indicative Prices

Get prices for selling a specific amount of an asset:

```php
<?php

use Soneso\StellarSDK\SEP\Quote\QuoteService;

$quoteService = QuoteService::fromDomain("anchor.example.com");

// What can I buy for 100 USD?
$prices = $quoteService->prices(
    sellAsset: "iso4217:USD",
    sellAmount: "100",
    jwt: $jwtToken
);

foreach ($prices->buyAssets as $buyAsset) {
    echo "Asset: " . $buyAsset->asset . "\n";
    echo "Price: " . $buyAsset->price . "\n";
    echo "Decimals: " . $buyAsset->decimals . "\n";
}
```

### Getting a Price for a Specific Pair

```php
<?php

use Soneso\StellarSDK\SEP\Quote\QuoteService;

$quoteService = QuoteService::fromDomain("anchor.example.com");

// Price for USD -> USDC exchange in SEP-24 context
$price = $quoteService->price(
    context: "sep24",
    sellAsset: "iso4217:USD",
    buyAsset: "stellar:USDC:GA5ZSEJYB37JRC5AVCIA5MOP4RHTM335X2KGX3IHOJAPP5RE34K4KZVN",
    sellAmount: "100",
    jwt: $jwtToken
);

echo "Price: " . $price->price . "\n";
echo "Sell amount: " . $price->sellAmount . "\n";
echo "Buy amount: " . $price->buyAmount . "\n";
```

You can also specify the buy amount instead:

```php
// How much USD for 50 USDC?
$price = $quoteService->price(
    context: "sep24",
    sellAsset: "iso4217:USD",
    buyAsset: "stellar:USDC:GA5ZSEJYB37JRC5AVCIA5MOP4RHTM335X2KGX3IHOJAPP5RE34K4KZVN",
    buyAmount: "50",
    jwt: $jwtToken
);
```

### Requesting a Firm Quote

Firm quotes lock in a price for a limited time. Authentication is required.

```php
<?php

use Soneso\StellarSDK\SEP\Quote\QuoteService;
use Soneso\StellarSDK\SEP\Quote\SEP38PostQuoteRequest;

$quoteService = QuoteService::fromDomain("anchor.example.com");

$request = new SEP38PostQuoteRequest(
    context: "sep24",
    sellAsset: "iso4217:USD",
    buyAsset: "stellar:USDC:GA5ZSEJYB37JRC5AVCIA5MOP4RHTM335X2KGX3IHOJAPP5RE34K4KZVN",
    sellAmount: "100"
);

$quote = $quoteService->postQuote($request, $jwtToken);

echo "Quote ID: " . $quote->id . "\n";
echo "Expires: " . $quote->expiresAt . "\n";
echo "Total price: " . $quote->totalPrice . "\n";
echo "You receive: " . $quote->buyAmount . " USDC\n";
```

### Retrieving a Previous Quote

```php
<?php

use Soneso\StellarSDK\SEP\Quote\QuoteService;

$quoteService = QuoteService::fromDomain("anchor.example.com");

// Use the ID from postQuote response
$quote = $quoteService->getQuote($quoteId, $jwtToken);

echo "Quote ID: " . $quote->id . "\n";
echo "Expires: " . $quote->expiresAt . "\n";
```

## Error Handling

```php
<?php

use Soneso\StellarSDK\SEP\Quote\QuoteService;
use Soneso\StellarSDK\SEP\Quote\SEP38BadRequestException;
use Soneso\StellarSDK\SEP\Quote\SEP38NotFoundException;
use Soneso\StellarSDK\SEP\Quote\SEP38PermissionDeniedException;
use Soneso\StellarSDK\SEP\Quote\SEP38PostQuoteRequest;

$quoteService = QuoteService::fromDomain("anchor.example.com");

try {
    $request = new SEP38PostQuoteRequest(
        context: "sep24",
        sellAsset: "iso4217:USD",
        buyAsset: "stellar:USDC:GA5ZSEJYB37JRC5AVCIA5MOP4RHTM335X2KGX3IHOJAPP5RE34K4KZVN",
        sellAmount: "100"
    );
    
    $quote = $quoteService->postQuote($request, $jwtToken);
    
} catch (SEP38BadRequestException $e) {
    // Invalid request parameters
    echo "Bad request: " . $e->getMessage() . "\n";
    
} catch (SEP38PermissionDeniedException $e) {
    // Authentication failed or not authorized
    echo "Permission denied: " . $e->getMessage() . "\n";
    
} catch (SEP38NotFoundException $e) {
    // Quote not found (for getQuote)
    echo "Quote not found: " . $e->getMessage() . "\n";
}
```

Common errors:

| Exception | Cause | Solution |
|-----------|-------|----------|
| `SEP38BadRequestException` | Invalid asset format, amount, or context | Check asset identifiers and required fields |
| `SEP38PermissionDeniedException` | Missing or invalid JWT | Re-authenticate with SEP-10/SEP-45 |
| `SEP38NotFoundException` | Quote ID doesn't exist | Verify quote ID, may have expired |

## Related SEPs

- [SEP-10](sep-10.md) - Authentication for traditional accounts
- [SEP-45](sep-45.md) - Authentication for contract accounts
- [SEP-24](sep-24.md) - Interactive deposit/withdrawal (uses quotes)
- [SEP-6](sep-06.md) - Programmatic deposit/withdrawal (uses quotes)
- [SEP-31](sep-31.md) - Cross-border payments (uses quotes)

## Reference

- [SEP-38 Specification](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0038.md)
- [SDK Test Cases](https://github.com/Soneso/stellar-php-sdk/blob/main/Soneso/StellarSDKTests/SEP038Test.php)
