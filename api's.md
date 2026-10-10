# A. App launch
GET     /api/master-data
GET     /api/content/app-version

# B. Sign up
POST    /api/auth/register
POST    /api/auth/verify-otp
POST    /api/auth/resend-otp

# C. Signup steps
GET     /api/signup/status
PUT     /api/signup/personal
POST    /api/profile/id-proof
PUT     /api/signup/social
PUT     /api/signup/preferences
POST    /api/profile/photos
POST    /api/signup/photos/skip

# D. Login
POST    /api/auth/login
POST    /api/auth/login/otp/send
POST    /api/auth/login/otp/verify

# E. Forgot password
POST    /api/auth/forgot-password
POST    /api/auth/forgot-password/verify
POST    /api/auth/reset-password

# F. After login
POST    /api/settings/device-token
POST    /api/settings/community-guidelines/accept

# G. Home, matches and search
GET     /api/home
GET     /api/matches
GET     /api/matches/newly-joined
GET     /api/matches/mutual
GET     /api/shortlist
GET     /api/search

# H. Viewing a profile and acting on it
GET     /api/profile/:id
POST    /api/interests/:userId
POST    /api/shortlist/:userId
DELETE  /api/shortlist/:userId
POST    /api/settings/hide/:userId
DELETE  /api/settings/hide/:userId
POST    /api/photo-requests/:userId
GET     /api/profile/:id/share
GET     /api/profile/code/:code
POST    /api/settings/block/:userId
POST    /api/settings/report/:userId

# I. Requests screen
GET     /api/interests/received
GET     /api/interests/sent
PUT     /api/interests/:interestId
DELETE  /api/interests/:interestId
GET     /api/photo-requests/received
GET     /api/photo-requests/sent
PUT     /api/photo-requests/:requestId
DELETE  /api/photo-requests/:requestId

# J. Notifications
GET     /api/settings/notifications
GET     /api/settings/notifications/unread-count
PUT     /api/settings/notifications/:id/read
PUT     /api/settings/notifications/read-all

# K. My profile and Edit Profile
GET     /api/profile/me
PUT     /api/profile/me/bio
PUT     /api/profile/me/basic
PUT     /api/profile/me/professional
PUT     /api/profile/me/family
PUT     /api/profile/me/religious
PUT     /api/profile/preferences
PUT     /api/profile/me/contact
POST    /api/profile/me/contact/verify-phone

# L. Photos management
POST    /api/profile/photos
PUT     /api/profile/photos/:photoId/main
PUT     /api/profile/photos/order
DELETE  /api/profile/photos/:photoId

# M. Settings
GET     /api/settings
PUT     /api/settings
PUT     /api/settings/hide-profile
GET     /api/settings/blocked
DELETE  /api/settings/block/:userId
GET     /api/settings/hidden

# N. Support and static pages
POST    /api/settings/support
GET     /api/content/:key

# O. Account: password, logout, delete
PUT     /api/auth/change-password
POST    /api/auth/logout
DELETE  /api/settings/profile

# P. Admin
GET     /api/admin/id-proofs
GET     /api/admin/id-proofs/:userId/file
PUT     /api/admin/id-proofs/:userId
GET     /api/admin/reports
PUT     /api/admin/reports/:id
GET     /api/admin/support-tickets
PUT     /api/admin/support-tickets/:id
PUT     /api/admin/content/:key
