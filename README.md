# GNG-Social-Media
const express = require("express");
const mongoose = require("mongoose");
const bcrypt = require("bcrypt");
const jwt = require("jsonwebtoken");
const cors = require("cors");

const app = express();
app.use(express.json());
app.use(cors());

// Replace with your MongoDB Atlas connection string
mongoose.connect("mongodb+srv://username:password@cluster.mongodb.net/mlmSystem?retryWrites=true&w=majority");

const UserSchema = new mongoose.Schema({
  name: String,
  email: String,
  password: String,
  referralCode: String,
  referredBy: String,
  points: { type: Number, default: 0 },
  wallet: { type: Number, default: 0 }
});

const User = mongoose.model("User", UserSchema);

// Register
app.post("/register", async (req, res) => {
  const { name, email, password, referredBy } = req.body;

  const hashed = await bcrypt.hash(password, 10);
  const referralCode = Math.random().toString(36).substring(2, 8);

  const user = new User({
    name,
    email,
    password: hashed,
    referralCode,
    referredBy
  });

  await user.save();

  // MLM Level 1 Commission
  if (referredBy) {
    const refUser = await User.findOne({ referralCode: referredBy });
    if (refUser) {
      refUser.wallet += 10; // Level 1 bonus
      await refUser.save();
    }
  }

  res.json({ message: "Registered Successfully" });
});

// Login
app.post("/login", async (req, res) => {
  const { email, password } = req.body;

  const user = await User.findOne({ email });
  if (!user) return res.status(400).json({ message: "User not found" });

  const match = await bcrypt.compare(password, user.password);
  if (!match) return res.status(400).json({ message: "Wrong password" });

  const token = jwt.sign({ id: user._id }, "secretKey");

  res.json({ token, referralCode: user.referralCode, wallet: user.wallet, points: user.points });
});

// Watch Ad
app.post("/watch-ad", async (req, res) => {
  const { email } = req.body;

  const user = await User.findOne({ email });
  if (!user) return res.status(400).json({ message: "User not found" });

  user.points += 5;
  user.wallet += 5;

  await user.save();

  res.json({ message: "Ad watched, points added", points: user.points, wallet: user.wallet });
});

app.listen(5000, () => console.log("Server running on port 5000"));
