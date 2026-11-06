# Feature Spec: Address Details Page

> **Type:** page
> **Figma:** _(not captured in this session — add the Address Details Figma link if available)_
> **Jira:**

## API Contract

### verifyWorkEmailService — POST `draft-application/verify-offical-email-otp-sync`

**Request**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| emailId | string | yes | Work email address the OTP was sent to |
| otp | string | yes | OTP entered by user, length `OTP_LENGTH` |
| _id | string | yes | Application ID, auto-attached from `getAppIdSelector` — not supplied by caller |

**Response — Success**

| Field | Type | Notes |
|-------|------|-------|
| *(no consumed fields)* | — | On success the caller only resets OTP counters (`resetAllOtpCountsAction`); the response body isn't read for any field |

**Response — Errors**

| Error Code | User-facing Message |
|------------|---------------------|
| CC_ACQUISITION_DRAFT_APPLICATION_35 (RETRY_EXHAUSTED) | Redirects to `COMMON_ERROR_TYPE.RETRY_CNT_EXAUST` error page (no inline message) |
| CC_ACQUISITION_DRAFT_APPLICATION_34 (EV_CHECK_FAILED) | Redirects to `CUSTOMER_ELIGIBILITY_ERROR_TYPE.EV_CHECK_FAILED` error page (no inline message) |
| CC_ACQUISITION_CHUB_03 (WRONG_OTP) | "Incorrect OTP.  `{N}` attempts left, please try again" (or "OTP limit is exhausted, please resend OTP to continue" once attempts hit `MAX_OTP_VERIFY_COUNT`) |
| NOTIF_MS_CHUB_04 (OTP_LIMIT_EXCEED) | "OTP limit is exhausted, please resend OTP to continue " |
| NOTIF_MS_CHUB_03 (OTP_EXPIRE) / CC_ACQUISITION_DRAFT_APPLICATION_10 (OTP_EXPIRE_ACQ) | "OTP has expired. Please tap resend OTP for a new one" |
| NOTIF_MS_CHUB_01 (HARD_BLOCK_FROM_CHUB) | No inline message — sets hard-block state (`showHardBlock`) via `error.hardBlock` |
| NOTIF_MS_CHUB_02 (GENERATE_OTP_LIMIT_REACH) | No inline message — sets soft-block state (`showSoftBlock`) |
| *(any other errorCode)* | Generic snackbar shown via `showSnackbar()` |

Note: `response.error.verifyCount` / `response.error.generateCount` (when present on any error) also update `otpVerifyCount`/`otpGenerateCount` in the store regardless of which branch above handles the message.

### saveApplicationDetailsService — POST `/draft-application/save`

**Request**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| submittedApplicationDetails._id | string | yes | Application ID |
| submittedApplicationDetails.addresses.communicationAddress | object | yes | `{ addressLine1, addressLine2, addressLine3, city, state, pin, stateCode, country, residenceType }` from the home address form |
| submittedApplicationDetails.addresses.officeAddress | object | no | Same shape as above, minus `residenceType`; omitted entirely when `isJourneyTypePA` |
| submittedApplicationDetails.addresses.cardDeliveryAddress | string | yes | Selected delivery address type (`Communication` / `Office`) |
| submittedApplicationDetails.currentApplicationStage | string | yes | From `getNextStageRedirect(ADDRESS_DETAILS, journeyType)` |
| submittedApplicationDetails.nextApplicationStage | string | yes | From `getNextStageRedirect(ADDRESS_DETAILS, journeyType)` |

**Response — Success**

| Field | Type | Notes |
|-------|------|-------|
| *(no consumed fields)* | — | Caller only checks `_isCustomError`; the page's overlay flow (EV checks / T&C / eKYC) reacts to redux stage flags updated elsewhere, not to this response body |

**Response — Errors**

| Error Code | User-facing Message |
|------------|---------------------|
| *(any errorCode)* | Generic snackbar shown via `showSnackbar()` — no field-level or code-specific handling |

### searchPinDetailsService — POST `/pinCode/search`

**Request**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| pinCode | string | yes | Triggered once the Pincode field reaches 6 digits |

**Response — Success**

| Field | Type | Notes |
|-------|------|-------|
| city | string | Auto-fills the City field |
| state | string | Auto-fills the State field |
| stateCode | string | Stored for downstream address payloads |

**Response — Errors**

| Error Code | User-facing Message |
|------------|---------------------|
| CC_ACQUISITION_PINCODE_01 (HTTP 422) | "Your pincode is not serviceable at the moment, please try another address" — Pincode, City, State fields are cleared and marked unserviceable |
| *(any other error)* | Generic snackbar shown via `showSnackbar()`; Pincode, City, State fields are cleared |

### validatePincodeDistanceService — POST `/pincode/validate-distance`

**Request**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| applicationId | string | yes | Current application ID |
| currentAddressPinCode | string | yes | Home address pincode |
| officeAddressPinCode | string | yes | Work address pincode |

**Response — Success**

| Field | Type | Notes |
|-------|------|-------|
| *(no consumed fields)* | — | Success just clears any distance-warning state |

**Response — Errors**

| Error Code | User-facing Message |
|------------|---------------------|
| CC_ACQUISITION_PINCODE_04 (PIN_DISTANCE_WARN) | "Your work address seems far. Get it delivered to your home address instead." — auto-selects home as delivery address |
| CC_ACQUISITION_PINCODE_05 (PIN_DISTANCE_ABOVE_200KM) | No inline message — auto-selects home (communication) address as delivery address |
| *(any other error)* | Silently retried up to 2 times, then falls back to work/office address as delivery address |

## Variant Conditions

<!--
  The Address Details page itself doesn't navigate away on save — EV checks, T&C, and eKYC
  are rendered as overlays on top of the same page, driven by redux stage flags.
-->

| Figma Variant | Condition |
|---|---|
| Default (address form) | `!showEVChecks && !showTnC && !showEKYC` |
| Loading | `isServiceLoading([saveApplicationDetailsServiceName, validatePincodeDistanceServiceName, employmentWorkEmailVerifyServiceName, skipWorkEmailVerifyServiceName, otpServiceName, searchPinDetailsServiceName])` |
| EVChecksOverlay | `showEVChecks = !isAllEvChecksCompleted && isNextStageTnC` |
| TnCOverlay | `showTnC = isAllEvChecksCompleted && isNextStageTnC` |
| EkycBottomSheet | `showEKYC = isFinalSubmitOtpVerified && (isRekycDue \|\| kycStatus === EKYCstatus.ACCEPTED)` |
| PrefetchedAddressSection | `isJourneyTypeNPA && isPrefetchedHomeAddress` (renders read-only prefetched addresses instead of the editable form) |
| PA journey (home-only) | `isJourneyTypePA` — only the home address section is rendered/validated; work address is skipped entirely |
| NpaSalariedBanner | `isJourneyTypeNPA && isSalaried` |
| SameAddressToggle | `isJourneyTypeNTB && !isSalaried && homeFormikBag.dirty && homeFormikBag.isValid` |
| DeliverySelectionSection | `isFormsValid && (!isJourneyTypePA \|\| (isJourneyTypeNPA && !!selectedDeliveryAddress))` |

## Business Logic

### Validation Rules

| Field | Rule | Error Message |
|-------|------|---------------|
| addressLine1 | `required`, `min(6)`, `max(35)` | "Please enter Address Line 1" / "Character limit: min 6, max 35" |
| addressLine1 | `matches(ADDRESS_LINE_ALLOWED_REGEX)` | "Numbers and/or special characters alone are not allowed" |
| addressLine1 | no consecutive repeated words | "Repeated words are not allowed" |
| addressLine1 | must differ from addressLine2/addressLine3/city/state | "Address line 1 cannot be same as line 2 or 3 and city/state fields" |
| addressLine1 | must differ from the other address type's line1/line2 (salaried only) | "Please ensure that the home/work address is different from the work/home address." |
| addressLine2 | optional, `max(40)`, same allowed-chars + no-repeat rules as line 1 | "Character limit: max 40" / "Numbers and/or special characters alone are not allowed" |
| addressLine3 | optional, `max(25)`, same allowed-chars + no-repeat rules | "Character limit: max 25" |
| city | `required`, `min(3)`, `max(50)` | "Please enter valid pincode" / "Character limit: min 3, max 50" |
| state | `required`, `min(3)`, `max(50)` | "Please enter valid pincode" / "Character limit: min 3, max 50" |
| pin | `required`, exactly 6 chars, must resolve as serviceable | "Please enter valid pincode" / "Your pincode is not serviceable at the moment, please try another address" |
| residenceType | `required` only when `requireResidenceType` (NTB journey) | "Home residence type is required" |
| addressLine1/2/3 (combined) | normalised alphabet count across all three lines must be ≥ 15 | Sets `formGlobalError`, rendered via `AddMoreDetailsError` |

### Conditional Field Behavior

| Field | Show When | Read-only When |
|-------|-----------|----------------|
| Work address section | `!isJourneyTypePA` | — (fully hidden, not just read-only, for PA journey) |
| addressLine3 | User taps "+ ADD ADDRESS LINE 3", or already has a value | — |
| residenceType | `isJourneyTypeNTB` (`showResidenceType` prop) | — |
| Home/Work address fields | — | `isPrefetchedHomeAddress` / `isPrefetchedWorkAddress` (rendered as read-only text instead of the form) |
| SameAddressToggle | `isJourneyTypeNTB && !isSalaried && homeFormikBag.dirty && homeFormikBag.isValid` | — |
| DeliverySelectionSection | `isFormsValid` (both addresses valid + pin serviceable) | — |

### Business Rules

- PA journey only collects and validates the home address; `isWorkFormValid` is hard-coded `true` and the work address section is never rendered.
- NPA journey with an already-prefetched home address renders `PrefetchedAddressSection` (read-only) instead of the editable form.
- Salaried users' home and work address lines are cross-validated against each other — the two addresses cannot match.
- "Same as home address" toggle auto-copies home address values into the work address form (NTB, non-salaried only); editing either form after toggling turns the toggle off.
- Pincode is only re-looked-up when its value changes from the initial value or when 6 digits are entered; a matching prefetched pin skips the lookup and is marked serviceable immediately.
- Delivery address distance validation (`validatePincodeDistanceService`) only runs automatically for NPA journeys once both home and work pins are known, and can be disabled entirely via `APP_DISABLED_VALIDATE_DISTANCE_CHECKS`.
- Save & Next is only enabled once the applicable address form(s) are valid, both pins are confirmed serviceable, and (for non-PA journeys) a delivery address is selected.
- After a successful save, the page does not navigate away — it re-renders the same route with an EV-checks, T&C, or eKYC overlay depending on updated redux stage flags.

## Navigation Flow

**Entry Point**

| Property | Value |
|----------|-------|
| Route | `ADDRESS_DETAILS` (`/application/address-details`) |
| Reached From | `EMPLOYMENT_DETAILS` (NTB/NPA) or `PA_DETAILS` (PA), via `getNextStageRedirect` |
| Guard | None explicit on this page; relies on prior stage's redirect |

**Exit Routes**

| Event | Destination |
|-------|-------------|
| Save & Next (address stage) | Stays on `ADDRESS_DETAILS` pathname; renders EV-checks or T&C overlay next based on stage flags |
| EV checks completed + T&C accepted | `APPLICATION_FINAL_STATUS.pathname?fetchStatusOnLoad=false` |
| eKYC re-verification resolved | `APPLICATION_FINAL_STATUS.pathname` (stage `EKYC` → `APPLICATION_STATUS`) |

## State Hints

| Property | Value |
|----------|-------|
| Redux Domain | `Application` |
| Reuse Selectors | `getApplicationDetailsSelector`, `getSelectedDeliveryAddressSelector`, `isJourneyTypeNPASelector`, `isJourneyTypeNTBSelector`, `isJourneyTypePASelector`, `isSalariedSelector`, `isAllEvChecksCompletedSelector`, `isFinalSubmitOtpVerifiedSelector`, `isNextStageTnCSelector`, `isRekycDueSelector`, `kycStatusSelector` |
| Reuse Services | `saveApplicationDetailsService`, `searchPinDetailsService`, `validatePincodeDistanceService`, `verifyWorkEmailService` (via `EvStatusCheck` overlay), `sendOtpService` |

## Scope

**Exclude:**
- eKYC OTP verification internals (separate `EkycBottomSheet` component)
- EV work-email verification internals (separate `EvStatusCheck` / `WorkEmailVerification` components)
- T&C acceptance flow internals (separate `TermsAndConditions` component)

**Feature Flags:**
- `APP_DISABLED_VALIDATE_DISTANCE_CHECKS` (env var — skips delivery-address distance validation entirely when enabled)

## Analytics

| Event | Trigger | Properties |
|-------|---------|------------|
| `olive_post_login_onload` (`onAddressDetailsPageLoad`) | Page load | `journeyType`, `customerType` |
| `olive_post_login_onclick` (`onSaveAndNextCTAClick`) | Save & Next CTA click | `customerType`, `pageTitle`, `deliveryAddressType` |
| `olive_post_login_onclick` (`onAddressPageEditCTAClick`) | Edit CTA click on a prefetched address | `customerType`, `addressType`, `pageTitle` |
