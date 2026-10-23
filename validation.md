# Pension Scheme Return SIPP – Data Validation

Inbound validation is **structural, not semantic**. Play JSON `Reads` macros (`Json.format`) check that required fields are present and have the right types, that enums take allowed values, and that lists marked non-empty are non-empty. The SRN header is the only value with a local format check. Formats, lengths and cross-field rules (NINO, PSTR, CRN, postcode, string lengths, "Yes requires details") are enforced only by the **outbound** JSON-schema check against the bundled ETMP API 1997 schema, which runs just before each POST, and by ETMP itself. Read operations (GET) send identifiers upstream without checking them at all.

## Validation layers at a glance

| # | Layer | Mechanism | Where | On failure |
|---|---|---|---|---|
| 1 | Route binding | Play binders; `QueryStringBindable[JourneyType/Journey]` (case-insensitive enumeratum) | `conf/app.routes`, `config/Binders` | 400 (Play client error) |
| 2 | Required header | `requiredHeaders("srn")` | `PsrBaseController` | `BadRequestException` → 400 (before auth) |
| 3 | SRN format | `^S[0-9]{10}$` | `SchemeId.Srn.apply` in `PsrAuth` | 400 `Invalid scheme reference number` |
| 4 | Role header | `requestRole` ∈ {PSA, PSP} (upper-cased) or absent | `PsrAuth.authorisedUser` | `BadRequestException` → 400 |
| 5 | Authorisation | Enrolment + scheme association | `PsrAuth`, `SchemeDetailsConnector` | 401 |
| 6 | Body parsing | `parse.json` (journey PUTs, member update) / `request.body.asJson` (others) | controllers | invalid JSON → 400; non-JSON body → 400 `Request does not contain Json body` |
| 7 | Body reading | `JsValue.as[T]` (throws) | controllers | `JsResultException` → **500** |
| 8 | Business lookups | existing PSR / member found | `SippPsrSubmissionService` | varies (404/204/500/201, see below) |
| 9 | Outbound size | JSON bytes > `etmpConfig.maxRequestSize` (512000) | `PsrConnector.submitSippPsr` | 413 + audit events |
| 10 | Outbound schema | networknt validator vs `api-1997-…-v1.5.0.json` | `JSONSchemaValidator`, `PsrConnector.submitSippPsr` | `PensionSchemeReturnValidationFailureException` → 400 |
| 11 | Upstream response | status mapping + `.as[...]` parsing | `PsrConnector`, `HttpResponseHelper` | 400/404/4xx/502/500 |
| 12 | Email callback | AES decrypt + email regex + `EmailEvents` reads | `EmailResponseController` | 500 / 403 / 400 |

## Route and path parameters

| Parameter | Rule | Code |
|---|---|---|
| `pstr` (path) | Any non-slash string. **No format check.** | `app.routes` (`String`) |
| `journeyType` (query) | Required; `Standard` or `Amend` (case-insensitive) | `Binders.journeyTypeBindable` |
| `journey` (query, delete-assets) | Required; one of the 6 `Journey` values (case-insensitive) | `Binders.journeyBindable` |
| `fbNumber` (query) | Optional (required for `PUT /psr/member-details`). **No format check.** | routes |
| `periodStartDate`, `psrVersion` (query) | Optional strings. **No format check.** Either `fbNumber`, or both of these, must be present for any ETMP read, otherwise `BadRequestException("Missing url parameters")` → 400. | `PsrConnector.buildParams` |
| `startDate` (query, versions) | Parsed as ISO date | `SippPsrSubmitController.getPsrVersions` → 400 `Invalid startDate …` |
| Email callback path segments | `submittedBy`, `requestId`, `taxYear` and `reportVersion` are free text. The other five must decrypt (see below). | `EmailResponseController` |

## Request headers

| Endpoint | Headers | Validation |
|---|---|---|
| All `/psr/**` except email callback | `srn` (required), `requestRole` (optional), `Authorization` (session) | presence → regex → enrolment → association |
| ETMP-backed calls | `CorrelationId` (optional) | Forwarded to IF **unchecked**; random UUID if absent |
| Email callback | none | – |

## Request bodies

All readers are `Json.format`-derived. `Option[...]` fields may be absent, and everything else is required. Lists typed `NonEmptyList` reject `[]` with `The list should not be empty`. Dates must be ISO `yyyy-MM-dd`, and enums must match exactly (`Yes`/`No`, `Compiled`/`Submitted`, `Cost Value`/`Market Value`).

| Flow | Reader | Failure |
|---|---|---|
| `POST /psr/empty/sipp` | `ReportDetails` | 500 (`JsResultException`) |
| `PUT /psr/{land-or-connected-property, land-arms-length}` | `LandOrConnectedPropertyRequest` | 500 |
| `PUT /psr/outstanding-loans` | `OutstandingLoansRequest` | 500 |
| `PUT /psr/assets-from-connected-party` | `AssetsFromConnectedPartyRequest` | 500 |
| `PUT /psr/tangible-moveable-property` | `TangibleMoveablePropertyRequest` | 500 |
| `PUT /psr/unquoted-shares` | `UnquotedShareRequest` | 500 |
| `PUT /psr/report-details/member-transactions/:pstr` | `MemberTransactions{value: YesNo}` | 500 |
| `PUT /psr/accounting-periods/:pstr` | `AccountingPeriodDetailsRequest` | 500 |
| `POST /psr/sipp` | `PsrSubmissionRequest` | 500 |
| `PUT /psr/member-details/:pstr` | `UpdateMemberDetailsRequest{current, updated: PersonalDetails}` | 500 |
| `PUT /psr/delete-member/:pstr` | `PersonalDetails` | missing/non-JSON body → 400 `Invalid personal details`; malformed JSON object → **500** (the `.as` runs outside the `recover`) |

**What is checked**

- Presence and JSON type of every non-optional field (e.g. `reportDetails.pstr`, `status`, `periodStart`, `periodEnd`, `memberTransactions`; per-transaction `nameDOB`, `nino` object, amounts, Yes/No answers).
- Enum membership and ISO date parsing.
- `transactions`, when present, is non-empty. Absent means "clear this asset class".
- `accountingPeriods`, when present, is non-empty.

**What is not checked locally**

- Formats: PSTR, NINO, PSA ID (`PsrSubmissionRequest.psaId`), CRN, postcode, country code, registry reference, version strings.
- Lengths (ETMP: names 35, free text 160), numeric ranges (e.g. `noOfShares` 0–9 999 999, currency), and negative amounts.
- Conditional rules:
  - `nino` vs `reasonNoNino` (exactly one)
  - `isPropertyDisposed = Yes` ⇒ `disposalDetails`
  - `isLeased = Yes` ⇒ `lesseeDetails`
  - `registryRefExist` ⇒ reference / reason
  - `jointlyHeld` ⇒ `noOfPersons`
  - `acquisitionOfShares` ⇒ `sharesCompanyDetails`
  - `arrearsOutstandingPrYears` ⇒ amount
- Cross-field rules: `periodStart ≤ periodEnd`, at most 3 accounting periods, `transactionCount` vs actual count (`transactionCount` is ignored on input).
- `reportDetails.pstr` in the body (journey PUTs), and `pstr` in `PsrSubmissionRequest`, against the authorised SRN.
- `ReportDetails.status` on create-empty: the client value is passed through to ETMP. Journey merges force `Compiled`.
- `auditContext` content. It is used only for audit.

**Normalisation and derivation**

| Rule | Where |
|---|---|
| Member identity = exact `(firstName, lastName, dob, nino)`. Case- and whitespace-sensitive, no NINO normalisation. | `EtmpMemberAndTransactionsUpdater.merge`, `MemberDetails.compare` (update/delete use `firstName, lastName, dateOfBirth, nino`; `reasonNoNINO` ignored) |
| Journey upload **replaces** the member's transactions for that asset class. Members absent from the upload lose theirs. Members with no remaining assets → `Deleted`, otherwise `Changed`; new members → `New`. | `EtmpMemberAndTransactionsUpdater.merge` |
| Duplicate members already in ETMP data collapse to the **first** record (`groupBy(...).mapValues(_.head)`). The others are dropped from the re-posted payload. ⚠️ | `EtmpMemberAndTransactionsUpdater.merge` |
| `noOfTransactions` = size of the uploaded list per member. `transactionCount` on output = ETMP `noOfTransactions`. | per-journey transformers |
| Asset-class declaration = `Yes` if transactions are present, otherwise `No`. On final submit, missing declarations are derived from data. On delete-assets the declaration is cleared (`None`). | `EtmpSippReportDetails.transactionsAssetClassDeclaration`, `completeOutstandingAssetDeclarations` |
| `version` stripped (`None`) on the report and on changed members before every POST | service / transformers |
| Create-empty: `memberTransactions` forced to `No`. New-PSR merge: forced to `Yes`, status `Compiled`. | `SippPsrSubmissionService.createEmptySippPsr`, `transformations.reportDetailsApiToEtmp`, `merge` |
| Submit: status `Submitted`, declaration `submittedBy`/`submitterID` from the **auth** identity, `psaID` from the **body**, declarations true for the submitter's role | `SippPsrSubmissionService.psrSubmission` |
| Delete member / assets: declarations reset to `false` | `SippPsrSubmissionService.deleteMember/deleteAssets` |
| Reads drop `Deleted` members (except asset declarations and the `memberDetails` version) | `filterDeletedMembers`, `PSRSubmissionTransformer` |

## External validation services

| Service | Request | Decision | Fail mode |
|---|---|---|---|
| pensions-scheme `is-psa-associated` | PSAID/PSPID + SRN headers | `true` → proceed | **Fail closed.** `false` → 401; upstream error → 4xx/502/500. |
| pension-administrator minimal details | psaId/pspId | details required for every write | **Fail closed** for writes (500). For the email step, `Left` → email skipped, but exceptions propagate (5xx after the return was already submitted ⚠️). |

## Outbound payload validation

- **When**: before every ETMP POST, after the payload size check (`PsrConnector.submitSippPsr`). The size check runs first and fires audit events. A schema failure fires **no** audit event.
- **Schema**: `conf/schemas/api-1997-submit-sipp-pension-scheme-return-request-schema-v1.5.0.json` (`SchemaPaths.API_1997`). It is the only schema bundled, and it is used.
- **How**: `JsonSchemaFactory.getInstance(SpecVersion.VersionFlag.V4)`.
  - ⚠️ The schema declares `$schema: draft-07`. Draft-07-only keywords may not be applied as intended.
  - ⚠️ The schema is re-read from the classpath and recompiled on **every** call (no caching).
- **Failure**: `logger.error("Validation has errors: …")` logs the validator messages, which can include field paths. No PII masking is applied. The caller then gets **400** with the messages in the body.
- **Key contract rules** (not enforced locally):

| Field | Contract rule |
|---|---|
| `pstr` | `^[0-9]{8}[A-Z]{2}$` |
| `nino` | `^[A-Z]{2}[0-9]{6}[A-Z]$` |
| `submitterID` | `^[A\|0\|1\|2][0-9]{7}$` |
| `psaID` | `^A[0-9]{7}$` |
| `companySharesCRN` | `^[A-Z0-9]{8,10}$` |
| `ukPostCode` | UK/BFPO regex |
| `countryCode` | ISO-like enum |
| `version` | `^[0-9]{3}$` |
| names | 1–35 chars |
| free text | 1–160 chars |
| `accountingPeriods` | 1–3 items |
| `personalDetails` | `additionalProperties: false` |

## Upstream response validation

| Response | Validation | Failure result |
|---|---|---|
| ETMP GET PSR 200 | `.as[SippPsrSubmissionEtmpResponse]` | `JsResultException` → 500 |
| ETMP GET PSR 404 / 422 + `PSR_NOT_FOUND` | treated as "no return" | GET journey → 204; GET PSR/members/declarations/counts → 404; writes → `RuntimeException` → 500; member update → 404 |
| ETMP POST 200 | `.json.as[SippPsrJourneySubmissionEtmpResponse]` (`formBundleNumber`) | 500 |
| ETMP versions 200 | `Try(.as[Seq[PsrVersionsResponse]])` | 500 |
| ETMP versions 404 | empty list | 200 `[]` |
| is-psa-associated 200 | `.as[Boolean]` | 500 |
| minimal details | typed `HttpReads[MinimalDetails]` | 4xx → `UpstreamErrorResponse(reportAs 500)`, 5xx → reportAs 502 (except the two mapped `Left`s) |
| email | status 202 only | `Left` (never fails the request) |

## Inbound callbacks

`POST /psr/email-response/...` (unauthenticated):

1. `email` is decrypted **outside** the `try`. A decryption failure (tampered or garbage value) throws → **500**.
2. The decrypted email must match `Constants.emailRegex`, otherwise **403** `Malformed email : <decrypted value>`. ⚠️ This echoes the decrypted value.
3. The PSA/PSP ID, PSTR, scheme name and user name are decrypted. A failure → 500.
4. The body is read with `parser.tolerantJson` + `validate[EmailEvents]`. Each event needs `event` ∈ {Sent, Delivered, PermanentBounce, Opened, Complained} and `detected: LocalDateTime`. Failure → 400.
5. `Opened` events are skipped. The rest are audited with the plaintext `submittedBy`, `requestId`, `taxYear` and `reportVersion` taken verbatim from the path. `submittedBy` ≠ `PSA` is treated as PSP in the audit shape.

## Cache / opaque stores

None. There is no local persistence.

## Error mapping summary

| Source | Exception / value | HTTP to caller |
|---|---|---|
| Missing `srn` header | `BadRequestException` | 400 |
| Invalid SRN | `BadRequest` result | 400 |
| Bad `requestRole` | `BadRequestException` | 400 |
| No enrolment / not logged in | `AuthorisationException` | 401 |
| Not associated / no externalId | `UnauthorizedException` | 401 |
| Enrolment without ID | `BadRequestException` | 400 |
| Invalid JSON / missing JSON body | Play parser / `BadRequestException` | 400 |
| Body shape wrong | `JsResultException` | **500** |
| Missing fb / period+version | `BadRequestException` | 400 |
| Payload > 512000 bytes | `RequestEntityTooLargeException` | 413 |
| Schema violation | `PensionSchemeReturnValidationFailureException` | 400 |
| Upstream 400 / 404 | `BadRequestException` / `NotFoundException` | 400 / 404 |
| Upstream other 4xx | `UpstreamErrorResponse(reportAs = status)` | same status |
| Upstream 5xx | `UpstreamErrorResponse(reportAs = 502)` | 502 |
| Minimal details not found / delimited | `RuntimeException` | 500 |
| PSR not found on write/submit | `Exception` / `RuntimeException` | 500 |
| Any failure in delete-member / delete-assets service call | caught by `recover` | **400** (masks upstream 5xx/401) |
| Email callback decrypt failure | crypto exception | 500 |
| Email callback bad email | – | 403 |

## Recommendations

1. Replace `request.body.as[T]` with `validate[T]` (or a `parse.json[T]` parser) so that malformed bodies return 400 with field errors instead of 500.
2. Validate PSTR (`^[0-9]{8}[A-Z]{2}$`), `fbNumber`, `periodStartDate` (ISO date) and `psrVersion` locally, and URL-encode them when building ETMP URLs. Also check that the PSTR belongs to the authorised SRN.
3. Validate `PsrSubmissionRequest.psaId` locally. Better, derive or verify it, for example as the authorising PSA from scheme details. For PSA submitters, it should equal the authenticated PSA ID.
4. Narrow the `recover` blocks in delete-member and delete-assets to genuine client errors, so that upstream 5xx are not reported as 400.
5. Cache the compiled JSON schema and use the draft-07 factory that matches its `$schema`.
6. Stop logging full payloads at INFO, and mask validator output and the 403 email echo.
7. Decrypt all callback values inside the error handling and return 400 or 403 rather than 500. Consider validating `submittedBy` ∈ {PSA, PSP}.
8. Treat a minimal-details failure in the post-submit email step as "email not sent" rather than propagating a 5xx.
