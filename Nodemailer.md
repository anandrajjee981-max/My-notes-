Nodemailer
This guide provides step-by-step instructions for setting up and using Nodemailer in a Node.js application with OAuth2 authentication using ClientID and ClientSecret, along with generating a refresh token using the OAuth 2.0 Playground.

Prerequisites
Node.js installed on your machine.
A Google account to generate OAuth2 credentials (ClientID and ClientSecret).
Access to the Google API Console to create OAuth2 credentials.
Getting OAuth2 Credentials
Go to the Google API Console:

Navigate to the Google API Console.
Create a new project or select an existing one.
Enable Gmail API:

Go to the Library section.
Search for Gmail API and enable it.
Create OAuth2 Credentials:
then go to oauth consent screen

Go to the Credentials section.
Click on Create Credentials and choose OAuth 2.0 Client IDs.
Set the application type to Web application.
Under Authorized redirect URIs, add http://localhost and https://developers.google.com/oauthplayground (or your application’s URL).
After creating, you'll get your ClientID and ClientSecret.
then again google console pay jao apne project select kro -> credential -> web client 1 -> audience -> test user -> apna email doo

Generating the Refresh Token Using OAuth 2.0 Playground
gamil api khojo and first wala link ko click kreke authorise kro
Access OAuth 2.0 Playground:

Open the OAuth 2.0 Playground in your web browser.

Configure OAuth 2.0 Playground:

In the top-right corner, click on the gear icon (settings).
Under OAuth 2.0 endpoints, select Use your own OAuth credentials.
Enter your ClientID and ClientSecret obtained from the Google Cloud Console.
Set the Access type to Offline to obtain a refresh token.
Select Scopes:

In Step 1 on the left panel, select the appropriate scopes for your application. For Gmail, choose:
https://mail.google.com/
Authorize APIs:

Click on Authorize APIs. You’ll be redirected to a Google login page to authorize the application.
Exchange Authorization Code for Tokens:

After authorizing, you'll be redirected back to the OAuth Playground.
Click on Exchange authorization code for tokens. This will generate an access token and a refresh token.
Copy Refresh Token:

Your refresh token will appear under the response for Step 2. Copy it and use it in your .env file.
Installation
Initialize a Node.js Project:

npm init -y
Install Nodemailer:

npm install nodemailer
Install dotenv (for environment variables):

npm install dotenv
Configuration
1. Create a .env File
Create a .env file in the root of your project to securely store your OAuth2 credentials:

CLIENT_ID=your-client-id
CLIENT_SECRET=your-client-secret
REFRESH_TOKEN=your-refresh-token
EMAIL_USER=your-email@example.com
Replace the placeholders with your actual OAuth2 credentials and the refresh token obtained from the OAuth 2.0 Playground.

2. Set Up Nodemailer with OAuth2
Create a new file email.js and set up the Nodemailer transporter with OAuth2:

require('dotenv').config();
const nodemailer = require('nodemailer');

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
    console.error('Error connecting to email server:', error);
  } else {
    console.log('Email server is ready to send messages');
  }
});

module.exports = transporter;
3. Create a Function to Send Emails
In the same email.js file, add a function to send emails:

// Function to send email
const sendEmail = async (to, subject, text, html) => {
  try {
    const info = await transporter.sendMail({
      from: `"Your Name" <${process.env.EMAIL_USER}>`, // sender address
      to, // list of receivers
      subject, // Subject line
      text, // plain text body
      html, // html body
    });

    console.log('Message sent: %s', info.messageId);
    console.log('Preview URL: %s', nodemailer.getTestMessageUrl(info));
  } catch (error) {
    console.error('Error sending email:', error);
  }
};

module.exports = sendEmail;
4. Use the Email Function in Your Application
In your main application file (e.g., app.js), import the sendEmail function and use it to send emails:

const sendEmail = require('./email');

// Example usage
sendEmail(
  'recipient@example.com',
  'Test Email Subject',
  'This is a test email sent with Nodemailer using OAuth2.',
  '<p>This is a test email sent with <b>Nodemailer</b> using OAuth2.</p>'
);
Running the Application
To send an email, simply run your Node.js application:

node app.js
If everything is configured correctly, you should see a message in your console indicating that the email was sent successfully.

Troubleshooting
Invalid Credentials Error: Ensure that your ClientID, ClientSecret, and RefreshToken are correct and match the Google account associated with EMAIL_USER.
Email Not Sent: Check if the OAuth2 scopes include Gmail API access and if the RefreshToken has the necessary permissions.
References
Nodemailer Documentation
Google OAuth2 Documentation
OAuth 2.0 Playground
