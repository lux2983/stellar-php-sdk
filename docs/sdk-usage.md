# SDK Usage Guide

This guide covers SDK features organized by use case. For detailed method signatures, see the [PHPDoc API Reference](https://soneso.github.io/stellar-php-sdk/packages/Soneso-StellarSDK.html).

## Table of Contents

- [Keypairs & Accounts](#keypairs--accounts)
- [Building Transactions](#building-transactions)
- [Operations](#operations)
- [Querying Horizon Data](#querying-horizon-data)
- [Streaming (SSE)](#streaming-sse)
- [Network Communication](#network-communication)
- [Assets](#assets)

---

## Keypairs & Accounts

### Creating Keypairs

```php
<?php
use Soneso\StellarSDK\Crypto\KeyPair;

// Generate new random keypair
$keyPair = KeyPair::random();
echo $keyPair->getAccountId();   // G... public key
echo $keyPair->getSecretSeed();  // S... secret seed

// Create from existing secret seed
$keyPair = KeyPair::fromSeed("SCZANGBA5YHTNYVVV3C7CAZMTQDBJHJG6C34JFD6XVEAEPTBED53FETV");

// Create public-key-only keypair (cannot sign)
$publicOnly = KeyPair::fromAccountId("GABC123...");
```

### Loading an Account

```php
<?php
use Soneso\StellarSDK\StellarSDK;

$sdk = StellarSDK::getTestNetInstance();

// Load account data from network
$account = $sdk->requestAccount("GABC123...");
echo "Sequence: " . $account->getSequenceNumber();

// Check balances
foreach ($account->getBalances() as $balance) {
    if ($balance->getAssetType() === 'native') {
        echo "XLM: " . $balance->getBalance();
    } else {
        echo $balance->getAssetCode() . ": " . $balance->getBalance();
    }
}

// Check if account exists
if ($sdk->accountExists("GABC123...")) {
    echo "Account exists";
}
```

### Funding Testnet Accounts

```php
<?php
use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Util\FriendBot;

$keyPair = KeyPair::random();
FriendBot::fundTestAccount($keyPair->getAccountId());
```

### HD Wallets (SEP-5)

```php
<?php
use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\SEP\Derivation\Mnemonic;

// Generate 24-word mnemonic
$mnemonic = Mnemonic::generate24WordsMnemonic();
echo implode(" ", $mnemonic->words);

// Restore from existing words
$mnemonic = Mnemonic::mnemonicFromWords("cable spray genius state float ...");

// Derive keypairs: m/44'/148'/{index}'
$account0 = KeyPair::fromMnemonic($mnemonic, 0);
$account1 = KeyPair::fromMnemonic($mnemonic, 1);

// With BIP-39 passphrase
$accountWithPass = KeyPair::fromMnemonic($mnemonic, 0, "my-passphrase");
```

### Muxed Accounts

Muxed accounts (M...) allow multiple users to share a single Stellar account by adding a 64-bit ID.

```php
<?php
use Soneso\StellarSDK\Asset;
use Soneso\StellarSDK\MuxedAccount;
use Soneso\StellarSDK\PaymentOperationBuilder;

// Create muxed account from base account + ID
$muxedAccount = MuxedAccount::fromAccountId("GABC...");
$muxedAccount->setMuxedAccountMed25519Id(123456789);

echo $muxedAccount->getAccountId();  // M... address
echo $muxedAccount->getMed25519Id(); // 123456789

// Parse existing muxed address
$muxed = MuxedAccount::fromMed25519AccountId("MABC...");
echo $muxed->getEd25519AccountId();  // Underlying G... address

// Use in payments
$paymentOp = PaymentOperationBuilder::forMuxedDestinationAccount(
    $muxedAccount,
    Asset::native(),
    "100"
)->build();
```

### Connecting to Networks

```php
<?php
use Soneso\StellarSDK\StellarSDK;
use Soneso\StellarSDK\Network;

// Testnet (development and testing)
$sdk = StellarSDK::getTestNetInstance();
$network = Network::testnet();

// Public network (production)
$sdk = StellarSDK::getPublicNetInstance();
$network = Network::public();

// Futurenet (preview upcoming features)
$sdk = StellarSDK::getFutureNetInstance();
$network = Network::futurenet();

// Custom Horizon server
$sdk = new StellarSDK("https://my-horizon-server.example.com");
$network = new Network("Custom Network Passphrase");
```

---

## Building Transactions

### Simple Payments

```php
<?php
use Soneso\StellarSDK\StellarSDK;
use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Asset;
use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\PaymentOperationBuilder;
use Soneso\StellarSDK\TransactionBuilder;

$sdk = StellarSDK::getTestNetInstance();

$senderKeyPair = KeyPair::fromSeed("SCZANGBA5YHTNYVVV3C7CAZMTQDBJHJG6C34JFD6XVEAEPTBED53FETV");
$sender = $sdk->requestAccount($senderKeyPair->getAccountId());

// Build payment
$paymentOp = (new PaymentOperationBuilder("GDEST...", Asset::native(), "100.50"))->build();

// Build, sign, submit
$transaction = (new TransactionBuilder($sender))
    ->addOperation($paymentOp)
    ->build();

$transaction->sign($senderKeyPair, Network::testnet());
$response = $sdk->submitTransaction($transaction);

if ($response->isSuccessful()) {
    echo "Payment sent! Hash: " . $response->getHash();
}
```

### Multi-Operation Transactions

```php
<?php
use Soneso\StellarSDK\StellarSDK;
use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\CreateAccountOperationBuilder;
use Soneso\StellarSDK\ChangeTrustOperationBuilder;
use Soneso\StellarSDK\PaymentOperationBuilder;
use Soneso\StellarSDK\TransactionBuilder;
use Soneso\StellarSDK\AssetTypeCreditAlphaNum4;

$sdk = StellarSDK::getTestNetInstance();

$funderKeyPair = KeyPair::fromSeed("SFUNDER...");
$newAccountKeyPair = KeyPair::random();
$funder = $sdk->requestAccount($funderKeyPair->getAccountId());

$usdAsset = new AssetTypeCreditAlphaNum4("USD", "GISSUER...");

// All operations execute atomically
$transaction = (new TransactionBuilder($funder))
    ->addOperation((new CreateAccountOperationBuilder($newAccountKeyPair->getAccountId(), "5"))->build())
    ->addOperation(
        (new ChangeTrustOperationBuilder($usdAsset, "10000"))
            ->setSourceAccount($newAccountKeyPair->getAccountId())
            ->build()
    )
    ->addOperation((new PaymentOperationBuilder($newAccountKeyPair->getAccountId(), $usdAsset, "100"))->build())
    ->build();

// Both accounts sign
$transaction->sign($funderKeyPair, Network::testnet());
$transaction->sign($newAccountKeyPair, Network::testnet());
$sdk->submitTransaction($transaction);
```

### Memos, Time Bounds, and Fees

```php
<?php
use Soneso\StellarSDK\Memo;
use Soneso\StellarSDK\TimeBounds;
use Soneso\StellarSDK\TransactionBuilder;

// Add memo
$transaction = (new TransactionBuilder($account))
    ->addOperation($operation)
    ->addMemo(Memo::text("Payment for invoice #1234"))
    ->build();

// Memo types: Memo::text(), Memo::id(), Memo::hash(), Memo::return()

// Time bounds (valid for next 5 minutes)
$timeBounds = new TimeBounds(new DateTime(), (new DateTime())->modify('+5 minutes'));
$transaction = (new TransactionBuilder($account))
    ->addOperation($operation)
    ->setTimeBounds($timeBounds)
    ->build();

// Custom fee (stroops per operation, default 100)
$transaction = (new TransactionBuilder($account))
    ->addOperation($operation)
    ->setMaxOperationFee(200)
    ->build();
```

### Fee Bump Transactions

```php
<?php
use Soneso\StellarSDK\FeeBumpTransactionBuilder;

// Wrap existing transaction with higher fee
$feeBumpTx = (new FeeBumpTransactionBuilder($innerTransaction))
    ->setBaseFee(500)
    ->setMuxedFeeAccount($feePayer->getMuxedAccount())
    ->build();

$feeBumpTx->sign($feePayerKeyPair, Network::testnet());
$sdk->submitTransaction($feeBumpTx);
```

---

## Operations

### Payment Operations

```php
<?php
use Soneso\StellarSDK\Asset;
use Soneso\StellarSDK\PaymentOperationBuilder;
use Soneso\StellarSDK\PathPaymentStrictSendOperationBuilder;
use Soneso\StellarSDK\PathPaymentStrictReceiveOperationBuilder;
use Soneso\StellarSDK\AssetTypeCreditAlphaNum4;

// Native XLM payment
$paymentOp = (new PaymentOperationBuilder("GDEST...", Asset::native(), "100"))->build();

// Custom asset payment
$usdAsset = new AssetTypeCreditAlphaNum4("USD", "GISSUER...");
$paymentOp = (new PaymentOperationBuilder("GDEST...", $usdAsset, "50.25"))->build();

// Path payment strict send: send exactly 100 XLM, receive at least 95 USD
$pathPaymentSend = (new PathPaymentStrictSendOperationBuilder(
    Asset::native(), "100", "GDEST...", $usdAsset, "95"
))->build();

// Path payment strict receive: receive exactly 100 USD, send at most 110 XLM
$pathPaymentReceive = (new PathPaymentStrictReceiveOperationBuilder(
    Asset::native(), "110", "GDEST...", $usdAsset, "100"
))->build();
```

### Account Operations

```php
<?php
use Soneso\StellarSDK\CreateAccountOperationBuilder;
use Soneso\StellarSDK\AccountMergeOperationBuilder;
use Soneso\StellarSDK\BumpSequenceOperationBuilder;
use Soneso\StellarSDK\ManageDataOperationBuilder;
use Soneso\StellarSDK\SetOptionsOperationBuilder;
use Soneso\StellarSDK\Signer;

// Create account
$createOp = (new CreateAccountOperationBuilder("GNEWACCOUNT...", "10"))->build();

// Merge account
$mergeOp = (new AccountMergeOperationBuilder("GDEST..."))->build();

// Bump sequence
$bumpOp = (new BumpSequenceOperationBuilder(123456789))->build();

// Set data
$setDataOp = (new ManageDataOperationBuilder("config", "production"))->build();
$deleteDataOp = (new ManageDataOperationBuilder("temp_key", null))->build();

// Set options
$setOptionsOp = (new SetOptionsOperationBuilder())
    ->setHomeDomain("example.com")
    ->setMasterKeyWeight(1)
    ->setLowThreshold(1)
    ->setMediumThreshold(2)
    ->setHighThreshold(3)
    ->build();

// Add signer
$signerKey = Signer::ed25519PublicKey("GSIGNER...");
$addSignerOp = (new SetOptionsOperationBuilder())->setSigner($signerKey, 1)->build();
```

### Asset Operations

```php
<?php
use Soneso\StellarSDK\ChangeTrustOperationBuilder;
use Soneso\StellarSDK\SetTrustLineFlagsOperationBuilder;
use Soneso\StellarSDK\AssetTypeCreditAlphaNum4;
use Soneso\StellarSDK\Xdr\XdrTrustLineFlags;

$usdAsset = new AssetTypeCreditAlphaNum4("USD", "GISSUER...");

// Create trustline
$trustOp = (new ChangeTrustOperationBuilder($usdAsset, "10000"))->build();

// Remove trustline (must have zero balance)
$removeTrustOp = (new ChangeTrustOperationBuilder($usdAsset, "0"))->build();

// Issuer: authorize trustline
$authorizeTrustOp = (new SetTrustLineFlagsOperationBuilder(
    "GTRUSTOR...", $usdAsset, XdrTrustLineFlags::AUTHORIZED_FLAG, 0
))->build();
```

### Trading Operations

```php
<?php
use Soneso\StellarSDK\Asset;
use Soneso\StellarSDK\ManageSellOfferOperationBuilder;
use Soneso\StellarSDK\ManageBuyOfferOperationBuilder;
use Soneso\StellarSDK\CreatePassiveSellOfferOperationBuilder;
use Soneso\StellarSDK\AssetTypeCreditAlphaNum4;

$usdAsset = new AssetTypeCreditAlphaNum4("USD", "GISSUER...");

// Create sell offer: sell 100 XLM at 0.20 USD per XLM
$sellOp = (new ManageSellOfferOperationBuilder(Asset::native(), $usdAsset, "100", "0.20"))->build();

// Create buy offer
$buyOp = (new ManageBuyOfferOperationBuilder(Asset::native(), $usdAsset, "50", "0.20"))->build();

// Update existing offer
$updateOp = (new ManageSellOfferOperationBuilder(Asset::native(), $usdAsset, "150", "0.22"))
    ->setOfferId(12345)
    ->build();

// Cancel offer (set amount to 0)
$cancelOp = (new ManageSellOfferOperationBuilder(Asset::native(), $usdAsset, "0", "0.20"))
    ->setOfferId(12345)
    ->build();

// Passive sell offer (won't match existing offers)
$passiveOp = (new CreatePassiveSellOfferOperationBuilder(Asset::native(), $usdAsset, "100", "0.20"))->build();
```

### Claimable Balance Operations

```php
<?php
use Soneso\StellarSDK\Asset;
use Soneso\StellarSDK\Claimant;
use Soneso\StellarSDK\CreateClaimableBalanceOperationBuilder;
use Soneso\StellarSDK\ClaimClaimableBalanceOperationBuilder;

// Claimants with predicates
$claimant1 = new Claimant("GCLAIMER1...", Claimant::predicateUnconditional());
$claimant2 = new Claimant("GCLAIMER2...", Claimant::predicateBeforeAbsoluteTime(strtotime("+30 days")));
$claimant3 = new Claimant("GCLAIMER3...", Claimant::predicateBeforeRelativeTime(3600));

// Complex: claimable after 1 day AND before 30 days
$afterOneDay = Claimant::predicateNot(Claimant::predicateBeforeRelativeTime(86400));
$beforeThirtyDays = Claimant::predicateBeforeRelativeTime(86400 * 30);
$complexPredicate = Claimant::predicateAnd($afterOneDay, $beforeThirtyDays);

// Create claimable balance
$createOp = (new CreateClaimableBalanceOperationBuilder([$claimant1, $claimant2], Asset::native(), "100"))->build();

// Claim balance
$claimOp = (new ClaimClaimableBalanceOperationBuilder("00000000abc123..."))->build();
```

### Liquidity Pool Operations

```php
<?php
use Soneso\StellarSDK\Asset;
use Soneso\StellarSDK\Price;
use Soneso\StellarSDK\ChangeTrustOperationBuilder;
use Soneso\StellarSDK\AssetTypeCreditAlphaNum4;
use Soneso\StellarSDK\AssetTypePoolShare;
use Soneso\StellarSDK\LiquidityPoolDepositOperationBuilder;
use Soneso\StellarSDK\LiquidityPoolWithdrawOperationBuilder;

$usdAsset = new AssetTypeCreditAlphaNum4("USD", "GISSUER...");

// Trust pool shares (assets in canonical order)
$poolShareAsset = new AssetTypePoolShare(Asset::native(), $usdAsset);
$trustPoolOp = (new ChangeTrustOperationBuilder($poolShareAsset))->build();

// Deposit liquidity
$depositOp = (new LiquidityPoolDepositOperationBuilder(
    "poolid123...", "1000", "500", Price::fromString("1.9"), Price::fromString("2.1")
))->build();

// Withdraw liquidity
$withdrawOp = (new LiquidityPoolWithdrawOperationBuilder("poolid123...", "100", "180", "90"))->build();
```

### Sponsorship Operations

```php
<?php
use Soneso\StellarSDK\BeginSponsoringFutureReservesOperationBuilder;
use Soneso\StellarSDK\EndSponsoringFutureReservesOperationBuilder;
use Soneso\StellarSDK\CreateAccountOperationBuilder;
use Soneso\StellarSDK\RevokeSponsorshipOperationBuilder;
use Soneso\StellarSDK\TransactionBuilder;

// Sponsor account creation
$transaction = (new TransactionBuilder($sponsor))
    ->addOperation((new BeginSponsoringFutureReservesOperationBuilder($sponsoredKeyPair->getAccountId()))->build())
    ->addOperation((new CreateAccountOperationBuilder($sponsoredKeyPair->getAccountId(), "0"))->build())
    ->addOperation(
        (new EndSponsoringFutureReservesOperationBuilder())
            ->setSourceAccount($sponsoredKeyPair->getAccountId())
            ->build()
    )
    ->build();

// Both must sign
$transaction->sign($sponsorKeyPair, Network::testnet());
$transaction->sign($sponsoredKeyPair, Network::testnet());

// Revoke sponsorship
$revokeOp = RevokeSponsorshipOperationBuilder::forAccount($sponsoredKeyPair->getAccountId())->build();
```

---

## Querying Horizon Data

All query builders support `limit()`, `order()`, `cursor()` for pagination.

### Pagination

```php
<?php
use Soneso\StellarSDK\StellarSDK;

$sdk = StellarSDK::getTestNetInstance();

// First page
$page = $sdk->transactions()
    ->forAccount("GABC...")
    ->limit(20)
    ->order("desc")
    ->execute();

// Process results
foreach ($page->getTransactions()->toArray() as $tx) {
    echo $tx->getHash();
}

// Get next page using cursor from last record
$transactions = $page->getTransactions()->toArray();
if (!empty($transactions)) {
    $lastTx = end($transactions);
    $nextPage = $sdk->transactions()
        ->forAccount("GABC...")
        ->limit(20)
        ->order("desc")
        ->cursor($lastTx->getPagingToken())
        ->execute();
}
```

### Account Queries

```php
<?php
use Soneso\StellarSDK\StellarSDK;
use Soneso\StellarSDK\Asset;

$sdk = StellarSDK::getTestNetInstance();

// Get single account
$account = $sdk->accounts()->account("GABC...");

// Query accounts by signer
$accountsPage = $sdk->accounts()->forSigner("GSIGNER...")->limit(50)->order("desc")->execute();

// Query accounts holding an asset
$usdAsset = Asset::createNonNativeAsset("USD", "GISSUER...");
$accountsPage = $sdk->accounts()->forAsset($usdAsset)->execute();

// Query by sponsor
$accountsPage = $sdk->accounts()->forSponsor("GSPONSOR...")->execute();

// Get account data entry
$dataValue = $sdk->accounts()->accountData("GABC...", "config");
```

### Transaction Queries

```php
<?php
$sdk = StellarSDK::getTestNetInstance();

// Get specific transaction
$tx = $sdk->transactions()->transaction("abc123hash...");

// Transactions for account
$txPage = $sdk->transactions()->forAccount("GABC...")->limit(20)->order("desc")->execute();

// Include failed transactions
$txPage = $sdk->transactions()->forAccount("GABC...")->includeFailed(true)->execute();

// For ledger, claimable balance, or liquidity pool
$txPage = $sdk->transactions()->forLedger("12345678")->execute();
$txPage = $sdk->transactions()->forClaimableBalance("00000000abc...")->execute();
$txPage = $sdk->transactions()->forLiquidityPool("poolid...")->execute();
```

### Operation & Effect Queries

```php
<?php
$sdk = StellarSDK::getTestNetInstance();

// Operations
$op = $sdk->operations()->operation("123456789");
$opsPage = $sdk->operations()->forAccount("GABC...")->execute();
$opsPage = $sdk->operations()->forTransaction("txhash...")->execute();

// Effects
$effectsPage = $sdk->effects()->forAccount("GABC...")->execute();
$effectsPage = $sdk->effects()->forOperation("123456789")->execute();
```

### Ledger & Payment Queries

```php
<?php
$sdk = StellarSDK::getTestNetInstance();

// Ledgers
$ledger = $sdk->requestLedger("12345678");
$ledgersPage = $sdk->ledgers()->limit(10)->order("desc")->execute();

// Payments (Payment, PathPayment, CreateAccount, AccountMerge)
$paymentsPage = $sdk->payments()->forAccount("GABC...")->execute();
```

### Offer Queries

```php
<?php
use Soneso\StellarSDK\Asset;

$sdk = StellarSDK::getTestNetInstance();

// Get specific offer
$offer = $sdk->requestOffer("12345");
echo "Selling: " . $offer->getAmount() . " " . $offer->getSellingAsset();
echo "Buying: " . $offer->getBuyingAsset();
echo "Price: " . $offer->getPrice();

// Offers for an account
$offersPage = $sdk->offers()
    ->forAccount("GABC...")
    ->limit(50)
    ->execute();

foreach ($offersPage->getOffers()->toArray() as $offer) {
    echo $offer->getId() . ": " . $offer->getAmount() . " at " . $offer->getPrice();
}

// Offers for selling asset
$offersPage = $sdk->offers()
    ->forSellingAsset(Asset::native())
    ->execute();

// Offers for buying asset
$usdAsset = Asset::createNonNativeAsset("USD", "GISSUER...");
$offersPage = $sdk->offers()
    ->forBuyingAsset($usdAsset)
    ->execute();

// Offers by sponsor
$offersPage = $sdk->offers()
    ->forSponsor("GSPONSOR...")
    ->execute();
```

### Trade & Asset Queries

```php
<?php
use Soneso\StellarSDK\Asset;

$sdk = StellarSDK::getTestNetInstance();
$usdAsset = Asset::createNonNativeAsset("USD", "GISSUER...");

// Trades for account
$tradesPage = $sdk->trades()
    ->forAccount("GABC...")
    ->limit(50)
    ->order("desc")
    ->execute();

foreach ($tradesPage->getTrades()->toArray() as $trade) {
    echo $trade->getBaseAmount() . " " . $trade->getBaseAssetCode();
    echo " for " . $trade->getCounterAmount() . " " . $trade->getCounterAssetCode();
}

// Trades for asset pair
$tradesPage = $sdk->trades()
    ->forAssetPair(Asset::native(), $usdAsset)
    ->execute();

// Trades for offer
$tradesPage = $sdk->trades()
    ->forOfferId("12345")
    ->execute();

// Trade aggregations (OHLCV candles)
$aggregations = $sdk->tradeAggregations()
    ->forAssetPair(Asset::native(), $usdAsset)
    ->resolution(3600000)  // 1 hour in ms
    ->limit(24)
    ->execute();

foreach ($aggregations->getRecords()->toArray() as $candle) {
    echo "Open: " . $candle->getOpen();
    echo "High: " . $candle->getHigh();
    echo "Low: " . $candle->getLow();
    echo "Close: " . $candle->getClose();
    echo "Volume: " . $candle->getBaseVolume();
}

// Assets by code
$assetsPage = $sdk->assets()
    ->forCode("USD")
    ->limit(20)
    ->execute();

foreach ($assetsPage->getAssets()->toArray() as $asset) {
    echo $asset->getAssetCode() . " by " . $asset->getAssetIssuer();
    echo " - Holders: " . $asset->getNumAccounts();
    echo " - Supply: " . $asset->getAmount();
}

// Assets by issuer
$assetsPage = $sdk->assets()
    ->forIssuer("GISSUER...")
    ->execute();
```

### Order Book & Path Queries

```php
<?php
use Soneso\StellarSDK\Asset;

$sdk = StellarSDK::getTestNetInstance();
$usdAsset = Asset::createNonNativeAsset("USD", "GISSUER...");

// Order book
$orderBook = $sdk->orderBook()->forSellingAsset(Asset::native())->forBuyingAsset($usdAsset)->execute();

// Strict send paths
$pathsPage = $sdk->findStrictSendPaths()
    ->forSourceAsset(Asset::native())
    ->forSourceAmount("100")
    ->forDestinationAssets([$usdAsset])
    ->execute();

// Strict receive paths
$pathsPage = $sdk->findStrictReceivePaths()
    ->forSourceAccount("GSENDER...")
    ->forDestinationAsset($usdAsset)
    ->forDestinationAmount("100")
    ->execute();
```

### Claimable Balance & Liquidity Pool Queries

```php
<?php
use Soneso\StellarSDK\Asset;

$sdk = StellarSDK::getTestNetInstance();

// Claimable balances
$balance = $sdk->requestClaimableBalance("00000000abc123...");
$balancesPage = $sdk->claimableBalances()->forClaimant("GCLAIMER...")->execute();
$balancesPage = $sdk->claimableBalances()->forSponsor("GSPONSOR...")->execute();

// Liquidity pools
$pool = $sdk->requestLiquidityPool("poolid123...");
$poolsPage = $sdk->liquidityPools()->forAccount("GABC...")->execute();

$usdAsset = Asset::createNonNativeAsset("USD", "GISSUER...");
$poolsPage = $sdk->liquidityPools()->forReserves(Asset::native(), $usdAsset)->execute();
```

---

## Streaming (SSE)

Real-time updates via Server-Sent Events.

```php
<?php
use Soneso\StellarSDK\StellarSDK;
use Soneso\StellarSDK\Responses\Operations\OperationResponse;
use Soneso\StellarSDK\Responses\Operations\PaymentOperationResponse;

$sdk = StellarSDK::getTestNetInstance();

// Stream payments
$sdk->payments()->forAccount("GABC...")->cursor("now")->stream(function(OperationResponse $payment) {
    if ($payment instanceof PaymentOperationResponse) {
        echo "Received " . $payment->getAmount() . " from " . $payment->getFrom() . "\n";
    }
    // Return false to stop streaming
});

// Stream transactions
$sdk->transactions()->forAccount("GABC...")->cursor("now")->stream(function($tx) {
    echo "New transaction: " . $tx->getHash() . "\n";
});

// Stream ledgers
$sdk->ledgers()->cursor("now")->stream(function($ledger) {
    echo "Ledger " . $ledger->getSequence() . " closed\n";
});

// Also available: operations(), effects(), trades()
```

---

## Network Communication

### Transaction Submission

```php
<?php
use Soneso\StellarSDK\StellarSDK;
use Soneso\StellarSDK\Exceptions\HorizonRequestException;

$sdk = StellarSDK::getTestNetInstance();

try {
    $response = $sdk->submitTransaction($transaction);
    
    if ($response->isSuccessful()) {
        echo "Hash: " . $response->getHash();
        echo "Ledger: " . $response->getLedger();
    }
} catch (HorizonRequestException $e) {
    echo "Error: " . $e->getMessage();
    $horizonError = $e->getHorizonErrorResponse();
    if ($horizonError) {
        echo "Result codes: " . json_encode($horizonError->getExtras()->getResultCodes());
    }
}

// Check memo requirements (SEP-29)
$memoRequired = $sdk->checkMemoRequired($transaction);
if ($memoRequired !== false) {
    echo "Account $memoRequired requires a memo";
}
```

### Fee Statistics

```php
<?php
$sdk = StellarSDK::getTestNetInstance();

$feeStats = $sdk->requestFeeStats();
echo "Min: " . $feeStats->getMinAcceptedFee();
echo "Mode: " . $feeStats->getModeAcceptedFee();
echo "P90: " . $feeStats->getP90AcceptedFee();
```

### Async Submission

```php
<?php
// Returns immediately after core accepts transaction
$response = $sdk->submitAsyncTransaction($transaction);
echo "Status: " . $response->getTxStatus();  // PENDING, DUPLICATE, TRY_AGAIN_LATER, ERROR
echo "Hash: " . $response->getHash();
```

### Error Handling

```php
<?php
use Soneso\StellarSDK\StellarSDK;
use Soneso\StellarSDK\Exceptions\HorizonRequestException;

$sdk = StellarSDK::getTestNetInstance();

try {
    $response = $sdk->submitTransaction($transaction);
    
    if ($response->isSuccessful()) {
        echo "Success! Hash: " . $response->getHash();
    }
} catch (HorizonRequestException $e) {
    // Get HTTP status code
    echo "Status: " . $e->getStatusCode();
    echo "Message: " . $e->getMessage();
    
    // Get detailed Horizon error response
    $horizonError = $e->getHorizonErrorResponse();
    if ($horizonError) {
        echo "Type: " . $horizonError->getType();
        echo "Title: " . $horizonError->getTitle();
        
        // Transaction-specific error details
        $extras = $horizonError->getExtras();
        if ($extras) {
            // Result codes explain why the transaction failed
            $resultCodes = $extras->getResultCodes();
            echo "Transaction result: " . $resultCodes->getTransaction();
            
            // Per-operation result codes
            foreach ($resultCodes->getOperations() as $opResult) {
                echo "Operation result: " . $opResult;
            }
        }
    }
}

// Common result codes:
// tx_insufficient_fee - Fee too low for current network load
// tx_bad_seq - Sequence number mismatch (stale account data)
// tx_insufficient_balance - Not enough XLM for operation + fees
// op_underfunded - Not enough balance for payment
// op_no_trust - Missing trustline for asset
// op_line_full - Trustline limit exceeded
```

### Message Signing (SEP-53)

```php
<?php
use Soneso\StellarSDK\Crypto\KeyPair;

$keyPair = KeyPair::fromSeed("SCZANGBA5YHTNYVVV3C7CAZMTQDBJHJG6C34JFD6XVEAEPTBED53FETV");

// Sign a message
$message = "Please sign this message to verify your identity";
$signature = $keyPair->signMessage($message);

// Encode for transmission
$signatureBase64 = base64_encode($signature);

// Verify a message signature
$isValid = $keyPair->verifyMessage($message, $signature);
if ($isValid) {
    echo "Signature is valid";
}

// Verify with public key only
$publicKey = KeyPair::fromAccountId($keyPair->getAccountId());
$isValid = $publicKey->verifyMessage($message, base64_decode($signatureBase64));
```

---

## Assets

### Creating Assets

```php
<?php
use Soneso\StellarSDK\Asset;
use Soneso\StellarSDK\AssetTypeCreditAlphaNum4;
use Soneso\StellarSDK\AssetTypeCreditAlphaNum12;

// Native XLM
$xlm = Asset::native();

// 4-character code
$usd = new AssetTypeCreditAlphaNum4("USD", "GISSUER...");

// 5-12 character code
$myToken = new AssetTypeCreditAlphaNum12("MYTOKEN", "GISSUER...");

// Auto-detect code length
$asset = Asset::createNonNativeAsset("USD", "GISSUER...");

// From/to canonical form
$asset = Asset::createFromCanonicalForm("USD:GISSUER...");
$canonical = Asset::canonicalForm($usd);  // "USD:GISSUER..."
```

### Trustlines

```php
<?php
use Soneso\StellarSDK\StellarSDK;
use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\ChangeTrustOperationBuilder;
use Soneso\StellarSDK\TransactionBuilder;
use Soneso\StellarSDK\AssetTypeCreditAlphaNum4;

$sdk = StellarSDK::getTestNetInstance();
$trustorKeyPair = KeyPair::fromSeed("STRUSTOR...");
$trustor = $sdk->requestAccount($trustorKeyPair->getAccountId());
$usdAsset = new AssetTypeCreditAlphaNum4("USD", "GISSUER...");

// Create trustline
$trustOp = (new ChangeTrustOperationBuilder($usdAsset, "10000"))->build();
$transaction = (new TransactionBuilder($trustor))->addOperation($trustOp)->build();
$transaction->sign($trustorKeyPair, Network::testnet());
$sdk->submitTransaction($transaction);

// Modify limit
$modifyOp = (new ChangeTrustOperationBuilder($usdAsset, "50000"))->build();

// Remove (balance must be zero)
$removeOp = (new ChangeTrustOperationBuilder($usdAsset, "0"))->build();
```

---

## Further Reading

- [Quick Start Guide](quick-start.md) — First transaction in 15 minutes
- [Getting Started](getting-started.md) — Installation and fundamentals
- [Soroban Guide](soroban.md) — Smart contract development
- [SEP Protocols](sep/README.md) — Stellar Ecosystem Proposals
- [PHPDoc Reference](https://soneso.github.io/stellar-php-sdk/packages/Soneso-StellarSDK.html) — Full API documentation
