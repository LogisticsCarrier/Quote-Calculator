# Quote-Calculator
Submit Quote
const express = require("express");
const Stripe = require("stripe");
const nodemailer = require("nodemailer");
const bodyParser = require("body-parser");
const cors = require("cors");
require("dotenv").config();

const app = express();
const stripe = Stripe(process.env.STRIPE_SECRET_KEY);

app.use(cors());
app.use(bodyParser.json());

// This tells server to load your frontend files
app.use(express.static("public"));

// EMAIL SETUP (hidden from users)
const mailer = nodemailer.createTransport({
    service: "gmail",
    auth: {
        user: process.env.EMAIL_USER,
        pass: process.env.EMAIL_PASS
    }
});

// MAIN BACKEND ENDPOINT
app.post("/create-quote", async (req, res) => {

    try {
        const { pickup, dropoff, weight, miles, total } = req.body;

        // Create Stripe payment session
        const session = await stripe.checkout.sessions.create({
            payment_method_types: ["card"],
            mode: "payment",
            line_items: [{
                price_data: {
                    currency: "usd",
                    product_data: {
                        name: "Delivery Quote",
                        description: `${pickup} → ${dropoff}`
                    },
                    unit_amount: Math.round(total * 100)
                },
                quantity: 1
            }],
            success_url: "http://localhost:3000/success",
            cancel_url: "http://localhost:3000/cancel"
        });

        // Send email
        await mailer.sendMail({
            from: process.env.EMAIL_USER,
            to: process.env.CUSTOMER_EMAIL,
            subject: "Your Quote",
            text: `Pickup: ${pickup}
Dropoff: ${dropoff}
Total: $${total}`
        });

        res.json({ url: session.url });

    } catch (err) {
        console.error(err);
        res.status(500).json({ error: "Server error" });
    }
});

// START SERVER
app.listen(3000, () => {
    console.log("Server running on http://localhost:3000");
});
