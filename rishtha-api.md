# Rishtha Online API: fields by app flow
Every endpoint the app calls, in the order a user moves through the app, with its request fields and which of them are required.
- **Base URL:** `/api`
- **Auth:** 🔓 Public = no token. 🔐 Token = `Authorization: Bearer <token>`. 🛡 Admin token = token of a user with `role: "admin"`.
- **Fields** go in the JSON body unless marked *(path)*, *(query)* or *(form-data)*.
- **Dropdown fields** take the option **value** from `GET /api/master-data` (for example `never_married`, not "Never Married"). Where a list is long, the table names the master-data key.
- **`any`** ("Doesn't matter") is accepted only in partner-preference fields, where the table says so.
- **Errors:** validation failures return `400` with `{ "success": false, "message": "...", "errors": ["..."] }`.
- Full response samples are in [API-v2.md](API-v2.md); a ready-to-run collection is in [Rishtha-Online-API-v2.postman_collection.json](Rishtha-Online-API-v2.postman_collection.json).

## Contents
1. [App launch](#a-app-launch)
2. [Sign up](#b-sign-up)
3. [Signup steps 1–3 and photos](#c-signup-steps)
4. [Login](#d-login)
5. [Forgot password](#e-forgot-password)
6. [After login](#f-after-login)
7. [Home, matches and search](#g-home-matches-and-search)
8. [Viewing a profile and acting on it](#h-viewing-a-profile-and-acting-on-it)
9. [Requests screen](#i-requests-screen)
10. [Notifications](#j-notifications)
11. [My profile and Edit Profile](#k-my-profile-and-edit-profile)
12. [Photos management](#l-photos-management)
13. [Settings](#m-settings)
14. [Support and static pages](#n-support-and-static-pages)
15. [Account: password, logout, delete](#o-account-password-logout-delete)
16. [Admin](#p-admin)

## A. App launch

### 1. Load dropdown options

`GET /api/master-data` · 🔓 Public

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `keys` | Text (query) | No | Comma-separated list keys, e.g. `heights,sects`. Omit to get everything |

**Returns:** `masterData` — every dropdown list as `[{ value, label }]`. Cache it in the app.

### 2. Check app version

`GET /api/content/app-version` · 🔓 Public

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `platform` | Text (query) | No | `android` (default) or `ios` |
| `version` | Text (query) | No | Installed version, e.g. `1.4.0` |

**Returns:** `appVersion` with `latestVersion`, `minSupportedVersion`, `storeUrl`, and (when `version` is sent) `updateAvailable`, `updateRequired`.

## B. Sign up

Create Account → Verify Mobile OTP → "You're Verified".

### 3. Create account (sends OTP)

`POST /api/auth/register` · 🔓 Public

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `profileCreatedFor` | Text | **Yes** | `masterData.profileCreatedFor` value |
| `phone` | Text | **Yes** | Indian mobile, 10 digits starting 6–9. `9876543210`, `+91 98765 43210` and similar are accepted. Must not be registered |
| `gender` | Text | **Yes** | `male`, `female` |
| `email` | Text | No | Valid email. Must not be registered |
| `password` | Text | No | Min 6 chars. Without it the user logs in by OTP |

**Returns:** `userId`, `phone`, `signupStep: "verify_phone"`, `otp: { expiresIn: 300, resendAfter: 60 }`. **No token yet.**

### 4. Verify mobile OTP

`POST /api/auth/verify-otp` · 🔓 Public

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `phone` | Text | **Yes** | Same number used to register |
| `otp` | Text | **Yes** | 4 digits |

Wrong code → `400` with `attemptsLeft`. After 5 wrong attempts or 5 minutes the user must resend.

**Returns:** `token`, `user` (with `signupStep: "personal"`). Save the token: every signup step needs it.

### 5. Resend OTP

`POST /api/auth/resend-otp` · 🔓 Public

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `phone` | Text | Yes, or `email` | Use `phone` for signup and login |
| `email` | Text | Yes, or `phone` | Only for `purpose: "reset_password"` |
| `purpose` | Text | No | `signup` (default), `login`, `reset_password` |

One OTP per 60 seconds and 5 per hour. Too soon → `429` with `retryAfter` (seconds).

## C. Signup steps

Order: `verify_phone` → `personal` → `social` → `preferences` → `photos` → `completed`.
- A step can only be submitted after the previous one (otherwise `400`, with the current `signupStep`).
- Send **all** fields of the step. Every step returns `{ signupStep, completed, steps: [{ step, done }] }`.
- If the user leaves midway, call **Signup status** (or read `signupStep` from the login response) and open that step.

### 6. Signup status (resume)

`GET /api/signup/status` · 🔐 Token

_No request fields._

**Returns:** `signupStep`, `completed`, `steps`.

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
| `city` | Text | No | Max 100 chars. Not on the design, but needed for location matching ("in Hyderabad"). Send it if the app can |
| `languages` | Array | **Yes** | One or more `masterData.languages` values |
| `sect` | Text | **Yes** | `masterData.sects` value |
| `jamaatAffiliation` | Text | **Yes** | `barelvi`, `deobandi`, `ahl_e_hadith`, `hanafi`, `other` |
| `religiosity` | Text | **Yes** | `very_practising`, `practising`, `moderate`, `liberal` |
| `prayerHabits` | Text | **Yes** | `five_times`, `mostly`, `sometimes`, `jummah_only`, `rarely` |
| `religiousEducation` | Text | **Yes** | `hifz`, `alimiyat`, `moderate_knowledge`, `self_taught`, `basic` |
| `isRevert` | Boolean | No | `true` / `false`. Defaults to `false` |
| `hijabLevel` | Text | No | `niqab`, `hijab_scarf`, `occasionally`, `no_hijab`. Optional; used in match scores for female profiles |

### 8. Upload ID proof (Step 1 screen)

`POST /api/profile/id-proof` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `idProofType` | Text (form-data) | **Yes** | `aadhaar`, `pan`, `passport`, `voter_id`, `driving_licence`, `other` |
| `idProof` | File (form-data) | **Yes** | JPG, PNG, WEBP or PDF, max 5 MB |

Optional for signup progress. Can be re-uploaded until approved.

**Returns:** `idProof: { type, status: "pending" }`. Status later becomes `approved` or `rejected` (with `rejectionReason`).

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
| `netWorth` | Text | No | Max 100 chars. Shown in Edit Profile; optional here |
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

The table follows the screen: Professional, then Family, then Marital.

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
| `education` | Text | **Yes** | `masterData.secularEducation` value, or `any`. Minimum level |
| `religiousEducation` | Array | **Yes** | One or more of: `hifz`, `alimiyat`, `moderate_knowledge`, `self_taught`, `basic`, or `any` |
| `religiosity` | Text | **Yes** | `very_practising`, `practising`, `moderate`, `liberal`, or `any` |
| `prayerHabits` | Text | **Yes** | `five_times`, `mostly`, `sometimes`, `jummah_only`, `rarely`, or `any` |
| `sect` | Text | **Yes** | `masterData.sects` value, or `any` |
| `jamaatAffiliation` | Text | **Yes** | `barelvi`, `deobandi`, `ahl_e_hadith`, `hanafi`, `other`, or `any` |
| `hijabLevel` | Text | **Yes** for male users, No for female | `niqab`, `hijab_scarf`, `occasionally`, `no_hijab`, or `any` |
| `professionalOrientation` | Text | **Yes** | `career_oriented`, `balanced`, `home_maker`, or `any` |
| `maritalHistory` | Text | **Yes** | `first_marriage_only`, `open_to_divorced`, `open_to_widowed`, `open_to_divorced_widowed`, `any` |

### 11. Upload photos (completes signup)

`POST /api/profile/photos` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `photos` | File(s) (form-data) | **Yes** | One or more images (JPG, PNG, WEBP), 5 MB each, max 6 photos per user. The first becomes the main photo |

**Returns:** `photos`, `signupStep: "completed"`.

### 12. Skip photos ("Add photos later")

`POST /api/signup/photos/skip` · 🔐 Token

_No request fields._

**Returns:** `signupStep: "completed"`.

## D. Login

Every login returns `token` and `user`. Use `user.signupStep`: if it isn't `completed`, open that signup step instead of Home.

### 13. Login with email + password

`POST /api/auth/login` · 🔓 Public

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `email` | Text | **Yes** |  |
| `password` | Text | **Yes** |  |

`401` wrong credentials · `400` account has no password (use OTP or Forgot password) · `403` with `signupStep: "verify_phone"` if the phone was never verified (call Resend OTP, then Verify OTP).

### 14. Login with phone – send OTP

`POST /api/auth/login/otp/send` · 🔓 Public

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `phone` | Text | **Yes** | Registered Indian mobile |

`404` if no account uses this number.

### 15. Login with phone – verify OTP

`POST /api/auth/login/otp/verify` · 🔓 Public

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `phone` | Text | **Yes** |  |
| `otp` | Text | **Yes** | 4 digits |

## E. Forgot password

Choose Email or Phone → Send OTP → Verify OTP → set new password. Send **either** `email` or `phone`, and use the same one in steps 1 and 2.

### 16. Send OTP

`POST /api/auth/forgot-password` · 🔓 Public

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `email` | Text | Yes, or `phone` |  |
| `phone` | Text | Yes, or `email` |  |

**Returns:** `sentTo` (masked, e.g. `imXXXXXX@XXXX.com`), `otp: { expiresIn, resendAfter }`.

### 17. Verify OTP

`POST /api/auth/forgot-password/verify` · 🔓 Public

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `email` | Text | Yes, or `phone` |  |
| `phone` | Text | Yes, or `email` |  |
| `otp` | Text | **Yes** | 4 digits |

**Returns:** `resetToken` (valid 10 minutes).

### 18. Set new password

`POST /api/auth/reset-password` · 🔓 Public

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `resetToken` | Text | **Yes** | From the previous step |
| `newPassword` | Text | **Yes** | Min 6 chars |
| `confirmPassword` | Text | **Yes** | Must equal `newPassword` |

## F. After login

### 19. Register device for push notifications

`POST /api/settings/device-token` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `token` | Text | **Yes** | FCM device token |
| `platform` | Text | **Yes** | `android`, `ios`, `web` |

Call after every login and whenever FCM gives a new token.

### 20. Accept community guidelines

`POST /api/settings/community-guidelines/accept` · 🔐 Token

_No request fields._

**Returns:** `communityGuidelinesAcceptedAt`. The text itself comes from `GET /api/content/community_guidelines`.

## G. Home, matches and search

### 21. Home

`GET /api/home` · 🔐 Token

_No request fields._

**Returns:** `home: { greetingName, location, matchesCount, matchesCountText ("24 matches in Hyderabad today"), matches[10], newlyJoined[10], unreadNotifications }`. Lists contain match cards.

### 22. Matches list

`GET /api/matches` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `page` | Integer (query) | No | Default 1 |
| `limit` | Integer (query) | No | Default 20, max 50 |
| `sortBy` | Text (query) | No | `lastSeen` (default), `newest` |
| `location` | Text (query) | No | City, e.g. `Hyderabad` |

**Returns:** `total`, `page`, `pages`, `profiles` (match cards with `matchScore`, `isShortlisted`, `interestStatus`, `photo`).

### 23. Newly joined

`GET /api/matches/newly-joined` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `page` | Integer (query) | No | Default 1 |
| `limit` | Integer (query) | No | Default 20, max 50 |

### 24. Mutual interests

`GET /api/matches/mutual` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `page` | Integer (query) | No | Default 1 |
| `limit` | Integer (query) | No | Default 20, max 50 |

### 25. My shortlist (favourites)

`GET /api/shortlist` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `page` | Integer (query) | No | Default 1 |
| `limit` | Integer (query) | No | Default 20, max 50 |

**Returns:** `total`, `page`, `shortlist: [{ _id, createdAt, shortlistedUser: <match card> }]`.

### 26. Search / filters

`GET /api/search` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `sect` | Text (query) | No | `masterData.sects` value(s). `sunni` also matches all Sunni sub-options |
| `religiosity` | Text (query) | No | `masterData.religiosityLevels` value(s) |
| `familyValues` | Text (query) | No | `masterData.familyValues` value(s) |
| `ageMin` | Integer (query) | No |  |
| `ageMax` | Integer (query) | No |  |
| `heightMin` | Integer (query) | No | Inches |
| `heightMax` | Integer (query) | No | Inches |
| `education` | Text (query) | No | `masterData.secularEducation` value(s) |
| `maritalStatus` | Text (query) | No | `masterData.maritalStatuses` value(s) |
| `profession` | Text (query) | No | `masterData.professions` value(s) |
| `professionCategory` | Text (query) | No | Category name, e.g. `IT & Engineering` |
| `incomeMin` | Text (query) | No | `masterData.incomeRanges` value (range start) |
| `incomeMax` | Text (query) | No | `masterData.incomeRanges` value (range end) |
| `location` | Text (query) | No | City or area |
| `page` | Integer (query) | No | Default 1 |
| `limit` | Integer (query) | No | Default 20, max 50 |
| `sortBy` | Text (query) | No | `lastSeen` (default), `newest`, `age_asc`, `age_desc` |

Comma-separated values mean "any of", e.g. `religiosity=practising,moderate`. **Clear All** = call with no filters.

## H. Viewing a profile and acting on it

### 27. View profile

`GET /api/profile/:id` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `id / userId` | ID (path) | **Yes** | The other user's `_id` (from a match card) |

Contact is masked until both users accept an interest. `404` if the profile is hidden, deleted or has blocked you; `403` if you blocked them.

**Returns:** `profile` with `sections` (basic, bio, professional, family, religious, preferences, contact), `photos`, `photoAccess`, `canRequestPhotos`, `relationship` (interest / shortlist / photo-request status), `matchScore` ("You match X/N" + per-field `breakdown`), `restricted`.

### 28. Send interest

`POST /api/interests/:userId` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `id / userId` | ID (path) | **Yes** | The other user's `_id` (from a match card) |
| `message` | Text | No | Max 300 chars |

`400` if already sent, or if they already sent you one (`interestId` returned — accept that instead).

### 29. Shortlist / favourite

`POST /api/shortlist/:userId` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `id / userId` | ID (path) | **Yes** | The other user's `_id` (from a match card) |

### 30. Remove from shortlist

`DELETE /api/shortlist/:userId` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `id / userId` | ID (path) | **Yes** | The other user's `_id` (from a match card) |

### 31. Not Now (hide from my suggestions)

`POST /api/settings/hide/:userId` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `id / userId` | ID (path) | **Yes** | The other user's `_id` (from a match card) |

### 32. Undo Not Now

`DELETE /api/settings/hide/:userId` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `id / userId` | ID (path) | **Yes** | The other user's `_id` (from a match card) |

### 33. Request photos

`POST /api/photo-requests/:userId` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `id / userId` | ID (path) | **Yes** | The other user's `_id` (from a match card) |

Only when the profile has `canRequestPhotos: true`.

### 34. Share profile

`GET /api/profile/:id/share` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `id / userId` | ID (path) | **Yes** | The other user's `_id` (from a match card) |

**Returns:** `share: { profileCode, shareUrl, shareText }`.

### 35. Open shared profile

`GET /api/profile/code/:code` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `code` | Text (path) | **Yes** | Profile code, e.g. `RO482913` |

**Returns:** Same as View profile.

### 36. Block user

`POST /api/settings/block/:userId` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `id / userId` | ID (path) | **Yes** | The other user's `_id` (from a match card) |

### 37. Report user

`POST /api/settings/report/:userId` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `id / userId` | ID (path) | **Yes** | The other user's `_id` (from a match card) |
| `reason` | Text | **Yes** | `masterData.reportReasons` value |
| `details` | Text | **If** `reason` is `others` | Max 1000 chars |

## I. Requests screen

| Tab | Received (Accept / Decline) | Accepted (View Profile) |
|---|---|---|
| Gallery | Received photo requests, `status=pending` | Received photo requests, `status=accepted` |
| Requests | Received interests, `status=pending` | Received interests, `status=accepted` |
| Interests | Sent interests | Mutual interests (section G) |

### 38. Received interests

`GET /api/interests/received` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `status` | Text (query) | No | `pending`, `accepted`, `declined` |
| `page` | Integer (query) | No | Default 1 |
| `limit` | Integer (query) | No | Default 20, max 50 |

### 39. Sent interests

`GET /api/interests/sent` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `status` | Text (query) | No | `pending`, `accepted`, `declined` |
| `page` | Integer (query) | No | Default 1 |
| `limit` | Integer (query) | No | Default 20, max 50 |

### 40. Accept / decline interest

`PUT /api/interests/:interestId` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `interestId` | ID (path) | **Yes** |  |
| `status` | Text | **Yes** | `accepted`, `declined` |

### 41. Cancel sent interest

`DELETE /api/interests/:interestId` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `interestId` | ID (path) | **Yes** |  |

### 42. Received photo requests

`GET /api/photo-requests/received` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `status` | Text (query) | No | `pending`, `accepted`, `declined` |
| `page` | Integer (query) | No | Default 1 |
| `limit` | Integer (query) | No | Default 20, max 50 |

### 43. Sent photo requests

`GET /api/photo-requests/sent` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `status` | Text (query) | No | `pending`, `accepted`, `declined` |
| `page` | Integer (query) | No | Default 1 |
| `limit` | Integer (query) | No | Default 20, max 50 |

### 44. Accept / decline photo request

`PUT /api/photo-requests/:requestId` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `requestId` | ID (path) | **Yes** |  |
| `status` | Text | **Yes** | `accepted`, `declined` |

### 45. Cancel sent photo request

`DELETE /api/photo-requests/:requestId` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `requestId` | ID (path) | **Yes** |  |

## J. Notifications

### 46. Notifications list

`GET /api/settings/notifications` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `page` | Integer (query) | No | Default 1 |
| `limit` | Integer (query) | No | Default 20, max 50 |

**Returns:** `unreadCount`, `notifications`, and the same items split into `recent` (last 24 h) and `older`. Each item has `type`, `title`, `body`, `timeAgo`, `isRead`, `data`.

### 47. Unread count (badge)

`GET /api/settings/notifications/unread-count` · 🔐 Token

_No request fields._

### 48. Mark one as read

`PUT /api/settings/notifications/:id/read` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `id` | ID (path) | **Yes** | Notification `_id` |

### 49. Mark all as read

`PUT /api/settings/notifications/read-all` · 🔐 Token

_No request fields._

## K. My profile and Edit Profile

Each Edit Profile section has its own **Save**. Send only the fields shown on that screen.
"Required" below means the field **can't be cleared**: if you send it, it must have a valid value.

### 50. Get my full profile

`GET /api/profile/me` · 🔐 Token

_No request fields._

**Returns:** `myProfile` with `age`, `signupStep`, `isVerified`, `idProof`, `photos` and `sections` (basic, bio, professional, family, religious, preferences, contact — unmasked).

### 51. Edit bio

`PUT /api/profile/me/bio` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `bio` | Text | No | Max 1000 chars |

### 52. Edit basic info

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

### 53. Edit professional info

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

### 54. Edit family info

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

### 55. Edit religious info

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

### 56. Edit partner preferences

`PUT /api/profile/preferences` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `ageMin` | Integer | **Yes** | Min 18, Max 80 |
| `ageMax` | Integer | **Yes** | Min 18, Max 80. Must be ≥ `ageMin` |
| `heightMinInches` | Number | **Yes** | Inches, 47–84 (`masterData.heights`) |
| `heightMaxInches` | Number | **Yes** | Inches, 47–84 (`masterData.heights`). Must be ≥ `heightMinInches` |
| `weightCategories` | Array | **Yes** | One or more `masterData.weights` values, or `any` |
| `complexion` | Array | **Yes** | One or more of: `very_fair`, `fair`, `wheatish`, `slightly_dark`, `dark`, or `any` |
| `education` | Text | **Yes** | `masterData.secularEducation` value, or `any`. Minimum level |
| `religiousEducation` | Array | **Yes** | One or more of: `hifz`, `alimiyat`, `moderate_knowledge`, `self_taught`, `basic`, or `any` |
| `religiosity` | Text | **Yes** | `very_practising`, `practising`, `moderate`, `liberal`, or `any` |
| `prayerHabits` | Text | **Yes** | `five_times`, `mostly`, `sometimes`, `jummah_only`, `rarely`, or `any` |
| `sect` | Text | **Yes** | `masterData.sects` value, or `any` |
| `jamaatAffiliation` | Text | **Yes** | `barelvi`, `deobandi`, `ahl_e_hadith`, `hanafi`, `other`, or `any` |
| `hijabLevel` | Text | **Yes** for male users, No for female | `niqab`, `hijab_scarf`, `occasionally`, `no_hijab`, or `any` |
| `professionalOrientation` | Text | **Yes** | `career_oriented`, `balanced`, `home_maker`, or `any` |
| `maritalHistory` | Text | **Yes** | `first_marriage_only`, `open_to_divorced`, `open_to_widowed`, `open_to_divorced_widowed`, `any` |

### 57. Edit contact details

`PUT /api/profile/me/contact` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `email` | Text | Yes, or `phone` | Valid, not used by another account. Saved immediately |
| `phone` | Text | Yes, or `email` | New Indian mobile, not used by another account |

**Returns:** If the phone changed: `phoneVerificationRequired: true` and an OTP is sent to the **new** number. Call this again to resend.

### 58. Verify new phone number

`POST /api/profile/me/contact/verify-phone` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `otp` | Text | **Yes** | 4 digits sent to the new number |

## L. Photos management

### 59. Add photos

`POST /api/profile/photos` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `photos` | File(s) (form-data) | **Yes** | Images (JPG, PNG, WEBP), 5 MB each, max 6 photos per user |

### 60. Set main photo

`PUT /api/profile/photos/:photoId/main` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `photoId` | ID (path) | **Yes** |  |

### 61. Reorder photos

`PUT /api/profile/photos/order` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `photoIds` | Array | **Yes** | Every one of the user's photo ids, exactly once, in the new order |

### 62. Delete photo

`DELETE /api/profile/photos/:photoId` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `photoId` | ID (path) | **Yes** |  |

## M. Settings

### 63. Get settings

`GET /api/settings` · 🔐 Token

_No request fields._

**Returns:** `settings: { isProfileHidden, photoVisibility, whoCanViewProfile, appLanguage, locationPreference, notificationSettings, communityGuidelinesAccepted, communityGuidelinesAcceptedAt }`.

### 64. Update settings

`PUT /api/settings` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `photoVisibility` | Text | No | `everyone`, `on_request`, `after_match`, `blurred_until_approved`, `hidden` |
| `whoCanViewProfile` | Text | No | `everyone`, `matches_only`, `after_interest_accepted`, `basic_info_only` |
| `appLanguage` | Text | No | `masterData.appLanguages` value |
| `locationPreference` | Text | No | Max 100 chars, e.g. `Hyderabad` |
| `notificationSettings` | Object | No | `{ matches, interests, acceptedInterest, photoRequest, photoAccessAccepted }` — each `true`/`false`; send only the toggles that changed |

Send at least one field. Each settings screen can send just its own field.

### 65. Hide / unhide my profile

`PUT /api/settings/hide-profile` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `hidden` | Boolean | **Yes** | `true` hides, `false` unhides |

### 66. Blocked users list

`GET /api/settings/blocked` · 🔐 Token

_No request fields._

**Returns:** `blockedUsers: [{ _id, name, age, city }]`.

### 67. Unblock user

`DELETE /api/settings/block/:userId` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `id / userId` | ID (path) | **Yes** | The other user's `_id` (from a match card) |

### 68. Not Now list

`GET /api/settings/hidden` · 🔐 Token

_No request fields._

**Returns:** `hiddenUsers` (match cards).

## N. Support and static pages

### 69. Contact support

`POST /api/settings/support` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `firstName` | Text | **Yes** | Max 50 chars |
| `lastName` | Text | **Yes** | Max 50 chars |
| `email` | Text | **Yes** | Valid email |
| `phone` | Text | No | Max 20 chars |
| `message` | Text | **Yes** | Max 1000 chars |
| `privacyConsent` | Boolean | **Yes** | Must be `true` |

### 70. Static page

`GET /api/content/:key` · 🔓 Public

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `key` | Text (path) | **Yes** | `faq`, `community_guidelines`, `about_us`, `privacy_policy`, `terms` |

**Returns:** `content: { title, body, items: [{ question, answer }] }` (`items` is used for the FAQ). `404` until an admin creates the page.

## O. Account: password, logout, delete

### 71. Change password

`PUT /api/auth/change-password` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `currentPassword` | Text | **Yes**, unless the account has no password yet |  |
| `newPassword` | Text | **Yes** | Min 6 chars |
| `confirmPassword` | Text | **Yes** | Must equal `newPassword` |

### 72. Logout

`POST /api/auth/logout` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `deviceToken` | Text | No | This device's FCM token, so it stops getting push notifications |

### 73. Delete profile permanently

`DELETE /api/settings/profile` · 🔐 Token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `confirm` | Text | **Yes** | Exactly `DELETE` (after the user confirms in the app) |

Removes interests, shortlists, photo requests and notifications, hides the profile and logs the user out.

## P. Admin

Not used by the mobile app. All require an admin token.

### 74. ID proofs to review

`GET /api/admin/id-proofs` · 🛡 Admin token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `status` | Text (query) | No | `pending` (default), `approved`, `rejected` |
| `page` | Integer (query) | No | Default 1 |
| `limit` | Integer (query) | No | Default 20, max 50 |

### 75. Download ID proof file

`GET /api/admin/id-proofs/:userId/file` · 🛡 Admin token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `userId` | ID (path) | **Yes** |  |

### 76. Approve / reject ID proof

`PUT /api/admin/id-proofs/:userId` · 🛡 Admin token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `userId` | ID (path) | **Yes** |  |
| `status` | Text | **Yes** | `approved`, `rejected` |
| `rejectionReason` | Text | **If** `status` is `rejected` | Shown to the user |

### 77. Reports

`GET /api/admin/reports` · 🛡 Admin token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `status` | Text (query) | No | `open`, `reviewed`, `actioned`, `dismissed` |
| `page` | Integer (query) | No | Default 1 |
| `limit` | Integer (query) | No | Default 20, max 50 |

### 78. Update report

`PUT /api/admin/reports/:id` · 🛡 Admin token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `id` | ID (path) | **Yes** |  |
| `status` | Text | **Yes** | `open`, `reviewed`, `actioned`, `dismissed` |
| `adminNote` | Text | No |  |

### 79. Support tickets

`GET /api/admin/support-tickets` · 🛡 Admin token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `status` | Text (query) | No | `open`, `in_progress`, `closed` |
| `page` | Integer (query) | No | Default 1 |
| `limit` | Integer (query) | No | Default 20, max 50 |

### 80. Update support ticket

`PUT /api/admin/support-tickets/:id` · 🛡 Admin token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `id` | ID (path) | **Yes** |  |
| `status` | Text | **Yes** | `open`, `in_progress`, `closed` |
| `adminNote` | Text | No |  |

### 81. Create / update static page or app version

`PUT /api/admin/content/:key` · 🛡 Admin token

| Field | Type | Required | Allowed values / rules |
|---|---|---|---|
| `key` | Text (path) | **Yes** | `faq`, `community_guidelines`, `about_us`, `privacy_policy`, `terms`, `app_version` |
| `title` | Text | No |  |
| `body` | Text | No | HTML or markdown |
| `items` | Array | No | FAQ entries `[{ question, answer }]` |
| `data` | Object | **If** `key` is `app_version` | `{ android: { latestVersion, minSupportedVersion, storeUrl }, ios: { … } }` |

## Not listed above

These older endpoints still work for the current app version but are not needed by the redesigned flow:
`GET /api/auth/me`, `DELETE /api/auth/account` (deactivate), `PUT /api/profile/me` (single-form update), `GET /api/profile/me/viewers` (who viewed me), `GET /api/search/suggestions`, `PUT /api/settings/visibility`, and the `/api/messages` chat endpoints.
