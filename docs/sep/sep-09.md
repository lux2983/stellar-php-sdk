# SEP-09: Standard KYC Fields

SEP-09 defines a standard vocabulary for KYC (Know Your Customer) and AML (Anti-Money Laundering) data fields. When different services need to exchange customer information—deposits, withdrawals, cross-border payments—they use these field names so everyone speaks the same language.

**Use SEP-09 when:**
- Submitting KYC data via SEP-12
- Providing customer info for SEP-24 interactive flows
- Sending receiver details for SEP-31 cross-border payments
- Building anchor services that collect customer information

**Spec:** [SEP-0009](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0009.md)

## Quick Example

```php
<?php

use Soneso\StellarSDK\SEP\StandardKYCFields\NaturalPersonKYCFields;
use Soneso\StellarSDK\SEP\StandardKYCFields\StandardKYCFields;

// Build KYC fields for an individual
$person = new NaturalPersonKYCFields();
$person->firstName = 'John';
$person->lastName = 'Doe';
$person->emailAddress = 'john@example.com';
$person->birthDate = '1990-05-15';

// Wrap in container
$kyc = new StandardKYCFields();
$kyc->naturalPersonKYCFields = $person;

// Get fields as array for API submission
$fields = $person->fields();
// Returns: ['first_name' => 'John', 'last_name' => 'Doe', ...]
```

## Detailed Usage

### Natural Person Fields

For individual customers:

```php
<?php

use DateTime;
use Soneso\StellarSDK\SEP\StandardKYCFields\NaturalPersonKYCFields;

$person = new NaturalPersonKYCFields();

// Personal identification
$person->firstName = 'Maria';
$person->lastName = 'Garcia';
$person->additionalName = 'Elena';  // Middle name
$person->birthDate = '1985-03-20';
$person->birthPlace = 'Madrid, Spain';
$person->birthCountryCode = 'ESP';  // ISO 3166-1 alpha-3
$person->sex = 'female';

// Contact information
$person->emailAddress = 'maria@example.com';
$person->mobileNumber = '+34612345678';  // E.164 format

// Current address
$person->addressCountryCode = 'ESP';
$person->stateOrProvince = 'Madrid';
$person->city = 'Madrid';
$person->postalCode = '28001';
$person->address = "Calle Mayor 10\n28001 Madrid\nSpain";

// Employment
$person->occupation = 2511;  // ISCO-08 code (Software developer)
$person->employerName = 'Tech Corp';
$person->employerAddress = 'Paseo de la Castellana 50, Madrid';

// Tax information
$person->taxId = '12345678Z';
$person->taxIdName = 'NIF';

// Identity document
$person->idType = 'passport';
$person->idNumber = 'AB1234567';
$person->idCountryCode = 'ESP';
$person->idIssueDate = new DateTime('2020-01-15');
$person->idExpirationDate = new DateTime('2030-01-14');

// Other
$person->languageCode = 'es';  // ISO 639-1
$person->ipAddress = '192.168.1.1';
$person->referralId = 'partner-12345';

// Convert to array for API submission
$fieldData = $person->fields();
```

### Document Uploads

Binary files (photos, documents) are handled separately via `files()`:

```php
<?php

use Soneso\StellarSDK\SEP\StandardKYCFields\NaturalPersonKYCFields;

$person = new NaturalPersonKYCFields();
$person->firstName = 'John';
$person->lastName = 'Doe';

// Base64-encode document images
$person->photoIdFront = base64_encode(file_get_contents('/path/to/passport-front.jpg'));
$person->photoIdBack = base64_encode(file_get_contents('/path/to/passport-back.jpg'));
$person->photoProofResidence = base64_encode(file_get_contents('/path/to/utility-bill.pdf'));
$person->proofOfIncome = base64_encode(file_get_contents('/path/to/payslip.pdf'));
$person->proofOfLiveness = base64_encode(file_get_contents('/path/to/selfie-video.mp4'));

// Get text fields and file fields separately
$textFields = $person->fields();
$fileFields = $person->files();
```

### Organization Fields

For business customers:

```php
<?php

use Soneso\StellarSDK\SEP\StandardKYCFields\OrganizationKYCFields;
use Soneso\StellarSDK\SEP\StandardKYCFields\StandardKYCFields;

$org = new OrganizationKYCFields();

// Corporate identity
$org->name = 'Acme Corporation S.L.';
$org->VATNumber = 'ESB12345678';
$org->registrationNumber = 'B-12345678';
$org->registrationDate = '2015-06-01';
$org->registeredAddress = 'Calle Gran Via 100, 28013 Madrid, Spain';

// Corporate structure
$org->numberOfShareholders = 3;
$org->shareholderName = 'John Smith';  // Query recursively for all UBOs
$org->directorName = 'Jane Doe';

// Contact details
$org->addressCountryCode = 'ESP';
$org->stateOrProvince = 'Madrid';
$org->city = 'Madrid';
$org->postalCode = '28013';
$org->website = 'https://acme-corp.example.com';
$org->email = 'compliance@acme-corp.example.com';
$org->phone = '+34911234567';

// Documents (base64 encoded)
$org->photoIncorporationDoc = base64_encode(file_get_contents('/path/to/incorporation.pdf'));
$org->photoProofAddress = base64_encode(file_get_contents('/path/to/business-utility-bill.pdf'));

// Wrap in container
$kyc = new StandardKYCFields();
$kyc->organizationKYCFields = $org;

// Organization fields use 'organization.' prefix
$fieldData = $org->fields();
// Returns: ['organization.name' => 'Acme Corporation S.L.', ...]
```

### Financial Account Fields

Bank accounts, crypto addresses, and mobile money for both individuals and organizations:

```php
<?php

use Soneso\StellarSDK\SEP\StandardKYCFields\FinancialAccountKYCFields;
use Soneso\StellarSDK\SEP\StandardKYCFields\NaturalPersonKYCFields;

$person = new NaturalPersonKYCFields();
$person->firstName = 'John';
$person->lastName = 'Doe';

// Add bank account details
$bankAccount = new FinancialAccountKYCFields();
$bankAccount->bankName = 'First National Bank';
$bankAccount->bankAccountType = 'checking';
$bankAccount->bankAccountNumber = '123456789012';
$bankAccount->bankNumber = '021000021';  // Routing number (US)
$bankAccount->bankBranchNumber = '001';

// Regional bank formats
$bankAccount->clabeNumber = '012345678901234567';  // Mexico
$bankAccount->cbuNumber = '0123456789012345678901';  // Argentina
$bankAccount->cbuAlias = 'john.doe.acme';

// Mobile money
$bankAccount->mobileMoneyNumber = '+254712345678';
$bankAccount->mobileMoneyProvider = 'M-Pesa';

// Crypto
$bankAccount->cryptoAddress = 'GBH4TZYZ4IRCPO44CBOLFUHULU2WGALXTAVESQA6432MBJMABBB4GIYI';
$bankAccount->externalTransferMemo = 'user-12345';

// Attach to person
$person->financialAccountKYCFields = $bankAccount;

// fields() includes nested financial account fields
$allFields = $person->fields();
```

### Card Fields

Credit and debit card information:

```php
<?php

use Soneso\StellarSDK\SEP\StandardKYCFields\CardKYCFields;
use Soneso\StellarSDK\SEP\StandardKYCFields\NaturalPersonKYCFields;

$person = new NaturalPersonKYCFields();
$person->firstName = 'John';
$person->lastName = 'Doe';

$card = new CardKYCFields();

// Card details
$card->number = '4111111111111111';
$card->expirationDate = '29-11';  // YY-MM format
$card->cvc = '123';
$card->holderName = 'JOHN DOE';
$card->network = 'Visa';

// Billing address
$card->address = "123 Main St\nApt 4B";
$card->city = 'New York';
$card->stateOrProvince = 'NY';
$card->postalCode = '10001';
$card->countryCode = 'US';  // ISO 3166-1 alpha-2

// Prefer tokens over raw card numbers
$card->token = 'tok_visa_4242';  // From Stripe, etc.

$person->cardKYCFields = $card;

// Card fields use 'card.' prefix
$allFields = $person->fields();
// Includes: ['card.number' => '4111...', 'card.expiration_date' => '29-11', ...]
```

### Combining with Organizations

Organizations can also have financial accounts and cards:

```php
<?php

use Soneso\StellarSDK\SEP\StandardKYCFields\FinancialAccountKYCFields;
use Soneso\StellarSDK\SEP\StandardKYCFields\OrganizationKYCFields;

$org = new OrganizationKYCFields();
$org->name = 'Acme Corp';
$org->VATNumber = 'US12-3456789';

$bankAccount = new FinancialAccountKYCFields();
$bankAccount->bankName = 'Business Bank';
$bankAccount->bankAccountNumber = '9876543210';
$bankAccount->bankNumber = '021000021';

$org->financialAccountKYCFields = $bankAccount;

// Organization financial fields get the 'organization.' prefix
$fields = $org->fields();
// Returns: ['organization.name' => 'Acme Corp', 'organization.bank_name' => 'Business Bank', ...]
```

## Field Reference

### Natural Person Fields

| Field | Type | Description |
|-------|------|-------------|
| `first_name`, `last_name` | string | Name fields |
| `email_address` | string | Email (RFC 5322) |
| `mobile_number` | string | Phone (E.164) |
| `birth_date` | string | ISO 8601 date |
| `birth_country_code` | string | ISO 3166-1 alpha-3 |
| `address`, `city`, `postal_code` | string | Address fields |
| `address_country_code` | string | ISO 3166-1 alpha-3 |
| `id_type` | string | passport, drivers_license, id_card |
| `id_number`, `id_country_code` | string | Document details |
| `id_issue_date`, `id_expiration_date` | DateTime | Document dates |
| `tax_id`, `tax_id_name` | string | Tax information |
| `occupation` | int | ISCO-08 code |

### Organization Fields

All prefixed with `organization.`:

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Legal name |
| `VAT_number`, `registration_number` | string | Corporate IDs |
| `registration_date` | string | Date registered |
| `registered_address` | string | Legal address |
| `number_of_shareholders` | int | Shareholder count |
| `shareholder_name`, `director_name` | string | Key persons |
| `website`, `email`, `phone` | string | Contact info |

### Financial Account Fields

| Field | Type | Description |
|-------|------|-------------|
| `bank_name` | string | Bank name |
| `bank_account_type` | string | checking, savings |
| `bank_account_number`, `bank_number` | string | Account/routing numbers |
| `clabe_number`, `cbu_number` | string | Regional formats (MX, AR) |
| `mobile_money_number`, `mobile_money_provider` | string | Mobile money |
| `crypto_address`, `external_transfer_memo` | string | Crypto details |

### Card Fields

All prefixed with `card.`:

| Field | Type | Description |
|-------|------|-------------|
| `number`, `cvc` | string | Card number and security code |
| `expiration_date` | string | YY-MM format |
| `holder_name`, `network` | string | Cardholder and brand |
| `token` | string | Payment processor token |
| `address`, `city`, `postal_code`, `country_code` | string | Billing address |

## Security Considerations

- **Transmit over HTTPS only** - KYC data contains sensitive PII
- **Encrypt at rest** - Store collected data encrypted
- **Card data requires PCI-DSS** - Prefer tokenization over raw card numbers
- **Minimize collection** - Only request fields you actually need
- **Respect data regulations** - GDPR, CCPA, and local privacy laws apply

## Related SEPs

- [SEP-12](sep-12.md) - KYC API (submits SEP-09 fields to anchors)
- [SEP-24](sep-24.md) - Interactive deposit/withdrawal (may collect SEP-09 data)
- [SEP-31](sep-31.md) - Cross-border payments (uses SEP-09 for receiver info)
