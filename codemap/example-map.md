This is an illustration of a map's shape and depth. The project, paths and
code are examples.

---

# Test document generation — code map

Mapped at `a1b2c3d`, 2026-10-26. Entry: `TestDocumentCreate.vue`, `POST /test_documents/generate`.

Generates Romanian test documents as PDF or PNG using GD and wkhtmltopdf, and
stores a document entity per generated file. This map covers bank statement
generation [1a–1f]. Identity card and certificate generation share the same
entry and generator but are not mapped here.

## 1. Bank statement generation

From the generate button to a stored document entity.

- **User clicks generate**
  - **1a Vue submits the request** · [TestDocumentCreate.vue:123](assets/vue/documents/TestDocumentCreate.vue#L123)

    Reads `this.selected.type` (one of the document types offered by the
    select) and `this.values`, the raw form values keyed by field name. Posts
    both as JSON. On success it pushes the returned path into
    `this.generated` so the list re-renders; on failure it shows the API
    error text in the alert box and keeps the form values.

        async generate() {
            const response = await this.makePostRequest('/test_documents/generate', {type: this.selected.type, fields: this.values});
            this.generated.unshift(response.data.path);
        }

  - *route `POST /test_documents/generate` → `TestDocumentController::generateAction`*
    - **1b Controller hands the fields to the generator** · [TestDocumentController.php:63](src/Controller/TestDocumentController.php#L63)

      Receives `type` and `fields` from the JSON body. Validates only that
      both keys exist; an empty `fields` array passes. Calls the generator
      and gets a path relative to the documents directory. Builds the
      `TestDocument` entity from the path and the type. Persists in 1f.

          $type   = $payload['type']   ?? throw new BadRequestHttpException('type missing');
          $fields = $payload['fields'] ?? throw new BadRequestHttpException('fields missing');

          $relativePath = $this->generator->generate($type, $fields);

    - **1c Generator routes to the bank statement service** · [TestDocumentGenerator.php:88](src/Service/TestDocumentGenerator.php#L88)

      Receives `$type` and `$values`. The `match` on `$type` picks the
      service; an unknown type throws `UnsupportedDocumentTypeException`,
      which the controller turns into a 400. For bank statements only `iban`
      and `holder` are read; every other key in `$values` is ignored from
      here on. Returns the relative path from the service unchanged.

          return match ($type) {
              'bank_statement' => $this->bankStatements->generate((string) $values['iban'], (string) $values['holder']),
              'identity_card'  => $this->identityCards->generate(...),   // flow 2
              'certificate'    => $this->certificates->generate(...),    // flow 3
              default          => throw new UnsupportedDocumentTypeException($type),
          };

    - *`BankStatementGenerator::generate()`*
      - **1d Build the statement and render HTML** · [BankStatementGenerator.php:53](src/Service/BankStatementGenerator.php#L53)

        Receives the IBAN and holder name. `buildStatement` fills a
        `Statement` value object with the holder, a masked IBAN, and twelve
        deterministic transactions from `TestDocumentPools`, so the same
        input always gives the same statement. `renderHtml` passes it to
        `bank_statement.html.twig` and writes the HTML next to the final PDF.

            $statement = $this->buildStatement($iban, $holderName);
            $html      = $this->renderHtml($statement);
            $htmlPath  = $this->writeTemp($html);

      - **1e Convert HTML to PDF** · [BankStatementGenerator.php:70](src/Service/BankStatementGenerator.php#L70)

        Runs wkhtmltopdf on the temp HTML. A non-zero exit throws
        `PdfRenderException` with the process output; nothing is persisted
        in that case. Returns the path relative to the documents directory.

            $this->renderPdf($htmlPath, $destination);

            return $this->relativize($destination);

    - **1f Persist the document entity** · [TestDocumentController.php:78](src/Controller/TestDocumentController.php#L78)

      Receives the relative path. Creates a `TestDocument` with the path,
      the type, and the current user, flushes, and returns the path as JSON
      for 1a to display.

          $document = new TestDocument($relativePath, $type, $this->getUser());
          $manager->persist($document);
          $manager->flush();

          return $this->json(['path' => $relativePath]);

## Unresolved

- `DocumentGeneratedEvent` is dispatched after 1f. Two subscribers found
  (`AuditSubscriber`, `NotificationSubscriber`); neither changes the
  document, so they are not hops.

## Nearby flows not mapped

- Identity card generation: `IdentityCardGenerator`, CNP-based, with MRZ.
- Certificate generation: `CertificateGenerator`, with CUI validation.
- Document listing and download: `TestDocumentController::listAction`,
  `downloadAction`.
