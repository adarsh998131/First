// 📁 File: server.js
const express = require("express");
const cors = require("cors");
const session = require("express-session");
const bodyParser = require("body-parser");
const stripe = require("stripe")(process.env.STRIPE_SECRET_KEY);
const twilio = require("twilio");
const nodemailer = require("nodemailer");
const db = require("./firebase");
const bcrypt = require("bcrypt");
const path = require("path");
require("dotenv").config();

const app = express();
const PORT = process.env.PORT || 3000;

app.use(cors());
app.use(bodyParser.json());
app.use(express.urlencoded({ extended: true }));
app.use(express.static("public"));

app.use(session({
  secret: process.env.SESSION_SECRET || "supersecret",
  resave: false,
  saveUninitialized: false
}));

const adminUser = {
  username: "admin",
  passwordHash: bcrypt.hashSync("password123", 10)
};

const twilioClient = twilio(process.env.TWILIO_SID, process.env.TWILIO_AUTH);
const transporter = nodemailer.createTransport({
  service: "gmail",
  auth: {
    user: process.env.EMAIL_ID,
    pass: process.env.EMAIL_PASS,
  },
});

// Serve pages
app.get("/admin.html", (req, res) => {
  if (req.session.admin) res.sendFile(path.join(__dirname, "/public/admin.html"));
  else res.redirect("/login.html");
});

app.get("/login.html", (req, res) => {
  res.sendFile(path.join(__dirname, "/public/login.html"));
});

// Admin login
app.post("/admin/login", (req, res) => {
  const { username, password } = req.body;
  if (username === adminUser.username && bcrypt.compareSync(password, adminUser.passwordHash)) {
    req.session.admin = true;
    res.json({ success: true });
  } else {
    res.json({ success: false });
  }
});

app.get("/admin/logout", (req, res) => {
  req.session.destroy();
  res.redirect("/login.html");
});

// Order form submission
app.post("/create-checkout-session", async (req, res) => {
  const { name, phone, address, pickupDate, service } = req.body;

  try {
    const newOrder = await db.collection("orders").add({
      name, phone, address, pickupDate, service,
      status: "Pending",
      timestamp: new Date()
    });

    await twilioClient.messages.create({
      body: `Hi ${name}, your laundry service (${service}) is scheduled for pickup on ${pickupDate}.`,
      from: process.env.TWILIO_NUMBER,
      to: `whatsapp:+91${phone}`
    });

    await transporter.sendMail({
      from: `FastClean <${process.env.EMAIL_ID}>`,
      to: process.env.EMAIL_ID, // or user's email if collected
      subject: "Laundry Pickup Confirmed",
      text: `Hi ${name}, we have scheduled your ${service} on ${pickupDate} at ${address}.`
    });

    const session = await stripe.checkout.sessions.create({
      payment_method_types: ["card"],
      mode: "payment",
      line_items: [
        {
          price_data: {
            currency: "inr",
            product_data: { name: `${service} - Laundry Service` },
            unit_amount: 10000,
          },
          quantity: 1,
        },
      ],
      success_url: "https://example.com/success",
      cancel_url: "https://example.com/cancel",
    });

    res.json({ url: session.url });
  } catch (err) {
    console.error(err);
    res.status(500).send("Error processing request");
  }
});

app.get("/admin/orders", async (req, res) => {
  if (!req.session.admin) return res.status(401).json({ error: "Unauthorized" });
  const snapshot = await db.collection("orders").orderBy("timestamp", "desc").get();
  const orders = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
  res.json(orders);
});

app.post("/admin/update-status/:id", async (req, res) => {
  if (!req.session.admin) return res.status(403).send("Unauthorized");
  const { id } = req.params;
  const { status } = req.body;

  try {
    await db.collection("orders").doc(id).update({ status });
    res.sendStatus(200);
  } catch (err) {
    res.status(500).send("Error updating status");
  }
});

app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
