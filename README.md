# Two-user-chatting-app-
// Server-side code (Node.js + Express + Socket.io) with Invite System, Chat History, and Message Deletion
const express = require("express");
const http = require("http");
const socketIo = require("socket.io");
const crypto = require("crypto");

const app = express();
const server = http.createServer(app);
const io = socketIo(server);

let users = {};
let inviteCodes = {}; // Stores invite codes and assigned users
let chatHistory = []; // Stores chat history

// Generate an invite code
app.get("/generate-invite", (req, res) => {
    const inviteCode = crypto.randomBytes(4).toString("hex");
    inviteCodes[inviteCode] = null; // No user assigned yet
    res.json({ inviteCode });
});

// Fetch chat history
app.get("/chat-history", (req, res) => {
    res.json(chatHistory);
});

io.on("connection", (socket) => {
    console.log("A user connected: " + socket.id);

    socket.on("register", (data) => {
        const { username, inviteCode } = data;
        
        if (!inviteCodes[inviteCode]) {
            socket.emit("error", "Invalid invite code");
            return;
        }
        
        if (inviteCodes[inviteCode] !== null) {
            socket.emit("error", "Invite code already used");
            return;
        }
        
        users[socket.id] = username;
        inviteCodes[inviteCode] = username; // Assign user to invite code
        io.emit("user-connected", username);
        socket.emit("chat-history", chatHistory); // Send chat history to user
    });

    socket.on("send-message", (data) => {
        const messageData = {
            id: crypto.randomUUID(), // Unique ID for each message
            sender: users[socket.id],
            message: data.message,
            status: "sent",
            timestamp: new Date().toLocaleString()
        };
        chatHistory.push(messageData);
        io.emit("receive-message", messageData);
    });

    socket.on("delete-message", (messageId) => {
        chatHistory = chatHistory.filter(msg => msg.id !== messageId);
        io.emit("message-deleted", messageId);
    });

    socket.on("message-read", (data) => {
        io.emit("update-status", { sender: data.sender, status: "read" });
    });

    socket.on("disconnect", () => {
        io.emit("user-disconnected", users[socket.id]);
        delete users[socket.id];
    });
});

server.listen(3000, () => {
    console.log("Server running on port 3000");
});
