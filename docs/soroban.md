# Soroban Smart Contracts

Deploy and interact with Soroban smart contracts using the Stellar PHP SDK.

**Protocol details**: [Soroban Documentation](https://developers.stellar.org/docs/smart-contracts)

## Quick Start

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\Soroban\Contract\ClientOptions;
use Soneso\StellarSDK\Soroban\Contract\DeployRequest;
use Soneso\StellarSDK\Soroban\Contract\InstallRequest;
use Soneso\StellarSDK\Soroban\Contract\SorobanClient;
use Soneso\StellarSDK\Xdr\XdrSCVal;

$keyPair = KeyPair::fromSeed('SXXX...');
$rpcUrl = 'https://soroban-testnet.stellar.org';

// 1. Install WASM
$wasmHash = SorobanClient::install(new InstallRequest(
    wasmBytes: file_get_contents('hello.wasm'),
    rpcUrl: $rpcUrl,
    network: Network::testnet(),
    sourceAccountKeyPair: $keyPair
));

// 2. Deploy
$client = SorobanClient::deploy(new DeployRequest(
    rpcUrl: $rpcUrl,
    network: Network::testnet(),
    sourceAccountKeyPair: $keyPair,
    wasmHash: $wasmHash
));

// 3. Invoke
$result = $client->invokeMethod('hello', [XdrSCVal::forSymbol('World')]);
echo $result->vec[0]->sym . ', ' . $result->vec[1]->sym; // Hello, World
```

## SorobanServer

Direct communication with Soroban RPC nodes.

```php
<?php

use Soneso\StellarSDK\Soroban\Responses\GetHealthResponse;
use Soneso\StellarSDK\Soroban\SorobanServer;
use Soneso\StellarSDK\Xdr\XdrContractDataDurability;
use Soneso\StellarSDK\Xdr\XdrSCVal;

$server = new SorobanServer('https://soroban-testnet.stellar.org');
$server->enableLogging = true; // Debug mode

// Health check
if ($server->getHealth()->status === GetHealthResponse::HEALTHY) {
    echo "Node healthy\n";
}

// Network info
$network = $server->getNetwork();
echo "Passphrase: {$network->passphrase}\n";

// Latest ledger
$ledger = $server->getLatestLedger();
echo "Sequence: {$ledger->sequence}\n";

// Account data
$account = $server->getAccount('GABC...');
echo "Sequence: {$account->getSequenceNumber()}\n";

// Contract data
$entry = $server->getContractData(
    contractId: 'CCXYZ...',
    key: XdrSCVal::forSymbol('counter'),
    durability: XdrContractDataDurability::PERSISTENT()
);

// Contract info (spec entries, metadata)
$info = $server->loadContractInfoForContractId('CCXYZ...');
```

## SorobanClient

High-level API for contract interaction.

### Creating a Client

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\Soroban\Contract\ClientOptions;
use Soneso\StellarSDK\Soroban\Contract\SorobanClient;

$client = SorobanClient::forClientOptions(new ClientOptions(
    sourceAccountKeyPair: KeyPair::fromSeed('SXXX...'),
    contractId: 'CCXYZ...',
    network: Network::testnet(),
    rpcUrl: 'https://soroban-testnet.stellar.org'
));

$methodNames = $client->getMethodNames();
$spec = $client->getContractSpec();
```

### Invoking Methods

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\Soroban\Address;
use Soneso\StellarSDK\Soroban\Contract\ClientOptions;
use Soneso\StellarSDK\Soroban\Contract\MethodOptions;
use Soneso\StellarSDK\Soroban\Contract\SorobanClient;
use Soneso\StellarSDK\Xdr\XdrSCVal;

$client = SorobanClient::forClientOptions(new ClientOptions(
    sourceAccountKeyPair: KeyPair::fromSeed('SXXX...'),
    contractId: 'CCXYZ...',
    network: Network::testnet(),
    rpcUrl: 'https://soroban-testnet.stellar.org'
));

// Read-only (returns simulation result)
$balance = $client->invokeMethod('balance', [
    Address::fromAccountId('GABC...')->toXdrSCVal()
]);

// Write (auto-signs and submits)
$result = $client->invokeMethod('transfer', [
    Address::fromAccountId('GFROM...')->toXdrSCVal(),
    Address::fromAccountId('GTO...')->toXdrSCVal(),
    XdrSCVal::forI128BigInt(1000)
]);

// Custom options
$methodOptions = new MethodOptions(
    fee: 10000,
    timeoutInSeconds: 30,
    restore: true  // Auto-restore expired state
);
$result = $client->invokeMethod('expensive_op', [], methodOptions: $methodOptions);
```

## Installing and Deploying

### Installation

Upload WASM bytecode (do once per contract version):

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\Soroban\Contract\InstallRequest;
use Soneso\StellarSDK\Soroban\Contract\SorobanClient;

$wasmHash = SorobanClient::install(new InstallRequest(
    wasmBytes: file_get_contents('contract.wasm'),
    rpcUrl: 'https://soroban-testnet.stellar.org',
    network: Network::testnet(),
    sourceAccountKeyPair: KeyPair::fromSeed('SXXX...')
));

// Returns existing hash if already installed (use force: true to re-install)
```

### Deployment

Create contract instance from installed WASM:

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\Soroban\Contract\DeployRequest;
use Soneso\StellarSDK\Soroban\Contract\SorobanClient;
use Soneso\StellarSDK\Xdr\XdrSCVal;

// Basic deployment
$client = SorobanClient::deploy(new DeployRequest(
    rpcUrl: 'https://soroban-testnet.stellar.org',
    network: Network::testnet(),
    sourceAccountKeyPair: KeyPair::fromSeed('SXXX...'),
    wasmHash: $wasmHash
));

// With constructor (protocol 22+)
$client = SorobanClient::deploy(new DeployRequest(
    rpcUrl: 'https://soroban-testnet.stellar.org',
    network: Network::testnet(),
    sourceAccountKeyPair: KeyPair::fromSeed('SXXX...'),
    wasmHash: $wasmHash,
    constructorArgs: [XdrSCVal::forSymbol('MyToken'), XdrSCVal::forU32(8)]
));
```

## AssembledTransaction

Fine-grained control over transaction lifecycle.

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Memo;
use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\Soroban\Contract\ClientOptions;
use Soneso\StellarSDK\Soroban\Contract\MethodOptions;
use Soneso\StellarSDK\Soroban\Contract\SorobanClient;
use Soneso\StellarSDK\Soroban\Responses\GetTransactionResponse;
use Soneso\StellarSDK\Xdr\XdrSCVal;

$client = SorobanClient::forClientOptions(new ClientOptions(
    sourceAccountKeyPair: KeyPair::fromSeed('SXXX...'),
    contractId: 'CCXYZ...',
    network: Network::testnet(),
    rpcUrl: 'https://soroban-testnet.stellar.org'
));

// Build without submitting
$tx = $client->buildInvokeMethodTx('transfer', [XdrSCVal::forSymbol('test')]);

// Access simulation
$simData = $tx->getSimulationData();
$returnValue = $simData->returnedValue;

// Check if read-only
if ($tx->isReadCall()) {
    $result = $simData->returnedValue;
} else {
    $response = $tx->signAndSend();
    $result = $response->getResultValue();
}

// Modify before simulation
$tx = $client->buildInvokeMethodTx(
    'my_method', 
    [], 
    methodOptions: new MethodOptions(simulate: false)
);
$tx->raw->addMemo(Memo::text('My memo'));
$tx->simulate();
$response = $tx->signAndSend();
```

## Authorization

Handle multi-party signing for operations like swaps and transfers.

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\Soroban\Address;
use Soneso\StellarSDK\Soroban\Contract\ClientOptions;
use Soneso\StellarSDK\Soroban\Contract\SorobanClient;
use Soneso\StellarSDK\Soroban\SorobanAuthorizationEntry;
use Soneso\StellarSDK\Xdr\XdrSCVal;

$alice = KeyPair::fromSeed('SALICE...');
$bob = KeyPair::fromSeed('SBOB...');

$client = SorobanClient::forClientOptions(new ClientOptions(
    sourceAccountKeyPair: $alice,
    contractId: 'CSWAP...',
    network: Network::testnet(),
    rpcUrl: 'https://soroban-testnet.stellar.org'
));

$tx = $client->buildInvokeMethodTx('swap', [
    Address::fromAccountId($alice->getAccountId())->toXdrSCVal(),
    Address::fromAccountId($bob->getAccountId())->toXdrSCVal(),
    XdrSCVal::forI128BigInt(1000),
    XdrSCVal::forI128BigInt(500)
]);

// Check who needs to sign
$neededSigners = $tx->needsNonInvokerSigningBy();

// Sign Bob's auth entries (same machine)
$tx->signAuthEntries(signerKeyPair: $bob);

// Remote signing (Bob's key on another server)
$bobPublicKey = KeyPair::fromAccountId('GBOB...');
$tx->signAuthEntries(
    signerKeyPair: $bobPublicKey,
    authorizeEntryCallback: function (
        SorobanAuthorizationEntry $entry,
        Network $network
    ): SorobanAuthorizationEntry {
        $base64Entry = $entry->toBase64Xdr();
        $signedBase64 = sendToRemoteServer($base64Entry); // Your implementation
        return SorobanAuthorizationEntry::fromBase64Xdr($signedBase64);
    }
);

// Submit (Alice signs envelope)
$response = $tx->signAndSend();
```

## Type Conversions

### Creating XdrSCVal

```php
<?php

use Soneso\StellarSDK\Soroban\Address;
use Soneso\StellarSDK\Xdr\XdrInt128Parts;
use Soneso\StellarSDK\Xdr\XdrSCMapEntry;
use Soneso\StellarSDK\Xdr\XdrSCVal;

// Primitives
$bool = XdrSCVal::forBool(true);
$u32 = XdrSCVal::forU32(42);
$i64 = XdrSCVal::forI64(-1000000);
$string = XdrSCVal::forString('Hello');
$symbol = XdrSCVal::forSymbol('transfer');
$bytes = XdrSCVal::forBytes(hex2bin('deadbeef'));
$void = XdrSCVal::forVoid();

// BigInt (128/256-bit)
$u128 = XdrSCVal::forU128BigInt('340282366920938463463374607431768211455');
$i128 = XdrSCVal::forI128BigInt(-1000);
$u256 = XdrSCVal::forU256BigInt(gmp_pow(2, 200));

// Legacy 128-bit
$i128Legacy = XdrSCVal::forI128(new XdrInt128Parts(hi: 0, lo: 1000));

// Addresses
$account = Address::fromAccountId('GABC...')->toXdrSCVal();
$contract = Address::fromContractId('CABC...')->toXdrSCVal();

// Vector
$vec = XdrSCVal::forVec([
    XdrSCVal::forSymbol('a'),
    XdrSCVal::forSymbol('b')
]);

// Map
$map = XdrSCVal::forMap([
    new XdrSCMapEntry(XdrSCVal::forSymbol('name'), XdrSCVal::forString('Alice')),
    new XdrSCMapEntry(XdrSCVal::forSymbol('age'), XdrSCVal::forU32(30))
]);
```

### Using ContractSpec

Auto-convert native PHP values based on contract types:

```php
<?php

use Soneso\StellarSDK\Soroban\Contract\ContractSpec;

$spec = $client->getContractSpec();

// Convert function arguments
$args = $spec->funcArgsToXdrSCValues('swap', [
    'a' => 'GALICE...',     // Auto-converts to Address
    'b' => 'GBOB...',
    'token_a' => 'CTOKEN1...',
    'token_b' => 'CTOKEN2...',
    'amount_a' => 1000,      // Auto-converts to i128
    'min_b_for_a' => 950,
    'amount_b' => 500,
    'min_a_for_b' => 450
]);

// Find functions and types
$functions = $spec->funcs();
$swapFunc = $spec->getFunc('swap');
$myUnion = $spec->findEntry('myUnion');
```

### Reading Return Values

```php
<?php

$result = $client->invokeMethod('get_data', []);

// By type
$count = $result->u32;
$name = $result->str;
$flag = $result->b;

// BigInt
$bigValue = $result->toBigInt();
echo gmp_strval($bigValue);

// Vector elements
foreach ($result->vec as $item) {
    echo $item->sym . "\n";
}
```

## Events

```php
<?php

use Soneso\StellarSDK\Crypto\StrKey;
use Soneso\StellarSDK\Soroban\Requests\EventFilter;
use Soneso\StellarSDK\Soroban\Requests\EventFilters;
use Soneso\StellarSDK\Soroban\Requests\GetEventsRequest;
use Soneso\StellarSDK\Soroban\Requests\TopicFilter;
use Soneso\StellarSDK\Soroban\Requests\TopicFilters;
use Soneso\StellarSDK\Soroban\SorobanServer;
use Soneso\StellarSDK\Xdr\XdrSCVal;

$server = new SorobanServer('https://soroban-testnet.stellar.org');

// Contract ID must be C-prefixed
$contractId = StrKey::encodeContractIdHex($hexContractId);

$topicFilter = new TopicFilter([
    '*',
    XdrSCVal::forSymbol('transfer')->toBase64Xdr()
]);

$eventFilter = new EventFilter(
    type: 'contract',
    contractIds: [$contractId],
    topics: new TopicFilters($topicFilter)
);

$filters = new EventFilters();
$filters->add($eventFilter);

$response = $server->getEvents(new GetEventsRequest(12345, $filters));

foreach ($response->events as $event) {
    echo "Ledger: {$event->ledger}, Value: {$event->value}\n";
}
```

## Error Handling

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\Soroban\Contract\ClientOptions;
use Soneso\StellarSDK\Soroban\Contract\MethodOptions;
use Soneso\StellarSDK\Soroban\Contract\SorobanClient;
use Soneso\StellarSDK\Soroban\Responses\GetTransactionResponse;

$client = SorobanClient::forClientOptions(new ClientOptions(
    sourceAccountKeyPair: KeyPair::fromSeed('SXXX...'),
    contractId: 'CCXYZ...',
    network: Network::testnet(),
    rpcUrl: 'https://soroban-testnet.stellar.org',
    enableServerLogging: true // Debug
));

try {
    $tx = $client->buildInvokeMethodTx('nonexistent', []);
} catch (\Exception $e) {
    echo "Error: {$e->getMessage()}\n";
}

// Simulation errors
$tx = $client->buildInvokeMethodTx('my_method', []);
if ($tx->simulationResponse->error !== null) {
    echo "Simulation failed: {$tx->simulationResponse->error}\n";
}

// Transaction failures
try {
    $response = $tx->signAndSend();
    if ($response->status === GetTransactionResponse::STATUS_FAILED) {
        echo "Failed: {$response->resultXdr}\n";
    }
} catch (\Exception $e) {
    echo "Submission error: {$e->getMessage()}\n";
}

// Auto-restore expired state
$result = $client->invokeMethod(
    'my_method',
    [],
    methodOptions: new MethodOptions(restore: true)
);
```

## Contract Bindings

Generate type-safe PHP classes using [stellar-contract-bindings](https://github.com/lightsail-network/stellar-contract-bindings):

```bash
pip install stellar-contract-bindings

stellar-contract-bindings php \
  --contract-id YOUR_CONTRACT_ID \
  --rpc-url https://soroban-testnet.stellar.org \
  --output ./generated \
  --namespace MyApp\\Contracts \
  --class-name TokenClient
```

Or use the [web interface](https://stellar-contract-bindings.fly.dev/).

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\Soroban\Contract\ClientOptions;
use MyApp\Contracts\TokenClient;

$client = TokenClient::forClientOptions(new ClientOptions(
    sourceAccountKeyPair: KeyPair::fromSeed('SXXX...'),
    contractId: 'CTOKEN...',
    network: Network::testnet(),
    rpcUrl: 'https://soroban-testnet.stellar.org'
));

// Type-safe calls
$balance = $client->balance('GABC...');
$client->transfer('GFROM...', 'GTO...', 1000);
```

## Low-Level Deployment

For SAC or custom workflows:

```php
<?php

use Soneso\StellarSDK\Asset;
use Soneso\StellarSDK\CreateContractHostFunction;
use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\DeploySACWithAssetHostFunction;
use Soneso\StellarSDK\InvokeHostFunctionOperationBuilder;
use Soneso\StellarSDK\Network;
use Soneso\StellarSDK\Soroban\Address;
use Soneso\StellarSDK\Soroban\Requests\SimulateTransactionRequest;
use Soneso\StellarSDK\Soroban\SorobanServer;
use Soneso\StellarSDK\TransactionBuilder;
use Soneso\StellarSDK\UploadContractWasmHostFunction;

$keyPair = KeyPair::fromSeed('SXXX...');
$server = new SorobanServer('https://soroban-testnet.stellar.org');

// Upload WASM
$uploadOp = (new InvokeHostFunctionOperationBuilder(
    new UploadContractWasmHostFunction(file_get_contents('contract.wasm'))
))->build();

$account = $server->getAccount($keyPair->getAccountId());
$tx = (new TransactionBuilder($account))->addOperation($uploadOp)->build();

$sim = $server->simulateTransaction(new SimulateTransactionRequest($tx));
$tx->setSorobanTransactionData($sim->transactionData);
$tx->addResourceFee($sim->minResourceFee);
$tx->sign($keyPair, Network::testnet());

$server->sendTransaction($tx);
// Poll getTransaction until STATUS_SUCCESS, then get $wasmHash

// Create instance
$createOp = (new InvokeHostFunctionOperationBuilder(
    new CreateContractHostFunction(
        Address::fromAccountId($keyPair->getAccountId()),
        $wasmHash
    )
))->build();
// ... simulate, sign, send

// Stellar Asset Contract
$asset = Asset::createNonNativeAsset('USDC', 'GISSUER...');
$sacOp = (new InvokeHostFunctionOperationBuilder(
    new DeploySACWithAssetHostFunction($asset)
))->build();
```

## Further Reading

- [Soroban Examples](https://github.com/stellar/soroban-examples) - Official contracts
- [Soroban Docs](https://developers.stellar.org/docs/smart-contracts) - Protocol details
- [RPC API Reference](https://developers.stellar.org/docs/data/rpc/api-reference)
- [SorobanClientTest.php](https://github.com/Soneso/stellar-php-sdk/blob/main/Soneso/StellarSDKTests/SorobanClientTest.php) - Test cases

---

[← Back to SDK Usage](sdk-usage.md) | [SEP Protocols →](sep/README.md)
