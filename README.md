// === backend/server.js ===
const express = require("express");
const mongoose = require("mongoose");
const cors = require("cors");
const http = require("http");
const socketIO = require("socket.io");
const cloudinary = require("cloudinary").v2;
const multer = require("multer");
const firebaseAdmin = require("firebase-admin");
const fs = require("fs");

require("dotenv").config();
const app = express();
const server = http.createServer(app);
const io = socketIO(server, { cors: { origin: "*" } });

app.use(cors());
app.use(express.json());

mongoose.connect(process.env.MONGODB_URI, {
  useNewUrlParser: true,
  useUnifiedTopology: true,
});

cloudinary.config({
  cloud_name: process.env.CLOUDINARY_CLOUD_NAME,
  api_key: process.env.CLOUDINARY_API_KEY,
  api_secret: process.env.CLOUDINARY_API_SECRET,
});

firebaseAdmin.initializeApp({
  credential: firebaseAdmin.credential.cert(
    JSON.parse(fs.readFileSync("firebase-service-account.json", "utf-8"))
  ),
});

const User = mongoose.model("User", new mongoose.Schema({
  username: String,
  interests: [String],
  location: String,
  video: String,
}));

const Message = mongoose.model("Message", new mongoose.Schema({
  from: String,
  to: String,
  text: String,
  timestamp: Date,
  read: Boolean,
}));

const upload = multer({ dest: "uploads/" });
app.post("/upload-video", upload.single("video"), async (req, res) => {
  const result = await cloudinary.uploader.upload(req.file.path, {
    resource_type: "video",
  });
  res.json({ url: result.secure_url });
});

app.get("/match/:userId", async (req, res) => {
  const currentUser = await User.findById(req.params.userId);
  const matches = await User.find({
    _id: { $ne: currentUser._id },
    interests: { $in: currentUser.interests },
    location: currentUser.location,
  });
  res.json(matches);
});

io.on("connection", (socket) => {
  socket.on("join", (userId) => socket.join(userId));

  socket.on("typing", ({ to }) => {
    io.to(to).emit("typing");
  });

  socket.on("message", async ({ from, to, text }) => {
    const message = new Message({ from, to, text, timestamp: new Date(), read: false });
    await message.save();
    io.to(to).emit("message", message);
  });

  socket.on("read", async ({ from, to }) => {
    await Message.updateMany({ from, to, read: false }, { read: true });
    io.to(from).emit("read-receipt", { from, to });
  });
});

app.get("/", (req, res) => res.send("Dating App Backend Running"));

const PORT = process.env.PORT || 5000;
server.listen(PORT, () => console.log(`Server running on ${PORT}`));

// === frontend/src/App.js ===
import React, { useEffect, useState } from "react";
import io from "socket.io-client";
import axios from "axios";

const socket = io("http://localhost:5000");

function App() {
  const [messages, setMessages] = useState([]);
  const [text, setText] = useState("");
  const [typing, setTyping] = useState(false);

  const sendMessage = () => {
    socket.emit("message", { from: "user1", to: "user2", text });
    setText("");
  };

  useEffect(() => {
    socket.emit("join", "user1");
    socket.on("message", (msg) => {
      setMessages((prev) => [...prev, msg]);
    });
    socket.on("typing", () => {
      setTyping(true);
      setTimeout(() => setTyping(false), 2000);
    });
  }, []);

  return (
    <div>
      <h1>Dating App Chat</h1>
      <div>{messages.map((m, i) => <p key={i}>{m.text}</p>)}</div>
      {typing && <p>User is typing...</p>}
      <input value={text} onChange={(e) => setText(e.target.value)} onKeyDown={() => socket.emit("typing", { to: "user2" })} />
      <button onClick={sendMessage}>Send</button>
    </div>
  );
}

export default App;

// === frontend/src/index.js ===
import React from "react";
import ReactDOM from "react-dom";
import App from "./App";

ReactDOM.render(<App />, document.getElementById("root"));

// === frontend/package.json ===
{
  "name": "dating-app-frontend",
  "version": "1.0.0",
  "dependencies": {
    "axios": "^0.21.1",
    "react": "^17.0.2",
    "react-dom": "^17.0.2",
    "socket.io-client": "^4.0.0"
  },
  "scripts": {
    "start": "react-scripts start",
    "build": "react-scripts build"
  }
}

// === Procfile ===
web: node backend/server.js

// === README.md ===
