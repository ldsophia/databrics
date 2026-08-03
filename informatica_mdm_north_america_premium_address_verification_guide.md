# Informatica MDM SaaS — North America Premium Address Verification Guide

**Research date:** 2026-08-03  
**Audience:** Informatica MDM SaaS / Customer 360 architects, MDM developers, Cloud Data Quality developers, Cloud Data Integration developers, data stewards, and platform administrators

---

## 1. Executive recommendation

### Recommended processing point

**Yes: postal-address verification and standardization should normally run on each source record before match, merge, survivorship, and golden-record generation.**

The preferred logical sequence is:

```text
Source data
  -> basic parsing and country normalization
  -> address verification / correction / standardization
  -> retain verification status and quality metadata
  -> route invalid or ambiguous records
  -> MDM source-record ingestion
  -> match
  -> merge and survivorship
  -> golden record
  -> downstream publication
```

This order is important because standardized values such as street name, locality, region, postal code, and unit information can improve matching consistency. It also prevents a badly formatted or undeliverable address from winning survivorship without a quality indicator.

### MDM SaaS recommendation

For **Multidomain MDM SaaS / Customer 360**, use the **Enrichment and Validation Orchestrator (EVO) Address Verification plugin on Source Records** when you want MDM-native validation for ingress, file imports, REST-created source records, and Business UI updates.

Do **not** configure only a Master Record objective for batch ingress. Informatica's EVO material states that:

- Source-record objectives can trigger for ingress, file import, business applications, and REST APIs.
- Master-record objectives can trigger for master records created or updated through business applications and REST APIs, but are **not triggered by ingress or file import**.

Therefore, an ingress-oriented address-verification objective should be associated with **Source Records**, optionally restricted by source system.

### CDI recommendation

For a large initial load, recurring batch ingestion, or when you want to reject/quarantine bad addresses before they reach MDM, use:

```text
CDI source -> Cloud Data Quality Verifier transformation -> decision/routing -> MDM SaaS target
```

Create the reusable Verifier asset in Cloud Data Quality and use it in a CDI mapping. This is generally preferable to making a row-by-row REST call from CDI.

### REST API recommendation

Use the **Cloud Address Verification REST API** when an external application requires synchronous point-of-entry verification, or when an API-led architecture needs address verification independently of MDM ingestion.

The North America Premium Address Cleansing subscription does **not by itself** entitle direct use of the Cloud Data Quality Address Verification REST API outside MDM SaaS. The current Informatica product schedule states that a separate **IDMC IPU subscription** is required for that REST API. Confirm the entitlement in your order before designing the API path.

---

## 2. Product-name and licensing clarification

The commercial name used in Informatica's current product schedule is **Premium Address Cleansing and Validation for Cloud**, available as country, regional, or subregional subscriptions. The schedule lists a **North American Regional Pack**.

Important points:

1. It is designed for use with **Cloud Data Quality Verifier or MDM SaaS**.
2. It includes address-standardization dictionaries, a validation engine, and required postal reference datasets for licensed countries.
3. It is subscribed per instance and by country, region, or subregion.
4. The current schedule explicitly says that the **North American Regional Pack does not include Mexico**.
5. Informatica says regional pack country details are available on request and can change. Obtain a written country list for your order rather than assuming the exact coverage.
6. The current schedule describes up to two million real-time Address Verification web-service transactions per year for use with MDM SaaS for subscribed premium countries, subject to the order and stated exclusions. These transactions do not authorize API use outside MDM SaaS.
7. Direct Cloud Data Quality Address Verification REST API access requires an **IDMC IPU subscription**.
8. Geocoding is a separate option and requires Premium Address Cleansing and Validation. Do not assume latitude/longitude enrichment is included in the regional address pack.
9. Nonproduction and production entitlements, instance counts, transaction limits, and expiration rules should be verified against your signed order.

---

## 3. Where address verification belongs in the MDM lifecycle

### 3.1 Primary control: before match and golden-record generation

Run verification and standardization on source-record address data before MDM matching when any address component participates in match rules, candidate selection, survivorship, trust, or stewardship decisions.

Example:

```text
Source A: 100 N Main St., Ste 200, Chicago, IL 60601
Source B: 100 North Main Street #200, Chicago, Illinois 60601-1234
```

Without standardization, these records may appear less similar. With postal standardization, the match engine receives more comparable values.

### 3.2 Secondary control: after manual edits or golden-record updates

A second real-time validation can be useful when a data steward or application user changes the master/golden record directly. This is a secondary safeguard, not a replacement for source-level verification.

Recommended model:

- **Source-record objective:** handles ingress, file import, source REST APIs, and source-record UI changes.
- **Master-record objective:** handles direct master-record edits through supported Business UI or REST API paths.
- Avoid invoking both for the same unchanged address unless there is a clear business reason, because duplicate calls increase cost and latency.

### 3.3 Address verification is not identity verification

A verified address means that the postal address is recognized and/or deliverable according to reference data. It does not prove that:

- the company currently operates there;
- the contact currently lives or works there;
- the named party owns the property;
- two parties at the same address are the same entity.

Do not use postal verification status as a unique identity key. Combine standardized address data with company/person name, tax identifiers, email, phone, source identifiers, and other domain attributes.

---

## 4. Architecture decision table

| Requirement | Preferred pattern | Reason |
|---|---|---|
| Initial historical load or high-volume recurring batch | CDQ Verifier in a CDI ingress mapping | Efficient batch processing, reusable asset, quarantine before MDM, detailed output mapping |
| MDM-native validation for ingress, file import, UI, and MDM REST paths | MDM SaaS EVO Address Verification plugin on Source Records | Centralized MDM rule association, source-system scoping, objectives, validation handling |
| Data steward edits a master record | EVO objective for Master Records | Validates direct master updates; not sufficient for ingress/file import |
| External website or application needs synchronous verification before submitting to MDM | Cloud Address Verification REST API, or application calls MDM and lets EVO execute | Direct API gives synchronous service integration; MDM/EVO keeps logic centralized |
| Expose an existing CDQ Verifier flow as a service | Put the Verifier in a mapplet and enable the mapplet as an API | Reuses the same Verifier asset and mapping logic |
| CDI batch process considering row-by-row REST calls | Prefer the Verifier transformation | Simpler operations, less custom token/retry logic, designed for mapping execution |
| Need geocodes | Add licensed geocoding option and map geocode status/accuracy | Geocoding is not automatically included with Premium Address Cleansing |

---

## 5. Data-model design for Company and Contact

### 5.1 Model the address as a reusable field group

For Company and Contact entities, use a consistent address field group or child entity where possible. Include:

- Address role/type: registered, headquarters, billing, shipping, mailing, home, work, other
- Address line 1
- Address line 2
- Building or premise
- Sub-building, suite, unit, or apartment
- House number
- Street/thoroughfare
- City/locality
- District/dependent locality
- State/province/region
- Postal code
- Country code
- Start and end dates, where addresses are temporal
- Primary-address indicator

A complete country value is essential. Prefer a governed ISO country code from Reference 360 or an equivalent controlled list.

### 5.2 Preserve raw and standardized values

Do not destroy source lineage. Preserve the original source values through the source-record/crosswalk model and store or expose standardized outputs separately where your model allows it.

Suggested result metadata:

- `AddressVerificationStatus`
- `AddressResultQuality`
- `AddressMatchPercentage`
- `AddressTypeIndicator`
- `AddressVerificationTimestamp`
- `AddressVerificationRuleVersion`
- `AddressVerificationProvider`
- `AddressVerificationMessage`
- `AddressCorrectionApplied`
- `AddressEnrichmentStatus`
- `GeocodeStatus` and `GeocodeAccuracy`, only when licensed

### 5.3 Minimum output fields to retain

At minimum, retain:

- standardized address lines or standardized discrete components;
- standardized city, region, postal code, and country;
- verification status code;
- result quality or equivalent confidence indicator;
- verification timestamp;
- original source record and source-system lineage.

### 5.4 Standard verification status codes

Cloud Data Quality Verifier documents these summary values:

| Code | Meaning | Typical MDM handling |
|---|---|---|
| `V` | All postally relevant elements verified | Accept, subject to other DQ and match rules |
| `A` | One or more relevant address elements added | Accept or review based on result quality and business policy |
| `C` | One or more input elements corrected | Accept or review; retain correction indicator |
| `I` | Address cannot be corrected and is not valid | Flag, quarantine, downgrade trust, or reject according to policy |
| `N` | Address cannot be verified because reference data is unavailable or country is unsupported | Do not label automatically as invalid; route based on cause |

Use the detailed result fields in addition to the single status code. A simple `V/A/C = good` and `I/N = bad` rule may be too coarse for production.

---

## 6. Option A — Configure Address Verification natively in MDM SaaS with EVO

> **Release note:** EVO is a newer MDM SaaS capability, and exact page names can differ by release and tenant entitlement. Informatica's public 2026 material identifies Address Verification as an EVO plugin. Validate the exact menu path in your tenant and current MDM documentation.

### Step 1 — Confirm product entitlement and tenant readiness

Ask Informatica or your CSM/support team to confirm:

- North American Regional Premium Address Cleansing is active for the correct production and nonproduction MDM SaaS instances.
- Exact countries included in your regional pack.
- Real-time transaction entitlement and annual limit.
- Batch license-file entitlement.
- Whether the EVO Address Verification plugin is enabled in your tenant release.
- Whether geocoding, CASS, SERP, or other certified output is licensed.

### Step 2 — Configure DaaS license details in MDM Global Settings

In the documented MDM SaaS flow:

1. Open **Global Settings**.
2. Click **Edit**.
3. For real-time Address Verification, enter the **Account ID** and **Cloud Token** supplied through Informatica.
4. For batch Address Verification, upload an Address Verification **6.5.0 or later license file** supplied by Informatica.
5. Save the settings.
6. Confirm that the DaaS services become active.

Do not rename the license file. Store credentials in platform-managed settings, not in mappings or plain-text parameters.

### Step 3 — Prepare the Company and Contact address field groups

1. Confirm that all address components have appropriate string lengths and data types.
2. Add country code as a required or strongly governed field for countries in scope.
3. Decide whether standardized output overwrites operational fields or populates dedicated standardized fields.
4. Add verification status and audit fields.
5. For multiple addresses, include address role and an address identifier so each address instance can be verified independently.
6. Publish/deploy the model changes before configuring rule associations.

### Step 4 — Create the EVO data-enhancement rule

In MDM Design / Business 360 Console, using the labels available in your release:

1. Create a **Data Enhancement Rule**.
2. Choose the **Address Verification** plugin under cleansing and standardization.
3. Give the rule a release-controlled name, for example:
   - `DER_AV_NA_COMPANY_ADDRESS_V1`
   - `DER_AV_NA_CONTACT_ADDRESS_V1`
4. Add a description containing scope, countries, expected input model, and owner.
5. Select batch or real-time behavior as supported by the plugin and intended trigger.
6. Configure standardization preferences such as output format, country handling, casing, language/script, and certification only where required and supported.

### Step 5 — Map plugin input fields

Map the business entity fields to the Address Verification inputs. Typical mapping:

| MDM field | Address Verification input |
|---|---|
| Address Line 1 | Delivery address line / address line 1 |
| Address Line 2 | Address line 2 / sub-building information |
| City | Locality |
| State/Province | Province / administrative division |
| Postal Code | Postal code |
| Country Code | Country ISO code |
| Company name, when appropriate | Organization name |
| Contact name, only when needed by the selected process | Contact element |

Guidelines:

- Prefer discrete input fields when the MDM model has separate components.
- Do not map a concatenated address to a discrete input field.
- Make country mapping explicit; do not rely on weak country inference for a mixed-country load.
- Ensure suite/unit data is not discarded during concatenation.

### Step 6 — Map plugin output fields

Map at least:

- standardized address line(s);
- standardized locality/city;
- standardized state/province;
- standardized postal code;
- standardized country code/name as needed;
- verification status;
- result quality and/or match percentage;
- address type;
- enrichment and certification status, where used;
- geocode fields only when licensed.

Retain the raw source value through source lineage. Do not silently overwrite a user's input without making the correction visible to stewards or auditable.

### Step 7 — Create the rule association

1. Create a **Rule Association** for the Company address field group.
2. Associate the Company address fields with the data-enhancement rule inputs and outputs.
3. Repeat for Contact if its field paths differ.
4. If the address model is identical and your release permits reuse, reuse the same core rule while creating entity-specific associations.
5. Add clear descriptions because they can be surfaced in error handling and future support.

### Step 8 — Create an objective and objective group

1. Create an objective such as `OBJ_VERIFY_NA_SOURCE_ADDRESS`.
2. Add the Company and/or Contact address rule associations.
3. Sequence rules if preliminary country normalization or required-field validation must run first.
4. Place the objective in an objective group for migration and lifecycle management.
5. Use sequential execution when the output of one rule becomes the input of the next. Use parallel execution only for independent operations.

Suggested sequence:

```text
1. Normalize country and state reference values
2. Validate required address components
3. Run Address Verification
4. Evaluate status/result quality
5. Set DQ flag, trust action, or rejection action
```

### Step 9 — Configure record type and trigger

For batch ingress and file import:

1. Select **Source Records**.
2. Select all relevant source systems or explicitly list approved source systems.
3. Enable the ingress/file-import trigger available in your release.
4. Include source REST API and source Business UI triggers where needed.

For direct master edits:

1. Create a separate objective or association for **Master Records**.
2. Use business-application submission/section-save and master REST update triggers as supported.
3. Do not expect a Master Record objective to process ordinary ingress/file imports.

Avoid field-focus triggers for expensive external verification unless immediate feedback is a hard requirement. Section save or record submission usually gives a better cost/latency balance.

### Step 10 — Configure error, rejection, and trust behavior

During the first rollout, use **observe/flag mode** rather than rejecting every unverified address.

Recommended baseline:

- `V`: pass.
- `A` or `C`: pass with correction metadata; optionally notify the steward.
- `I`: mark invalid, lower trust, and send to an exception queue; reject only when the business process requires a deliverable address.
- `N`: flag as unverifiable/unsupported, not automatically invalid.
- Service unavailable: generally fail open for bulk ingress with a retry flag; consider fail closed for a critical point-of-entry transaction only when the user can correct or retry.

Use a dedicated `AddressDQDisposition`, for example:

- `ACCEPTED_VERIFIED`
- `ACCEPTED_CORRECTED`
- `REVIEW_REQUIRED`
- `INVALID`
- `UNSUPPORTED_OR_NO_REFERENCE`
- `SERVICE_ERROR_RETRY`

### Step 11 — Activate, test, and deploy

1. Validate all mappings.
2. Publish or activate the rule, association, objective, and group.
3. Run unit tests in nonproduction.
4. Confirm the source record contains standardized values before the match outcome is finalized.
5. Test Company and Contact separately, including multiple-address child records.
6. Migrate the objective group and related assets through your normal deployment process.
7. Re-enter production credentials/license details through approved secret-management procedures; do not export them as ordinary configuration data.

---

## 7. Option B — Configure a CDQ Verifier asset and use it in the CDI ingress mapping

This option is recommended for high-volume batch processing, a backfill of existing MDM data, or a design that requires invalid records to be quarantined before MDM ingestion.

### Step 1 — Configure the Secure Agent for Address Verification

In **Administrator**:

1. Open **Runtime Environments**.
2. Select the Secure Agent that will run the mapping.
3. Choose **Edit Secure Agent**.
4. Under **System Configuration Details**:
   - Service: `Data Integration Server`
   - Type: `CDQAV`
5. Review the Address Verification properties.
6. Provide the license file for each reference-data type you will download and use.
7. Place license files in a directory readable by the Secure Agent, such as an approved `avLicenseFile` directory.
8. Accept defaults initially unless capacity or support guidance requires changes.
9. For a Linux Secure Agent in a container, Informatica documents at least 1 GB of `/dev/shm` memory for address-verification operations.

The Secure Agent downloads applicable reference data when the mapping runs. Plan disk, network, memory, and first-run time accordingly.

### Step 2 — Create the Verifier asset in Cloud Data Quality

1. Open **Data Quality**.
2. Create a new **Verifier** asset.
3. Name it clearly, for example `VRF_NA_POSTAL_ADDRESS_V1`.
4. Choose an input model:
   - **Discrete** when street, city, state, postal code, and country are in separate fields.
   - **Hybrid** when the source has a mixture of discrete and combined fields.
   - **Multiline** when one or more lines contain multiple address elements.
5. Select the input fields that correspond exactly to your source structure.
6. Select output fields.

Recommended output groups:

- Single address elements
- Preformatted address fields, when needed for labels/display
- Status Codes
- Result Quality
- Address Type
- Match Percentage or detailed result indicators where available
- Certification fields only when required
- Country-specific enrichments only when there is a defined business use
- Geocoding fields only when licensed

### Step 3 — Configure Verifier properties

Configure only the properties required by your use case, such as:

- destination country or country derivation;
- verification level;
- output casing;
- preferred language and script;
- descriptor standardization;
- invalid-address standardization policy;
- suggestions for ambiguous/incomplete addresses;
- certified processing where required.

Avoid retrieving every enrichment field. Extra outputs add mapping complexity, storage, and potentially licensing considerations.

### Step 4 — Validate and test the Verifier asset

Test representative addresses for:

- United States and Canada or other explicitly licensed countries;
- valid exact addresses;
- misspellings;
- missing postal codes;
- missing unit/suite;
- PO boxes;
- rural routes;
- ambiguous street names;
- unsupported countries;
- Mexico, because the North American Regional Pack is documented as excluding Mexico;
- empty or incorrect country codes.

Review status, result quality, standardized fields, and any suggestions rather than checking only whether the mapping completed.

### Step 5 — Add the Verifier transformation to the CDI mapping

Recommended mapping structure:

```text
Source
  -> Expression: trim fields, normalize nulls, map country code
  -> Verifier transformation: VRF_NA_POSTAL_ADDRESS_V1
  -> Expression: derive DQ disposition and changed-address hash
  -> Router:
       accepted -> MDM SaaS source target
       review   -> MDM plus exception flag, or stewardship target
       rejected -> quarantine table/file
       retry    -> retry queue
```

In the Verifier transformation:

1. Select the Verifier asset.
2. Map incoming source fields to the Verifier inputs.
3. Confirm that discrete fields map to discrete inputs and combined lines map to multiline/hybrid inputs.
4. Map standardized outputs and status fields to downstream fields.
5. Connect approved outputs to the MDM ingress target.
6. Validate the mapping and synchronize the transformation if the Verifier asset changes.

### Step 6 — Map results into MDM

Map standardized address fields used by match and survivorship. Preserve:

- source-system identifier;
- source primary key;
- raw address lineage;
- standardized address;
- verification status and result quality;
- verification date/time;
- processing disposition.

If MDM match rules use address fields, verify that they reference the standardized fields populated by the mapping or the MDM-native rule.

### Step 7 — Prevent unnecessary repeat transactions

Calculate a stable hash over materially relevant input fields, for example:

```text
UPPER(TRIM(AddressLine1)) |
UPPER(TRIM(AddressLine2)) |
UPPER(TRIM(City)) |
UPPER(TRIM(StateProvince)) |
UPPER(TRIM(PostalCode)) |
UPPER(TRIM(CountryCode))
```

Re-run verification only when this hash changes, when the verification has expired under your policy, or when a significant engine/reference-data change requires reprocessing. Persist only your own record-level results and confirm contractual restrictions before implementing any broader cache.

### Step 8 — Monitor operational behavior

Track:

- total submitted records;
- `V`, `A`, `C`, `I`, and `N` counts;
- service failures and retries;
- processing time and throughput;
- reference-data download or license errors;
- transaction consumption;
- percentage of addresses changed by verification;
- match-rate and false-positive changes after standardization.

---

## 8. Option C — Call the Cloud Address Verification REST API

### When to use it

Use the direct REST API for:

- e-commerce or customer-registration point-of-entry validation;
- synchronous external application workflows;
- API-first services that verify an address before calling MDM;
- an Application Integration process or custom service that needs direct control over the request and response.

Do not default to row-by-row REST calls inside a large CDI batch mapping when the Verifier transformation satisfies the requirement.

### Licensing requirement

The current product schedule says the Premium Address Cleansing and Validation subscription does not itself provide access to the Cloud Data Quality Address Verification REST API. An **IDMC IPU subscription** is required. Verify the order, tenant access, and IPU metering model.

### Endpoint pattern

The current documentation shows:

```http
POST <baseApiUrl>/dataquality/addressverification/api/v1/process
Content-Type: application/json
Authorization: Bearer <jwt_token>
```

For the North America IDMC API region, the documented base URL is:

```text
https://idmc-api.dm-us.informaticacloud.com/
```

Use your tenant's actual pod/region and current documentation. Obtain a JWT through the supported IDMC authentication flow. Do not store access tokens in source code.

### Swagger

The current documentation provides Swagger at a tenant/service URL similar to:

```text
https://<service-url>/dataquality/addressverification/docs/swagger-ui/index.html
```

A JWT is required. Use the Swagger definition from your tenant to build request bodies rather than copying an old payload from an example.

### CDI implementation only when required

When a CDI flow must call the API:

1. Confirm IPU and Address Verification API entitlements.
2. Create a managed REST V2/Application Integration connection.
3. Use secure credential/token handling.
4. Configure request timeouts.
5. Implement bounded retries with backoff for transient errors.
6. Handle HTTP 401/403, 429, and 5xx separately.
7. Route permanent address failures differently from service failures.
8. Add idempotency or a changed-address check.
9. Avoid logging full postal addresses and tokens.
10. Monitor IPU consumption and response latency.

### Alternative: expose a Verifier mapplet as an API

Cloud Data Quality documentation states that a mapplet containing a Verifier transformation can be enabled as an API. This can be useful when you want one governed Verifier asset used by both mappings and real-time consumers.

---

## 9. Match, merge, trust, and survivorship design

### 9.1 Match against standardized fields

Use standardized address components in candidate selection or fuzzy match rules, but combine them with other attributes.

For Company:

- standardized legal/trading name;
- standardized postal address;
- tax/registration identifier;
- website/domain;
- phone;
- source-system keys.

For Contact:

- standardized person name;
- email and phone;
- standardized address;
- associated company;
- source identifiers.

### 9.2 Handle shared addresses carefully

Many valid entities share an address:

- apartment buildings;
- office towers;
- campuses;
- coworking locations;
- registered-agent addresses;
- family households.

Missing unit or suite data can create false matches. Do not auto-merge solely because two addresses standardize to the same building.

### 9.3 Use verification status in trust/survivorship

Possible policy:

- Verified/corrected address with high result quality receives higher trust.
- Invalid address receives lower trust.
- Unsupported/unverifiable status does not necessarily receive the same penalty as a confirmed invalid address.
- Recent verification may rank above stale verification when all other factors are equal.
- A source with authoritative legal-address data can still outrank another source, but its address quality should be visible.

Do not let the correction service alone overwrite an authoritative regulatory value without an approved business rule.

### 9.4 Avoid golden-only cleansing

If only the golden record is verified after match/merge:

- raw variants may reduce match quality;
- bad source addresses may create incorrect clusters;
- survivorship may choose a poor value before validation;
- source-level quality issues become harder to trace and remediate.

Golden-record verification is useful as a final control, but source-level standardization is the primary design.

---

## 10. Rollout and test strategy

### Phase 1 — Profiling and shadow mode

1. Profile source address completeness and country distribution.
2. Identify null country, invalid state/province, malformed postal code, and missing unit rates.
3. Run verification without rejecting records.
4. Store results in temporary or nonauthoritative fields.
5. Compare corrected values with source values.
6. Review a stratified sample with data stewards.

### Phase 2 — Match-impact test

Create a nonproduction experiment:

1. Load a baseline sample without address verification.
2. Capture match pairs and merge results.
3. Reload the same sample with source-level verification.
4. Compare:
   - candidate counts;
   - auto-merge counts;
   - manual-review counts;
   - false positives;
   - false negatives;
   - cluster changes;
   - survivorship outcomes.
5. Adjust match thresholds only after reviewing these effects.

### Phase 3 — Controlled enforcement

1. Enable automatic acceptance for clear `V/A/C` results that meet your detailed quality criteria.
2. Route invalid and ambiguous results to stewardship.
3. Enable rejection only for use cases that truly require a deliverable address.
4. Monitor service availability and transactions daily during rollout.

### Minimum test cases

- Exact valid US address
- Valid Canadian address
- Correctable spelling error
- Missing ZIP/postal code
- Wrong postal code with correct city
- Wrong city with correct postal code
- Missing state/province
- Missing or invalid country
- Apartment/suite present
- Apartment/suite missing
- PO box
- Rural route
- New construction/not yet in reference data
- Unsupported country
- Mexico record under the North America regional pack
- Multiple candidate/suggestion result
- Empty address
- Service timeout
- Invalid/expired license
- Duplicate source records with differently formatted addresses
- Contact and Company at the same building but different units

---

## 11. Operational and governance considerations

### Transaction and cost control

- Verify only changed addresses.
- Avoid firing verification on every UI field focus unless required.
- Use section save or record submit for most UI scenarios.
- Do not run both CDI Verifier and EVO on the same unchanged source record without a reason.
- Track MDM SaaS real-time transactions, batch transactions, and IPUs separately.
- Confirm what happens to unused transaction entitlements at term end.

### Failure policy

Define explicitly:

- fail open versus fail closed;
- retry count and delay;
- who owns the retry queue;
- whether a service failure lowers trust;
- whether MDM ingestion continues when reference data is unavailable;
- how stewards distinguish invalid addresses from unavailable service/reference data.

### Security and privacy

Postal addresses are personal or commercially sensitive data in many contexts.

- Use least-privilege service accounts.
- Store credentials in Informatica-managed settings/connections.
- Mask tokens and sensitive address values in logs.
- Limit payload retention.
- Review cross-border processing and data-residency requirements.
- Document the vendor service in privacy and data-processing inventories.

### Audit and lineage

Retain:

- original source value;
- standardized value;
- fields added or corrected;
- verification outcome;
- processing time;
- service/rule version where available;
- source system and source key;
- steward override and reason.

### Reference-data and release management

- Monitor license expiration.
- Confirm reference-data downloads after Secure Agent upgrades.
- Regression-test after Verifier engine, reference data, MDM, CDI, or EVO release changes.
- Version Verifier assets and EVO rules.
- Migrate objective groups and dependent assets together.

### Certified postal processing

US CASS and Canada SERP outputs have specialized fields and requirements. Enable certified processing only when the business needs postal certification and the necessary reference data/licenses are in place. Certification settings can impose additional operational or reporting requirements.

### Geocoding

Geocoding is a separate licensed option. Decide what precision is acceptable:

- country/region;
- postal code;
- locality;
- street;
- parcel/rooftop where supported.

Always retain a geocode status/accuracy field. Never treat every returned latitude/longitude as rooftop-accurate.

---

## 12. Recommended implementation blueprint

For most Company and Contact MDM programs, use this combined design:

```text
A. Initial migration and high-volume feeds
   Source -> CDI -> CDQ Verifier -> DQ router -> MDM Source Records

B. Ongoing MDM-native ingress and source updates
   MDM Source Record -> EVO Address Verification objective -> match/merge

C. Steward/direct master edits
   MDM Master Record -> separate EVO Master objective

D. External point-of-entry application
   Either:
     application -> MDM source API -> EVO
   Or, when synchronous pre-check is required and licensed:
     application -> Cloud Address Verification API -> MDM API
```

Choose one authoritative verification point for each transaction path. The same record should not be charged and transformed repeatedly simply because several layers can call the service.

### Recommended first production scope

1. Start with one entity and one address role, such as Company primary mailing address.
2. Start with explicitly confirmed countries in the North American pack.
3. Store standardized output and status without hard rejection.
4. Measure match impact.
5. Add Contact and secondary address roles after the first policy is stable.
6. Add real-time UI validation after batch behavior and transaction consumption are understood.

---

## 13. Questions to confirm with Informatica before configuration

Send these questions to your Informatica CSM/support/licensing contact:

1. What exact countries are included in our North American Regional Pack for each instance?
2. Is Mexico excluded from our order, and what separate subscription is needed for Mexico?
3. What real-time and batch transaction quantities are included?
4. Is our entitlement limited to MDM SaaS, or do we also have CDQ/Verifier rights?
5. Do we have IDMC IPUs for the Cloud Address Verification REST API?
6. Is the EVO Address Verification plugin enabled in our current MDM SaaS release and pod?
7. What credentials are supplied: Account ID, Cloud Token, batch license file, or another mechanism?
8. Which Address Verification engine/license-file version is required?
9. Are US CASS and Canada SERP reference data included?
10. Is geocoding included? At what accuracy levels?
11. What are the production and nonproduction instance restrictions?
12. What monitoring page reports MDM web-service transactions and IPU usage?
13. What is the supported behavior when the service is unavailable during ingress?
14. Are there payload, batch-size, concurrency, or rate limits for our tenant?
15. Are there country-specific pass-through terms or reporting obligations?

---

# 14. Learning resource list

The following resources are primarily official Informatica documentation, training, webinars, and support videos. Some Success Portal or University pages may require an Informatica login.

## A. Licensing and product scope

1. [Informatica Cloud and Product Description Schedule — current PDF](https://www.informatica.com/content/dam/informatica-com/en/docs/informatica-cloud-and-product-description-schedule.pdf)  
   Read the **Premium Address Cleansing and Validation for Cloud** section for regional-pack scope, MDM/CDQ use, transactions, API entitlement, geocoding prerequisites, and the Mexico exclusion.

2. [FAQs on Address Verification on the IPU subscription model](https://knowledge.informatica.com/s/article/FAQs-on-Address-Verification-on-the-IPU-subscription-model?language=en_US)  
   IPU subscription and metering considerations.

3. [Informatica Data as a Service solutions](https://www.informatica.com/products/data-quality/data-as-a-service.html)  
   Product overview for address, email, and phone verification.

4. [Informatica Address Validation and Verification product page](https://www.informatica.com/products/data-quality/data-as-a-service/address-verification.html)  
   Address-verification capabilities and product material.

## B. MDM SaaS and EVO

5. [MDM SaaS EVO Address Verification — Success Accelerator](https://success.informatica.com/success-accelerators/psu/mdm-saas-evo-address-verification.html)  
   Current official description of configuring the Address Verification plugin on an MDM SaaS business entity.

6. [MDM SaaS Enrichment and Validation Orchestration — Informatica University](https://now.informatica.com/MDM-SaaS-Enrichment-and-Validation-Orchestration-onDemand-eLearning.html)  
   Course covering rule associations, objectives, objective groups, data-enhancement rules, plugins, triggers, error handling, and trust-score behavior.

7. [Data enrichment with EVO and MDM objective groups — webinar page](https://success.informatica.com/explore/tt-webinars/data-enrichment-to-boost-your-mdm-data-leveraging-the-enrichment.html)  
   Recording and agenda with MDM SaaS EVO concepts and demonstrations.

8. [Data enrichment with EVO and MDM objective groups — slide deck PDF](https://www.informatica.com/content/dam/informatica-cxp/techtuesdays-slides-pdf/Data%20enrichment%20to%20boost%20your%20MDM%20data%20leveraging%20the%20Enrichment%20and%20Validation%20Orchestrator%20and%20MDM%20objective%20groups.pdf)  
   Useful diagrams for plugins, data-enhancement rules, rule associations, objectives, source/master records, and triggers.

9. [Setting license details in MDM SaaS Global Settings](https://onlinehelp.informatica.com/iics/prod/b360/en/mm-b360-configure-global-settings/Setting_license_details.html)  
   Account ID/cloud token for real-time verification and batch license-file configuration.

10. [MDM SaaS Technical Design Document example](https://www.informatica.com/content/dam/informatica-cxp/informatica-gateway-partner-portal/documents/MDM%20SaaS%20Technical%20Design%20Document%20Example.pdf)  
    Includes a section explaining DaaS rules at Business Entity Field Group level and the CDI/CDQ alternative.

11. [Design and Development: MDM SaaS UI — Part 3](https://success.informatica.com/explore/tt-webinars/design-and-development--mdm-saas-ui---part-3.html)  
    MDM SaaS UI design webinar that includes DaaS integration considerations.

12. [MDM SaaS Development learning path](https://success.informatica.com/learning-path/mdm-saas/development.html)  
    MDM SaaS development resources, including DaaS and configuration topics.

13. [MDM SaaS EVO Introduction Workshop](https://success.informatica.com/success-accelerators/psu/mdm-saas-evo-introduction-workshop.html)  
    Official workshop offering for understanding EVO and prioritizing use cases.

## C. Cloud Data Quality Verifier documentation

14. [Introduction to Verifier assets](https://docs.informatica.com/data-quality-cloud/cloud-data-quality/current-version/verifier-assets/introduction-to-verifier-assets.html)  
    Core concepts, comparison to reference data, correction, enrichment, certification, and API-enabled mapplets.

15. [Verifier asset inputs](https://docs.informatica.com/data-governance-and-quality-cloud/data-quality/current-version/verifier-assets/introduction-to-verifier-assets/inputs.html)  
    Discrete, hybrid, and multiline input models.

16. [Verifier asset outputs](https://docs.informatica.com/data-governance-and-quality-cloud/data-quality/current-version/verifier-assets/introduction-to-verifier-assets/outputs.html)  
    Single address elements, preformatted data, status codes, and enrichments.

17. [Verifier configuration guide](https://docs.informatica.com/data-governance-and-quality-cloud/data-quality/current-version/verifier-assets/verifier-configuration.html)  
    Input model, input fields, output fields, suggestions, quality, geocodes, and certification configuration.

18. [Configuring Address Verification properties on the Secure Agent](https://docs.informatica.com/data-governance-and-quality-cloud/data-quality/current-version/verifier-assets/address-verification-properties/configuring-the-address-verification-properties.html)  
    Administrator, Runtime Environments, Secure Agent, Data Integration Server, and `CDQAV` property-set steps.

19. [Verification Status Codes](https://docs.informatica.com/data-governance-and-quality-cloud/data-quality/current-version/verifier-assets/output-address-fields/status-code-fields/verification-status-codes.html)  
    Meaning of `V`, `A`, `C`, `I`, and `N`.

20. [Verifier assets and mappings](https://docs.informatica.com/data-quality-cloud/cloud-data-quality/current-version/verifier-assets/introduction-to-verifier-assets/verifier-assets-and-mappings.html)  
    How the Verifier asset is applied in a mapping.

21. [Address reference data](https://docs.informatica.com/integration-cloud/data-integration/current-version/transformations/verifier-transformation/address-reference-data.html)  
    Reference-data behavior and Secure Agent considerations.

22. [Cloud Data Quality learning path — Define](https://success.informatica.com/learning-path/cloud-data-quality/define.html)  
    Verifier and Data Quality asset learning material.

## D. Cloud Data Integration documentation

23. [Verifier transformation](https://docs.informatica.com/integration-cloud/cloud-data-integration/current-version/transformations/verifier-transformation.html)  
    Adds a Verifier asset to a CDI mapping and explains correction, completion, formatting, deliverability, and suggestions.

24. [Understanding Verifier input and output mappings](https://docs.informatica.com/integration-cloud/data-integration/current-version/transformations/verifier_transformation/verifier-transformation-field-mappings/understanding-input-and-output-mappings.html)  
    Ensures mapping fields correspond to the asset's expected address information.

25. [Cloud Data Integration transformations guide PDF](https://docs.informatica.com/content/dam/source/GUID-2/GUID-2B5E2342-3D75-4BCE-B092-A78F85631717/65/en/CDI_April2025_Transformations_en.pdf)  
    Downloadable guide containing the Verifier transformation chapter.

## E. Cloud Address Verification REST API

26. [Cloud Address Verification API — Introduction](https://docs.informatica.com/data-governance-and-quality-cloud/data-quality/current-version/cloud-address-verification-api/introduction.html)  
    Real-time REST API overview and IPU monitoring context.

27. [Cloud Address Verification API — Send requests](https://docs.informatica.com/data-governance-and-quality-cloud/data-quality/current-version/cloud-address-verification-api/introduction/send-requests.html)  
    Endpoint pattern, region base URLs, JSON, and bearer token headers.

28. [Swagger documentation for POST requests](https://docs.informatica.com/data-governance-and-quality-cloud/data-quality/current-version/cloud-address-verification-api/rest-call-creation/swagger-documentation-for-post-requests.html)  
    How to open the JWT-protected Swagger UI.

29. [Cloud Address Verification API — Enrichments overview](https://docs.informatica.com/data-governance-and-quality-cloud/data-quality/current-version/cloud-address-verification-api/address-enrichments/enrichments-overview.html)  
    Global, country-specific, certification, and geocode enrichment concepts.

30. [Cloud Address Verification API — Address status values and return codes](https://docs.informatica.com/data-governance-and-quality-cloud/data-quality/current-version/cloud-address-verification-api/address-status-values-and-return-codes/address-status-values-and-return-codes-overview.html)  
    Detailed interpretation of API output.

31. [Cloud Address Verification API — Standardization overview](https://docs.informatica.com/data-governance-and-quality-cloud/data-quality/current-version/cloud-address-verification-api/standardization-parameters/standardization-overview.html)  
    Output standardization policies.

32. [Real-time Execution of Address Verification — Success Accelerator](https://success.informatica.com/success-accelerators/psu/real-time-execution-of-address-verification.html)  
    Official services offering for implementing real-time Address Verification API requests.

## F. Official support videos

33. [How to Create a Verifier asset in CDQ](https://success.informatica.com/videos/support-videos/A0_4B9KeuxI.html)

34. [Select the Input Fields in a Verifier Asset](https://success.informatica.com/videos/support-videos/7MJYYb7Nr9Y.html)

35. [Select the Output Fields in a Verifier Asset](https://success.informatica.com/videos/support-videos/RFflSMlwMKM.html)

36. [Configure Address Verification Properties in Cloud Data Quality](https://success.informatica.com/videos/support-videos/lr0m_WrBNZI.html)

37. [Address Verification Modes — YouTube, Informatica Support](https://www.youtube.com/watch?v=O-9aA2ZYwRI)

38. [How to Use a CDQ Verifier Asset to Evaluate Address Accuracy and Deliverability](https://success.informatica.com/videos/support-videos/d4-W-f-wTyQ.html)

39. [How to Optimize an Address Verifier Mapping in CDI — YouTube, Informatica Support](https://www.youtube.com/watch?v=iLwzCF-4NoI)

40. [Deploying an Address Validation Mapplet as a Web Service](https://success.informatica.com/videos/support-videos/8CI1dOA5TBw.html)

41. [Address Verification Overview — YouTube](https://www.youtube.com/watch?v=zbpiDaRbTxY)

42. [Address Verification and Correction with Informatica Cloud — YouTube](https://www.youtube.com/watch?v=Bytnvt1jwQM)

---

## 15. Final recommendation for your project

For Company and Contact business entities:

1. **Confirm license scope first**, especially exact North American countries, transaction limits, EVO availability, IPUs, and geocoding.
2. **Standardize and verify source-record addresses before match/merge.**
3. For normal MDM SaaS operation, configure the **EVO Address Verification plugin on Source Records** so ingress and file-import data are covered.
4. For initial migration and large batch loads, use a **CDQ Verifier transformation in the CDI ingress mapping** before the MDM target.
5. Add a separate Master Record objective only for direct master/golden edits.
6. Use the direct REST API only for synchronous external use cases and only after confirming the separate IPU entitlement.
7. Preserve raw values, standardized values, status, quality, timestamp, and lineage.
8. Start in shadow mode, measure match impact, and enable rejection only after steward review.

