# Google OAuth 2.0 Authentication with JWT (Node.js + Express)

This guide explains how to implement **Google Login** using **Passport.js**, **Google OAuth 2.0**, and **JWT Authentication** in a Node.js application.

---

# 📌 Features

- Google OAuth 2.0 Authentication
- JWT Token Generation
- Passport.js Integration
- Environment Variable Support
- Express.js Backend
- Secure Authentication Flow

---

# 📋 Prerequisites

Before getting started, ensure you have:

- Node.js (v18+ recommended)
- npm
- A Google Account
- A Google Cloud Project
- Basic knowledge of:
  - JavaScript
  - Node.js
  - Express.js
  - REST APIs

---

# 📁 Project Structure

```text
google-auth-jwt/
│
├── app.js
├── package.json
├── package-lock.json
├── .env
├── .gitignore
└── README.md
```

---

# Step 1 — Create Google OAuth Credentials

## 1. Open Google Cloud Console

Visit:

https://console.cloud.google.com/

Sign in with your Google account.

---

## 2. Create a New Project

- Click **Project**
- Select **New Project**
- Enter your project name
- Click **Create**

---

## 3. Configure OAuth Consent Screen

Navigate to:

APIs & Services → OAuth Consent Screen

Choose:

- External

Fill in:

- App Name
- User Support Email
- Developer Contact Email

Click:

Save and Continue

No additional scopes are required for basic authentication.

---

## 4. Enable Required APIs

Navigate to:

APIs & Services → Library

Enable:

- Google People API

(Some projects work without enabling it, but enabling is recommended.)

---

## 5. Create OAuth Client ID

Navigate to:

APIs & Services → Credentials

Click:

Create Credentials

Choose:

OAuth Client ID

Application Type:

```
Web Application
```

Authorized JavaScript Origins

```text
http://localhost:3000
```

Authorized Redirect URI

```text
http://localhost:3000/auth/google/callback
```

Click **Create**

Google will provide:

- Client ID
- Client Secret

Save both securely.

---

# Step 2 — Create Project

```bash
mkdir google-auth-jwt
cd google-auth-jwt
```

Initialize Node project

```bash
npm init -y
```

---

# Step 3 — Install Dependencies

```bash
npm install express passport passport-google-oauth20 jsonwebtoken dotenv
```

### Optional (Recommended)

```bash
npm install cors helmet morgan cookie-parser
```

### Development Dependency

```bash
npm install --save-dev nodemon
```

---

# Step 4 — Configure Environment Variables

Create a `.env` file.

```env
PORT=3000

GOOGLE_CLIENT_ID=your-google-client-id
GOOGLE_CLIENT_SECRET=your-google-client-secret

JWT_SECRET=your-super-secret-key
```

Generate a strong JWT secret:

```bash
node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"
```

---

# Step 5 — Ignore Sensitive Files

Create a `.gitignore`

```gitignore
node_modules
.env
```

---

# Step 6 — Create Express Application

Create:

```text
app.js
```

---

## Import Packages

```javascript
require("dotenv").config();

const express = require("express");
const passport = require("passport");
const jwt = require("jsonwebtoken");

const { Strategy: GoogleStrategy } = require("passport-google-oauth20");

const app = express();
```

---

## Initialize Passport

```javascript
app.use(passport.initialize());
```

---

## Configure Google Strategy

```javascript
passport.use(
  new GoogleStrategy(
    {
      clientID: process.env.GOOGLE_CLIENT_ID,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET,
      callbackURL: "/auth/google/callback",
    },
    (accessToken, refreshToken, profile, done) => {
      /*
        Normally you would:

        1. Search user in database
        2. Create user if not exists
        3. Return database user

        For demo:
      */

      return done(null, profile);
    }
  )
);
```

---

## Login Route

```javascript
app.get(
  "/auth/google",
  passport.authenticate("google", {
    scope: ["profile", "email"],
  })
);
```

---

## Callback Route

```javascript
app.get(
  "/auth/google/callback",
  passport.authenticate("google", {
    session: false,
  }),
  (req, res) => {
    const token = jwt.sign(
      {
        id: req.user.id,
        displayName: req.user.displayName,
      },
      process.env.JWT_SECRET,
      {
        expiresIn: "1h",
      }
    );

    res.json({
      success: true,
      token,
    });
  }
);
```

---

# Step 7 — Start Server

```javascript
const PORT = process.env.PORT || 3000;

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

---

# Step 8 — Run Application

Using Node

```bash
node app.js
```

Using Nodemon

```bash
npx nodemon app.js
```

---

# Authentication Flow

```text
Client
   │
   ▼
GET /auth/google
   │
   ▼
Google Login Page
   │
   ▼
User Grants Permission
   │
   ▼
Google Redirects
/auth/google/callback
   │
   ▼
Passport Verifies User
   │
   ▼
Generate JWT
   │
   ▼
Return Token
```

---

# Example Success Response

```json
{
  "success": true,
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI..."
}
```

---

# JWT Payload Example

```json
{
  "id": "123456789",
  "displayName": "John Doe",
  "iat": 1710000000,
  "exp": 1710003600
}
```

---

# Security Best Practices

- Never commit `.env` to GitHub.
- Use HTTPS in production.
- Store JWT secrets securely.
- Set a reasonable token expiration.
- Verify JWT on protected routes.
- Validate Google profile data before creating users.
- Use a database instead of returning the raw Google profile.

---

# Production Notes

In production, update your Google OAuth credentials:

Authorized JavaScript Origin

```text
https://yourdomain.com
```

Authorized Redirect URI

```text
https://yourdomain.com/auth/google/callback
```

---

# Recommended Improvements

Instead of returning the Google profile directly:

- Save users in MongoDB or PostgreSQL.
- Generate your own User ID.
- Store:
  - Google ID
  - Name
  - Email
  - Avatar
  - Created At
- Issue your own JWT after successful authentication.

---

# Tech Stack

- Node.js
- Express.js
- Passport.js
- Google OAuth 2.0
- JSON Web Token (JWT)
- dotenv

---

# Useful Commands

Install packages

```bash
npm install
```

Run project

```bash
node app.js
```

Run with Nodemon

```bash
npx nodemon app.js
```

Generate JWT Secret

```bash
node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"
```

---

# Resources

- https://developers.google.com/identity
- https://console.cloud.google.com/
- https://www.passportjs.org/
- https://jwt.io/
- https://expressjs.com/

---

# License

MIT License
