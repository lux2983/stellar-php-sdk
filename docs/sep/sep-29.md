# SEP-29: Memo Required

SEP-29 prevents lost funds by allowing accounts to require incoming payments include a memo. Exchanges and custodians use this to identify which customer a payment belongs to. Without a memo, deposits can't be credited to the right user.

**Use SEP-29 when:**
- Sending payments to exchanges or custodial services
- Building a payment flow that needs to validate destinations before submission
- Running an exchange and requiring memos on incoming deposits

**Spec:** [SEP-0029](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0029.md)

## Quick Example

```php
<?php

use Soneso\StellarSDK\Asset;
use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Memo;
use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\PaymentOperationBuilder;
use Soneso\StellarSDK\StellarSDK;
use Soneso\StellarSDK\TransactionBuilder;

$sdk = StellarSDK::getTestNetInstance();
$senderKeyPair = KeyPair::fromSeed("SCZANGBA5YHTNYVVV3C7CAZMTQDBJHJG6C34CJDQ66EQ7DZTPBRJFN4A");
$destinationId = "GDQP2KPQGKIHYJGXNUIYOMHARUARCA7DJT5FO2FFOOUJ3UBEZ3ENO5GT";

$senderAccount = $sdk->requestAccount($senderKeyPair->getAccountId());

$paymentOp = (new PaymentOperationBuilder($destinationId, Asset::native(), "100.0"))
    ->build();

$transaction = (new TransactionBuilder($senderAccount))
    ->addOperation($paymentOp)
    ->build();

// Check if destination requires a memo
$requiresMemo = $sdk->checkMemoRequired($transaction);

if ($requiresMemo !== false) {
    echo "Account {$requiresMemo} requires a memo. Rebuild with one.";
    $transaction = (new TransactionBuilder($senderAccount))
        ->addOperation($paymentOp)
        ->addMemo(Memo::text("user-123"))
        ->build();
}

$transaction->sign($senderKeyPair, Network::testnet());
$response = $sdk->submitTransaction($transaction);
```

## How It Works

Accounts signal memo requirement by setting a data entry with key `config.memo_required` and value `1`.

When you call `checkMemoRequired()`, the SDK:
1. Returns `false` immediately if the transaction already has a memo
2. Extracts destinations from payment-type operations
3. Queries each destination's account data
4. Returns the first account ID requiring a memo, or `false` if none do

Checked operation types: `PaymentOperation`, `PathPaymentStrictSendOperation`, `PathPaymentStrictReceiveOperation`, `AccountMergeOperation`.

## Detailed Usage

### Setting Memo Requirement on Your Account

If you run an exchange or custodial service:

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\ManageDataOperationBuilder;
use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\StellarSDK;
use Soneso\StellarSDK\TransactionBuilder;

$sdk = StellarSDK::getTestNetInstance();
$exchangeKeyPair = KeyPair::fromSeed("SBMSVD4KKELKGZXHBUQTIROWUAPQASDX7KEJITARP4VMZ6KLUHOGPTYW");
$exchangeAccount = $sdk->requestAccount($exchangeKeyPair->getAccountId());

// Set memo_required flag
$setMemoRequired = (new ManageDataOperationBuilder("config.memo_required", "1"))
    ->build();

$transaction = (new TransactionBuilder($exchangeAccount))
    ->addOperation($setMemoRequired)
    ->build();

$transaction->sign($exchangeKeyPair, Network::testnet());
$sdk->submitTransaction($transaction);
```

To remove the requirement, pass `null` as the value (deletes the data entry):

```php
$removeMemoRequired = (new ManageDataOperationBuilder("config.memo_required", null))
    ->build();
```

### Transactions with Multiple Destinations

The check validates all destination accounts and returns the first one requiring a memo:

```php
<?php

use Soneso\StellarSDK\Asset;
use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\PaymentOperationBuilder;
use Soneso\StellarSDK\StellarSDK;
use Soneso\StellarSDK\TransactionBuilder;

$sdk = StellarSDK::getTestNetInstance();
$senderKeyPair = KeyPair::fromSeed("SCZANGBA5YHTNYVVV3C7CAZMTQDBJHJG6C34CJDQ66EQ7DZTPBRJFN4A");
$senderAccount = $sdk->requestAccount($senderKeyPair->getAccountId());

// Batch payment to multiple recipients
$transaction = (new TransactionBuilder($senderAccount))
    ->addOperation((new PaymentOperationBuilder(
        "GDQP2KPQGKIHYJGXNUIYOMHARUARCA7DJT5FO2FFOOUJ3UBEZ3ENO5GT",
        Asset::native(), "100.0"))->build())
    ->addOperation((new PaymentOperationBuilder(
        "GCKUD4BHIYSBER7DI6TPMYQ4KNDEUKVMN44VKSUQGEFXWLNTHIIQE7FB",
        Asset::native(), "50.0"))->build())
    ->build();

$accountRequiringMemo = $sdk->checkMemoRequired($transaction);

if ($accountRequiringMemo !== false) {
    echo "Cannot batch: {$accountRequiringMemo} requires a memo.";
}
```

## Integration with Payment Flows

Check memo requirements before showing the confirmation screen:

```php
<?php

use Soneso\StellarSDK\Asset;
use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Memo;
use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\PaymentOperationBuilder;
use Soneso\StellarSDK\StellarSDK;
use Soneso\StellarSDK\TransactionBuilder;

function sendPayment(
    StellarSDK $sdk,
    KeyPair $senderKeyPair,
    string $destinationId,
    string $amount,
    ?string $memo = null
): array {
    $senderAccount = $sdk->requestAccount($senderKeyPair->getAccountId());

    $paymentOp = (new PaymentOperationBuilder($destinationId, Asset::native(), $amount))
        ->build();

    $builder = (new TransactionBuilder($senderAccount))
        ->addOperation($paymentOp);

    if ($memo !== null) {
        $builder->addMemo(Memo::text($memo));
    }

    $transaction = $builder->build();
    $requiresMemo = $sdk->checkMemoRequired($transaction);

    if ($requiresMemo !== false && $memo === null) {
        return [
            'success' => false,
            'error' => 'memo_required',
            'account' => $requiresMemo,
        ];
    }

    $transaction->sign($senderKeyPair, Network::testnet());
    $response = $sdk->submitTransaction($transaction);

    return ['success' => true, 'hash' => $response->getHash()];
}
```

## Error Handling

The method throws `HorizonRequestException` if it fails to query a destination account:

```php
<?php

use Soneso\StellarSDK\Asset;
use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Exceptions\HorizonRequestException;
use Soneso\StellarSDK\PaymentOperationBuilder;
use Soneso\StellarSDK\StellarSDK;
use Soneso\StellarSDK\TransactionBuilder;

$sdk = StellarSDK::getTestNetInstance();
$senderKeyPair = KeyPair::fromSeed("SCZANGBA5YHTNYVVV3C7CAZMTQDBJHJG6C34CJDQ66EQ7DZTPBRJFN4A");
$senderAccount = $sdk->requestAccount($senderKeyPair->getAccountId());

$paymentOp = (new PaymentOperationBuilder(
    "GDQP2KPQGKIHYJGXNUIYOMHARUARCA7DJT5FO2FFOOUJ3UBEZ3ENO5GT",
    Asset::native(), "50.0"))->build();

$transaction = (new TransactionBuilder($senderAccount))
    ->addOperation($paymentOp)
    ->build();

try {
    $requiresMemo = $sdk->checkMemoRequired($transaction);
} catch (HorizonRequestException $e) {
    // Account might not exist yet or Horizon unavailable
    echo "Could not verify memo requirement: " . $e->getMessage();
}
```

Fee bump transactions always return `false` — check the inner transaction before wrapping.

## Related SEPs

- **[SEP-10](sep-10.md)** - Web authentication (often used by exchanges that require memos)
- **[SEP-24](sep-24.md)** - Interactive deposit/withdrawal (anchors provide deposit memos)
- **[SEP-31](sep-31.md)** - Cross-border payments (uses memos for transaction tracking)
