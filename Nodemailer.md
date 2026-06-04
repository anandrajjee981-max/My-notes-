
---

# 🚀 Nodemailer Setup Guide with OAuth2 & Gmail

This guide provides step-by-step instructions for setting up and using **Nodemailer** in a Node.js application with **OAuth2 authentication** using `ClientID` and `ClientSecret`, along with generating a refresh token using the **OAuth 2.0 Playground**.

---

## 📋 Prerequisites

* **Node.js** installed on your machine.
* A **Google account** to generate OAuth2 credentials.
* Access to the **[Google API Console](https://console.cloud.google.com/)**.

---

## 🔐 1. Getting OAuth2 Credentials

### Step A: Create Project & Enable Gmail API

1. Go to the **Google API Console**.
2. Create a new project or select an existing one.
3. Navigate to the **Library** section.
4. Search for **Gmail API** and click **Enable**.

### Step B: Configure OAuth Consent Screen

1. Go to the **OAuth consent screen** option from the sidebar.
2. Configure it by setting up the application type (External/Internal).
3. **Important:** Under the **Test users** section, click **Add Users** and add your own Gmail address (the one you will use to send emails).
4. then again google console pay jao apne project select kro -> credential -> web client 1 -> audience -> test user -> apna email doo

### Step C: Create OAuth2 Client IDs

1. Go to the **Credentials** section.
2. Click on **+ Create Credentials** and choose **OAuth 2.0 Client IDs**.
3. Set the application type to **Web application**.
4. Under **Authorized redirect URIs**, add the following URLs:
* `http://localhost`
* `https://developers.google.com/oauthplayground`


5. Click **Create** and copy your **ClientID** and **ClientSecret**.

---

## 🔄 2. Generating the Refresh Token Using OAuth 2.0 Playground

1. Open the **[OAuth 2.0 Playground](https://developers.google.com/oauthplayground)**.
2. In the top-right corner, click on the ⚙️ **Gear Icon (Settings)**.
3. Check the box **"Use your own OAuth credentials"**.
4. Enter your **ClientID** and **ClientSecret** obtained from the Google Cloud Console.
5. Set the **Access type** to **Offline** (essential to get a refresh token).
6. On the left panel (**Step 1**), search/scroll down for **Gmail API v1** and select:
* `https://mail.google.com/`


7. Click on **Authorize APIs**. You’ll be redirected to the Google login page to authorize the application.
8. After authorization, you will return to the Playground (**Step 2**). Click on **Exchange authorization code for tokens**.
9. Your **Refresh Token** will appear in the JSON response block. Copy it for your configuration.

---

## 💻 3. Installation

Initialize a Node.js Project:

```bash
npm init -y

```

Install Nodemailer and Dotenv (for managing environment variables safely):

```bash
npm install nodemailer dotenv

```

---

## ⚙️ 4. Configuration

### Create a `.env` File

Create a `.env` file in the root directory of your project to securely store your credentials:

```env
CLIENT_ID=your-client-id-here
CLIENT_SECRET=your-client-secret-here
REFRESH_TOKEN=your-refresh-token-here
EMAIL_USER=your-email@example.com

```

> ⚠️ *Replace placeholders with your actual Google OAuth2 credentials.*

### Set Up Nodemailer Transporter (`email.js`)

Create a new file named `email.js` and add the following code:

```javascript
require('dotenv').config();
const nodemailer = require('nodemailer');

// Initialize the transporter with OAuth2 configuration
const transporter = nodemailer.createTransport({
    service: 'gmail',
    auth: {
        type: 'OAuth2',
        user: process.env.EMAIL_USER,
        clientId: process.env.CLIENT_ID,
        clientSecret: process.env.CLIENT_SECRET,
        refreshToken: process.env.REFRESH_TOKEN,
    },
});

// Verify the connection configuration
transporter.verify((error, success) => {
    if (error) {
        console.error('❌ Error connecting to email server:', error);
    } else {
        console.log('✅ Email server is ready to send messages');
    }
});

// Function to send email
const sendEmail = async (to, subject, text, html) => {
    try {
        const info = await transporter.sendMail({
            from: `"Your Name" <${process.env.EMAIL_USER}>`, // sender address
            to,                                            // list of receivers
            subject,                                       // Subject line
            text,                                          // plain text body
            html,                                          // html body
        });

        console.log('✉️ Message sent: %s', info.messageId);
    } catch (error) {
        console.error('❌ Error sending email:', error);
    }
};

module.exports = sendEmail;

```

### Use the Email Function (`app.js`)

In your main application file (e.g., `app.js`), import and execute the `sendEmail` function:

```javascript
const sendEmail = require('./email');

// Example usage
sendEmail(
    'recipient@example.com',
    'Test Email Subject',
    'This is a test email sent with Nodemailer using OAuth2.',
    '<p>This is a test email sent with <b>Nodemailer</b> using <b>OAuth2</b>.</p>'
);

```

---

## 🏃‍♂️ 5. Running the Application

To send your email, run the application in your terminal:

```bash
node app.js

```

If everything is configured correctly, you should see a success message in your console logs.

---

## 🛠️ Troubleshooting

* **`Invalid Credentials Error`**: Ensure that your `ClientID`, `ClientSecret`, and `RefreshToken` are completely correct and match the specific Google account associated with `EMAIL_USER`.
* **`Email Not Sent`**: Check if the OAuth2 scopes inside the Playground included the exact `https://mail.google.com/` access scope and that your Google Cloud Project status is active.

---

## 📚 References

* [Nodemailer Documentation](https://nodemailer.com/)
* [Google OAuth2 Documentation](https://developers.google.com/identity/protocols/oauth2)
* [OAuth 2.0 Playground](https://developers.google.com/oauthplayground)
