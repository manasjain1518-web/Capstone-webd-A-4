// npm init -y
// npm i socket.io
const express = require("express");
const http = require("http");

const { Server } = require("socket.io");

const app = express();

// ---- ADDED: serve the frontend ----
const path = require("path");
app.get("/", (req, res) => res.sendFile(path.join(__dirname, "client.html")));
const httpserver = http.createServer(app);

const io = new Server(httpserver);

// ---- ADDED: room helpers ----
const LOBBY = "LOBBY";
const rooms = new Map([[LOBBY, { code: LOBBY, name: "Public Lobby", owner: null, history: [] }]]);
const clean = (v, max) => (typeof v === "string" ? v.trim().slice(0, max) : "");

function membersOf(code) {
  const set = io.sockets.adapter.rooms.get(code) || new Set();
  return [...set].map((id) => ({ id, name: io.sockets.sockets.get(id)?.data.username || "?" }));
}
function emitMembers(code) {
  io.to(code).emit("members", { room: code, members: membersOf(code) });
}
function joinRoomInternal(socket, code) {
  const r = rooms.get(code);
  if (!socket.data.rooms.has(code)) {
    socket.join(code);
    socket.data.rooms.add(code);
    socket.to(code).emit("system", { room: code, text: `${socket.data.username} joined`, ts: Date.now() });
  }
  emitMembers(code);
  return { room: { code: r.code, name: r.name, owner: r.owner === socket.id }, history: r.history };
}
function leaveRoomInternal(socket, code, disconnecting) {
  socket.leave(code);
  socket.data.rooms.delete(code);
  io.to(code).emit("system", { room: code, text: `${socket.data.username} left`, ts: Date.now() });
  emitMembers(code);
  const r = rooms.get(code);
  // personal rooms are deleted once empty
  if (r && code !== LOBBY && !(io.sockets.adapter.rooms.get(code)?.size)) rooms.delete(code);
}

let count = 0;
io.on("connection", (socket) => {
  count++;
  console.log("Active user: ", count);

  // Sending
  socket.emit("gyankibaat", "welcome to socket.io server");

  socket.broadcast.emit("join", "New User Joined");

  //   Reading
  socket.on("kuchbhi", (msg) => {
    console.log(msg);
  });

  socket.on("disconnect", () => {
    count--;
    console.log("Active user: ", count);
  });


  // =====================================================================
  // ADDED FEATURES (original events above/below are untouched)
  // Usernames, chat messages, personal (private) rooms, typing indicator
  // =====================================================================
  socket.data.username = "Guest" + Math.floor(1000 + Math.random() * 9000);
  socket.data.rooms = new Set();
  joinRoomInternal(socket, LOBBY);

  socket.on("setUsername", (name, cb) => {
    name = clean(name, 20);
    if (!name) return cb && cb({ ok: false, error: "Name cannot be empty" });
    const old = socket.data.username;
    socket.data.username = name;
    socket.data.rooms.forEach((code) => {
      io.to(code).emit("system", { room: code, text: `${old} is now ${name}`, ts: Date.now() });
      emitMembers(code);
    });
    cb && cb({ ok: true, username: name });
  });

  // Create a personal room, creator joins automatically, others join with the code
  socket.on("createRoom", (data, cb) => {
    const name = clean(data && data.name, 30);
    if (!name) return cb && cb({ ok: false, error: "Room name required" });
    let code;
    do { code = Math.random().toString(36).slice(2, 8).toUpperCase(); } while (rooms.has(code));
    rooms.set(code, { code, name, owner: socket.id, history: [] });
    cb && cb({ ok: true, ...joinRoomInternal(socket, code) });
  });

  // Join an existing room by its code (multiple users can join the same room)
  socket.on("joinRoom", (data, cb) => {
    const code = clean(data && data.code, 10).toUpperCase();
    if (!rooms.has(code)) return cb && cb({ ok: false, error: "Room not found" });
    cb && cb({ ok: true, ...joinRoomInternal(socket, code) });
  });

  socket.on("leaveRoom", (code, cb) => {
    if (code === LOBBY) return cb && cb({ ok: false, error: "Cannot leave the lobby" });
    if (!socket.data.rooms.has(code)) return cb && cb({ ok: false, error: "Not in room" });
    leaveRoomInternal(socket, code);
    cb && cb({ ok: true });
  });

  socket.on("chatMessage", (data, cb) => {
    const room = data && data.room;
    const text = clean(data && data.text, 1000);
    if (!text || !socket.data.rooms.has(room)) return cb && cb({ ok: false });
    const msg = { id: Date.now() + "-" + socket.id.slice(0, 4), room, user: socket.data.username,
                  from: socket.id, text, ts: Date.now() };
    const r = rooms.get(room);
    r.history.push(msg);
    if (r.history.length > 100) r.history.shift();
    io.to(room).emit("chatMessage", msg);
    cb && cb({ ok: true });
  });

  socket.on("typing", (data) => {
    if (data && socket.data.rooms.has(data.room))
      socket.to(data.room).emit("typing", { room: data.room, user: socket.data.username, typing: !!data.typing });
  });

  socket.on("disconnect", () => {
    [...socket.data.rooms].forEach((code) => leaveRoomInternal(socket, code, true));
  });
  // =====================================================================

  io.emit("announcement", "Today is last time we study web dev -3");
});

httpserver.listen(8080, () => {
  console.log("Server started");
});
