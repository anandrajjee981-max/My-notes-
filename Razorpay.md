# 💳 Razorpay Payment Integration with Express.js & React

A complete example of integrating **Razorpay Checkout** with a **Node.js + Express.js backend**, **MongoDB**, and a **React frontend**.

This project demonstrates:

* Razorpay order creation
* Razorpay Checkout integration
* Payment verification using signature validation
* Payment information storage in MongoDB
* React payment button
* Backend API integration
* Payment failure handling
* Environment variable configuration
* Webhook-based payment confirmation
* Refund handling
* Idempotency and duplicate-payment protection
* Basic production security considerations

---

## 📌 Tech Stack

### Frontend

* React.js
* Axios
* Razorpay Checkout.js

### Backend

* Node.js
* Express.js
* Razorpay Node.js SDK
* MongoDB
* Mongoose
* dotenv

### Payment Gateway

* Razorpay

---

## 📁 Suggested Project Structure

```text
razorpay-payment/
│
├── backend/
│   ├── models/
│   │   └── Payment.js
│   │
│   ├── routes/
│   │   └── paymentRoutes.js
│   │
│   ├── controllers/
│   │   └── paymentController.js
│   │
│   ├── middleware/
│   │   └── authMiddleware.js
│   │
│   ├── config/
│   │   └── razorpay.js
│   │
│   ├── .env
│   ├── index.js
│   └── package.json
│
└── frontend/
    ├── src/
    │   └── components/
    │       └── PaymentButton.jsx
    │
    ├── public/
    │   └── index.html
    │
    └── package.json
```

---

# 🚀 Prerequisites

Before starting, make sure you have:

1. Node.js installed
2. MongoDB installed or a MongoDB Atlas database
3. A Razorpay account
4. Razorpay API credentials
5. React.js environment for the frontend

---

# 1. Install Razorpay

Inside the backend project:

```bash
npm install razorpay
```

Install the other required packages:

```bash
npm install express mongoose dotenv cors axios
```

For development:

```bash
npm install -D nodemon
```

---

# 2. Razorpay Configuration

Create a Razorpay instance in your backend.

```js
require("dotenv").config();

const Razorpay = require("razorpay");

const razorpay = new Razorpay({
  key_id: process.env.RAZORPAY_KEY_ID,
  key_secret: process.env.RAZORPAY_KEY_SECRET,
});

module.exports = razorpay;
```

---

# 3. Environment Variables

Create a `.env` file:

```env
PORT=5000

MONGODB_URI=mongodb://127.0.0.1:27017/razorpay

RAZORPAY_KEY_ID=your_razorpay_key_id
RAZORPAY_KEY_SECRET=your_razorpay_key_secret
```

### ⚠️ Important

Never expose:

```text
RAZORPAY_KEY_SECRET
```

to the frontend.

The `RAZORPAY_KEY_ID` can be used by the frontend because Razorpay Checkout requires it, but the secret must remain on the backend.

Also add `.env` to `.gitignore`:

```gitignore
node_modules/
.env
```

---

# 4. MongoDB Connection

```js
const mongoose = require("mongoose");

mongoose
  .connect(process.env.MONGODB_URI)
  .then(() => console.log("MongoDB connected"))
  .catch((error) => console.error("MongoDB connection error:", error));
```

---

# 5. Payment Schema

Create:

```text
models/Payment.js
```

```js
const mongoose = require("mongoose");

const paymentSchema = new mongoose.Schema(
  {
    orderId: {
      type: String,
      required: true,
      unique: true,
    },

    paymentId: {
      type: String,
    },

    signature: {
      type: String,
    },

    amount: {
      type: Number,
      required: true,
    },

    currency: {
      type: String,
      required: true,
      default: "INR",
    },

    status: {
      type: String,
      enum: [
        "pending",
        "created",
        "completed",
        "failed",
        "refunded",
      ],
      default: "pending",
    },

    refundId: {
      type: String,
    },
  },
  {
    timestamps: true,
  }
);

module.exports = mongoose.model("Payment", paymentSchema);
```

---

# 6. Create Razorpay Order

Create an API endpoint:

```http
POST /api/payment/orders
```

Example:

```js
router.post("/api/payment/orders", async (req, res) => {
  try {
    const amount = 500;

    const options = {
      amount: amount * 100,
      currency: "INR",
      receipt: `receipt_${Date.now()}`,
    };

    const order = await razorpay.orders.create(options);

    await Payment.create({
      orderId: order.id,
      amount: order.amount,
      currency: order.currency,
      status: "created",
    });

    res.status(201).json(order);
  } catch (error) {
    console.error(error);

    res.status(500).json({
      message: "Unable to create Razorpay order",
    });
  }
});
```

### Why `amount * 100`?

Razorpay expects the amount in the smallest currency unit.

For INR:

```text
₹500 = 50000 paise
```

Therefore:

```js
amount: 500 * 100
```

---

# 7. Add Razorpay Checkout

For a React application, include Razorpay Checkout in:

```text
public/index.html
```

```html
<script src="https://checkout.razorpay.com/v1/checkout.js"></script>
```

Install Axios:

```bash
npm install axios
```

---

# 8. React Payment Button

```jsx
import React from "react";
import axios from "axios";

function PaymentButton() {
  const handlePayment = async () => {
    try {
      const { data: order } = await axios.post(
        "http://localhost:5000/api/payment/orders",
        {
          amount: 500,
        }
      );

      const options = {
        key: "YOUR_RAZORPAY_KEY_ID",

        amount: order.amount,

        currency: order.currency,

        name: "My Company",

        description: "Test Transaction",

        order_id: order.id,

        handler: async function (response) {
          try {
            await axios.post(
              "http://localhost:5000/api/payment/verify",
              {
                razorpayOrderId: response.razorpay_order_id,
                razorpayPaymentId: response.razorpay_payment_id,
                signature: response.razorpay_signature,
              }
            );

            alert("Payment successful!");
          } catch (error) {
            console.error(error);
            alert("Payment verification failed!");
          }
        },

        prefill: {
          name: "Test User",
          email: "test@example.com",
          contact: "9999999999",
        },

        theme: {
          color: "#3399cc",
        },
      };

      const razorpay = new window.Razorpay(options);

      razorpay.on("payment.failed", function (response) {
        console.error("Payment failed:", response.error);

        alert("Payment failed!");
      });

      razorpay.open();
    } catch (error) {
      console.error("Order creation failed:", error);
    }
  };

  return (
    <button onClick={handlePayment}>
      Pay Now
    </button>
  );
}

export default PaymentButton;
```

---

# 9. Verify Payment Signature

After successful checkout, Razorpay returns:

```text
razorpay_order_id
razorpay_payment_id
razorpay_signature
```

These values must be verified on the backend.

```http
POST /api/payment/verify
```

Example:

```js
const {
  validatePaymentVerification,
} = require("razorpay/dist/utils/razorpay-utils");

router.post("/api/payment/verify", async (req, res) => {
  try {
    const {
      razorpayOrderId,
      razorpayPaymentId,
      signature,
    } = req.body;

    const isValid = validatePaymentVerification(
      {
        order_id: razorpayOrderId,
        payment_id: razorpayPaymentId,
      },
      signature,
      process.env.RAZORPAY_KEY_SECRET
    );

    if (!isValid) {
      return res.status(400).json({
        message: "Invalid payment signature",
      });
    }

    const payment = await Payment.findOne({
      orderId: razorpayOrderId,
    });

    if (!payment) {
      return res.status(404).json({
        message: "Payment record not found",
      });
    }

    payment.paymentId = razorpayPaymentId;
    payment.signature = signature;
    payment.status = "completed";

    await payment.save();

    res.json({
      status: "success",
      message: "Payment verified successfully",
    });
  } catch (error) {
    console.error(error);

    res.status(500).json({
      message: "Payment verification failed",
    });
  }
});
```

---

# 🔐 10. Payment Verification Flow

The recommended flow is:

```text
React Frontend
      │
      │ Create Order
      ▼
Express Backend
      │
      │ Razorpay API
      ▼
Razorpay
      │
      │ Order ID
      ▼
Express Backend
      │
      ▼
React
      │
      │ Open Checkout
      ▼
Razorpay Checkout
      │
      │ Payment
      ▼
Razorpay
      │
      │ Payment ID + Signature
      ▼
React
      │
      │ Send verification data
      ▼
Express Backend
      │
      │ Verify signature
      ▼
MongoDB
      │
      ▼
Payment Completed
```

---

# 🆕 Additional Production Features

The basic integration above is enough for learning and testing. For a real application, several additional concepts should be considered.

## 11. Webhooks

Frontend verification alone should not be treated as the only source of payment status.

Razorpay Webhooks allow Razorpay to notify your backend about payment events.

Common events include:

```text
payment.captured
payment.failed
order.paid
refund.created
refund.processed
```

A webhook endpoint can look like:

```http
POST /api/payment/webhook
```

Your backend can update MongoDB when Razorpay sends a verified webhook event.

### Why Webhooks Matter?

A user may:

* close the browser after payment
* lose internet connection
* refresh the page
* never execute the frontend success callback

The backend can still receive the payment event from Razorpay.

---

# 12. Webhook Signature Verification

Webhook requests should be verified using the Razorpay webhook secret.

Do not blindly trust incoming webhook requests.

Conceptually:

```text
Razorpay
   │
   │ Webhook + Signature
   ▼
Express Backend
   │
   │ Verify Signature
   ▼
Process Event
   │
   ▼
MongoDB
```

This prevents unauthorized requests from pretending to be Razorpay events.

---

# 13. Payment Status Handling

A production application should not only have:

```text
pending
completed
```

A better lifecycle can be:

```text
created
   ↓
pending
   ↓
completed
```

or:

```text
created
   ↓
failed
```

and:

```text
completed
   ↓
refunded
```

This makes payment tracking easier.

---

# 14. Refund Support

A real application may also need refunds.

Razorpay provides APIs for creating refunds against captured payments.

Conceptually:

```text
Customer Payment
       ↓
Captured
       ↓
Refund Requested
       ↓
Refund Created
       ↓
Refund Processed
```

Store the refund ID and update the payment status accordingly.

Example database fields:

```js
refundId: String,
refundStatus: String,
```

---

# 15. Prevent Duplicate Payments

Your backend should protect against duplicate order creation and duplicate payment processing.

For example, the database can enforce:

```js
orderId: {
  type: String,
  required: true,
  unique: true
}
```

You should also check whether a payment has already been marked as completed before processing the same payment again.

---

# 16. Never Trust the Frontend Amount

This is extremely important.

Avoid blindly trusting:

```js
req.body.amount
```

for the actual amount of a product/order.

For example, a malicious client could send:

```json
{
  "amount": 1
}
```

instead of the actual product price.

Instead:

```text
Frontend
   ↓
Product / Cart ID
   ↓
Backend
   ↓
Database
   ↓
Calculate actual amount
   ↓
Create Razorpay Order
```

The backend should be the source of truth for the final payable amount.

---

# 17. Associate Payments With Users

For authenticated applications, the payment should ideally be associated with the user who created the order.

Example:

```js
userId: {
  type: mongoose.Schema.Types.ObjectId,
  ref: "User",
  required: true,
}
```

This allows you to query:

```text
Which payments belong to this user?
```

and:

```text
Which user made this payment?
```

---

# 18. Use Product / Order References

For an e-commerce or subscription application, the payment should not exist independently.

A better structure is:

```text
User
  │
  ▼
Application Order
  │
  ▼
Razorpay Order
  │
  ▼
Razorpay Payment
```

For example:

```js
orderId
razorpayOrderId
razorpayPaymentId
userId
amount
currency
status
```

This makes payment reconciliation much easier.

---

# 19. CORS Configuration

If React and Express are running on different ports:

```text
React     → localhost:5173
Express   → localhost:5000
```

configure CORS on the backend.

```js
const cors = require("cors");

app.use(
  cors({
    origin: "http://localhost:5173",
    credentials: true,
  })
);
```

For production, replace the localhost origin with your actual frontend domain.

---

# 20. Error Handling

Do not expose sensitive backend errors directly to users.

Instead of:

```js
res.status(500).send(error);
```

prefer:

```js
res.status(500).json({
  message: "Something went wrong while processing the payment",
});
```

Log detailed errors on the server.

---

# 21. Security Checklist

Before deploying the application:

* Never expose `RAZORPAY_KEY_SECRET`
* Never commit `.env`
* Validate payment signatures on the backend
* Validate webhook signatures
* Never trust the frontend amount
* Authenticate payment/order APIs where required
* Prevent duplicate payment processing
* Validate request bodies
* Configure CORS correctly
* Use HTTPS in production
* Keep Razorpay credentials in environment variables
* Store only the payment information actually required by your application

---

# 22. API Endpoints

| Method | Endpoint                | Purpose                         |
| ------ | ----------------------- | ------------------------------- |
| POST   | `/api/payment/orders`   | Create Razorpay order           |
| POST   | `/api/payment/verify`   | Verify payment signature        |
| POST   | `/api/payment/webhook`  | Receive Razorpay webhook events |
| POST   | `/api/payment/refund`   | Initiate refund                 |
| GET    | `/api/payment/:orderId` | Get payment information         |

The last three endpoints are production-oriented additions and require their corresponding implementation.

---

# 23. Important Razorpay IDs

During a payment flow, you may encounter several different IDs:

### Razorpay Order ID

Created when your backend creates an order.

```text
order_xxxxxxxxx
```

### Razorpay Payment ID

Generated after the customer completes payment.

```text
pay_xxxxxxxxx
```

### Razorpay Signature

Used to verify that the payment response came from Razorpay and has not been tampered with.

### Receipt

A merchant-defined reference associated with the Razorpay order.

---

# 24. Development vs Production

### Development

```text
React → localhost
Express → localhost
MongoDB → local/Atlas
Razorpay → Test Mode
```

### Production

```text
React → HTTPS Domain
        ↓
Express API → HTTPS
        ↓
MongoDB Atlas
        ↓
Razorpay Live Mode
```

Before going live, replace test credentials with the appropriate live credentials and thoroughly test the payment, failure, webhook, and refund flows.

---

# 🧪 Testing

Use Razorpay's test/sandbox environment while developing.

Test:

* Successful payment
* Failed payment
* Invalid signature
* Missing order
* Duplicate verification
* Webhook events
* Refund flow
* Network interruption
* User closing checkout
* Invalid amount
* Unauthorized API requests

Do not test your first implementation directly with real money.

---

# 🐛 Common Mistakes

### 1. Exposing the secret key

❌ Never put this in React:

```js
RAZORPAY_KEY_SECRET
```

### 2. Verifying payment on frontend

❌ Never rely on frontend verification alone.

Verification must happen on the backend.

### 3. Trusting frontend amount

❌ Don't use the frontend amount as the final source of truth.

### 4. Forgetting to save Razorpay Order ID

Always store the Razorpay order ID so the payment can be reconciled later.

### 5. Not handling failed payments

A payment can fail even after an order is successfully created.

### 6. Treating checkout success as final settlement

For production systems, webhook/event-based confirmation should also be incorporated into the payment lifecycle.

---

# 📚 Payment Lifecycle

```text
Create Application Order
          ↓
Calculate Amount on Backend
          ↓
Create Razorpay Order
          ↓
Save Razorpay Order ID
          ↓
Open Razorpay Checkout
          ↓
Customer Pays
          ↓
Razorpay Returns Payment Details
          ↓
Backend Signature Verification
          ↓
Webhook Confirmation
          ↓
Update Payment Status
          ↓
Fulfill Order / Subscription
```

---

# 🎯 Learning Outcomes

After implementing this project, you will understand:

* How payment gateways work
* Razorpay order creation
* Razorpay Checkout
* Payment IDs
* Order IDs
* Payment signatures
* Backend payment verification
* MongoDB payment records
* React payment integration
* Payment failure handling
* Webhooks
* Refund concepts
* Duplicate payment prevention
* Secure payment architecture
* Production payment considerations

---

# 📄 License

This project is intended for learning and development purposes. Refer to Razorpay's official documentation and terms before using the integration in a production application.
