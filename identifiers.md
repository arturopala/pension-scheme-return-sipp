# Pension Scheme Return SIPP – Business Identifiers

This document lists every identifier the service receives, derives or sends, and compares the validation rules that come from three sources:

- **Local**: checks in this codebase (binders, regexes, constructors).
- **Library**: checks inside dependency types. The service uses **no** library identifier types such as `uk.gov.hmrc.domain.Nino` or `PsaId`. `PsaId` and `PspId` are local case classes without a `require`, so this column is empty throughout.
- **Contract**: patterns in the bundled ETMP API 1997 request schema (`conf/schemas/api-1997-submit-sipp-pension-scheme-return-request-schema-v1.5.0.json`). This schema is enforced only on outbound POSTs, never on GETs.

Terminology:
- *SRN* (scheme reference number) is the identifier the frontend uses for a scheme. *PSTR* (pension scheme tax reference) is the identifier ETMP uses. The codebase never maps one to the other; both are supplied independently by the caller.
- *fbNumber* is the same thing as *psrFormBundleNumber* and *formBundleNumber*.
- *psaPspId* / `PensionSchemeId` means "PSA ID or PSP ID, whichever the user is acting as".

## Summary table

| Identifier | Identifies | Format (contract) | Local validation | Enters via | Main uses |
|---|---|---|---|---|---|
| SRN | Registered pension scheme (MDTP) | – | `^S[0-9]{10}$` | `srn` header (untrusted) | Association check with pensions-scheme |
| PSTR | Scheme in ETMP | `^[0-9]{8}[A-Z]{2}$` | none | path `:pstr`, body `reportDetails.pstr`, body `PsrSubmissionRequest.pstr` (untrusted); ETMP response | ETMP URL path, payload `reportDetails.pstr`, audit, encrypted email callback |
| Form bundle number | One stored version of a return | – (not in request schema) | none | query `fbNumber` (untrusted); ETMP POST response | ETMP GET query `psrFormBundleNumber`; returned to frontend |
| Period start date | Reporting period of a return | – | none (raw string) | query `periodStartDate` | ETMP GET query |
| PSR version | Version of a return / section | `^[0-9]{3}$` (in payload) | none | query `psrVersion`; body `reportDetails.version`; ETMP response | ETMP GET query; stripped before POST; `toInt > 1` → audit "amendment"; email callback |
| PSA ID | Pension scheme administrator | `submitterID ^[A\|0\|1\|2][0-9]{7}$`; `psaID ^A[0-9]{7}$` | none | auth enrolment `HMRC-PODS-ORG/PSAID` (trusted); body `PsrSubmissionRequest.psaId` (untrusted) | Association check, minimal details, declaration, audit, email callback |
| PSP ID | Pension scheme practitioner | `submitterID` as above | none | auth enrolment `HMRC-PODSPP-ORG/PSPID` (trusted) | as PSA ID |
| externalId | Auth user | – | presence only | auth retrieval | Carried in `PsrAuthContext`, otherwise unused |
| NINO | Scheme member | `^[A-Z]{2}[0-9]{6}[A-Z]$` | none | body (`nino.nino`, `PersonalDetails.nino`) | Member identity key, ETMP payload |
| Member key | Scheme member within a return | – | none | derived (first name, last name, DOB, NINO) | Merge / update / delete matching |
| Company CRN | Company whose shares are held | `^[A-Z0-9]{8,10}$` | none | body `sharesCompanyDetails.companySharesCRN` | ETMP payload |
| Land registry reference | Property title | string35 | none | body `registryDetails.registryReference` | ETMP payload |
| CorrelationId | IF call trace | – | none | `CorrelationId` header or random UUID | IF request header |
| requestId | Platform request (X-Request-ID) | – | none | `HeaderCarrier.requestId`; callback path (untrusted) | Email callback path → email audit |
| Email address | Submitter's contact | – | `Constants.emailRegex` (callback only) | minimal details (trusted); callback path (encrypted) | Email send, email audit |
| File reference | Uploaded file (upscan) | – | none | body `auditContext.fileReference` | Upload audit only |

## Scheme identifiers (SRN, PSTR)

**Lifecycle.** The SRN and PSTR are both issued at scheme registration, by the scheme registry and ETMP respectively. The service never derives one from the other and never fetches scheme details to link them. `auditContext.schemeDetails.pstr`, which the frontend supplies, is used only in the upload audit event.

**Sources**

- SRN: the `srn` header only.
- PSTR, by flow:
  - path segment `:pstr` on all `/psr/*/:pstr` routes;
  - body `reportDetails.pstr` on the six journey `PUT`s and on `POST /psr/empty/sipp`;
  - body `pstr` on `POST /psr/sipp`.

**Uses**

- SRN: the `schemeReferenceNumber` header to `is-psa-associated` (`SchemeDetailsConnector.checkAssociation`).
- PSTR:
  - `String.format` into `serviceUrls.get-sipp-psr`, `submit-sipp-psr` and `get-psr-versions`;
  - ETMP payload `reportDetails.pstr`. On updates, the payload copies the PSTR from the **ETMP response**, not from the caller;
  - audit `pensionSchemeTaxReference`;
  - encrypted into the email callback URL.

**Validation**

| Source | SRN | PSTR |
|---|---|---|
| Local | `SchemeId.Srn.apply`: `^S[0-9]{10}$` → 400 | none (`SchemeId.Pstr` exists but is unused) |
| Library | – | – |
| Contract | – | `pstrType ^([0-9]{8}[A-Z]{2})$` (POST only) |

**Mismatches and risks**

- ⚠️ **Authorised on SRN, acted on PSTR.** `PsrAuth` proves that the user is associated with the scheme in the `srn` header. Every ETMP operation then uses a PSTR from the path or body that is never linked to that SRN. An authenticated PSA or PSP associated with *any* scheme can therefore name a different scheme's PSTR, and read or overwrite that scheme's return.
- ⚠️ The path PSTR is not URL-encoded. It is interpolated with `String.format` into a template that also contains the query string, so characters such as `?`, `&`, `/` and `#` would change the ETMP request. GETs never pass through the schema.
- On journey `PUT`s there is no path PSTR; the PSTR comes only from the body.

## User / enrolment identifiers (PSA ID, PSP ID, externalId)

**Lifecycle.** Issued at PSA or PSP registration and held as auth enrolments: `HMRC-PODS-ORG` with identifier `PSAID`, and `HMRC-PODSPP-ORG` with identifier `PSPID`. The service picks the role as follows:

- `requestRole: PSA` → PSA ID, and requires that enrolment.
- `requestRole: PSP` → PSP ID, and requires that enrolment.
- No header → PSA ID if present, otherwise PSP ID.

**Uses (trusted, auth-derived)**

- The association-check header name/value pair: `PSAID: …` or `PSPID: …`.
- Minimal-details headers `psaId` / `pspId` plus `loggedInAsPsa`.
- Declaration `submittedBy` (`PSA`/`PSP`) and `submitterID`.
- Audit: `pensionSchemeAdministratorId` or `pensionSchemePractitionerId`, plus `credentialRolePsaPsp`.
- Encrypted into the email callback. The role is in plaintext as the `:submittedBy` segment.

**Untrusted copy.** `PsrSubmissionRequest.psaId`, from the body, is written to declaration `psaID`, the "PSA who approved the return". It is not compared with the auth identity: not for a PSA submitter, and not against the scheme's authorising PSA for a PSP submitter. The only guard is the contract regex `^A[0-9]{7}$`.

**Validation**

| Source | PSA ID | PSP ID |
|---|---|---|
| Local | none (`PensionSchemeId.PsaId` has no check) | none |
| Library | – | – |
| Contract | `submitterID ^([A\|0\|1\|2]{1}[0-9]{7})$`, `psaID ^(A[0-9]{7})$` | `submitterID` as left |

⚠️ The `submitterID` character class `[A|0|1|2]` also admits a literal `|`. This looks like an error in the contract.

**externalId** is retrieved and required (otherwise 401), then placed in `PsrAuthContext` and never used.

## Return identifiers (form bundle number, period start date, version)

- **Issuer**: ETMP. Every successful POST returns `{formBundleNumber}`, which goes to the frontend as `201 {formBundleNumber}`. The versions endpoint returns `reportFormBundleNumber`, `reportVersion` (Int) and `reportStatus`.
- **Addressing a return**: either `fbNumber`, or `periodStartDate` + `psrVersion` (`PsrConnector.buildParams`). `fbNumber` wins when both are given. `PUT /psr/member-details` requires `fbNumber`.
- **Version handling**:
  - `version` is always stripped to `None` before a POST.
  - The PSR version from the query or body, or from the ETMP response on submit, is passed to audit. `maybeVersion.exists(_.toInt > 1)` marks "submissionAmendment". A non-numeric value throws inside the `andThen` callback, so that audit event is silently lost.
  - Section versions in `GET /psr/sipp` are computed as `maxOption` over `Option[String]`. This is a lexicographic comparison, which is safe only while every version is zero-padded to three digits as the contract requires.
- **Member update with no match**: the endpoint echoes the caller's `fbNumber` back as if it were a new form bundle number, without writing.
- **Validation**: there is no local check on `fbNumber`, `periodStartDate` or `psrVersion`. None of them is URL-encoded, and they go upstream on GETs only, so the contract never applies. ⚠️ A query value such as `fbNumber=1&psrVersion=…` changes the ETMP query.

## Party identifiers (members, companies, property)

| Identifier | Frontend key(s) | Upstream key (flow) | Local rule | Contract rule |
|---|---|---|---|---|
| Member NINO | `nino.nino` (journeys); `nino` (`PersonalDetails` in update/delete) | `memberDetails.personalDetails.nino` (all writes) | none | `^[A-Z]{2}[0-9]{6}[A-Z]$` |
| No-NINO reason | `nino.reasonNoNino`; `reasonNoNINO` | `personalDetails.reasonNoNINO` | none | string160 |
| Member name / DOB | `nameDOB.{firstName,lastName,dob}`; `firstName/lastName/dateOfBirth` | `personalDetails.{firstName,lastName,dateOfBirth}` | presence, ISO date | names string35; date |
| Company CRN | `sharesCompanyDetails.companySharesCRN` (unquoted shares, connected-party assets) | same | none | `^[A-Z0-9]{8,10}$` |
| No-CRN reason | `reasonNoCRN` | same | none | string160 |
| Land registry ref | `registryDetails.registryReference` | same | none | string35 |
| UK postcode | `addressDetails.ukPostCode` | same | none | UK/BFPO regex, max 10 |

**Member key / normalisation.** No normalisation is applied. A member is identified by the exact tuple `(firstName, lastName, dob, nino)` in `EtmpMemberAndTransactionsUpdater.merge`, and by the same tuple via `MemberDetails.compare` for update and delete, where `reasonNoNINO` is ignored. The consequences:

- `AB123456C` and `ab 12 34 56 c` are different members. The lower-case or spaced form will then fail the contract regex on POST, giving a 400.
- A case or whitespace change in a name in an uploaded file creates a **new** member. The original member's transactions for that asset class are removed; if it has no others, it is soft-deleted.
- Members without a NINO are keyed by name and DOB only (`nino = None`). Two such members with the same name and DOB collide.
- ⚠️ If ETMP already holds two records with the same key, only the first is kept in the re-posted payload.

⚠️ **CRN contract inconsistency.** The schema description says "Min 7, Max 8 characters (NNNNNNN / ANNNNNN / AANNNNNN)", but the pattern requires 8–10 characters. A valid 7-character CRN is rejected, and 9–10 character values are accepted.

## Technical identifiers

| Identifier | Source | Use | Notes |
|---|---|---|---|
| `CorrelationId` | inbound header, otherwise `UUID.randomUUID` per call | IF header | Not validated. IF-side format expectations are not enforced. GET and POST in one request get different IDs unless the caller supplies one. |
| `requestId` | `hc.requestId` (X-Request-ID); empty string if absent | email callback path → `emailInitiationRequestId` in audit | Unencoded in the path. On callback it is untrusted and audited verbatim. |
| Email callback tokens | AES (`queryParameter.encryption`) over PSA/PSP ID, PSTR, email, scheme name, user name | path segments | Not authenticated beyond decryptability. There is no expiry or binding. The plaintext `submittedBy`, `taxYear` and `reportVersion` can be altered freely. |
| `fileReference` | body `auditContext` | upload audit | Untrusted, audit-only |

## Validation coverage matrix

| Identifier | Format checked locally? | Checked against the authenticated user? | Encoded when put in a URL? |
|---|---|---|---|
| SRN | ✅ regex | ✅ association check | ✅ (header, not URL) |
| PSTR | ❌ | ❌ not linked to SRN | ❌ `String.format` |
| fbNumber | ❌ | ❌ | ❌ |
| periodStartDate | ❌ | ❌ | ❌ |
| psrVersion | ❌ | n/a | ❌ |
| startDate (versions) | ✅ ISO date | n/a | ✅ interpolator |
| PSA/PSP ID (enrolment) | ❌ (trusted source) | ✅ is the auth identity | ✅ encrypted + URL-encoded |
| `psaId` (submit body) | ⚠️ contract only on POST | ❌ | n/a |
| NINO | ⚠️ contract only on POST | n/a | n/a |
| CRN | ⚠️ contract only (inconsistent) | n/a | n/a |
| CorrelationId | ❌ | n/a | n/a (header) |
| requestId | ❌ | n/a | ❌ |
| Callback email | ✅ regex after decrypt | ❌ (no auth) | ✅ encrypted + URL-encoded |

## Recommendations

1. Bind the PSTR to the authorised SRN. Either resolve the PSTR from the SRN through scheme details, or verify that the caller-supplied PSTR belongs to the SRN, and reject a mismatch with 403. Then use that PSTR on every route, path or body.
2. Add local validation for PSTR, fbNumber, periodStartDate and psrVersion, and build ETMP URLs with the `url"…/$pstr?psrFormBundleNumber=$fb"` interpolator so that values are encoded.
3. For PSA submitters, set or verify declaration `psaID` from the authenticated PSA ID. For PSP submitters, verify it against the scheme's authorising PSA.
4. Normalise the NINO (trim, remove spaces, upper-case) and consider case-insensitive, trimmed name matching for the member key. Document the replace-not-append merge semantics for the frontend.
5. Raise the CRN contract inconsistency with the API owners. Until it is resolved, consider a local check aligned with the description.
6. Pass a single correlation ID across all IF calls in one inbound request, and validate or generate it in the IF-required format.
7. Remove the unused `externalId` retrieval, or use it, for example in audit.
