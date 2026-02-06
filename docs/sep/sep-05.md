# SEP-05: Key Derivation for Stellar

SEP-05 defines how to generate Stellar keypairs from mnemonic phrases (seed words) using hierarchical deterministic (HD) key derivation. This allows users to back up their wallet with a simple word list and derive multiple accounts from a single seed.

**When to use:** Building wallets that support mnemonic backup phrases, recovering accounts from seed words, or generating multiple related accounts from a single master seed.

See the [SEP-05 specification](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0005.md) for protocol details.

## Quick Example

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\SEP\Derivation\Mnemonic;

// Generate a new 24-word mnemonic
$mnemonic = Mnemonic::generate24WordsMnemonic();
echo implode(' ', $mnemonic->words) . PHP_EOL;

// Derive the first account
$keyPair = KeyPair::fromMnemonic($mnemonic, 0);
echo "Account: " . $keyPair->getAccountId() . PHP_EOL;
```

## Generating Mnemonics

### 12-Word Mnemonic

Standard security for most use cases:

```php
<?php

use Soneso\StellarSDK\SEP\Derivation\Mnemonic;

$mnemonic = Mnemonic::generate12WordsMnemonic();
echo implode(' ', $mnemonic->words) . PHP_EOL;
// bind struggle sausage repair machine fee setup finish transfer stamp benefit economy
```

### 24-Word Mnemonic

Higher security for larger holdings:

```php
<?php

use Soneso\StellarSDK\SEP\Derivation\Mnemonic;

$mnemonic = Mnemonic::generate24WordsMnemonic();
echo implode(' ', $mnemonic->words) . PHP_EOL;
// cabbage verb depart erase cable eye crowd approve tower umbrella violin tube 
// island tortoise suspect resemble harbor twelve romance away rug current robust practice
```

### 15-Word Mnemonic

```php
<?php

use Soneso\StellarSDK\SEP\Derivation\Mnemonic;

$mnemonic = Mnemonic::generate15WordsMnemonic();
echo implode(' ', $mnemonic->words) . PHP_EOL;
```

## Mnemonics in Other Languages

The SDK supports BIP-39 word lists in multiple languages:

```php
<?php

use Soneso\StellarSDK\SEP\Derivation\Mnemonic;
use Soneso\StellarSDK\SEP\Derivation\WordList;

// French
$french = Mnemonic::generate12WordsMnemonic(WordList::LANGUAGE_FRENCH);
echo implode(' ', $french->words) . PHP_EOL;
// traction maniable punaise flasque digital maussade usuel joueur volcan vaccin tasse concert

// Korean
$korean = Mnemonic::generate24WordsMnemonic(WordList::LANGUAGE_KOREAN);
echo implode(' ', $korean->words) . PHP_EOL;

// Spanish
$spanish = Mnemonic::generate12WordsMnemonic(WordList::LANGUAGE_SPANISH);
echo implode(' ', $spanish->words) . PHP_EOL;
```

**Supported languages:**
- `WordList::LANGUAGE_ENGLISH` (default)
- `WordList::LANGUAGE_FRENCH`
- `WordList::LANGUAGE_SPANISH`
- `WordList::LANGUAGE_ITALIAN`
- `WordList::LANGUAGE_KOREAN`
- `WordList::LANGUAGE_JAPANESE`
- `WordList::LANGUAGE_CHINESE_SIMPLIFIED`
- `WordList::LANGUAGE_CHINESE_TRADITIONAL`
- `WordList::LANGUAGE_MALAY`

## Deriving Keypairs from Mnemonics

### Basic Derivation

The derivation path is `m/44'/148'/index'` where 148 is Stellar's coin type:

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\SEP\Derivation\Mnemonic;

$words = 'shell green recycle learn purchase able oxygen right echo claim hill again hidden evidence nice decade panic enemy cake version say furnace garment glue';
$mnemonic = Mnemonic::mnemonicFromWords($words);

// First account (index 0)
$keyPair0 = KeyPair::fromMnemonic($mnemonic, 0);
echo "Account 0: " . $keyPair0->getAccountId() . PHP_EOL;
// GCVSEBHB6CTMEHUHIUY4DDFMWQ7PJTHFZGOK2JUD5EG2ARNVS6S22E3K

echo "Secret 0: " . $keyPair0->getSecretSeed() . PHP_EOL;
// SATLGMF3SP2V47SJLBFVKZZJQARDOBDQ7DNSSPUV7NLQNPN3QB7M74XH

// Second account (index 1)  
$keyPair1 = KeyPair::fromMnemonic($mnemonic, 1);
echo "Account 1: " . $keyPair1->getAccountId() . PHP_EOL;
// GBPHPX7SZKYEDV5CVOA5JOJE2RHJJDCJMRWMV4KBOIE5VSDJ6VAESR2W
```

### Derivation with Passphrase

An optional passphrase adds extra security. Different passphrases produce completely different accounts:

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\SEP\Derivation\Mnemonic;

$words = 'cable spray genius state float twenty onion head street palace net private method loan turn phrase state blanket interest dry amazing dress blast tube';
$mnemonic = Mnemonic::mnemonicFromWords($words);
$passphrase = 'p4ssphr4se';

// With passphrase
$keyPair0 = KeyPair::fromMnemonic($mnemonic, 0, $passphrase);
echo "Account: " . $keyPair0->getAccountId() . PHP_EOL;
// GDAHPZ2NSYIIHZXM56Y36SBVTV5QKFIZGYMMBHOU53ETUSWTP62B63EQ

$keyPair1 = KeyPair::fromMnemonic($mnemonic, 1, $passphrase);
echo "Account: " . $keyPair1->getAccountId() . PHP_EOL;
// GDY47CJARRHHL66JH3RJURDYXAMIQ5DMXZLP3TDAUJ6IN2GUOFX4OJOC
```

### Derivation from Non-English Mnemonic

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\SEP\Derivation\Mnemonic;
use Soneso\StellarSDK\SEP\Derivation\WordList;

$koreanWords = '절차 튀김 건강 평가 테스트 민족 몹시 어른 주민 형제 발레 만점 산길 물고기 방면 여학생 결국 수명 애정 정치 관심 상자 축하 고무신';
$mnemonic = Mnemonic::mnemonicFromWords($koreanWords, WordList::LANGUAGE_KOREAN);

$keyPair = KeyPair::fromMnemonic($mnemonic, 0);
echo "Account: " . $keyPair->getAccountId() . PHP_EOL;
// GBEAH7ADD5NRYA5YGXDMSWB7PK7J44DYG5I7SVL2FYHCPH5ZH4EJC3YP
```

## Working with BIP-39 Seeds

### Getting the Seed Hex

If you need the raw BIP-39 seed (for example, for use with other tools):

```php
<?php

use Soneso\StellarSDK\SEP\Derivation\Mnemonic;
use Soneso\StellarSDK\SEP\Derivation\WordList;

$mnemonic = Mnemonic::generate24WordsMnemonic(WordList::LANGUAGE_ITALIAN);

// BIP-39 seed (128 hex characters)
$seedHex = $mnemonic->bip39SeedHex();
echo "Seed: " . $seedHex . PHP_EOL;

// With passphrase
$seedWithPassphrase = $mnemonic->bip39SeedHex('p4ssphr4se');
echo "Seed (with passphrase): " . $seedWithPassphrase . PHP_EOL;
```

### Getting the m/44'/148' Key

The Stellar derivation path key:

```php
<?php

use Soneso\StellarSDK\SEP\Derivation\Mnemonic;

$mnemonic = Mnemonic::generate24WordsMnemonic();

$keyHex = $mnemonic->m44148keyHex();
echo "Key: " . $keyHex . PHP_EOL;

// With passphrase
$keyWithPassphrase = $mnemonic->m44148keyHex('p4ssphr4se');
echo "Key (with passphrase): " . $keyWithPassphrase . PHP_EOL;
```

### Deriving from Seed Hex Directly

If you already have the BIP-39 seed as hex:

```php
<?php

use Soneso\StellarSDK\Crypto\KeyPair;

$bip39SeedHex = 'e4a5a632e70943ae7f07659df1332160937fad82587216a4c64315a0fb39497ee4a01f76ddab4cba68147977f3a147b6ad584c41808e8238a07f6cc4b582f186';

$keyPair0 = KeyPair::fromBip39SeedHex($bip39SeedHex, 0);
echo "Account: " . $keyPair0->getAccountId() . PHP_EOL;
// GDRXE2BQUC3AZNPVFSCEZ76NJ3WWL25FYFK6RGZGIEKWE4SOOHSUJUJ6

$keyPair1 = KeyPair::fromBip39SeedHex($bip39SeedHex, 1);
echo "Account: " . $keyPair1->getAccountId() . PHP_EOL;
// GBAW5XGWORWVFE2XTJYDTLDHXTY2Q2MO73HYCGB3XMFMQ562Q2W2GJQX
```

## Restoring from Words

Convert a space-separated word string back to a Mnemonic object:

```php
<?php

use Exception;
use Soneso\StellarSDK\Crypto\KeyPair;
use Soneso\StellarSDK\SEP\Derivation\Mnemonic;

$words = 'illness beef work lemon dove route health way penalty sort merge purchase';

try {
    // Third parameter enables checksum verification (default: true)
    $mnemonic = Mnemonic::mnemonicFromWords($words);
    
    $keyPair = KeyPair::fromMnemonic($mnemonic, 0);
    echo "Recovered account: " . $keyPair->getAccountId() . PHP_EOL;
} catch (Exception $e) {
    echo "Invalid mnemonic: " . $e->getMessage() . PHP_EOL;
}
```

## Error Handling

```php
<?php

use Exception;
use Soneso\StellarSDK\SEP\Derivation\Mnemonic;
use Soneso\StellarSDK\SEP\Derivation\WordList;

// Invalid word in mnemonic
try {
    $mnemonic = Mnemonic::mnemonicFromWords('invalid words that are not in the wordlist');
} catch (Exception $e) {
    echo "Invalid words: " . $e->getMessage() . PHP_EOL;
}

// Wrong word count
try {
    $mnemonic = Mnemonic::mnemonicFromWords('one two three');
} catch (Exception $e) {
    echo "Invalid word count: " . $e->getMessage() . PHP_EOL;
}

// Invalid checksum (words are valid but combination is wrong)
try {
    $mnemonic = Mnemonic::mnemonicFromWords(
        'abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon',
        WordList::LANGUAGE_ENGLISH,
        true  // Enable checksum verification
    );
} catch (Exception $e) {
    echo "Checksum failed: " . $e->getMessage() . PHP_EOL;
}

// Unsupported language
try {
    $mnemonic = Mnemonic::generate12WordsMnemonic('klingon');
} catch (Exception $e) {
    echo "Language error: " . $e->getMessage() . PHP_EOL;
}
```

## Security Notes

- **Never share your mnemonic** - Anyone with your words can access all derived accounts
- **Store mnemonics offline** - Write them on paper, use a hardware wallet, or use encrypted storage
- **Use passphrases for extra security** - A passphrase creates a completely different set of accounts
- **Verify checksums** - The SDK validates mnemonics by default to catch typos
- **Test recovery** - Before using an account for real funds, verify you can recover it from the mnemonic

## Related SEPs

- [SEP-30 Account Recovery](sep-30.md) - Uses mnemonics for account recovery flows
