# Pension Scheme Return SIPP – Summary

## Purpose

This is the backend for the **Pension Scheme Return (PSR) for Self-Invested Personal Pensions (SIPP)**. Pension Scheme Administrators (PSAs) and Pension Scheme Practitioners (PSPs) use it to compile, amend and submit the annual SIPP return for a registered pension scheme. The return covers each scheme member and the assets the member held during the tax year: land and property, connected-party assets, tangible moveable property, outstanding loans and unquoted shares.

It has no UI. Its only caller is the *pension-scheme-return-sipp-frontend*, which sends both screen answers and the results of bulk file uploads (CSV). The service does not store return data itself. Every change is a read-modify-write against the tax platform's system of record (ETMP, reached through the Integration Framework). Each write creates a new compiled version of the return there. When the return is finally submitted, the service sends a confirmation email and records email-delivery events in the audit trail.

## Main Capabilities

### 1. Creating a return

- Creates an empty, *compiled* return for a scheme and reporting period. The return is marked as having no member transactions yet.
- Returns the form bundle number that the tax platform assigns.

### 2. Recording member transactions per asset class

There are six asset classes ("journeys"): interest in land or property from a connected party, arm's-length land or property, assets acquired from a connected party, tangible moveable property, outstanding loans, and unquoted shares.

- For each asset class the caller sends the complete list of transactions for all members. The service fetches the current return, merges the list in, and writes the whole return back as *compiled*.
- **The merge replaces, it does not append.** It groups transactions by member, using first name + last name + date of birth + NINO as an exact-match key:
  - A member who already has transactions for this asset class has them replaced.
  - A member who exists in the return but is absent from the upload loses their transactions for this asset class.
  - New members are added with status *New*.
  - A member left with no transactions in any asset class is marked *Deleted* (soft delete).
- An empty upload clears the asset class. The asset-class declaration ("did any member hold X?") is set to Yes or No to match.
- Per-asset-class reads return the transactions flattened per member, excluding soft-deleted members.

### 3. Return-level details

- Toggle the overall "any member transactions?" answer.
- Set or replace the scheme's accounting periods.
- Read the full return, the member list, the transaction counts per asset class, and the asset-class declarations.
- List all versions of returns for a scheme from a given period start date, including status and submitter.

### 4. Member maintenance

- **Update a member's personal details** (name, date of birth, NINO or reason for no NINO). The member is found by exact match on the current details and marked *Changed*. If no member matches, nothing is written.
- **Delete a member.** This is a soft delete (status *Deleted*), and any existing declarations are reset to "not declared".
- **Delete all assets of one class.** Every live member's transactions for that class are zeroed. Members left with no assets are soft-deleted, the asset-class declaration is cleared, and declarations are reset.

### 5. Submission and declaration

- Takes the latest compiled return and sets it to *Submitted*. It fills in any asset-class declarations still missing, based on whether transactions exist, and adds a declaration naming the submitter (PSA or PSP), the submitter's ID and the approving PSA ID.
- Submissions can be *Standard* or *Amend*. An amendment is flagged in audit as "changed compiled" or "changed submitted".

### 6. Notifications

- After a successful submission, the service looks up the submitter's name and email address with the pension-administrator service. It then sends a "return submitted" email, giving the scheme name, the period of the return and the submission time.
- The email service is given a callback URL. Scheme ID, PSTR, email address, scheme name and user name are encrypted inside that URL. Delivery events (sent, delivered, bounced, complained; *opened* is ignored) are posted back to the service and written as audit events.
- The submit response tells the caller whether the email was sent. An email failure does not undo the submission.

## Cross-cutting Concerns

- **Authentication and authorisation.** The caller must be logged in with a PSA enrolment (`HMRC-PODS-ORG`) or a PSP enrolment (`HMRC-PODSPP-ORG`). A `requestRole` header can force PSA or PSP. The scheme reference number (SRN) from the `srn` header must be associated with that PSA or PSP, which is checked against the pensions-scheme service. The email callback endpoint is unauthenticated.
- **Auditing.** Every return write is audited (`PensionSchemeReturnPost`), including the full payload and the upstream outcome. Final submissions are audited separately (`PensionSchemeReturnSubmitted`). File uploads that exceed the size limit (`PensionSchemeReturnFileUpload`) and email events (`PensionSchemeReturnEmailEvent`) are also audited. Several events are deliberately attributed to the frontend as audit source.
- **Validation and transformation.** Incoming JSON is checked for structure only: required fields and types. Formats are not checked. Before anything is sent upstream, the outbound payload is checked against the bundled API 1997 JSON schema and a 500 KB size limit. The service maps between a member-centric frontend model and the platform's member-and-transactions model.
- **Data protection.** There is no local persistence. Values carried in the email callback URL are AES-encrypted. ⚠️ Full return payloads, including NINOs, names and dates of birth, are written to INFO logs.
- **Feature toggles.** `email.force` is passed through to the email service. No other toggles exist.

## Dependencies (generic)

| Kind | Role |
|---|---|
| Auth platform | Identifies the user and their PSA/PSP enrolment |
| Scheme registry (pensions-scheme) | Confirms that the PSA or PSP is associated with the scheme (SRN) |
| Tax platform via Integration Framework (ETMP) | System of record for SIPP returns: read, write/submit, list versions |
| Pension administrator service | Submitter name and email address for the confirmation email and for audit |
| Email service | Sends the submission confirmation and calls back with delivery events |
| Audit platform | Audit trail of writes, submissions, upload failures and email events |

## In One Sentence

A stateless backend that lets authorised pension scheme administrators and practitioners build, amend and submit the annual SIPP Pension Scheme Return. It merges per-asset-class transactions into the return held by the tax platform, validates the payload against the platform's schema, and confirms submission by email with audited delivery tracking.
