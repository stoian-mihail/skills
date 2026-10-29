This is an illustration of a map's shape and depth, at module scale. The
project, paths and code are examples.

---

# Test document generation — Romanian document generation system

Created 2026-10-29, mapped at `a1b2c3d`.

This module generates realistic Romanian test documents (bank statements,
identity cards, company registration certificates) in a PHP/Symfony backend
with a Vue frontend, using GD for image documents and wkhtmltopdf for PDF
ones, and stores one `TestDocument` entity per generated file. Flows: bank
statement generation [1a–1f], identity card generation [2a–2g], certificate
generation [3a–3e], document listing and download [4a–4e], and the
deterministic data pools [5a–5e] that the three generators draw from.

## Module

- Flows 1, 2 and 3 share one entry: `TestDocumentCreate.vue` posts to
  `TestDocumentController::generateAction`, which calls
  `TestDocumentGenerator::generate`. The `match` on type in 1c is where they
  split. The shared part is detailed in flow 1 and shown as context in 2 and 3.
- Flows 1, 2 and 3 share one exit: the controller persists the entity (1f)
  and returns the path.
- Flows 2 and 3 draw their fake personal and company data from the pools in
  flow 5 (2b uses 5b and 5e; 3c uses 5c). Flow 1 uses 5d for transactions.
- Flow 4 lists and downloads the entities that 1f stores.

## 1. Bank statement generation

From the generate button to a stored PDF and its entity.

<details><summary>See more</summary>

Input: an IBAN and a holder name typed by the user. Output: a PDF under
`var/documents/bank_statement/` and a `TestDocument` row pointing at it.
Deterministic: the same IBAN always yields the same twelve transactions
(5d). Errors: a missing `type` or `fields` is a 400 from 1b; an unknown type
is a 400 from 1c; a wkhtmltopdf failure is a 500 from 1e and nothing is
persisted. Shares the entry (1a–1c) and the exit (1f) with flows 2 and 3.

</details>

- *User clicks generate*
  - **1a** Vue submits the generation request · [TestDocumentCreate.vue:123](assets/vue/documents/TestDocumentCreate.vue#L123)
    Reads the selected type and the form values keyed by field name and posts
    both as JSON. On success it pushes the returned path onto the list so 4a
    shows it without a reload. On failure it shows the API error text and
    keeps the form values.
    `await this.makePostRequest('/test_documents/generate', {type: this.selected.type, fields: this.values})`
  - *route `POST /test_documents/generate` → `TestDocumentController::generateAction`*
    - **1b** Controller hands the fields to the generator · [TestDocumentController.php:63](src/Controller/TestDocumentController.php#L63)
      Receives `type` and `fields` from the JSON body and checks only that both
      keys exist; an empty `fields` array passes. Hands them to the generator
      and gets back a path relative to the documents directory, which 1f turns
      into an entity.
      `$relativePath = $this->generator->generate($type, $fields);`
    - **1c** Generator routes by document type · [TestDocumentGenerator.php:88](src/Service/TestDocumentGenerator.php#L88)
      Receives the type and the raw values. The `match` on type picks the
      service: `bank_statement` continues here, `identity_card` is flow 2,
      `certificate` is flow 3, anything else throws
      `UnsupportedDocumentTypeException`, which becomes a 400. Only `iban` and
      `holder` are read for this arm; other keys are ignored from here on.
      `return $this->bankStatements->generate((string) $values['iban'], (string) $values['holder']);`
      - *`BankStatementGenerator::generate()`*
        - **1d** Build the statement and render HTML · [BankStatementGenerator.php:53](src/Service/BankStatementGenerator.php#L53)
          Receives the IBAN and holder. Fills a `Statement` value object with
          the holder, the IBAN masked to its last four digits, and twelve
          transactions from the pools (uses 5d, seeded by the IBAN). Renders
          `bank_statement.html.twig` with it and writes the HTML to a temp
          file next to the final PDF location.
          `$html = $this->renderHtml($this->buildStatement($iban, $holderName));`
        - **1e** Convert HTML to PDF · [BankStatementGenerator.php:70](src/Service/BankStatementGenerator.php#L70)
          Runs wkhtmltopdf on the temp HTML. A non-zero exit throws
          `PdfRenderException` carrying the process output; the controller
          turns it into a 500 and nothing is persisted. On success the temp
          HTML is deleted and the path relative to the documents directory is
          returned up through 1c to 1b.
          `$this->renderPdf($htmlPath, $destination);`
    - **1f** Persist the document entity · [TestDocumentController.php:78](src/Controller/TestDocumentController.php#L78)
      Receives the relative path. Creates a `TestDocument` with the path, the
      type and the current user, flushes, and returns the path as JSON for 1a.
      Stores the row that 4b lists and 4e streams. Flows 2 and 3 end here too.
      `$manager->persist($document); $manager->flush();`

## 2. Identity card generation

From the generate button to a stored PNG with MRZ and diacritics.

<details><summary>See more</summary>

Input: a CNP, a first name and a last name. Output: a PNG under
`var/documents/identity_card/` drawn with GD on a blank card template, and a
`TestDocument` row. The county, birth date and sex are derived from the CNP,
the address and issuing office from the pools, so the same CNP always gives
the same card (5a, 5b, 5e). Errors: an invalid CNP checksum is a 422 from
2b. Everything before 2a and after 2g is shared with flow 1.

</details>

- *User clicks generate (1a)*
  - *`TestDocumentController::generateAction` (1b)*
    - *`TestDocumentGenerator::generate` dispatches on type (1c)*
      - **2a** Generator routes to the identity card service · [TestDocumentGenerator.php:94](src/Service/TestDocumentGenerator.php#L94)
        The `identity_card` arm of the dispatch in 1c. Reads `nume`,
        `prenume` and `cnp` from the values and hands them to the service as
        strings; other keys are ignored.
        `return $this->identityCards->generate((string) $values['nume'], (string) $values['prenume'], (string) $values['cnp']);`
      - *`IdentityCardGenerator::generate()`*
        - **2b** Pools build the identity fields from the CNP · [TestDocumentPools.php:41](src/Service/TestDocumentPools.php#L41)
          Receives the CNP and the names. Validates the CNP checksum and
          throws `InvalidCnpException` (a 422) when it fails. Seeds the pools
          with the CNP (uses 5a) and picks address, issuing office and
          expiry (uses 5b). Returns an `IdentityFields` value object.
          `$fields = $this->pools->identityFields($cnp, $nume, $prenume);`
          - **2c** Pools derive county and birth date from the CNP · [TestDocumentPools.php:26](src/Service/TestDocumentPools.php#L26)
            Digits 8 and 9 of the CNP are the county code, digits 2 to 7 the
            birth date, digit 1 the sex and century. Maps the county code to
            a county name and a city through the locations table (uses 5e).
            `$jj = (int) substr($cnp, 7, 2); $birth = $this->locationForCounty($locations, $jj, $cnp, 'birth');`
        - *`IdentityCardTestDocument::draw()`*
          - **2d** Paste the portrait photo · [IdentityCardTestDocument.php:66](src/Document/IdentityCardTestDocument.php#L66)
            Opens the blank card template as a GD image and pastes the
            portrait chosen by the pools (uses 5b, by sex) at the photo box.
            `$this->pastePhoto($image, $root . '/portrait.png');`
          - **2e** Draw tracked text with diacritics · [IdentityCardTestDocument.php:71](src/Document/IdentityCardTestDocument.php#L71)
            Draws each field with a TrueType font and manual letter tracking
            so Romanian diacritics (ș, ț, ă) keep their width. Names are
            upper-cased with a multibyte-safe helper.
            `$this->drawTracked($image, $font, 495, 379, $this->upper($nume), 32, self::BLACK);`
          - **2f** Generate the MRZ lines · [IdentityCardTestDocument.php:83](src/Document/IdentityCardTestDocument.php#L83)
            Builds the three machine-readable lines from the names, the CNP
            and the expiry, with ICAO check digits, and draws them in the OCR
            font at the card's bottom band.
            `$this->drawMrz($image, $font, $lines);`
          - **2g** Save the PNG · [IdentityCardTestDocument.php:406](src/Document/IdentityCardTestDocument.php#L406)
            Writes the image to the destination path and frees the GD
            resource. Returns the relative path up through 2a to 1b.
            `imagepng($image, $destination);`
    - *`TestDocumentController` persists the entity and responds (1f)*

## 3. Certificate generation

From the generate button to a stored PDF registration certificate.

<details><summary>See more</summary>

Input: a CUI and a company name. Output: a PDF under
`var/documents/certificate/` with a barcode of the registration number, and
a `TestDocument` row. Registration number, registered office and NACE code
come from the pools seeded by the CUI (5a, 5c). Errors: an invalid CUI
checksum is a 422 from 3b; a wkhtmltopdf failure is a 500 from 3e. Shares
entry and exit with flow 1.

</details>

- *User clicks generate (1a)*
  - *`TestDocumentController::generateAction` (1b)*
    - *`TestDocumentGenerator::generate` dispatches on type (1c)*
      - **3a** Generator routes to the certificate service · [TestDocumentGenerator.php:100](src/Service/TestDocumentGenerator.php#L100)
        The `certificate` arm of the dispatch in 1c. Reads `cui` and
        `denumire` and hands them to the service as strings.
        `return $this->certificates->generate((string) $values['cui'], (string) $values['denumire']);`
      - *`CertificateGenerator::generate()`*
        - **3b** Validate the CUI checksum · [CertificateGenerator.php:38](src/Service/CertificateGenerator.php#L38)
          Strips an optional `RO` prefix and checks the control digit with
          the ANAF weights. Throws `InvalidCuiException` (a 422) on failure,
          so no file is produced for a bad CUI.
          `if (!$this->cuiValidator->isValid($cui)) { throw new InvalidCuiException($cui); }`
        - **3c** Pools build the company fields · [TestDocumentPools.php:120](src/Service/TestDocumentPools.php#L120)
          Seeds the pools with the CUI (uses 5a) and picks the registration
          number, registered office and NACE code (uses 5c). Returns a
          `CompanyFields` value object.
          `$fields = $this->pools->companyFields($cui, $denumire);`
        - **3d** Render HTML with the barcode · [CertificateGenerator.php:57](src/Service/CertificateGenerator.php#L57)
          Encodes the registration number as a Code 128 barcode PNG, inlines
          it as a data URI, and renders `certificate.html.twig` with the
          fields. Writes the HTML to a temp file.
          `$html = $this->renderHtml($fields, $this->barcode->png($fields->registrationNumber));`
        - **3e** Convert HTML to PDF · [CertificateGenerator.php:66](src/Service/CertificateGenerator.php#L66)
          Same wkhtmltopdf call and failure behaviour as 1e. Returns the
          relative path up through 3a to 1b.
          `$this->renderPdf($htmlPath, $destination);`
    - *`TestDocumentController` persists the entity and responds (1f)*

## 4. Document listing and download

From opening the page to streaming a stored file back to the browser.

<details><summary>See more</summary>

Input: the current user. Output: the list of their generated documents,
newest first, and on click the file itself. Reads the rows written by 1f.
Errors: a document id that is not the user's is a 404 from 4e, never a 403,
so ids cannot be probed.

</details>

- *User opens the test documents page*
  - **4a** Vue loads the document list · [TestDocumentList.vue:48](assets/vue/documents/TestDocumentList.vue#L48)
    On mount, requests the list and stores it; 1a pushes new entries onto
    the same array, so the list and the form share state.
    `this.documents = (await this.makeGetRequest('/test_documents')).data;`
  - *route `GET /test_documents` → `TestDocumentController::listAction`*
    - **4b** Controller lists the user's documents · [TestDocumentController.php:41](src/Controller/TestDocumentController.php#L41)
      Asks the repository for the current user's documents and serialises
      id, type, path and creation time. Reads what 1f stored.
      `$documents = $this->repository->findForUser($this->getUser());`
      - **4c** Repository queries by user, newest first · [TestDocumentRepository.php:22](src/Repository/TestDocumentRepository.php#L22)
        Filters on the owning user and orders by `createdAt` descending. No
        pagination; the test tool is expected to hold tens of rows per user.
        `->andWhere('d.user = :user')->orderBy('d.createdAt', 'DESC')`
  - *User clicks a document*
    - **4d** Vue opens the download URL · [TestDocumentList.vue:71](assets/vue/documents/TestDocumentList.vue#L71)
      Navigates the browser to the download route in a new tab, so the
      Content-Disposition header from 4e triggers the save dialog.
      `window.open(`/test_documents/${doc.id}/download`, '_blank');`
    - *route `GET /test_documents/{id}/download` → `TestDocumentController::downloadAction`*
      - **4e** Controller streams the file · [TestDocumentController.php:96](src/Controller/TestDocumentController.php#L96)
        Loads the document by id and user together, so another user's id is
        a 404. Resolves the relative path against the documents directory
        and returns a `BinaryFileResponse` as an attachment named after the
        type and the id.
        `return $this->file($this->documentsDir . '/' . $document->getPath(), $document->downloadName());`

## 5. Data pools

The deterministic fake data every generator draws from.

<details><summary>See more</summary>

A subsystem, not a request flow. `TestDocumentPools` holds fixed lists
(streets, cities, offices, company suffixes, NACE codes, transaction
descriptions) and picks from them with a random generator seeded from the
document's identifier, so the same CNP, CUI or IBAN always produces the same
document. Nothing here touches the database or the filesystem. Used by 1d,
2b, 2c and 3c.

</details>

- *A generator asks for fields*
  - **5a** Seed the generator from the identifier · [TestDocumentPools.php:18](src/Service/TestDocumentPools.php#L18)
    Receives the CNP, CUI or IBAN. Seeds a Mersenne Twister instance with
    its CRC32, so every later pick in this request is reproducible. Called
    by 2b, 3c and 1d before any pick.
    `$this->rng = new Randomizer(new Mt19937(crc32($identifier)));`
  - **5b** Pick the identity fields · [TestDocumentPools.php:41](src/Service/TestDocumentPools.php#L41)
    Picks a street and number, an issuing office, a portrait by sex and an
    expiry date five to ten years out. Combines them with the county and
    birth date from 5e into `IdentityFields`. Used by 2b.
    `return new IdentityFields($nume, $prenume, $cnp, $birth, $county, $this->pick($this->streets), ...);`
  - **5c** Pick the company fields · [TestDocumentPools.php:120](src/Service/TestDocumentPools.php#L120)
    Picks a registration number in `J<county>/<n>/<year>` form, a
    registered office and a NACE code. Used by 3c.
    `return new CompanyFields($cui, $denumire, $this->registrationNumber($county), $this->pick($this->offices), $this->pick($this->nace));`
  - **5d** Pick the transactions · [TestDocumentPools.php:164](src/Service/TestDocumentPools.php#L164)
    Produces twelve transactions over the last ninety days with amounts,
    descriptions from the pool and a running balance that never goes
    negative. Used by 1d.
    `$transactions[] = new Transaction($date, $this->pick($this->descriptions), $amount, $balance);`
  - **5e** Map a county code to a location · [TestDocumentPools.php:26](src/Service/TestDocumentPools.php#L26)
    Receives the two-digit county code from the CNP. Returns the county
    name and a city from the locations table; codes 51 and 52 (the
    historical Bucharest sectors) map to Bucharest. Used by 2c.
    `return $locations[$jj] ?? $locations[40];`

## Unresolved

- `DocumentGeneratedEvent` is dispatched after 1f. Two subscribers found
  (`AuditSubscriber`, `NotificationSubscriber`); neither changes the
  document, so they are not hops.
- `TestDocumentGenerator` is bound to `DocumentGeneratorInterface` in
  `services.yaml`; one other implementation exists (`LegacyDocumentGenerator`)
  but nothing is wired to it. Treated as dead.
