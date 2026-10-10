# Rishtha Online API: fields and flow

Every API the app calls (admin APIs excluded), in the order a member moves through the app. Each one lists its request fields, which are required, and **Next**: the API to call after it.

- **Base URL:** `/api` (local: `http://localhost:5000/api`)
- **Auth:** 🔓 Public = no token. 🔐 Token = `Authorization: Bearer <token>`.
- **Fields** go in the JSON body unless marked *(path)*, *(query)* or *(form-data)*.
- **Dropdown fields** take the option **value** from master data (e.g. `never_married`, not "Never Married").
- **`any`** ("Doesn't matter") is accepted only in partner-preference fields.
- **Errors:** validation → `400` `{ success: false, message, errors: [...] }`. Plan limits → `403` with a `code` (e.g. `PLAN_MESSAGING_REQUIRED`): show the upgrade screen.
- Postman collection: [Rishtha-Online-App-API.postman_collection.json](Rishtha-Online-App-API.postman_collection.json).

## Main flow at a glance

```
App launch:  master-data → app-version → (sign up | login | /auth/me)
Sign up:     register → register/verify-otp → signup/personal (+ id-proof) → signup/social
             → signup/preferences → profile/photos | signup/photos/skip
Login:       auth/login  |  login/otp/send → login/otp/verify   (resume signupStep if not completed)
After login: settings/device-token → community-guidelines/accept → home
Discover:    home → matches | newly-joined | search → profile/:id
Connect:     profile/:id → interests/:userId → (other member) interests/received → interests/:id accept
             → matches/mutual → messages/conversations → messages/:conversationId
Photos:      profile/:id → photo-requests/:userId → (owner) photo-requests/received → accept → profile/:id
Upgrade:     payments/plans → payments/initiate → (PhonePe) → payments/status/:txn
```

## A. App launch

### 1. Load dropdown options

`GET /api/master-data` · 🔓 Public

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `keys` | Text (query) | No | Comma-separated list keys, e.g. `heights,sects`. Omit to get everything |

**Returns:** `masterData`: every dropdown list as `[{ value, label }]`. Cache it in the app.

**Next →** [#2 Check app version](#2-check-app-version) (`GET /api/content/app-version`)

### 2. Check app version

`GET /api/content/app-version` · 🔓 Public

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `platform` | Text (query) | No | `android` (default) or `ios` |
| `version` | Text (query) | No | Installed version, e.g. `1.4.0` |

**Returns:** `appVersion` with `latestVersion`, `minSupportedVersion`, `storeUrl`, `updateAvailable`, `updateRequired`.

**Next →**

- New user ("Sign up for Free"): [#3 Create account (sends OTP)](#3-create-account-sends-otp) (`POST /api/auth/register`)
- Existing user ("Login"): [#13 Login with email + password](#13-login-with-email--password) (`POST /api/auth/login`)
- A token is already saved on the device: [#19 Get logged-in member](#19-get-logged-in-member) (`GET /api/auth/me`)

## B. Sign up

Create Account → Verify Mobile OTP → "You're Verified".

### 3. Create account (sends OTP)

`POST /api/auth/register` · 🔓 Public

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `profileCreatedFor` | Text | **Yes** | `masterData.profileCreatedFor` value. **Sending this field selects the new signup** |
| `phone` | Text | **Yes** | Indian mobile, 10 digits starting 6–9 (`9876543210`, `+91 98765 43210`…). Must not be registered |
| `gender` | Text | **Yes** | `male`, `female` |
| `email` | Text | No | Valid email. Must not be registered |
| `password` | Text | No | Min 6 chars. Without it the member logs in by OTP |

**Returns:** `userId`, `phone`, `signupStep: "verify_phone"`, `otp: { expiresIn: 300, resendAfter: 60 }`. **No token yet.**

**Next →** [#4 Verify mobile OTP](#4-verify-mobile-otp) (`POST /api/auth/register/verify-otp`)

### 4. Verify mobile OTP

`POST /api/auth/register/verify-otp` · 🔓 Public

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `phone` | Text | **Yes** | Same number used to register |
| `otp` | Text | **Yes** | 4 digits |

**Returns:** `token` and `user` (with `signupStep: "personal"`). Save the token: every request after this needs it.

Wrong code → `400` with `attemptsLeft`. After 5 wrong attempts or 5 minutes, resend.

**Next →**

- Success: [#7 Step 1 – Personal details](#7-step-1--personal-details) (`PUT /api/signup/personal`)
- Code expired, not received, or 5 wrong attempts: [#5 Resend OTP](#5-resend-otp) (`POST /api/auth/register/resend-otp`)

### 5. Resend OTP

`POST /api/auth/register/resend-otp` · 🔓 Public

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `phone` | Text | Yes, or `email` | Use `phone` for signup and login |
| `email` | Text | Yes, or `phone` | Only for `purpose: "reset_password"` |
| `purpose` | Text | No | `signup` (default), `login`, `reset_password` |

One OTP per 60 seconds, 5 per hour. Too soon → `429` with `retryAfter` (seconds). `POST /api/auth/resend-otp` does the same.

**Next →**

- purpose = signup: [#4 Verify mobile OTP](#4-verify-mobile-otp) (`POST /api/auth/register/verify-otp`)
- purpose = login: [#15 Login with phone – verify OTP](#15-login-with-phone--verify-otp) (`POST /api/auth/login/otp/verify`)
- purpose = reset_password: [#17 Verify OTP](#17-verify-otp) (`POST /api/auth/forgot-password/verify`)

## C. Signup steps

Order: `verify_phone` → `personal` → `social` → `preferences` → `photos` → `completed`.
- A step can only be submitted after the previous one (otherwise `400` with the current `signupStep`).
- Send **all** fields of the step. Every step returns `{ signupStep, completed, steps: [{ step, done }] }`.

### 6. Signup status (resume)

`GET /api/signup/status` · 🔐 Token

_No request fields._

**Returns:** `signupStep`, `completed`, `steps`.

**Next →**

- signupStep = personal: [#7 Step 1 – Personal details](#7-step-1--personal-details) (`PUT /api/signup/personal`)
- signupStep = social: [#9 Step 2 – Social details](#9-step-2--social-details) (`PUT /api/signup/social`)
- signupStep = preferences: [#10 Step 3 – Partner preferences](#10-step-3--partner-preferences) (`PUT /api/signup/preferences`)
- signupStep = photos: [#11 Upload photos (completes signup)](#11-upload-photos-completes-signup) (`POST /api/profile/photos`)
- signupStep = completed: [#20 Register device for push notifications](#20-register-device-for-push-notifications) (`POST /api/settings/device-token`)

### 7. Step 1 – Personal details

`PUT /api/signup/personal` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `name` | Text | **Yes** | Max 100 chars |
| `dateOfBirth` | Date | **Yes** | `YYYY-MM-DD`, age 18+ |
| `heightInches` | Number | **Yes** | Inches, 47–84 (`masterData.heights`) |
| `complexion` | Text | **Yes** | `very_fair`, `fair`, `wheatish`, `slightly_dark`, `dark` |
| `weight` | Text | **Yes** | `masterData.weights` value |
| `address` | Text | **Yes** | Max 300 chars |
| `city` | Text | No | Max 100 chars. Not on the design, but needed for location matching ("in Hyderabad") |
| `languages` | Array | **Yes** | One or more `masterData.languages` values |
| `sect` | Text | **Yes** | `masterData.sects` value |
| `jamaatAffiliation` | Text | **Yes** | `barelvi`, `deobandi`, `ahl_e_hadith`, `hanafi`, `other` |
| `religiosity` | Text | **Yes** | `very_practising`, `practising`, `moderate`, `liberal` |
| `prayerHabits` | Text | **Yes** | `five_times`, `mostly`, `sometimes`, `jummah_only`, `rarely` |
| `religiousEducation` | Text | **Yes** | `hifz`, `alimiyat`, `moderate_knowledge`, `self_taught`, `basic` |
| `isRevert` | Boolean | No | `true` / `false`. Defaults to `false` |
| `hijabLevel` | Text | No | `niqab`, `hijab_scarf`, `occasionally`, `no_hijab`. Used in match scores for female profiles |

**Next →**

- Member attaches an ID proof on this screen (optional): [#8 Upload ID proof (Step 1 screen)](#8-upload-id-proof-step-1-screen) (`POST /api/profile/id-proof`)
- Continue: [#9 Step 2 – Social details](#9-step-2--social-details) (`PUT /api/signup/social`)

### 8. Upload ID proof (Step 1 screen)

`POST /api/profile/id-proof` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `idProofType` | Text (form-data) | **Yes** | `aadhaar`, `pan`, `passport`, `voter_id`, `driving_licence`, `other` |
| `idProof` | File (form-data) | **Yes** | JPG, PNG, WEBP or PDF, max 5 MB |

**Returns:** `idProof: { type, status: "pending" }`. An admin later approves or rejects it (the member gets a notification).

**Next →** [#9 Step 2 – Social details](#9-step-2--social-details) (`PUT /api/signup/social`)

### 9. Step 2 – Social details

`PUT /api/signup/social` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `education` | Text | No | `masterData.secularEducation` value |
| `profession` | Text | **Yes** | `masterData.professions` value |
| `jobLocation` | Text | **Yes** | Max 100 chars |
| `designation` | Text | **Yes** | Max 100 chars |
| `yearsOfExperience` | Integer | **Yes** | Min 0, Max 60 |
| `income` | Text | No | `masterData.incomeRanges` value |
| `workingAbroadStatus` | Text | **Yes** | `no_india`, `job_visa`, `permanent_residency`, `open_to_abroad` |
| `netWorth` | Text | No | Max 100 chars. Optional at signup |
| `professionalOrientation` | Text | No | `career_oriented`, `balanced`, `home_maker` |
| `fatherProfession` | Text | **Yes** | Max 100 chars |
| `motherProfession` | Text | **Yes** | Max 100 chars |
| `familyValues` | Text | **Yes** | `traditional`, `moderate`, `liberal` |
| `siblings` | Integer | **Yes** | Min 0, Max 20 |
| `siblingsDetails` | Text | No | Max 300 chars. e.g. "2 Brothers (Married)" |
| `residenceType` | Text | **Yes** | `flat`, `independent_house` |
| `familyLivesIn` | Text | **Yes** | Max 100 chars |
| `nativePlace` | Text | **Yes** | Max 100 chars |
| `maritalStatus` | Text | **Yes** | `never_married`, `divorced`, `widowed`, `still_married` |
| `isFirstMarriage` | Boolean | No | `true` / `false`. If not sent: `true` for `never_married`, otherwise `false`. Must be `false` for divorced / widowed / still married |
| `previousMarriageStatus` | Text | **If** `isFirstMarriage` is `false` | `still_married`, `divorced`, `widowed` |
| `hasChildren` | Boolean | **If** `isFirstMarriage` is `false` | `true` / `false`. Children from previous marriage |

Fields follow the screen: Professional, then Family, then Marital.

**Next →** [#10 Step 3 – Partner preferences](#10-step-3--partner-preferences) (`PUT /api/signup/preferences`)

### 10. Step 3 – Partner preferences

`PUT /api/signup/preferences` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `ageMin` | Integer | **Yes** | Min 18, Max 80 |
| `ageMax` | Integer | **Yes** | Min 18, Max 80. Must be ≥ `ageMin` |
| `heightMinInches` | Number | **Yes** | Inches, 47–84 (`masterData.heights`) |
| `heightMaxInches` | Number | **Yes** | Inches, 47–84 (`masterData.heights`). Must be ≥ `heightMinInches` |
| `weightCategories` | Array | **Yes** | One or more `masterData.weights` values, or `any` |
| `complexion` | Array | **Yes** | One or more of: `very_fair`, `fair`, `wheatish`, `slightly_dark`, `dark`, or `any` |
| `education` | Array | **Yes** | One or more `masterData.secularEducation` values, or `any`. Minimum level. One value or several |
| `religiousEducation` | Array | **Yes** | One or more of: `hifz`, `alimiyat`, `moderate_knowledge`, `self_taught`, `basic`, or `any` |
| `religiosity` | Text | **Yes** | `very_practising`, `practising`, `moderate`, `liberal`, or `any` |
| `prayerHabits` | Text | **Yes** | `five_times`, `mostly`, `sometimes`, `jummah_only`, `rarely`, or `any` |
| `sect` | Array | **Yes** | One or more `masterData.sects` values, or `any`. One value or several. `sunni` includes all Sunni sub-options |
| `jamaatAffiliation` | Text | **Yes** | `barelvi`, `deobandi`, `ahl_e_hadith`, `hanafi`, `other`, or `any` |
| `hijabLevel` | Text | **Yes** for male members, No for female | `niqab`, `hijab_scarf`, `occasionally`, `no_hijab`, or `any` |
| `professionalOrientation` | Text | **Yes** | `career_oriented`, `balanced`, `home_maker`, or `any` |
| `maritalHistory` | Text | **Yes** | `first_marriage_only`, `open_to_divorced`, `open_to_widowed`, `open_to_divorced_widowed`, `any` |

**Next →**

- "Upload Photos": [#11 Upload photos (completes signup)](#11-upload-photos-completes-signup) (`POST /api/profile/photos`)
- "Add photos later": [#12 Skip photos ("Add photos later")](#12-skip-photos-add-photos-later) (`POST /api/signup/photos/skip`)

### 11. Upload photos (completes signup)

`POST /api/profile/photos` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `photos` | File(s) (form-data) | **Yes** | One or more images (JPG, PNG, WEBP), 5 MB each. Plan limit: free 3, basic 6, premium 15. The first becomes the main photo |

**Returns:** `photos`, `signupStep: "completed"`. Over the plan limit → `403` `PLAN_PHOTO_LIMIT_REACHED`.

**Next →** [#20 Register device for push notifications](#20-register-device-for-push-notifications) (`POST /api/settings/device-token`)

### 12. Skip photos ("Add photos later")

`POST /api/signup/photos/skip` · 🔐 Token

_No request fields._

**Returns:** `signupStep: "completed"`.

**Next →** [#20 Register device for push notifications](#20-register-device-for-push-notifications) (`POST /api/settings/device-token`)

## D. Login

Every login returns `token` and `user`. If `user.signupStep` is not `completed`, open that signup step instead of Home.

### 13. Login with email + password

`POST /api/auth/login` · 🔓 Public

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `email` | Text | **Yes** |  |
| `password` | Text | **Yes** |  |

`401` wrong credentials · `400` account has no password (use OTP or Forgot password) · `403` suspended/deactivated, or `signupStep: "verify_phone"` (resend OTP, then verify).

**Next →**

- signupStep = completed: [#20 Register device for push notifications](#20-register-device-for-push-notifications) (`POST /api/settings/device-token`)
- signup not finished: [#6 Signup status (resume)](#6-signup-status-resume) (`GET /api/signup/status`)
- "Login with OTP" instead: [#14 Login with phone – send OTP](#14-login-with-phone--send-otp) (`POST /api/auth/login/otp/send`)
- "Forgot password?": [#16 Send OTP](#16-send-otp) (`POST /api/auth/forgot-password`)

### 14. Login with phone – send OTP

`POST /api/auth/login/otp/send` · 🔓 Public

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `phone` | Text | **Yes** | Registered Indian mobile |

`404` if no account uses this number.

**Next →** [#15 Login with phone – verify OTP](#15-login-with-phone--verify-otp) (`POST /api/auth/login/otp/verify`)

### 15. Login with phone – verify OTP

`POST /api/auth/login/otp/verify` · 🔓 Public

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `phone` | Text | **Yes** |  |
| `otp` | Text | **Yes** | 4 digits |

**Next →**

- signupStep = completed: [#20 Register device for push notifications](#20-register-device-for-push-notifications) (`POST /api/settings/device-token`)
- signup not finished: [#6 Signup status (resume)](#6-signup-status-resume) (`GET /api/signup/status`)
- Code expired (purpose = login): [#5 Resend OTP](#5-resend-otp) (`POST /api/auth/register/resend-otp`)

## E. Forgot password

Choose Email or Phone → Send OTP → Verify OTP → set new password. Use the same email or phone in both steps.

### 16. Send OTP

`POST /api/auth/forgot-password` · 🔓 Public

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `email` | Text | Yes, or `phone` |  |
| `phone` | Text | Yes, or `email` |  |

**Returns:** `sentTo` (masked, e.g. `imXXXXXX@XXXX.com`), `otp: { expiresIn, resendAfter }`.

**Next →** [#17 Verify OTP](#17-verify-otp) (`POST /api/auth/forgot-password/verify`)

### 17. Verify OTP

`POST /api/auth/forgot-password/verify` · 🔓 Public

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `email` | Text | Yes, or `phone` |  |
| `phone` | Text | Yes, or `email` |  |
| `otp` | Text | **Yes** | 4 digits |

**Returns:** `resetToken` (valid 10 minutes).

**Next →** [#18 Set new password](#18-set-new-password) (`POST /api/auth/reset-password`)

### 18. Set new password

`POST /api/auth/reset-password` · 🔓 Public

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `resetToken` | Text | **Yes** | From the previous step |
| `newPassword` | Text | **Yes** | Min 6 chars |
| `confirmPassword` | Text | **Yes** | Must equal `newPassword` |

**Next →** [#13 Login with email + password](#13-login-with-email--password) (`POST /api/auth/login`)

## F. After login

### 19. Get logged-in member

`GET /api/auth/me` · 🔐 Token

_No request fields._

**Returns:** `user` (includes `signupStep`, `membership`). `401` means the token expired or was logged out.

**Next →**

- signupStep = completed: [#20 Register device for push notifications](#20-register-device-for-push-notifications) (`POST /api/settings/device-token`)
- signup not finished: [#6 Signup status (resume)](#6-signup-status-resume) (`GET /api/signup/status`)
- 401: [#13 Login with email + password](#13-login-with-email--password) (`POST /api/auth/login`)

### 20. Register device for push notifications

`POST /api/settings/device-token` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `token` | Text | **Yes** | FCM device token |
| `platform` | Text | **Yes** | `android`, `ios`, `web` |

Call after every login and whenever FCM issues a new token.

**Next →**

- Community guidelines not accepted yet: [#21 Accept community guidelines](#21-accept-community-guidelines) (`POST /api/settings/community-guidelines/accept`)
- [#22 Home](#22-home) (`GET /api/home`)

### 21. Accept community guidelines

`POST /api/settings/community-guidelines/accept` · 🔐 Token

_No request fields._

**Returns:** `communityGuidelinesAcceptedAt`. Show the text from Static page (`community_guidelines`) first.

**Next →** [#22 Home](#22-home) (`GET /api/home`)

## G. Home, matches and search

### 22. Home

`GET /api/home` · 🔐 Token

_No request fields._

**Returns:** `home: { greetingName, location, matchesCount, matchesCountText ("24 matches in Hyderabad today"), matches[10], newlyJoined[10], unreadNotifications }`. Lists contain match cards.

**Next →**

- Tap a card: [#26 View profile](#26-view-profile) (`GET /api/profile/:id`)
- "See all" matches: [#23 Matches list](#23-matches-list) (`GET /api/matches`)
- "See all" newly joined: [#24 Newly joined](#24-newly-joined) (`GET /api/matches/newly-joined`)
- Filter icon: [#25 Search / filters](#25-search--filters) (`GET /api/search`)
- Bell icon: [#52 Notifications list](#52-notifications-list) (`GET /api/notifications`)

### 23. Matches list

`GET /api/matches` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `page` | Integer (query) | No | Default 1 |
| `limit` | Integer (query) | No | Default 20 |
| `sortBy` | Text (query) | No | `lastSeen` (default), `newest` |
| `location` | Text (query) | No | City, e.g. `Hyderabad` |

**Returns:** `total`, `page`, `pages`, `profiles` (match cards: `matchScore`, `isShortlisted`, `interestStatus`, `photo`, `photoAccess`).

**Next →**

- Tap a card: [#26 View profile](#26-view-profile) (`GET /api/profile/:id`)
- Interest button on the card: [#27 Send interest](#27-send-interest) (`POST /api/interests/:userId`)
- Heart / favourite: [#29 Shortlist / favourite](#29-shortlist--favourite) (`POST /api/shortlist/:userId`)
- "Not Now": [#32 Not Now (hide from my suggestions)](#32-not-now-hide-from-my-suggestions) (`POST /api/settings/hide/:userId`)

### 24. Newly joined

`GET /api/matches/newly-joined` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `page` | Integer (query) | No | Default 1 |
| `limit` | Integer (query) | No | Default 20 |

**Next →**

- Tap a card: [#26 View profile](#26-view-profile) (`GET /api/profile/:id`)

### 25. Search / filters

`GET /api/search` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `sect` | Text (query) | No | `masterData.sects` value(s). `sunni` matches all Sunni sub-options. **Premium filter** |
| `religiosity` | Text (query) | No | `masterData.religiosityLevels` value(s) |
| `familyValues` | Text (query) | No | `masterData.familyValues` value(s) |
| `ageMin` | Integer (query) | No |  |
| `ageMax` | Integer (query) | No |  |
| `heightMin` | Integer (query) | No | Inches. **Premium filter** |
| `heightMax` | Integer (query) | No | Inches. **Premium filter** |
| `education` | Text (query) | No | `masterData.secularEducation` value(s) |
| `maritalStatus` | Text (query) | No | `masterData.maritalStatuses` value(s) |
| `profession` | Text (query) | No | `masterData.professions` value(s) |
| `professionCategory` | Text (query) | No | Category name, e.g. `IT & Engineering` |
| `incomeMin` | Text (query) | No | `masterData.incomeRanges` value (range start). **Premium filter** |
| `incomeMax` | Text (query) | No | `masterData.incomeRanges` value (range end). **Premium filter** |
| `location` | Text (query) | No | City or area |
| `page` | Integer (query) | No | Default 1 |
| `limit` | Integer (query) | No | Default 20 |
| `sortBy` | Text (query) | No | `lastSeen` (default), `newest`, `featured`, `age_asc`, `age_desc` |

Comma-separated values mean "any of". **Clear All** = call with no filters. Premium filters on a free plan → `403` `PLAN_ADVANCED_SEARCH_REQUIRED` (show the upgrade screen → Membership plans).

**Next →**

- Tap a result: [#26 View profile](#26-view-profile) (`GET /api/profile/:id`)
- 403 PLAN_ADVANCED_SEARCH_REQUIRED: [#70 Membership plans](#70-membership-plans) (`GET /api/payments/plans`)

## H. Viewing a profile and acting on it

### 26. View profile

`GET /api/profile/:id` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `id` | ID (path) | **Yes** | The other member's `_id` |

**Returns:** `profile` with `sections` (basic, bio, professional, family, religious, preferences, contact), `photos`, `photoAccess`, `canRequestPhotos`, `relationship` (interest / shortlist / photo-request status), `matchScore` ("You match X/N" + `breakdown`), `restricted`.

Contact is masked until both members accept an interest. Free plans see only the main photo (`photosLimitedByPlan: true`). `404` hidden/deleted/blocked you; `403` you blocked them.

**Next →**

- "Send Interest": [#27 Send interest](#27-send-interest) (`POST /api/interests/:userId`)
- "Request photos" (when `canRequestPhotos`): [#28 Request photos](#28-request-photos) (`POST /api/photo-requests/:userId`)
- Favourite: [#29 Shortlist / favourite](#29-shortlist--favourite) (`POST /api/shortlist/:userId`)
- Share: [#35 Share profile](#35-share-profile) (`GET /api/profile/:id/share`)
- Menu → Block: [#37 Block member](#37-block-member) (`POST /api/settings/block/:userId`)
- Menu → Report: [#38 Report member](#38-report-member) (`POST /api/reports/:userId`)
- Already connected → Message: [#48 Conversations](#48-conversations) (`GET /api/messages/conversations`)

### 27. Send interest

`POST /api/interests/:userId` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `userId` | ID (path) | **Yes** | The other member's `_id` (from a match card) |
| `message` | Text | No | Max 300 chars |

`400` if already sent, or if they already sent you one (`interestId` returned: accept that instead). Plan quota: free 5 → `403` `PLAN_CONTACT_LIMIT_REACHED`.

**Next →**

- Refresh the profile (status becomes `sent`): [#26 View profile](#26-view-profile) (`GET /api/profile/:id`)
- 403 PLAN_CONTACT_LIMIT_REACHED: [#70 Membership plans](#70-membership-plans) (`GET /api/payments/plans`)

### 28. Request photos

`POST /api/photo-requests/:userId` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `userId` | ID (path) | **Yes** | The other member's `_id` (from a match card) |

Needs a plan with photo access (`403` `PLAN_PHOTO_ACCESS_REQUIRED`). Only when the owner's photo visibility is "on request" or "blurred until approved".

**Next →**

- Track it: [#46 Sent photo requests](#46-sent-photo-requests) (`GET /api/photo-requests/sent`)
- 403 PLAN_PHOTO_ACCESS_REQUIRED: [#70 Membership plans](#70-membership-plans) (`GET /api/payments/plans`)

### 29. Shortlist / favourite

`POST /api/shortlist/:userId` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `userId` | ID (path) | **Yes** | The other member's `_id` (from a match card) |

**Next →**

- Open favourites: [#31 My shortlist (favourites)](#31-my-shortlist-favourites) (`GET /api/shortlist`)

### 30. Remove from shortlist

`DELETE /api/shortlist/:userId` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `userId` | ID (path) | **Yes** | The other member's `_id` (from a match card) |

**Next →** [#31 My shortlist (favourites)](#31-my-shortlist-favourites) (`GET /api/shortlist`)

### 31. My shortlist (favourites)

`GET /api/shortlist` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `page` | Integer (query) | No | Default 1 |
| `limit` | Integer (query) | No | Default 20 |

**Returns:** `total`, `page`, `shortlist: [{ _id, createdAt, shortlistedUser: <match card> }]`.

**Next →**

- Tap a card: [#26 View profile](#26-view-profile) (`GET /api/profile/:id`)
- Remove: [#30 Remove from shortlist](#30-remove-from-shortlist) (`DELETE /api/shortlist/:userId`)

### 32. Not Now (hide from my suggestions)

`POST /api/settings/hide/:userId` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `userId` | ID (path) | **Yes** | The other member's `_id` (from a match card) |

**Next →**

- Card disappears from lists: [#23 Matches list](#23-matches-list) (`GET /api/matches`)

### 33. Not Now list

`GET /api/settings/hidden` · 🔐 Token

_No request fields._

**Returns:** `hiddenUsers` (match cards).

**Next →** [#34 Undo Not Now](#34-undo-not-now) (`DELETE /api/settings/hide/:userId`)

### 34. Undo Not Now

`DELETE /api/settings/hide/:userId` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `userId` | ID (path) | **Yes** | The other member's `_id` (from a match card) |

**Next →** [#33 Not Now list](#33-not-now-list) (`GET /api/settings/hidden`)

### 35. Share profile

`GET /api/profile/:id/share` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `id` | ID (path) | **Yes** | Member's `_id` (or your own) |

**Returns:** `share: { profileCode, shareUrl, shareText }`. Open the phone's share sheet with it.

**Next →**

- Someone opens a shared link/code: [#36 Open shared profile](#36-open-shared-profile) (`GET /api/profile/code/:code`)

### 36. Open shared profile

`GET /api/profile/code/:code` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `code` | Text (path) | **Yes** | Profile code, e.g. `RO482913` |

**Returns:** Same as View profile.

**Next →** [#27 Send interest](#27-send-interest) (`POST /api/interests/:userId`)

### 37. Block member

`POST /api/settings/block/:userId` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `userId` | ID (path) | **Yes** | The other member's `_id` (from a match card) |

Also removes pending interests, pending photo requests and favourites between you.

**Next →**

- Back to lists (member no longer appears): [#22 Home](#22-home) (`GET /api/home`)
- Settings → Blocked users: [#77 Blocked members](#77-blocked-members) (`GET /api/settings/blocked`)

### 38. Report member

`POST /api/reports/:userId` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `userId` | ID (path) | **Yes** | The other member's `_id` (from a match card) |
| `reason` | Text | **Yes** | `masterData.reportReasons` value |
| `details` | Text | **If** `reason` is `others` | Max 1000 chars |

One report per member per 24 hours. `POST /api/settings/report/:userId` does the same.

**Next →**

- Offer "Block this member too": [#37 Block member](#37-block-member) (`POST /api/settings/block/:userId`)
- [#22 Home](#22-home) (`GET /api/home`)

## I. Requests screen

| Tab | Received (Accept / Decline) | Accepted (View Profile) |
|---|---|---|
| Gallery | Received photo requests, `status=pending` | Received photo requests, `status=accepted` |
| Requests | Received interests, `status=pending` | Received interests, `status=accepted` |
| Interests | Sent interests | Mutual interests |

### 39. Received interests

`GET /api/interests/received` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `status` | Text (query) | No | `pending`, `accepted`, `declined` |
| `page` | Integer (query) | No | Default 1 |
| `limit` | Integer (query) | No | Default 20 |

**Returns:** `interests: [{ _id, status, message, sender: <match card> }]`.

**Next →**

- Accept / Decline: [#40 Accept / decline interest](#40-accept--decline-interest) (`PUT /api/interests/:interestId`)
- View Profile: [#26 View profile](#26-view-profile) (`GET /api/profile/:id`)

### 40. Accept / decline interest

`PUT /api/interests/:interestId` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `interestId` | ID (path) | **Yes** |  |
| `status` | Text | **Yes** | `accepted`, `declined` |

Accepting connects both members: contact details unlock and a conversation is created.

**Next →**

- Accepted: [#43 Mutual interests](#43-mutual-interests) (`GET /api/matches/mutual`)
- Accepted → start chatting: [#48 Conversations](#48-conversations) (`GET /api/messages/conversations`)
- Declined: [#39 Received interests](#39-received-interests) (`GET /api/interests/received`)

### 41. Sent interests

`GET /api/interests/sent` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `status` | Text (query) | No | `pending`, `accepted`, `declined` |
| `page` | Integer (query) | No | Default 1 |
| `limit` | Integer (query) | No | Default 20 |

**Next →**

- Cancel a pending one: [#42 Cancel sent interest](#42-cancel-sent-interest) (`DELETE /api/interests/:interestId`)
- [#26 View profile](#26-view-profile) (`GET /api/profile/:id`)

### 42. Cancel sent interest

`DELETE /api/interests/:interestId` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `interestId` | ID (path) | **Yes** |  |

**Next →** [#41 Sent interests](#41-sent-interests) (`GET /api/interests/sent`)

### 43. Mutual interests

`GET /api/matches/mutual` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `page` | Integer (query) | No | Default 1 |
| `limit` | Integer (query) | No | Default 20 |

**Next →**

- Contact details now visible: [#26 View profile](#26-view-profile) (`GET /api/profile/:id`)
- [#48 Conversations](#48-conversations) (`GET /api/messages/conversations`)

### 44. Received photo requests

`GET /api/photo-requests/received` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `status` | Text (query) | No | `pending`, `accepted`, `declined` |
| `page` | Integer (query) | No | Default 1 |
| `limit` | Integer (query) | No | Default 20 |

**Next →**

- Accept / Decline: [#45 Accept / decline photo request](#45-accept--decline-photo-request) (`PUT /api/photo-requests/:requestId`)
- [#26 View profile](#26-view-profile) (`GET /api/profile/:id`)

### 45. Accept / decline photo request

`PUT /api/photo-requests/:requestId` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `requestId` | ID (path) | **Yes** |  |
| `status` | Text | **Yes** | `accepted`, `declined` |

Accepting lets the requester see your photos; they get a notification.

**Next →** [#44 Received photo requests](#44-received-photo-requests) (`GET /api/photo-requests/received`)

### 46. Sent photo requests

`GET /api/photo-requests/sent` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `status` | Text (query) | No | `pending`, `accepted`, `declined` |
| `page` | Integer (query) | No | Default 1 |
| `limit` | Integer (query) | No | Default 20 |

**Next →**

- Accepted → photos visible: [#26 View profile](#26-view-profile) (`GET /api/profile/:id`)
- Cancel a pending one: [#47 Cancel sent photo request](#47-cancel-sent-photo-request) (`DELETE /api/photo-requests/:requestId`)

### 47. Cancel sent photo request

`DELETE /api/photo-requests/:requestId` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `requestId` | ID (path) | **Yes** |  |

**Next →** [#46 Sent photo requests](#46-sent-photo-requests) (`GET /api/photo-requests/sent`)

## J. Messages

Needs a plan with messaging (`403` `PLAN_MESSAGING_REQUIRED` → Membership plans). A conversation exists once an interest is accepted.

### 48. Conversations

`GET /api/messages/conversations` · 🔐 Token

_No request fields._

**Returns:** `conversations` (participants, last message).

**Next →**

- Open a chat: [#49 Messages in a conversation](#49-messages-in-a-conversation) (`GET /api/messages/:conversationId`)
- 403 PLAN_MESSAGING_REQUIRED: [#70 Membership plans](#70-membership-plans) (`GET /api/payments/plans`)

### 49. Messages in a conversation

`GET /api/messages/:conversationId` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `conversationId` | ID (path) | **Yes** |  |
| `page` | Integer (query) | No | Default 1 |
| `limit` | Integer (query) | No | Default 30 |

**Returns:** `messages` (oldest first). Marks the other member's messages as read.

**Next →** [#50 Send message](#50-send-message) (`POST /api/messages/:conversationId`)

### 50. Send message

`POST /api/messages/:conversationId` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `conversationId` | ID (path) | **Yes** |  |
| `content` | Text | **Yes**, unless `mediaUrl` | Message text (`text` also accepted) |
| `type` | Text | No | Default `text` |
| `mediaUrl` | Text | No | For media messages |
| `mediaType` | Text | No | `image`, `audio`, `video` |

**Next →**

- Refresh the chat: [#49 Messages in a conversation](#49-messages-in-a-conversation) (`GET /api/messages/:conversationId`)
- Long-press → Delete: [#51 Delete message](#51-delete-message) (`DELETE /api/messages/:messageId`)

### 51. Delete message

`DELETE /api/messages/:messageId` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `messageId` | ID (path) | **Yes** | Your own message |

**Next →** [#49 Messages in a conversation](#49-messages-in-a-conversation) (`GET /api/messages/:conversationId`)

## K. Notifications

### 52. Notifications list

`GET /api/notifications` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `page` | Integer (query) | No | Default 1 |
| `limit` | Integer (query) | No | Default 20 |

**Returns:** `unreadCount`, `notifications`, and the same items split into `recent` (last 24 h) and `older`. Each has `type`, `title`, `body`, `timeAgo`, `isRead`, `data` (e.g. `userId`, `interestId`).

Types: `new_matches_today`, `preference_match`, `interest_received`, `interest_accepted`, `photo_request_received`, `photo_access_accepted`, `profile_viewed`, `shortlisted`, `message_received`, `system`. Same list at `GET /api/settings/notifications`.

**Next →**

- Tap a notification: [#54 Mark one as read](#54-mark-one-as-read) (`PUT /api/notifications/:notificationId/read`)
- "Mark all as read": [#55 Mark all as read](#55-mark-all-as-read) (`PUT /api/notifications/read-all`)

### 53. Unread count (badge)

`GET /api/settings/notifications/unread-count` · 🔐 Token

_No request fields._

**Next →** [#52 Notifications list](#52-notifications-list) (`GET /api/notifications`)

### 54. Mark one as read

`PUT /api/notifications/:notificationId/read` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `notificationId` | ID (path) | **Yes** |  |

**Next →**

- Open the member in `data.userId`: [#26 View profile](#26-view-profile) (`GET /api/profile/:id`)
- interest_received: [#39 Received interests](#39-received-interests) (`GET /api/interests/received`)
- photo_request_received: [#44 Received photo requests](#44-received-photo-requests) (`GET /api/photo-requests/received`)

### 55. Mark all as read

`PUT /api/notifications/read-all` · 🔐 Token

_No request fields._

**Next →** [#52 Notifications list](#52-notifications-list) (`GET /api/notifications`)

## L. My profile and Edit Profile

Each Edit Profile section has its own **Save**; send only that screen's fields. "Required" here means the field **can't be cleared**.

### 56. Get my full profile

`GET /api/profile/me` · 🔐 Token

_No request fields._

**Returns:** `myProfile` with `age`, `signupStep`, `membership`, `isIdVerified`, `isPhoneVerified`, `idProof`, `photos`, `sections` (unmasked contact). `profile` holds the raw record.

**Next →**

- [#57 Edit bio](#57-edit-bio) (`PUT /api/profile/me/bio`)
- [#58 Edit basic info](#58-edit-basic-info) (`PUT /api/profile/me/basic`)
- [#59 Edit professional info](#59-edit-professional-info) (`PUT /api/profile/me/professional`)
- [#60 Edit family info](#60-edit-family-info) (`PUT /api/profile/me/family`)
- [#61 Edit religious info](#61-edit-religious-info) (`PUT /api/profile/me/religious`)
- [#62 Edit partner preferences](#62-edit-partner-preferences) (`PUT /api/profile/me/preferences`)
- [#63 Edit contact details](#63-edit-contact-details) (`PUT /api/profile/me/contact`)
- Manage photos: [#66 Add photos](#66-add-photos) (`POST /api/profile/photos`)

### 57. Edit bio

`PUT /api/profile/me/bio` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `bio` | Text | No | Max 1000 chars |

**Next →** [#56 Get my full profile](#56-get-my-full-profile) (`GET /api/profile/me`)

### 58. Edit basic info

`PUT /api/profile/me/basic` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `name` | Text | **Yes** | Max 100 chars |
| `gender` | Text | **Yes** | `male`, `female` |
| `dateOfBirth` | Date | **Yes** | `YYYY-MM-DD`, age 18+ |
| `heightInches` | Number | **Yes** | Inches, 47–84 (`masterData.heights`) |
| `weight` | Text | **Yes** | `masterData.weights` value |
| `complexion` | Text | **Yes** | `very_fair`, `fair`, `wheatish`, `slightly_dark`, `dark` |
| `languages` | Array | **Yes** | One or more `masterData.languages` values |
| `address` | Text | **Yes** | Max 300 chars |
| `city` | Text | No | Max 100 chars |
| `maritalStatus` | Text | **Yes** | `never_married`, `divorced`, `widowed`, `still_married` |
| `isFirstMarriage` | Boolean | No | `true` / `false`. If not sent: `true` for `never_married`, otherwise `false`. Must be `false` for divorced / widowed / still married |
| `previousMarriageStatus` | Text | **If** `isFirstMarriage` is `false` | `still_married`, `divorced`, `widowed` |
| `hasChildren` | Boolean | **If** `isFirstMarriage` is `false` | `true` / `false`. Children from previous marriage |

**Next →** [#56 Get my full profile](#56-get-my-full-profile) (`GET /api/profile/me`)

### 59. Edit professional info

`PUT /api/profile/me/professional` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `education` | Text | No | `masterData.secularEducation` value |
| `profession` | Text | **Yes** | `masterData.professions` value |
| `jobLocation` | Text | **Yes** | Max 100 chars |
| `designation` | Text | **Yes** | Max 100 chars |
| `yearsOfExperience` | Integer | **Yes** | Min 0, Max 60 |
| `income` | Text | No | `masterData.incomeRanges` value |
| `workingAbroadStatus` | Text | **Yes** | `no_india`, `job_visa`, `permanent_residency`, `open_to_abroad` |
| `netWorth` | Text | No | Max 100 chars |
| `professionalOrientation` | Text | No | `career_oriented`, `balanced`, `home_maker` |

**Next →** [#56 Get my full profile](#56-get-my-full-profile) (`GET /api/profile/me`)

### 60. Edit family info

`PUT /api/profile/me/family` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `fatherProfession` | Text | **Yes** | Max 100 chars |
| `motherProfession` | Text | **Yes** | Max 100 chars |
| `familyValues` | Text | **Yes** | `traditional`, `moderate`, `liberal` |
| `siblings` | Integer | **Yes** | Min 0, Max 20 |
| `siblingsDetails` | Text | No | Max 300 chars |
| `residenceType` | Text | **Yes** | `flat`, `independent_house` |
| `familyLivesIn` | Text | **Yes** | Max 100 chars |
| `nativePlace` | Text | **Yes** | Max 100 chars |

**Next →** [#56 Get my full profile](#56-get-my-full-profile) (`GET /api/profile/me`)

### 61. Edit religious info

`PUT /api/profile/me/religious` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `sect` | Text | **Yes** | `masterData.sects` value |
| `jamaatAffiliation` | Text | **Yes** | `barelvi`, `deobandi`, `ahl_e_hadith`, `hanafi`, `other` |
| `religiosity` | Text | **Yes** | `very_practising`, `practising`, `moderate`, `liberal` |
| `prayerHabits` | Text | **Yes** | `five_times`, `mostly`, `sometimes`, `jummah_only`, `rarely` |
| `religiousEducation` | Text | **Yes** | `hifz`, `alimiyat`, `moderate_knowledge`, `self_taught`, `basic` |
| `isRevert` | Boolean | No | `true` / `false` |
| `hijabLevel` | Text | No | `niqab`, `hijab_scarf`, `occasionally`, `no_hijab` |

**Next →** [#56 Get my full profile](#56-get-my-full-profile) (`GET /api/profile/me`)

### 62. Edit partner preferences

`PUT /api/profile/me/preferences` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `ageMin` | Integer | **Yes** | Min 18, Max 80 |
| `ageMax` | Integer | **Yes** | Min 18, Max 80. Must be ≥ `ageMin` |
| `heightMinInches` | Number | **Yes** | Inches, 47–84 (`masterData.heights`) |
| `heightMaxInches` | Number | **Yes** | Inches, 47–84 (`masterData.heights`). Must be ≥ `heightMinInches` |
| `weightCategories` | Array | **Yes** | One or more `masterData.weights` values, or `any` |
| `complexion` | Array | **Yes** | One or more of: `very_fair`, `fair`, `wheatish`, `slightly_dark`, `dark`, or `any` |
| `education` | Array | **Yes** | One or more `masterData.secularEducation` values, or `any`. Minimum level. One value or several |
| `religiousEducation` | Array | **Yes** | One or more of: `hifz`, `alimiyat`, `moderate_knowledge`, `self_taught`, `basic`, or `any` |
| `religiosity` | Text | **Yes** | `very_practising`, `practising`, `moderate`, `liberal`, or `any` |
| `prayerHabits` | Text | **Yes** | `five_times`, `mostly`, `sometimes`, `jummah_only`, `rarely`, or `any` |
| `sect` | Array | **Yes** | One or more `masterData.sects` values, or `any`. One value or several. `sunni` includes all Sunni sub-options |
| `jamaatAffiliation` | Text | **Yes** | `barelvi`, `deobandi`, `ahl_e_hadith`, `hanafi`, `other`, or `any` |
| `hijabLevel` | Text | **Yes** for male members, No for female | `niqab`, `hijab_scarf`, `occasionally`, `no_hijab`, or `any` |
| `professionalOrientation` | Text | **Yes** | `career_oriented`, `balanced`, `home_maker`, or `any` |
| `maritalHistory` | Text | **Yes** | `first_marriage_only`, `open_to_divorced`, `open_to_widowed`, `open_to_divorced_widowed`, `any` |

**Next →**

- [#56 Get my full profile](#56-get-my-full-profile) (`GET /api/profile/me`)
- Matches update to the new preferences: [#23 Matches list](#23-matches-list) (`GET /api/matches`)

### 63. Edit contact details

`PUT /api/profile/me/contact` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `email` | Text | Yes, or `phone` | Valid, not used by another account. Saved immediately |
| `phone` | Text | Yes, or `email` | New Indian mobile, not used by another account |

**Returns:** If the phone changed: `phoneVerificationRequired: true` and an OTP goes to the **new** number. Call again to resend.

**Next →**

- Phone changed: [#64 Verify new phone number](#64-verify-new-phone-number) (`POST /api/profile/me/contact/verify-phone`)
- Only email changed: [#56 Get my full profile](#56-get-my-full-profile) (`GET /api/profile/me`)

### 64. Verify new phone number

`POST /api/profile/me/contact/verify-phone` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `otp` | Text | **Yes** | 4 digits sent to the new number |

**Next →** [#56 Get my full profile](#56-get-my-full-profile) (`GET /api/profile/me`)

### 65. Who viewed my profile

`GET /api/profile/me/viewers` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `page` | Integer (query) | No | Default 1 |
| `limit` | Integer (query) | No | Default 20 |

Paid plans only (`403` `PLAN_PAID_REQUIRED`).

**Next →**

- Tap a viewer: [#26 View profile](#26-view-profile) (`GET /api/profile/:id`)
- 403 PLAN_PAID_REQUIRED: [#70 Membership plans](#70-membership-plans) (`GET /api/payments/plans`)

## M. Photos management

### 66. Add photos

`POST /api/profile/photos` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `photos` | File(s) (form-data) | **Yes** | Images (JPG, PNG, WEBP), 5 MB each. Plan limit: free 3, basic 6, premium 15 |

**Next →**

- Choose the main photo: [#67 Set main photo](#67-set-main-photo) (`PUT /api/profile/photos/:photoId/main`)
- Drag to reorder: [#68 Reorder photos](#68-reorder-photos) (`PUT /api/profile/photos/order`)
- Who can see my photos: [#75 Update settings](#75-update-settings) (`PUT /api/settings`)

### 67. Set main photo

`PUT /api/profile/photos/:photoId/main` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `photoId` | ID (path) | **Yes** |  |

**Next →** [#56 Get my full profile](#56-get-my-full-profile) (`GET /api/profile/me`)

### 68. Reorder photos

`PUT /api/profile/photos/order` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `photoIds` | Array | **Yes** | Every one of the member's photo ids, exactly once, in the new order |

**Next →** [#56 Get my full profile](#56-get-my-full-profile) (`GET /api/profile/me`)

### 69. Delete photo

`DELETE /api/profile/photos/:photoId` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `photoId` | ID (path) | **Yes** |  |

**Next →** [#56 Get my full profile](#56-get-my-full-profile) (`GET /api/profile/me`)

## N. Membership and payments (PhonePe)

### 70. Membership plans

`GET /api/payments/plans` · 🔐 Token

_No request fields._

**Returns:** `plans` (name, type, price in ₹, duration, features). Show these on the upgrade screen.

**Next →**

- Member picks a plan: [#71 Start payment](#71-start-payment) (`POST /api/payments/initiate`)

### 71. Start payment

`POST /api/payments/initiate` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `planId` | ID | **Yes** | `_id` from Membership plans |

**Returns:** `paymentUrl` (open it in the browser / PhonePe), `merchantTransactionId`.

After paying, PhonePe sends the member back to the app through the deep link (`APP_DEEP_LINK/result?txn=…`).

**Next →**

- Member returns to the app: [#72 Payment status](#72-payment-status) (`GET /api/payments/status/:merchantTransactionId`)

### 72. Payment status

`GET /api/payments/status/:merchantTransactionId` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `merchantTransactionId` | Text (path) | **Yes** | From Start payment |

**Returns:** `status` (`pending`, `success`, `failed`), `transaction` (amount, plan, `activatedAt`, `expiresAt`).

**Next →**

- Still pending → poll again after a few seconds: [#72 Payment status](#72-payment-status) (`GET /api/payments/status/:merchantTransactionId`)
- success → reload membership: [#19 Get logged-in member](#19-get-logged-in-member) (`GET /api/auth/me`)
- [#73 My transactions](#73-my-transactions) (`GET /api/payments/history`)

### 73. My transactions

`GET /api/payments/history` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `page` | Integer (query) | No | Default 1 |
| `limit` | Integer (query) | No | Default 10 |

**Next →**

- Renew / upgrade: [#70 Membership plans](#70-membership-plans) (`GET /api/payments/plans`)

## O. Settings

### 74. Get settings

`GET /api/settings` · 🔐 Token

_No request fields._

**Returns:** `settings: { isProfileHidden, photoVisibility, whoCanViewProfile, appLanguage, locationPreference, notificationSettings, communityGuidelinesAccepted, communityGuidelinesAcceptedAt }`.

**Next →**

- [#75 Update settings](#75-update-settings) (`PUT /api/settings`)
- [#76 Hide / unhide my profile](#76-hide--unhide-my-profile) (`PUT /api/settings/hide-profile`)
- [#77 Blocked members](#77-blocked-members) (`GET /api/settings/blocked`)
- [#79 Contact support](#79-contact-support) (`POST /api/settings/support`)
- [#83 Delete profile permanently](#83-delete-profile-permanently) (`DELETE /api/settings/profile`)

### 75. Update settings

`PUT /api/settings` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `photoVisibility` | Text | No | `everyone`, `on_request`, `after_match`, `blurred_until_approved`, `hidden` |
| `whoCanViewProfile` | Text | No | `everyone`, `matches_only`, `after_interest_accepted`, `basic_info_only` |
| `appLanguage` | Text | No | `masterData.appLanguages` value |
| `locationPreference` | Text | No | Max 100 chars, e.g. `Hyderabad` |
| `notificationSettings` | Object | No | `{ matches, interests, acceptedInterest, photoRequest, photoAccessAccepted }`, each `true`/`false`; send only what changed |

Send at least one field. Each settings screen can send just its own field.

**Next →** [#74 Get settings](#74-get-settings) (`GET /api/settings`)

### 76. Hide / unhide my profile

`PUT /api/settings/hide-profile` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `hidden` | Boolean | **Yes** | `true` hides, `false` unhides |

**Next →** [#74 Get settings](#74-get-settings) (`GET /api/settings`)

### 77. Blocked members

`GET /api/settings/blocked` · 🔐 Token

_No request fields._

**Returns:** `blockedUsers: [{ _id, name, age, city }]`.

**Next →** [#78 Unblock member](#78-unblock-member) (`DELETE /api/settings/block/:userId`)

### 78. Unblock member

`DELETE /api/settings/block/:userId` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `userId` | ID (path) | **Yes** | The other member's `_id` (from a match card) |

**Next →** [#77 Blocked members](#77-blocked-members) (`GET /api/settings/blocked`)

## P. Support and static pages

### 79. Contact support

`POST /api/settings/support` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `firstName` | Text | **Yes** | Max 50 chars |
| `lastName` | Text | **Yes** | Max 50 chars |
| `email` | Text | **Yes** | Valid email |
| `phone` | Text | No | Max 20 chars |
| `message` | Text | **Yes** | Max 1000 chars |
| `privacyConsent` | Boolean | **Yes** | Must be `true` |

**Returns:** `ticketId`. Show a thank-you message.

**Next →** [#74 Get settings](#74-get-settings) (`GET /api/settings`)

### 80. Static page

`GET /api/content/:key` · 🔓 Public

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `key` | Text (path) | **Yes** | `faq`, `community_guidelines`, `about_us`, `privacy_policy`, `terms` |

**Returns:** `content: { title, body, items: [{ question, answer }] }` (`items` is the FAQ list). `404` until it has been published.

**Next →**

- key = community_guidelines → "I accept": [#21 Accept community guidelines](#21-accept-community-guidelines) (`POST /api/settings/community-guidelines/accept`)

## Q. Account: password, logout, delete

### 81. Change password

`PUT /api/auth/change-password` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `currentPassword` | Text | **Yes**, unless the account has no password yet |  |
| `newPassword` | Text | **Yes** | Min 6 chars |
| `confirmPassword` | Text | **Yes** | Must equal `newPassword` |

**Next →** [#74 Get settings](#74-get-settings) (`GET /api/settings`)

### 82. Logout

`POST /api/auth/logout` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `deviceToken` | Text | No | This device's FCM token, so it stops getting push notifications |

**Next →**

- Show the login screen: [#13 Login with email + password](#13-login-with-email--password) (`POST /api/auth/login`)

### 83. Delete profile permanently

`DELETE /api/settings/profile` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `confirm` | Text | **Yes** | Exactly `DELETE` (after the member confirms in the app) |

Removes interests, favourites, photo requests and notifications, hides the profile and logs the member out.

**Next →**

- Back to the welcome screen: [#3 Create account (sends OTP)](#3-create-account-sends-otp) (`POST /api/auth/register`)

## Older app endpoints (still supported)

Kept so app versions released before the redesign keep working. The redesigned app does not need them.

| Method & URL | Body / notes |
|---|---|
| `POST /api/auth/register` | `{ fullName, email, password, gender, phone?, dateOfBirth? }` (no `profileCreatedFor`) → token straight away |
| `POST /api/auth/send-otp` 🔐 | `{ phone }`: verify a phone after login |
| `POST /api/auth/verify-otp` 🔐 | `{ otp }` |
| `POST /api/auth/reset-password` | `{ email, otp, newPassword }` in one step |
| `PUT /api/profile/me` 🔐 | Single-form profile update (no longer changes phone) |
| `PUT /api/profile/update/basic-info`, `/physical-info`, `/education-career`, `/family-info`, `/lifestyle`, `/partner-preference`, `/privacy` 🔐 | Original section edits |
| `PUT /api/profile/preferences` 🔐 | Replaces the whole partner-preferences object |
| `DELETE /api/profile/account` 🔐 | Deactivate account |
| `PUT /api/settings/visibility` 🔐 | `{ profileVisibility: public \| members \| hidden }` |
| `POST /api/matches/request/:userId`, `PUT /api/matches/:matchId/respond`, `GET /api/matches/sent`, `/received`, `/accepted` 🔐 | Original match-request flow (same records as interests) |
| `POST /api/matches/shortlist/:userId`, `GET /api/matches/shortlist` 🔐 | Toggle / list favourites |
| `GET /api/search/suggestions`, `/matches`, `/new-members` 🔐 | Original suggestion lists (now return match cards) |
| `GET /api/settings/notifications`, `PUT /api/settings/notifications/read-all`, `PUT /api/settings/notifications/:id/read` 🔐 | Same as the `/api/notifications` endpoints |
| `POST /api/settings/report/:userId` 🔐 | Same as `POST /api/reports/:userId` |
| `POST /api/auth/resend-otp` | Same as `POST /api/auth/register/resend-otp` |

`POST /api/payments/callback` and `GET /api/payments/redirect/:merchantTransactionId` are called by PhonePe and the browser, not by the app.
