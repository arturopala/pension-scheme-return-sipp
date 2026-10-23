# Pension Scheme Return SIPP – External Dependencies

All outbound HTTP calls use the HMRC `HttpClientV2`, enabled through `play.modules.enabled += "uk.gov.hmrc.play.bootstrap.HttpClientV2Module"`. Base URLs are resolved in `AppConfig` in one of two ways:

- `ServicesConfig.baseUrl("<name>")` reads `microservice.services.<name>.{protocol,host,port}`. The protocol defaults to `http`.
- The custom `Service` config loader, used only for `pensionAdministrator`, reads the same keys but defaults the protocol to **`https`** when none is set.

Path templates for the Integration Framework and pensions-scheme calls live under `serviceUrls.*`. They are `String.format`-ed with request values (`%s`) and appended to the base URL. They are computed **once** at startup as `val`s in `AppConfig`; only the `%s` substitution happens per request. Responses are read as raw `HttpResponse` through `HttpResponseHelper.httpResponseReads`, and non-success statuses are mapped by `HttpResponseHelper.handleErrorResponse`:

| Upstream status | Exception thrown | HTTP to our caller (via bootstrap `JsonErrorHandler`) |
|---|---|---|
| 400 | `BadRequestException` | 400 |
| 404 | `NotFoundException` | 404 |
| other 4xx | `UpstreamErrorResponse(statusCode = s, reportAs = s)` | same 4xx (passed through) |
| 5xx | `UpstreamErrorResponse(reportAs = 502)` | 502 |
| anything else (1xx/3xx/unexpected 2xx) | `UnrecognisedHttpResponseException` | 500 |

## Overview

| # | Dependency | Type | Config key | Accessed via |
|---|---|---|---|---|
| 1 | auth | HMRC platform (auth-client) | `microservice.services.auth` | `PsrAuth` (`AuthorisedFunctions`), `DefaultAuthConnector` bound in `Module` |
| 2 | pensions-scheme (`is-psa-associated`) | HMRC microservice (REST) | `microservice.services.pensionsScheme`, `serviceUrls.is-psa-associated` | `SchemeDetailsConnector` |
| 3 | Integration Framework → ETMP (SIPP PSR APIs) | Backend system (REST, bearer token) | `microservice.services.if-hod`, `serviceUrls.submit-sipp-psr`, `get-sipp-psr`, `get-psr-versions` | `PsrConnector` |
| 4 | pension-administrator (minimal details) | HMRC microservice (REST) | `microservice.services.pensionAdministrator` | `MinimalDetailsConnector` / `MinimalDetailsConnectorImpl` |
| 5 | email | HMRC platform (REST) | `microservice.services.email`, `email.force` | `EmailConnector` |
| 6 | this service (email callback base) | Self-reference | `microservice.services.pensionSchemeReturn`, `serviceUrls.email-callback` | `AppConfig.emailCallback` |
| 7 | datastream (auditing) | HMRC platform (play-auditing) | bootstrap `auditing.*` (from `backend.conf`) | `AuditService` / `AuditServiceImpl`, `ApiAuditUtil` |

---

## 1. Auth

- **Base URL**: `microservice.services.auth`, local default `localhost:8500`.
- **Predicate**: `Enrolment("HMRC-PODS-ORG") or Enrolment("HMRC-PODSPP-ORG")`.
- **Retrievals**: `externalId` and `allEnrolments`.
- **Usage**: every endpoint except the email callback calls `PsrAuth.authorisedAsPsrUser(srn)`. It extracts the PSA ID (identifier `PSAID`) or the PSP ID (identifier `PSPID`) and checks scheme association through dependency #2. See the [Auth section](#auth-guards).

## 2. Pensions Scheme – association check

- **Base URL**: `microservice.services.pensionsScheme`, local default `http://localhost:8203`. The path comes from `serviceUrls.is-psa-associated`.
- **Headers**: `<idType> -> <id>`, where idType is `PSAID` or `PSPID`; `schemeReferenceNumber -> <SRN>`; `Content-Type: application/json`. The caller's `HeaderCarrier` is forwarded, so the user's bearer token goes with it.
- **Timeout**: platform default. `ifs.timeout` is **not** applied.

| Operation | Method & path | Code | Notes / validation |
|---|---|---|---|
| Is PSA/PSP associated with scheme | `GET /pensions-scheme/is-psa-associated` | `SchemeDetailsConnector.checkAssociation` | 200 → body read as JSON `Boolean`. A non-boolean body throws `JsResultException` (500). Non-200 → `handleErrorResponse`. |

**Usage**: called by `PsrAuth.checkPsa`, `checkPsp` and `checkPsaOrPsp`. `false` → `UnauthorizedException` → **401**.

## 3. Integration Framework → ETMP (SIPP PSR)

- **Base URL**: `microservice.services.if-hod`, local default `http://localhost:10702`.
- **Headers** (`PsrConnector.integrationFrameworkHeaders`):
  - `Environment`: `microservice.services.if-hod.env`, default `local`.
  - `Authorization`: `Bearer <microservice.services.if-hod.authorizationToken>`, default `local`. Read lazily.
  - `Content-Type: application/json`.
  - `CorrelationId`: taken from the inbound `CorrelationId` header if present, otherwise a random UUID **per call**. A single inbound request that does GET-then-POST therefore sends two different correlation IDs unless the caller supplies one.
- **Timeout**: `ifs.timeout` (40s) on all three operations.

| Operation | Method & path | Code | Notes / validation |
|---|---|---|---|
| Get SIPP PSR | `GET /pension-online/scheme-return/SIPP/{pstr}?psrFormBundleNumber={fb}` **or** `…/{pstr}?periodStartDate={d}&psrVersion={v}` | `PsrConnector.getSippPsr` | `buildParams` prefers fbNumber, otherwise needs both periodStartDate and psrVersion, otherwise throws `BadRequestException("Missing url parameters")`. 200 → `as[SippPsrSubmissionEtmpResponse]`. 404, or 422 with a body containing `PSR_NOT_FOUND` → `None`. Other statuses → `handleErrorResponse`. |
| Submit / compile SIPP PSR (API 1997) | `POST /pension-online/scheme-return/SIPP/{pstr}` | `PsrConnector.submitSippPsr` | 1) Payload size > `etmpConfig.maxRequestSize` (512000 bytes) → audit events, then **413** `RequestEntityTooLargeException`. 2) JSON-schema validation against `/schemas/api-1997-…-v1.5.0.json`; on errors → `PensionSchemeReturnValidationFailureException` (**400**). 3) Only 200 counts as success. Others → `handleErrorResponse`. The response body is parsed by the service as `{formBundleNumber}`. |
| Get PSR versions | `GET /pension-online/reports/{pstr}/PSR/versions?startDate={yyyy-MM-dd}` | `PsrConnector.getPsrVersions` | 200 → `Seq[PsrVersionsResponse]` (parse failure → failed future → 500). 404 → empty list. Others → `handleErrorResponse`. |

**Usage notes**

- Every write in `SippPsrSubmissionService` is GET followed by POST. The service reads the current return, modifies it and posts the whole return back, with `version = None` on the report and on changed members.
- **Audit**:
  - Every POST (success or failure) fires `PensionSchemeReturnPost` via `ApiAuditUtil.firePsrPostAuditEvent`.
  - A successful POST that carries a `psrDeclaration`, meaning a final submission, also fires `PensionSchemeReturnSubmitted` (audit source `pension-scheme-return-sipp-frontend`).
  - The oversize branch fires `PensionSchemeReturnPost` and, when an `auditContext` was supplied, `PensionSchemeReturnFileUpload`.
  - The GET only logs (`firePsrGetAuditEvent`). It sends no audit event.
- ⚠️ **Observation:** `pstr`, `fbNumber`, `periodStartDate` and `psrVersion` are `String.format`-ed into the path and query **without URL encoding** before the result is passed to `url"$url"`. Only `startDate` in `getPsrVersions` goes through the interpolator as a separate query value.
- ⚠️ **Observation:** `firePsrPostAuditEvent` logs `Json.prettyPrint(payload)` at **INFO**, and `firePSRSubmissionEvent` does the same. That payload is the full return, including member names, NINOs and dates of birth.

## 4. Pension Administrator – minimal details

- **Base URL**: `microservice.services.pensionAdministrator` via the `Service` loader. Local: `http://localhost:8205`. In environments where `protocol` is absent it defaults to **https**.
- **Headers**: `psaId: <id>` or `pspId: <id>`, plus `loggedInAsPsa: true|false`. The caller's `HeaderCarrier` is forwarded.
- **Timeout**: `ifs.timeout`.

| Operation | Method & path | Code | Notes / validation |
|---|---|---|---|
| Get minimal details for self | `GET /pension-administrator/get-minimal-details-self` | `MinimalDetailsConnectorImpl.fetch(PsaId)` / `fetch(PspId)` | Typed read `execute[MinimalDetails]`. A 404 whose message contains `no match found` → `Left(DetailsNotFound)`. A 403 whose message contains `DELIMITED_PSAID` → `Left(DelimitedAdmin)`. Any other error is logged at ERROR and rethrown. |

**Usage**

- `SippPsrSubmissionService.submitWithRequest` calls this before **every** POST, for audit details. A `Left` becomes `RuntimeException("Failed tp submit psr: …")` → **500**.
- `EmailSubmissionService.getMinimumDetails` calls it again after a final submission, to get the email address and name. A `Left` there means "email not sent" (`emailSent: false`).
- ⚠️ **Observation:** in the email step, a *thrown* error such as a 5xx, a timeout or a parse error is not caught. A return that ETMP has already accepted can therefore be reported to the frontend as a 5xx.

## 5. Email

- **Base URL**: `microservice.services.email`, local default `localhost:8300`.
- **Auth/headers**: platform default (forwarded `HeaderCarrier`).

| Operation | Method & path | Code | Notes / validation |
|---|---|---|---|
| Send templated email | `POST /hmrc/email` | `EmailConnector.sendEmail` | Body is `SendEmailRequest{to, templateId = pods_pension_scheme_return_sipp_submitted, parameters{psaName, schemeName, periodOfReturn, dateSubmitted}, force = email.force (default false), eventUrl}`. 202 → `Right(Done)`. Any other status or exception → `Left(...)`. Never fails the request. |

**Callback URL** (`EmailConnector.callBackUrl` + `AppConfig.emailCallback`):
`{pensionSchemeReturn base}/pension-scheme-return-sipp/psr/email-response/{PSA|PSP}/{requestId}/{enc(email)}/{enc(psa/pspId)}/{enc(pstr)}/{enc(schemeName)}/{enc(userName)}/{taxYear}/{reportVersion}`.

- Encrypted values use AES (`SymmetricCryptoFactory.aesCryptoFromConfig("queryParameter.encryption")`) and are then URL-encoded.
- `requestId`, `taxYear` and `reportVersion` are plaintext and unencoded.
- The full callback URL is logged at INFO. It contains encrypted values only.
- ⚠️ **Observation:** `application.conf` commits a literal value for `queryParameter.encryption.key`. It must be overridden per environment.

## 6. Self-reference (callback base)

`microservice.services.pensionSchemeReturn`, local `http://localhost:10704`, is used only to build the email `eventUrl` above.

## 7. Auditing

`AuditServiceImpl.sendEventWithSource` sends `ExtendedDataEvent`s through the bootstrap `AuditConnector`. The default `auditSource` is `appName`. Failures are only logged.

| Audit type | Trigger | Audit source |
|---|---|---|
| `PensionSchemeReturnPost` | every ETMP POST, including the oversize short-circuit | `pension-scheme-return-sipp` |
| `PensionSchemeReturnSubmitted` | successful POST with a declaration (final submit) | `pension-scheme-return-sipp-frontend` |
| `PensionSchemeReturnFileUpload` | oversize payload with an `auditContext` | `pension-scheme-return-sipp-frontend` |
| `PensionSchemeReturnEmailEvent` | each non-`Opened` email event received on the callback | `pension-scheme-return-sipp` |

---

## Datastore

**None.** The service has no MongoDB or other persistence dependency, and all return state lives in ETMP.

⚠️ **Observation:** `config/Crypto.scala` (`CryptoImpl`, which reads `mongodb.encryption`) is dead code. Nothing injects it, and no `mongodb.encryption` config exists.

## Auth guards

| Guard | Requires | Retrieves / yields |
|---|---|---|
| `requiredHeaders("srn")` (`PsrBaseController`) | `srn` header present. Runs **before** auth. | SRN string |
| `PsrAuth.authorisedAsPsrUser(srn)` | SRN matches `^S[0-9]{10}$`, otherwise 400 `Invalid scheme reference number`. Active session with the PSA or PSP enrolment. | `externalId`, `allEnrolments` → `PsrAuthContext(externalId, psaPspId)` |
| ↳ `requestRole: PSA` | `PSAID` identifier on `HMRC-PODS-ORG` + association(PSAID, SRN) | `PsaId` |
| ↳ `requestRole: PSP` | `PSPID` identifier on `HMRC-PODSPP-ORG` + association(PSPID, SRN) | `PspId` |
| ↳ no `requestRole` | PSA ID if present, otherwise PSP ID + association | `PsaId` or `PspId` |
| ↳ any other `requestRole` | → `BadRequestException` (400) | – |
| Email callback | **none** | – |

⚠️ **Observation:** authorisation is done against the **SRN header**. The **PSTR** actually used for ETMP comes from the path or body and is never checked against that SRN. See `identifiers.md`.

## Inbound callers and callbacks

- **pension-scheme-return-sipp-frontend** calls every `/pension-scheme-return-sipp/psr/**` route. The service also attributes some audit events to the frontend.
- **Email service** calls back with `POST /pension-scheme-return-sipp/psr/email-response/:submittedBy/:requestId/:email/:psaId/:pstr/:schemeName/:userName/:taxYear/:reportVersion`. This route is **unauthenticated**, so the only guard is successful AES decryption plus an email regex check. The service registers this URL as `eventUrl` on every send.
- Platform health: `/ping/ping` (`health.Routes`).

## Libraries

| Library | Version | Backs |
|---|---|---|
| `bootstrap-backend-play-30` | 10.7.0 | Play wiring, `JsonErrorHandler`, `AuditConnector` |
| ↳ `http-verbs-play-30` | 15.8.0 | `HttpClientV2`, `HttpReads`, `UpstreamErrorResponse` |
| ↳ `auth-client-play-30` | 8.8.0 | `AuthorisedFunctions`, enrolment retrievals |
| ↳ `play-auditing-play-30` | 9.7.0 | `ExtendedDataEvent` auditing |
| ↳ `crypto` | 8.4.0 | AES encryption of callback URL values |
| `json-schema-validator` (networknt) | 1.5.9 | Outbound API 1997 payload validation |
| `jackson-module-scala` | 2.21.1 | `JsValue` → `JsonNode` for the validator |
| `enumeratum-play-json` | 1.9.5 | Enums (`JourneyType`, `Journey`, `YesNo`, statuses, email events) |
| `chimney` | 1.8.2 | API ↔ ETMP model transformations |
| `cats-core` | 2.13.0 | `NonEmptyList`, `EitherT`, `OptionT` |

## Local development

The service runs on port 10704 (`PlayKeys.playDefaultPort`). `sm2 -start PSR_ALL` starts the related services. Local defaults from `application.conf`:

| Dependency | Local target |
|---|---|
| auth | `localhost:8500` |
| pensions-scheme | `http://localhost:8203` |
| Integration Framework (`if-hod`) | `http://localhost:10702`, `env = ist0`, token placeholder |
| pension-administrator | `http://localhost:8205` |
| email | `localhost:8300` |
| callback base (self) | `http://localhost:10704` |

The integration tests (`it/test/.../connectors/*Spec`) exercise `EmailConnector`, `PsrConnector` and `SchemeDetailsConnector` against WireMock.
