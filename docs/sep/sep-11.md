# SEP-11: Txrep (Transaction Representation)

Txrep is a human-readable text format for Stellar transactions. It converts the binary XDR transaction format into a structured key-value format that's easy to read, edit, and audit.

**When to use:** Debugging transactions, displaying transaction details to users, creating transaction templates, or building tools that need to inspect/modify transactions in a readable format.

See the [SEP-11 specification](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0011.md) for protocol details.

## Quick Example

```php
<?php

use Soneso\StellarSDK\SEP\TxRep\TxRep;

// Convert XDR to human-readable format
$xdrBase64 = 'AAAAAgAAAAArFkuQQ4QuQY6SkLc5xxSdwpFOvl7VqKVvrfkPSqB+0AAAAGQApSmNAAAAAQAAAAEAAAAAW4nJgAAAAABdav0AAAAAAQAAABZFbmpveSB0aGlzIHRyYW5zYWN0aW9uAAAAAAABAAAAAAAAAAEAAAAAQF827djPIu+/gHK5hbakwBVRw03TjBN6yNQNQCzR97QAAAABVVNEAAAAAAAyUlQyIZKfbs+tUWuvK7N0nGSCII0/Go1/CpHXNW3tCwAAAAAX15OgAAAAAAAAAAFKoH7QAAAAQN77Tx+tHCeTJ7Va8YT9zd9z9Peoy0Dn5TSnHXOgUSS6Np23ptMbR8r9EYWSJGqFdebCSauU7Ddo3ttikiIc5Qw=';

$txRep = TxRep::fromTransactionEnvelopeXdrBase64($xdrBase64);
echo $txRep;
```

Output:
```
type: ENVELOPE_TYPE_TX
tx.sourceAccount: GAVRMS4QIOCC4QMOSKILOOOHCSO4FEKOXZPNLKFFN6W7SD2KUB7NBPLN
tx.fee: 100
tx.seqNum: 46489056724385793
tx.cond.type: PRECOND_TIME
tx.cond.timeBounds.minTime: 1535756672
tx.cond.timeBounds.maxTime: 1567292672
tx.memo.type: MEMO_TEXT
tx.memo.text: "Enjoy this transaction"
tx.operations.len: 1
tx.operations[0].sourceAccount._present: false
tx.operations[0].body.type: PAYMENT
tx.operations[0].body.paymentOp.destination: GBAF6NXN3DHSF357QBZLTBNWUTABKUODJXJYYE32ZDKA2QBM2H33IK6O
tx.operations[0].body.paymentOp.asset: USD:GAZFEVBSEGJJ63WPVVIWXLZLWN2JYZECECGT6GUNP4FJDVZVNXWQWMYI
tx.operations[0].body.paymentOp.amount: 400004000
tx.ext.v: 0
signatures.len: 1
signatures[0].hint: 4aa07ed0
signatures[0].signature: defb4f1fad1c279327b55af184fdcddf73f4f7a8cb40e7e534a71d73a05124ba369db7a6d31b47cafd118592246a8575e6c249ab94ec3768dedb6292221ce50c
```

## Converting XDR to Txrep

### Standard Transaction

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Memo;
use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\PaymentOperationBuilder;
use Soneso\StellarSDK\SEP\TxRep\TxRep;
use Soneso\StellarSDK\StellarSDK;
use Soneso\StellarSDK\TransactionBuilder;

$sdk = StellarSDK::getTestNetInstance();

// Build a transaction
$sourceKeyPair = KeyPair::fromSeed('SCZANGBA5YHTNYVVV3C7CAZMTQDBJHJG6C34CPMLIHJPFV5RXN5M6CSS');
$sourceAccount = $sdk->requestAccount($sourceKeyPair->getAccountId());

$payment = (new PaymentOperationBuilder(
    'GDGUF4SCNINRDCRUIVOMDYGIMXOWVP3ZLMTL2OGQIWMFDDSECZSFQMQV',
    Asset::native(),
    '100'
))->build();

$transaction = (new TransactionBuilder($sourceAccount))
    ->addOperation($payment)
    ->addMemo(Memo::text('Test payment'))
    ->build();

$transaction->sign($sourceKeyPair, Network::testnet());

// Convert to Txrep
$txRep = TxRep::fromTransactionEnvelopeXdrBase64($transaction->toEnvelopeXdrBase64());
echo $txRep;
```

### Fee Bump Transaction

Fee bump transactions have a different structure:

```php
<?php

use Soneso\StellarSDK\SEP\TxRep\TxRep;

$feeBumpXdr = 'AAAABQAAAABkfT0dQuoYYNgStwXg4RJV62+W1uApFc4NpBdc2iHu6AAAAAAAAAGQAAAAAgAAAAAx5Qe+wF5jJp3kYrOZ2zBOQOcTHjtRBuR/GrBTLYydyQAAAGQAAVlhAAAAAQAAAAEAAAAAAAAAAAAAAAAAAAAAAAAAAQAAAAVoZWxsbwAAAAAAAAEAAAAAAAAAAAAAAABkfT0dQuoYYNgStwXg4RJV62+W1uApFc4NpBdc2iHu6AAAAAAL68IAAAAAAAAAAAEtjJ3JAAAAQFzU5qFDIaZRUzUxf0BrRO2abx0PuMn3WKM7o8NXZvmB7K0zvS+HBlmDo2P/M3IZpF5Riax21neE0N9/WiHRuAoAAAAAAAAAAdoh7ugAAABARiKZWxfy8ZOPRj6yZRTKXAp1Aw6SoEn5OvnFbOmVztZtSRUaVOaCnBpdDWFBNJ6xBwsm7lMxvomMaOyNM3T/Bg==';

$txRep = TxRep::fromTransactionEnvelopeXdrBase64($feeBumpXdr);
echo $txRep;
```

Output shows the nested structure:
```
type: ENVELOPE_TYPE_TX_FEE_BUMP
feeBump.tx.feeSource: GBSH2PI5ILVBQYGYCK3QLYHBCJK6W34W23QCSFOOBWSBOXG2EHXOQIV3
feeBump.tx.fee: 400
feeBump.tx.innerTx.type: ENVELOPE_TYPE_TX
feeBump.tx.innerTx.tx.sourceAccount: GAY6KB56YBPGGJU54RRLHGO3GBHEBZYTDY5VCBXEP4NLAUZNRSO4SSMH
feeBump.tx.innerTx.tx.fee: 100
...
```

## Converting Txrep to XDR

Parse Txrep text back into a transaction:

```php
<?php

use Soneso\StellarSDK\SEP\TxRep\TxRep;

$txRep = 'type: ENVELOPE_TYPE_TX
tx.sourceAccount: GAVRMS4QIOCC4QMOSKILOOOHCSO4FEKOXZPNLKFFN6W7SD2KUB7NBPLN
tx.fee: 100
tx.seqNum: 46489056724385793
tx.cond.type: PRECOND_TIME
tx.cond.timeBounds.minTime: 1535756672
tx.cond.timeBounds.maxTime: 1567292672
tx.memo.type: MEMO_TEXT
tx.memo.text: "Enjoy this transaction"
tx.operations.len: 1
tx.operations[0].sourceAccount._present: false
tx.operations[0].body.type: PAYMENT
tx.operations[0].body.paymentOp.destination: GBAF6NXN3DHSF357QBZLTBNWUTABKUODJXJYYE32ZDKA2QBM2H33IK6O
tx.operations[0].body.paymentOp.asset: USD:GAZFEVBSEGJJ63WPVVIWXLZLWN2JYZECECGT6GUNP4FJDVZVNXWQWMYI
tx.operations[0].body.paymentOp.amount: 400004000
tx.ext.v: 0
signatures.len: 1
signatures[0].hint: 4aa07ed0
signatures[0].signature: defb4f1fad1c279327b55af184fdcddf73f4f7a8cb40e7e534a71d73a05124ba369db7a6d31b47cafd118592246a8575e6c249ab94ec3768dedb6292221ce50c';

$xdrBase64 = TxRep::transactionEnvelopeXdrBase64FromTxRep($txRep);
echo $xdrBase64 . PHP_EOL;
```

## Txrep Format Reference

### Transaction Fields

```
tx.sourceAccount: G...           # Source account ID
tx.fee: 100                      # Total fee in stroops
tx.seqNum: 123456789             # Sequence number

# Time bounds (optional)
tx.cond.type: PRECOND_TIME
tx.cond.timeBounds.minTime: 0
tx.cond.timeBounds.maxTime: 0

# Memo types
tx.memo.type: MEMO_NONE
tx.memo.type: MEMO_TEXT
tx.memo.text: "Hello"
tx.memo.type: MEMO_ID
tx.memo.id: 12345
tx.memo.type: MEMO_HASH
tx.memo.hash: <64 hex chars>
tx.memo.type: MEMO_RETURN
tx.memo.retHash: <64 hex chars>
```

### Operation Types

Common operation formats:

```
# Payment
tx.operations[0].body.type: PAYMENT
tx.operations[0].body.paymentOp.destination: G...
tx.operations[0].body.paymentOp.asset: native
tx.operations[0].body.paymentOp.asset: USD:GISSUER...
tx.operations[0].body.paymentOp.amount: 10000000  # 1 unit = 10000000 stroops

# Create Account
tx.operations[0].body.type: CREATE_ACCOUNT
tx.operations[0].body.createAccountOp.destination: G...
tx.operations[0].body.createAccountOp.startingBalance: 100000000

# Change Trust
tx.operations[0].body.type: CHANGE_TRUST
tx.operations[0].body.changeTrustOp.line: USD:GISSUER...
tx.operations[0].body.changeTrustOp.limit: 10000000000

# Set Options
tx.operations[0].body.type: SET_OPTIONS
tx.operations[0].body.setOptionsOp.homeDomain: "example.com"
```

### Signatures

```
signatures.len: 2
signatures[0].hint: 4aa07ed0     # Last 4 bytes of public key (hex)
signatures[0].signature: def...  # 64-byte Ed25519 signature (hex)
signatures[1].hint: 1234abcd
signatures[1].signature: abc...
```

## Practical Examples

### Inspecting a Transaction Before Signing

```php
<?php

use Soneso\StellarSDK\SEP\TxRep\TxRep;

// Received from an external source
$xdrBase64 = 'AAAAAgAAAAD...';

// Convert to readable format
$txRep = TxRep::fromTransactionEnvelopeXdrBase64($xdrBase64);

// Display to user for review
echo "=== Transaction Details ===" . PHP_EOL;
echo $txRep . PHP_EOL;
echo "==========================" . PHP_EOL;

// Parse specific fields
$lines = explode(PHP_EOL, $txRep);
foreach ($lines as $line) {
    if (str_contains($line, 'tx.fee:')) {
        $fee = trim(explode(':', $line)[1]);
        echo "Fee: " . ((int)$fee / 10000000) . " XLM" . PHP_EOL;
    }
    if (str_contains($line, 'paymentOp.amount:')) {
        $amount = trim(explode(':', $line)[1]);
        echo "Amount: " . ((int)$amount / 10000000) . PHP_EOL;
    }
}
```

### Round-Trip Conversion

Verify that conversions are lossless:

```php
<?php

use Soneso\StellarSDK\SEP\TxRep\TxRep;

$originalXdr = 'AAAAAgAAAAArFkuQQ4QuQY6SkLc5xxSdwpFOvl7VqKVvrfkPSqB+0AAAAGQApSmNAAAAAQAAAAEAAAAAW4nJgAAAAABdav0AAAAAAQAAABZFbmpveSB0aGlzIHRyYW5zYWN0aW9uAAAAAAABAAAAAAAAAAEAAAAAQF827djPIu+/gHK5hbakwBVRw03TjBN6yNQNQCzR97QAAAABVVNEAAAAAAAyUlQyIZKfbs+tUWuvK7N0nGSCII0/Go1/CpHXNW3tCwAAAAAX15OgAAAAAAAAAAFKoH7QAAAAQN77Tx+tHCeTJ7Va8YT9zd9z9Peoy0Dn5TSnHXOgUSS6Np23ptMbR8r9EYWSJGqFdebCSauU7Ddo3ttikiIc5Qw=';

// XDR -> Txrep -> XDR
$txRep = TxRep::fromTransactionEnvelopeXdrBase64($originalXdr);
$reconstructedXdr = TxRep::transactionEnvelopeXdrBase64FromTxRep($txRep);

if ($originalXdr === $reconstructedXdr) {
    echo "Round-trip successful!" . PHP_EOL;
} else {
    echo "Mismatch detected" . PHP_EOL;
}
```

## Error Handling

```php
<?php

use InvalidArgumentException;
use Soneso\StellarSDK\SEP\TxRep\TxRep;

// Invalid XDR
try {
    $txRep = TxRep::fromTransactionEnvelopeXdrBase64('not-valid-base64!');
} catch (\Exception $e) {
    echo "Invalid XDR: " . $e->getMessage() . PHP_EOL;
}

// Invalid Txrep format
try {
    $invalidTxrep = 'this is not valid txrep';
    $xdr = TxRep::transactionEnvelopeXdrBase64FromTxRep($invalidTxrep);
} catch (InvalidArgumentException $e) {
    echo "Parse error: " . $e->getMessage() . PHP_EOL;
}

// Missing required fields
try {
    $incompleteTxrep = 'type: ENVELOPE_TYPE_TX
tx.sourceAccount: GAVRMS4QIOCC4QMOSKILOOOHCSO4FEKOXZPNLKFFN6W7SD2KUB7NBPLN';
    // Missing fee, seqNum, operations, etc.
    $xdr = TxRep::transactionEnvelopeXdrBase64FromTxRep($incompleteTxrep);
} catch (InvalidArgumentException $e) {
    echo "Missing field: " . $e->getMessage() . PHP_EOL;
}
```

## Notes on Amounts

Txrep shows amounts in stroops (1 XLM = 10,000,000 stroops):

```php
<?php

// Convert stroops to display units
$stroops = 400004000;
$displayAmount = $stroops / 10000000;  // 40.0004

// Convert display units to stroops
$amount = 25.5;
$stroops = (int)($amount * 10000000);  // 255000000
```

## Related SEPs

- [SEP-07 URI Scheme](sep-07.md) - Uses Txrep in the `replace` parameter for field substitution
