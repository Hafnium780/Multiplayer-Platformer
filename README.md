# Creating a Multiplayer Platformer "Game"
This is a document showing my journey in creating an online multiplayer game, using p5.js, Node.js, Express, and Socket.io.
## The idea
**So, why even do this?**<br>
...<br>
Why not?<br><br>
One day, I wanted to try creating some sort of multiplayer game. It would be a cool little project that would be fun to do. I used glitch.com to 
## Using Node.js and Socket.io to create a basic multiplayer interface
First of all, we have to make sure the connection between client and server works. On the server side, a simple socket connection is created.<br><br>
**server.js**
```javascript
const express = require("express"); // Load the express module
const app = express(); // Create an instance of the express module
const http = require("http").createServer(app); // Allow clients to connect to server
const io = require("socket.io")(http, { // Create an input/output connection through socket.io
  transports: ["websocket"]
});

const port = process.env.PORT || 3000; // Set the port to be used, if none specified use 3000

app.get("/", (req, res) => { // Whenever a request is recieved, send the main index.html file.
  res.sendFile("/public/index.html");
});

http.listen(port, () => { // Listen for incoming requests
  console.log(`Server port: ${port}`);
});

io.on("connection", (socket) => { // Called when a client connects
  console.log(`${socket.id} connected`);
  
  socket.on("disconnect", () => { // Called when a client disconnects
    console.log(`${socket.id} (${names[socket.id]}) disconnected`);
  });
});
```
On the client side, a standard p5.js setup is used, a socket instance is created, and the socket.io module is included in the html. <br><br>
**script.js**
```javascript
const socket = io({
  transports: ["websocket"],
});

function setup() {
}

function draw() {
}
```
**index.html**
```html
<!-- Hafnium780 -->
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>game</title>
    <script src="p5.min.js"></script>
    <style>
      body {
        background-color: grey;
      }
    </style>
  </head>
  <body>
    <!-- The sketch! -->
    <div id="sketch-container">
      <!-- canvas will be inserted here -->
    </div>
    <script src="socket.io.min.js"></script>
    <script src="script.js"></script>
  </body>
</html>
```
Using node to run this, we can see that the client is able to connect.
```
Server port:3000
LYZqhscpYD9Rju8sAAAB connected
Zg3hT83WKHCDeum-AAAD connected
```
## Creating the character
Now that we have the basic connection established, let's actually add some movement to the character. I decided to send the inputs over to the server, do the calculations there, then send the game state back to the client. This is typically better if the connection is not slow, since there is no way to manipulate the calculations to cheat (i.e. giving yourself a large upwards velocity to fly). <br>
The character will be a square to make collisions easier, and represented with a position (x, y) and half-side length r.<br><br>
**script.js**
```javascript
const socket = io({
  transports: ["websocket"],
});

let r = 10; // Size of the square, changing this has no effect on calculations, only changes how large the players are drawn

let positions;

function setup() {
  frameRate(30); // Lower frame rate allows for a more stable connection
  
  socket.on("positions", (data) => { // Whenever new positions are available, store them.
    positions = data;
  });
}

function draw() {
  let pl = false, pr = false, pu = false, pd = false; // Inputs
  if (keyIsDown(LEFT_ARROW) || keyIsDown(65)) pl = true;
  if (keyIsDown(RIGHT_ARROW) || keyIsDown(68)) pr = true;
  if (keyIsDown(UP_ARROW) || keyIsDown(87)) pu = true;
  if (keyIsDown(DOWN_ARROW) || keyIsDown(83)) pd = true;
  
  socket.emit("update", { // Update the server with the inputs
    l: pl,
    r: pr,
    u: pu,
    d: pd,
  });
  
  translate(0, height); 
  scale(1, -1); // Move and scale the plane to make the origin in the bottom left corner
  background(0);
  
  fill(100);
  for (const id in positions) {
    const position = positions[id];
    rect(position.x - r, position.y - r, r * 2, r * 2); // Draw each square
  }
}
```
## Platforms
### Adding platforms
### Collision detection
### Cutting platforms
## In game chat
### XSS attacks
