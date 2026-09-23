# Socket.io Complete Guide — Installation se WhatsApp-like Chat App tak (Full Detailed)

> MERN Stack Developer ke liye — crisp, cut-to-cut, koi concept loose nahi.

---

## 1. Fundamentals — Pehle Concept Clear Karo

### 1.1 Socket.io Kya Hai
- Real-time, bidirectional, event-based communication library.
- Do parts: `socket.io` (server) aur `socket.io-client` (browser/mobile).
- Har connected client ek unique `socket.id` leke aata hai.

### 1.2 WebSocket vs Socket.io vs SSE — Difference Clear Karo
| | WebSocket (raw) | Socket.io | Server-Sent Events (SSE) |
|---|---|---|---|
| Direction | Bidirectional | Bidirectional | Server → Client only |
| Fallback | Nahi hota | HTTP long-polling pe auto-fallback | N/A |
| Reconnection | Manual likhna padta hai | Built-in auto-reconnect | Built-in |
| Rooms/Namespaces | Nahi | Built-in | Nahi |
| Overhead | Kam | Thoda zyada (protocol layer) | Kam |

**Socket.io raw WebSocket ka wrapper nahi hai** — apna protocol (Engine.io) use karta hai jo WebSocket + polling dono handle karta hai. Isliye plain `ws` library ka client Socket.io server se directly connect nahi ho sakta.

### 1.3 Engine.io — Neeche Ka Layer
- Socket.io do layers pe bana hai: **Engine.io** (transport: polling/websocket handling, heartbeat) aur **Socket.io** (events, rooms, namespaces, acks — upar ka layer).
- Isse samajhna important hai kyunki connection-level tuning (timeouts, transports) Engine.io options se hota hai.

---

## 2. Installation

### Server side
```bash
npm install express socket.io
```

### Client side
```bash
npm install socket.io-client
```

**Version compatibility table:**
| socket.io (server) | socket.io-client |
|---|---|
| v4.x | v4.x |
| v3.x | v3.x (v2.x client bhi partial support) |

Mismatch hone par handshake silently fail ho sakta hai ya `connect_error` aayega — pehla debugging step hamesha version check karo.

---

## 3. Basic Server Setup — Full Options

```js
const express = require("express");
const http = require("http");
const { Server } = require("socket.io");

const app = express();
const server = http.createServer(app);

const io = new Server(server, {
  cors: {
    origin: "http://localhost:5173",
    methods: ["GET", "POST"],
    credentials: true,
  },
  transports: ["websocket", "polling"], // order matters — websocket priority
  pingTimeout: 20000,   // itne time tak pong na aaye toh disconnect maan lo
  pingInterval: 25000,  // kitni der mein ping bhejna hai
  maxHttpBufferSize: 1e6, // 1MB — max message size (default), DOS se bachne ke liye
  connectionStateRecovery: {
    maxDisconnectionDuration: 2 * 60 * 1000, // temp disconnect pe state recover
  },
});

io.on("connection", (socket) => {
  console.log("New client connected:", socket.id);
  socket.on("disconnect", (reason) => {
    console.log("Disconnected:", socket.id, reason);
  });
});

server.listen(5000, () => console.log("Server running on 5000"));
```

**`connectionStateRecovery`** (v4.6+) — agar client thodi der ke liye disconnect hoke reconnect ho, toh missed events/rooms automatically restore ho jate hain. Chat apps ke liye bahut useful (network blip pe messages miss nahi hote).

**`app.listen()` galat hai** — hamesha `server.listen()` use karo, kyunki `io` `server` object pe attach hota hai.

---

## 4. Basic Client Setup — Full Options

```js
import { io } from "socket.io-client";

export const socket = io("http://localhost:5000", {
  autoConnect: false,
  withCredentials: true,
  reconnection: true,
  reconnectionAttempts: Infinity,
  reconnectionDelay: 1000,       // pehla retry 1s baad
  reconnectionDelayMax: 5000,    // max delay between retries (exponential backoff)
  timeout: 20000,                // connection attempt timeout
  transports: ["websocket", "polling"],
});
```

```jsx
useEffect(() => {
  socket.connect();

  socket.on("connect", () => console.log("Connected:", socket.id));
  socket.on("connect_error", (err) => console.log("Connection failed:", err.message));
  socket.on("disconnect", (reason) => {
    console.log("Disconnected:", reason);
    // reason === "io server disconnect" → server ne forcefully disconnect kiya,
    // manually reconnect karna padega (auto-reconnect nahi chalega)
    if (reason === "io server disconnect") socket.connect();
  });

  return () => {
    socket.off("connect");
    socket.off("connect_error");
    socket.off("disconnect");
    socket.disconnect();
  };
}, []);
```

**Rule:** Har `socket.on()` ke saath specific `socket.off()` cleanup karo — generic `socket.disconnect()` sirf connection band karta hai, listeners memory mein reference rakh sakte hain agar socket instance reuse ho raha ho.

---

## 5. Core Event Model

```js
socket.emit("send_message", { text: "hello" });
socket.on("receive_message", (data) => console.log(data));
```

### Reserved/built-in events (khud define mat karo):
| Event | Side | Kab fire hota hai |
|---|---|---|
| `connect` | Client | Connect hone par |
| `disconnect` | Both | Disconnect hone par |
| `connect_error` | Client | Connection fail |
| `connecting` | Client | Reconnect attempt ho raha ho |
| `reconnect` | Client | Successful reconnect |
| `reconnect_attempt` | Client | Har retry attempt pe |
| `reconnect_failed` | Client | Saare attempts fail |
| `error` | Both | Generic error |

---

## 6. Acknowledgements — Server Se Confirmation Lena

Har emit ka reply chahiye ho (REST ke response jaisa) toh callback pass karo:

```js
// Client
socket.emit("send_message", { text: "hi" }, (response) => {
  console.log("Server confirmed:", response.status);
});

// Server
socket.on("send_message", (data, callback) => {
  // process...
  callback({ status: "ok", messageId: savedMsg._id });
});
```

### Timeout ke saath (v4.4+)
```js
socket.timeout(5000).emit("send_message", data, (err, response) => {
  if (err) {
    // server ne 5s mein respond nahi kiya
  } else {
    console.log(response);
  }
});
```

**Use case:** "message sent" confirmation, ya koi bhi action jahan client ko pakka jaanna ho ki server ne process kar liya — retry logic isi pe based hota hai.

---

## 7. Volatile Events aur Binary Data

### Volatile emit
```js
socket.volatile.emit("cursor_position", { x, y });
```
Agar client abhi connected nahi hai toh ye event **drop** ho jayega (queue nahi hoga). Non-critical, high-frequency data ke liye use karo — jaise live cursor tracking, typing indicator ka raw movement. Chat messages ke liye kabhi mat use karo (loss acceptable nahi hai wahan).

### Binary data
Socket.io binary data (Buffer, ArrayBuffer, Blob) ko directly support karta hai:
```js
socket.emit("upload_chunk", fileBuffer);
```
Lekin bade files ke liye ye best practice nahi — chunking manual karna padega aur memory pressure badhta hai. Isliye media messages REST upload (S3/Cloudinary) se bhejna better hai (Section 15.8 dekho).

---

## 8. Broadcasting Patterns — Sab Combinations

```js
socket.emit("event", data);              // sirf isi socket ko
socket.broadcast.emit("event", data);    // is socket ko chhodkar sabko
io.emit("event", data);                  // literally sabko
io.to(roomId).emit("event", data);       // ek room ko (sender included agar room mein hai)
socket.to(roomId).emit("event", data);   // room mein sabko, sender ko chhodkar
io.to(room1).to(room2).emit("event");    // multiple rooms ko ek saath
io.except(roomId).emit("event", data);   // is room ko chhodkar sabko
io.in(roomId).fetchSockets();            // room ke saare sockets ka array (async)
```

Ye poora table yaad rakho — real chat app logic isi ke combinations se banta hai.

---

## 9. Rooms — Deep Dive

```js
socket.on("join_room", (roomId) => {
  socket.join(roomId);
});

socket.on("leave_room", (roomId) => {
  socket.leave(roomId);
});
```

### Auto room
Har socket apne `socket.id` naam ka ek room bhi auto-join karta hai — isse individual targeting hoti hai (`io.to(socket.id).emit()`).

### Room membership check
```js
const room = io.sockets.adapter.rooms.get(roomId);
const size = room ? room.size : 0; // kitne clients room mein hain
```

### Disconnect pe auto-cleanup
Socket disconnect hote hi wo automatically saari rooms se leave ho jata hai — manual cleanup nahi chahiye.

**1-on-1 chat room ID pattern:** dono userIds sort karke concatenate karo (`[id1, id2].sort().join("_")`) — taaki dono directions se same room resolve ho.

---

## 10. Namespaces

```js
const chatNamespace = io.of("/chat");
const notifNamespace = io.of("/notifications");

chatNamespace.use((socket, next) => {
  // is namespace ke liye alag middleware
  next();
});

chatNamespace.on("connection", (socket) => { ... });
```

Client side:
```js
const chatSocket = io("http://localhost:5000/chat");
```

**Rooms vs Namespaces:**
- Namespace = app-level separation, alag connection pool, alag middleware chain possible.
- Room = namespace ke andar user-level grouping.

Chote apps mein sirf rooms kaafi hain; bade apps (chat + notifications + admin-panel alag concerns) mein namespaces se code organize hota hai.

---

## 11. Middleware — Connection Aur Per-Event Dono

### Connection-level middleware (sabse common — auth ke liye)
```js
io.use((socket, next) => {
  const token = socket.handshake.auth.token;
  if (!token) return next(new Error("Authentication error"));

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    socket.userId = decoded.id;
    next();
  } catch (err) {
    next(new Error("Invalid token"));
  }
});
```

Client:
```js
const socket = io("http://localhost:5000", {
  auth: { token: localStorage.getItem("token") },
});
```

### Multiple middleware chain karna
```js
io.use(authMiddleware);
io.use(rateLimitMiddleware);
io.use(loadUserProfileMiddleware);
```
Order matters — sequentially chalte hain, koi bhi `next(new Error())` bulaye toh chain wahin ruk jati hai aur `connect_error` client pe fire hota hai.

### Per-packet middleware (v4.x)
```js
socket.use(([event, ...args], next) => {
  console.log("Incoming event:", event);
  next();
});
```
Har incoming event pe validation/logging ke liye — jaise Express ka route-level middleware.

**Critical practice:** Client se `userId` seedha kabhi trust mat karo. JWT se decode karke `socket.userId` set karo, baaki logic usi se chalao.

---

## 12. Error Handling Patterns

```js
socket.on("send_message", async (data, callback) => {
  try {
    const message = await Message.create(data);
    callback({ status: "ok", message });
  } catch (err) {
    callback({ status: "error", message: err.message });
    // ya:
    socket.emit("error_event", { context: "send_message", error: err.message });
  }
});
```

- Har async handler ko try/catch mein wrap karo — warna unhandled promise rejection server crash kar sakta hai.
- Client pe generic `error` listener zaroor rakho:
```js
socket.on("error", (err) => {
  console.error("Socket error:", err);
});
```
- Middleware se aane wala error `connect_error` event se catch hota hai, event-handler ka error alag se emit karna padta hai (auto-propagate nahi hota).

---

## 13. Disconnecting Aur Managing Sockets Manually

```js
socket.disconnect(true); // true = underlying connection bhi close karo (close vs socket-only)

// Kisi specific user ko force-disconnect karna (ban/logout scenario)
const targetSocketId = onlineUsers.get(userId);
if (targetSocketId) {
  io.sockets.sockets.get(targetSocketId)?.disconnect(true);
}

// Saare sockets ek room ke fetch karke kuch karna
const sockets = await io.in(roomId).fetchSockets();
sockets.forEach((s) => console.log(s.id, s.rooms));
```

**Use case:** Admin panel se user ko force logout karna, ban lagana, ya multi-device session limit enforce karna.

---

## 14. Scaling — Multiple Server Instances

Single server pe socket.io state in-memory hota hai. Horizontal scaling (PM2 cluster, multiple containers, load balancer) mein ek instance ka socket dusre instance ko nahi dikhta.

### Redis Adapter (mandatory for production scaling)
```bash
npm install @socket.io/redis-adapter redis
```
```js
const { createAdapter } = require("@socket.io/redis-adapter");
const { createClient } = require("redis");

const pubClient = createClient({ url: "redis://localhost:6379" });
const subClient = pubClient.duplicate();

await Promise.all([pubClient.connect(), subClient.connect()]);
io.adapter(createAdapter(pubClient, subClient));
```

Sab instances shared pub/sub layer pe events broadcast karte hain — isse `io.to(room).emit()` sabhi instances ke connected clients tak pahunchta hai, chahe wo kisi bhi instance se connected ho.

### Nginx / Load Balancer — Sticky Sessions Zaroori
Agar polling fallback allow hai, toh same client ke saare HTTP requests **same server instance** pe jaane chahiye (sticky session / IP hash), warna handshake fail hoga.

```nginx
upstream socket_nodes {
  ip_hash; # sticky session
  server 127.0.0.1:5000;
  server 127.0.0.1:5001;
}

location /socket.io/ {
  proxy_pass http://socket_nodes;
  proxy_http_version 1.1;
  proxy_set_header Upgrade $http_upgrade;
  proxy_set_header Connection "upgrade";
}
```

Agar sirf `transports: ["websocket"]` force kar do (polling disable), toh sticky session ki zaroorat kam ho jati hai — lekin fir purane browsers/restrictive networks (corporate proxies) WebSocket block kar sakte hain.

---

## 15. Building a WhatsApp-Like Chat App — Full Architecture

### 15.1 Data Models (MongoDB)

```js
// User Schema
{
  _id, username, email,
  isOnline: Boolean,
  lastSeen: Date,
  blockedUsers: [userId],
}

// Conversation Schema
{
  _id,
  participants: [userId1, userId2],
  isGroup: Boolean,
  groupName: String,
  groupAdmins: [userId],
  lastMessage: { text, sender, timestamp },
}

// Message Schema
{
  _id, conversationId, sender: userId,
  text: String,
  mediaUrl: String,
  replyTo: messageId,      // reply feature
  reactions: [{ userId, emoji }],
  status: "sent" | "delivered" | "read",
  deletedFor: [userId],    // "delete for me"
  isDeletedForEveryone: Boolean,
  editedAt: Date,
  createdAt: Date,
}
```

### 15.2 Connection + Online Status Tracking

```js
const onlineUsers = new Map(); // userId -> socketId

io.on("connection", (socket) => {
  const userId = socket.userId;

  onlineUsers.set(userId, socket.id);
  io.emit("user_online", userId);

  socket.on("disconnect", async () => {
    onlineUsers.delete(userId);
    await User.findByIdAndUpdate(userId, { isOnline: false, lastSeen: new Date() });
    io.emit("user_offline", { userId, lastSeen: new Date() });
  });
});
```

**Multi-instance note:** `onlineUsers` Map single-instance ke liye theek hai. Scaling mein ye Redis Hash mein store karo (`HSET online_users userId socketId`) taaki saare instances access kar sakein.

**Multi-device consideration:** Ek user do devices se login kar sakta hai — isliye `onlineUsers` ko `Map<userId, Set<socketId>>` banao, single socketId nahi. Disconnect pe check karo ki Set khaali hua ya nahi, tabhi `user_offline` emit karo.

### 15.3 Joining Conversation Rooms

```js
socket.on("join_conversation", (conversationId) => {
  socket.join(conversationId);
});
```

### 15.4 Sending a Message — Full Flow

```js
socket.on("send_message", async ({ conversationId, senderId, text, replyTo }, callback) => {
  try {
    const message = await Message.create({
      conversationId, sender: senderId, text, replyTo, status: "sent",
    });

    await Conversation.findByIdAndUpdate(conversationId, {
      lastMessage: { text, sender: senderId, timestamp: message.createdAt },
    });

    socket.to(conversationId).emit("receive_message", message);
    callback({ status: "ok", message }); // ack — client optimistic UI replace karega
  } catch (err) {
    callback({ status: "error", error: err.message });
  }
});
```

**Golden rule:** DB write pehle, socket emit baad mein. Sirf socket state pe app design mat karo — refresh pe data loss hoga.

### 15.5 Delivered / Read Receipts

```js
socket.on("message_delivered", async ({ messageId, conversationId }) => {
  await Message.findByIdAndUpdate(messageId, { status: "delivered" });
  io.to(conversationId).emit("status_update", { messageId, status: "delivered" });
});

socket.on("message_read", async ({ messageId, conversationId }) => {
  await Message.findByIdAndUpdate(messageId, { status: "read" });
  io.to(conversationId).emit("status_update", { messageId, status: "read" });
});
```

Bulk read (jab user pura chat kholta hai):
```js
socket.on("mark_all_read", async ({ conversationId, userId }) => {
  await Message.updateMany(
    { conversationId, sender: { $ne: userId }, status: { $ne: "read" } },
    { status: "read" }
  );
  io.to(conversationId).emit("conversation_read", { conversationId, readBy: userId });
});
```

### 15.6 Typing Indicator (Debounced)

```js
socket.on("typing", ({ conversationId, userId }) => {
  socket.to(conversationId).emit("user_typing", { userId });
});

socket.on("stop_typing", ({ conversationId, userId }) => {
  socket.to(conversationId).emit("user_stop_typing", { userId });
});
```

Frontend debounce (React):
```js
let typingTimeout;
const handleInputChange = (text) => {
  socket.emit("typing", { conversationId, userId });
  clearTimeout(typingTimeout);
  typingTimeout = setTimeout(() => {
    socket.emit("stop_typing", { conversationId, userId });
  }, 1500);
};
```

### 15.7 Group Chat — Full Feature Set

```js
socket.on("add_participant", async ({ conversationId, newUserId, adminId }) => {
  const convo = await Conversation.findById(conversationId);
  if (!convo.groupAdmins.includes(adminId)) return; // permission check
  convo.participants.push(newUserId);
  await convo.save();
  io.to(conversationId).emit("participant_added", { newUserId });
});

socket.on("remove_participant", async ({ conversationId, targetUserId, adminId }) => {
  const convo = await Conversation.findById(conversationId);
  if (!convo.groupAdmins.includes(adminId)) return;
  convo.participants = convo.participants.filter(id => id.toString() !== targetUserId);
  await convo.save();
  io.to(conversationId).emit("participant_removed", { targetUserId });
});

socket.on("group_name_updated", async ({ conversationId, newName }) => {
  await Conversation.findByIdAndUpdate(conversationId, { groupName: newName });
  io.to(conversationId).emit("group_name_changed", { newName });
});
```

**Permission model:** Sab destructive/admin actions server-side check karo (`groupAdmins.includes(adminId)`) — client se aayi permission ko kabhi trust mat karo.

### 15.8 Media Messages (Images/Files/Voice Notes)

- Socket.io se raw file bhejna avoid karo — payload size aur memory issues.
- Flow: Client → REST API (multipart upload to S3/Cloudinary/ImageKit) → URL milta hai → us URL ko normal `send_message` event ke `mediaUrl` field mein bhejo.
- Voice notes: record karke Blob banao → REST upload → URL socket se bhejo, same pattern.
- Upload progress ke liye REST call (axios `onUploadProgress`) use karo, socket ka isme role nahi hai.

### 15.9 Message Edit, Delete, Reactions, Forward

```js
socket.on("edit_message", async ({ messageId, newText, conversationId }) => {
  await Message.findByIdAndUpdate(messageId, { text: newText, editedAt: new Date() });
  io.to(conversationId).emit("message_edited", { messageId, newText });
});

socket.on("delete_message", async ({ messageId, conversationId, forEveryone, userId }) => {
  if (forEveryone) {
    await Message.findByIdAndUpdate(messageId, { isDeletedForEveryone: true, text: "" });
    io.to(conversationId).emit("message_deleted_everyone", { messageId });
  } else {
    await Message.findByIdAndUpdate(messageId, { $addToSet: { deletedFor: userId } });
    socket.emit("message_deleted_me", { messageId }); // sirf khud ko dikhna band
  }
});

socket.on("react_message", async ({ messageId, userId, emoji, conversationId }) => {
  await Message.findByIdAndUpdate(messageId, {
    $pull: { reactions: { userId } }, // pehle purana reaction hatao
  });
  await Message.findByIdAndUpdate(messageId, {
    $push: { reactions: { userId, emoji } },
  });
  io.to(conversationId).emit("message_reacted", { messageId, userId, emoji });
});

socket.on("forward_message", async ({ originalMessageId, targetConversationIds, senderId }) => {
  const original = await Message.findById(originalMessageId);
  for (const convoId of targetConversationIds) {
    const forwarded = await Message.create({
      conversationId: convoId, sender: senderId,
      text: original.text, mediaUrl: original.mediaUrl,
    });
    io.to(convoId).emit("receive_message", forwarded);
  }
});
```

### 15.10 Block / Unblock User

```js
socket.on("block_user", async ({ userId, targetUserId }) => {
  await User.findByIdAndUpdate(userId, { $addToSet: { blockedUsers: targetUserId } });
});
```
Server-side `send_message` handler mein check add karo: agar receiver ne sender ko block kiya hai, toh message deliver mat karo (silently ya error callback ke saath).

### 15.11 Offline Message Delivery

Agar receiver offline hai:
1. Message DB mein `status: "sent"` ke saath save ho jata hai (Section 15.4).
2. Socket emit sirf online participants tak pahunchta hai.
3. Jab user next connect kare, unread messages fetch karne ke liye REST endpoint use karo (`GET /messages/unread?conversationId=`) — normal DB query, socket ka kaam nahi.

**Distinction yaad rakho:** Socket.io real-time delivery ke liye hai; persistence aur "catch-up on reconnect" REST + DB ka kaam hai.

### 15.12 Push Notifications (Background/Closed App)

Socket.io sirf active connection (foreground) mein kaam karta hai. Background ke liye:

```js
// server — jab receiver offline ho
const admin = require("firebase-admin");

if (!onlineUsers.has(receiverId)) {
  const receiver = await User.findById(receiverId);
  await admin.messaging().send({
    token: receiver.fcmToken,
    notification: { title: senderName, body: text },
    data: { conversationId },
  });
}
```

Web ke liye **Web Push API + Service Worker**, mobile ke liye **Firebase Cloud Messaging (FCM)** — dono cases mein device token DB mein store karna padega (`user.fcmToken`).

### 15.13 Message Pagination / Infinite Scroll

Socket sirf naye messages ke liye hai; purana history REST se load karo (cursor-based pagination):

```js
// REST endpoint
app.get("/messages/:conversationId", async (req, res) => {
  const { before, limit = 30 } = req.query;
  const query = { conversationId: req.params.conversationId };
  if (before) query.createdAt = { $lt: new Date(before) };

  const messages = await Message.find(query)
    .sort({ createdAt: -1 })
    .limit(Number(limit));

  res.json(messages.reverse());
});
```
Frontend scroll-up pe purane messages ka last `createdAt` `before` param mein bhejo.

### 15.14 Voice/Video Call Signaling (WebRTC + Socket.io)

Socket.io khud audio/video stream carry nahi karta — sirf **signaling** (offer/answer/ICE candidates exchange) ke liye use hota hai, actual media peer-to-peer WebRTC se jaata hai.

```js
socket.on("call_user", ({ toUserId, offer }) => {
  const targetSocketId = onlineUsers.get(toUserId);
  io.to(targetSocketId).emit("incoming_call", { from: socket.userId, offer });
});

socket.on("answer_call", ({ toUserId, answer }) => {
  const targetSocketId = onlineUsers.get(toUserId);
  io.to(targetSocketId).emit("call_answered", { answer });
});

socket.on("ice_candidate", ({ toUserId, candidate }) => {
  const targetSocketId = onlineUsers.get(toUserId);
  io.to(targetSocketId).emit("ice_candidate", { candidate });
});
```

Ye advanced feature hai — agar app mein calling chahiye toh alag se WebRTC (`simple-peer` library) seekhna padega, Socket.io sirf message-passing layer hai isme.

### 15.15 Privacy Settings (Last Seen, Read Receipts Toggle)

```js
// User Schema mein add karo
{
  privacySettings: {
    lastSeenVisible: Boolean,
    readReceiptsEnabled: Boolean,
  }
}
```
`message_read` aur `user_offline` emit karne se pehle sender/receiver dono ki settings check karo — agar disabled hai toh event hi mat bhejo.

---

## 16. Security Checklist

1. **JWT verify** har connection pe (Section 11) — client-trusted data kabhi mat lo.
2. **Rate limiting** events pe — `express-rate-limit` jaisa concept socket ke liye khud implement karo (per-userId counter with sliding window, Redis mein).
3. **`maxHttpBufferSize`** set karo — bina limit ke bade payloads DOS attack ban sakte hain.
4. **CORS strict rakho** — production mein `origin: "*"` mat use karo agar `credentials: true` hai (browser reject karega, aur security risk bhi hai).
5. **Input validation** — Zod/Joi se har incoming event payload validate karo, DB mein directly mat daalo.
6. **Block-list check** server-side, message delivery se pehle.
7. **XSS sanitization** — message text render karte waqt frontend pe escape karo (React default escape karta hai, lekin `dangerouslySetInnerHTML` avoid karo).

---

## 17. Testing Socket.io

```js
// Jest + socket.io-client
const { createServer } = require("http");
const { Server } = require("socket.io");
const Client = require("socket.io-client");

let io, serverSocket, clientSocket;

beforeAll((done) => {
  const httpServer = createServer();
  io = new Server(httpServer);
  httpServer.listen(() => {
    const port = httpServer.address().port;
    clientSocket = Client(`http://localhost:${port}`);
    io.on("connection", (socket) => { serverSocket = socket; });
    clientSocket.on("connect", done);
  });
});

afterAll(() => {
  io.close();
  clientSocket.close();
});

test("message event works", (done) => {
  clientSocket.on("receive_message", (data) => {
    expect(data.text).toBe("hello");
    done();
  });
  serverSocket.emit("receive_message", { text: "hello" });
});
```

---

## 18. Debugging Tools

- **DEBUG env variable:** `DEBUG=socket.io:* node server.js` — poora internal logging dikhata hai (handshake, transport upgrade, disconnect reason).
- **Browser dev tools → Network tab → WS filter** — actual frames dekh sakte ho.
- **Admin UI (`@socket.io/admin-ui`)** — production-grade dashboard, connected clients, rooms, events real-time monitor karne ke liye.

---

## 19. Performance Tuning

- **Compression:** Socket.io default per-message-deflate use karta hai; bade payloads ke liye helpful, chhote frequent events (typing) ke liye overhead ban sakta hai — `perMessageDeflate: false` set kar sakte ho agar zaroorat na ho.
- **`transports: ["websocket"]` force karna** polling handshake latency avoid karta hai — lekin restrictive networks mein fail ho sakta hai, fallback allow karna safer hai.
- **Batch events** — agar bahut saare chhote updates ek saath aa rahe hain, unhe client-side buffer karke ek batch event mein bhejo (jaise typing cursor positions).
- **Redis adapter ka overhead** — pub/sub har broadcast pe extra hop add karta hai, lekin scaling ke liye trade-off zaroori hai.

---

## 20. React Native Considerations

```js
import { io } from "socket.io-client";

const socket = io("http://YOUR_SERVER_IP:5000", {
  transports: ["websocket"], // RN mein polling se issues aate hain, websocket force karo
});
```
- App background jaane par (iOS/Android) socket disconnect ho jata hai — foreground return pe manually `socket.connect()` call karo (`AppState` listener use karo).
- Background messages ke liye push notifications (FCM) hi bharosemand tarika hai (Section 15.12).

---

## 21. Common Production Mistakes (Inse Bacho)

1. **Multiple socket connections** — React StrictMode/missing cleanup se duplicate connections. Always cleanup in `useEffect`.
2. **Client-trusted data** — `senderId`/`userId` client se directly lena dangerous.
3. **No DB fallback** — sirf socket state pe app design (refresh pe data loss).
4. **Missing reconnection handling** — `reason === "io server disconnect"` pe auto-reconnect nahi chalta, manual `socket.connect()` chahiye.
5. **CORS misconfiguration** — `origin: "*"` + `credentials: true` combo browser reject karega.
6. **No rate limiting** — unthrottled `typing` events server flood kar sakte hain.
7. **In-memory state without Redis** — scaling ke time online-users Map, socket-room mapping sab break ho jayega.
8. **Media directly via socket** — payload size aur memory pressure, hamesha REST upload use karo.

---

## 22. Quick Reference — Full Event Checklist for Chat App

| Event | Direction | Purpose |
|---|---|---|
| `connect` / `disconnect` | Both | Session lifecycle |
| `join_conversation` | Client → Server | Room join |
| `send_message` | Client → Server | Naya message |
| `receive_message` | Server → Client | Message deliver |
| `message_delivered` / `message_read` | Client → Server | Delivery/read ack |
| `mark_all_read` | Client → Server | Bulk read |
| `status_update` | Server → Client | Tick status |
| `typing` / `stop_typing` | Client → Server | Typing state |
| `user_typing` / `user_stop_typing` | Server → Client | Typing UI |
| `user_online` / `user_offline` | Server → Client | Presence |
| `add_participant` / `remove_participant` | Client → Server | Group management |
| `edit_message` / `delete_message` | Client → Server | Message edit/delete |
| `react_message` | Client → Server | Emoji reaction |
| `forward_message` | Client → Server | Forward to other chats |
| `block_user` | Client → Server | Block |
| `call_user` / `answer_call` / `ice_candidate` | Both | WebRTC signaling |

---

**Ab ye guide sach mein cut-to-cut complete hai — fundamentals, options, middleware, scaling, security, testing, aur poora WhatsApp-level feature set (edit/delete/react/forward/block/calls/privacy/pagination) sab cover ho gaya.**
