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
The server will be in charge of processing the inputs.<br><br>
**server.js**
```javascript

let sx = 1, sy = 10; // Speed of moving in x/y directions
let g = 1; // Gravity
let r = 10; // Size of squares
let pos = {}; // Position of every player
let vel = {}; // Velocity of every player
let ground = {}; // Whether or not the player is on the ground
let inp = {}; // Every player's inputs

io.on("connection", (socket) => {
  ...
  socket.on("update", (data) => { // Recieve the update from clients
    inp[socket.id] = data; // Every player gets a unique id, to distinguish between two players who might choose the same name.
  });
}

const framerate = 30; // Same framerate to sync
setInterval(() => { // Run this once every 1000/framerate milliseconds
  move();

  io.emit("positions", pos); // Send the positions back
}, 1000 / framerate);
function move() {
  for (const id in inp) { // Go through every player's inputs
    const input = inp[id];
    if (input.l) { // Move left
      vel[id].x -= sx;
    }
    if (input.r) { // Move right
      vel[id].x += sx;
    }
    if (input.u) { 
      if (ground[id]) { // Jump if on the ground
        vel[id].y = sy;
        ground[id] = false;
      } else vel[id].y += sy * 0.05; // Add a little bit of velocity if up is held, for a higher jump
    }
    if (input.d) { // Fall down faster
      vel[id].y -= sy * 0.05;
    }
    vel[id].y -= g; // Gravity
    vel[id].x *= 0.9; // Damping to make movement more controllable

    pos[id].x += vel[id].x; // Move in the x direction
    if (pos[id].x < r) { // Check for hitting the boundaries
      pos[id].x = r;
      vel[id].x = 0;
    } else if (pos[id].x > width - r) {
      pos[id].x = width - r;
      vel[id].x = 0;
    }
    pos[id].y += vel[id].y; // Move in the y direction
    if (pos[id].y < r) { // Check for hitting boundaries
      pos[id].y = r;
      ground[id] = true;
      vel[id].y = 0;
    } else if (pos[id].y > height - r) {
      pos[id].y = height - r;
      vel[id].y = 0;
    }
    if (vel[id].y != 0) ground[id] = false; // If moving in the y direction, the player is no longer on the ground.
  }
}
```
This all allows the players to move around, and see the other players. There isn't much to do except move around, so let's add some platforms.
## Platforms
There are many ways to describe platforms, but I decided to do it with two corners.
```javascript
let platform = {x1:x1, y1:y1, x2:x2, y2:y2};
```
One thing to be aware of is that whenever a new platform is created, the coordinates must be changed so that x1 < x2 and y1 < y2. This will become important later.
### Adding platforms
The simplest way to create a set of platforms is to make the player do it :). <br><br>
**script.js**
```javascript
function mouseClicked() {
  if (mouseX < 0 || mouseX > width || mouseY < 0 || mouseY > height) return;
  if (c1) { // Are we on corner 1?
    x1 = mouseX;
    y1 = height - mouseY;
  } else { // We are on corner 2
    x2 = mouseX;
    y2 = height - mouseY;
    if (x1 > x2) { // Swap x and y coordinates if they are in the wrong order.
      let t = x2;
      x2 = x1;
      x1 = t;
    }
    if (y1 > y2) {
      let t = y2;
      y2 = y1;
      y1 = t;
    }
    if (x1 != x2 && y1 != y2)
      socket.emit("platform", { // Verify the platform does not have 0 thickness, then send it over to the server.
        x1: x1,
        y1: y1,
        x2: x2,
        y2: y2,
      });
  }
  c1 = !c1; // Switch the state of which corner is active.
}

function draw() {
  ...
  strokeWeight(5);
  noFill();
  if (!c1) { // Show a rectangle where the platform will be placed
    stroke(0, 255, 0);
    rect(x1, y1, mouseX - x1, height - mouseY - y1);
  }
  noStroke();
  fill(200);
  for (const id in plat) {
    rect(plat[id].x1, plat[id].y1, plat[id].x2 - plat[id].x1, plat[id].y2 - plat[id].y1);
  }
  ...
}
```
**server.js**
```javascript
let plat = [];
io.on("connection", (socket) => {
  ...
  socket.on("platform", (platform) => {
    plat.push(platform); // Recieve new platforms
  });
}
const framerate = 30;
setInterval(() => {
  move();

  io.emit("positions", pos);
  io.emit("plat", plat); // Send platforms to all clients
}, 1000 / framerate);
...
```
Now, platforms can be made and seen by all the clients. They still don't do anything, though.
### Collision detection
Because the players and platforms are axis aligned rectangles, detecting whether or not they collide is pretty simple.<br><br>
**server.js**
```javascript
function colCheck(r1, r2) {
  return r1.x2 > r2.x1 && r1.x1 < r2.x2 && r1.y2 > r2.y1 && r1.y1 < r2.y2;
}
```
But why does this actually work? Remember that x2 is always the right side of a rectangle, and x1 is always the left.<br>
The first check asks, is part of the first rectangle on the right side of the left side of the second rectangle?<br>
![Image failed to load](assets/collision1.png) <br>
Then, the second check narrows down rectangle 1 to intersecting the x position of rectangle 2.<br>
![Image failed to load](assets/collision2.png)<br>
The y position checks does the same in the y direction.<br>
![Image failed to load](assets/collision3.png)<br>
This allows for a check as to whether the rectangles intersect or not.<br>
![Image failed to load](assets/collision4.png)<br><br>
So we can now determine if the player is touching a platform, but this isn't completely done yet. How would you figure out the side the player collided with, and move it to the correct spot? Instead of trying to see if each side intersects with the player, we can just move in the x direction and y direction seperately, then check for collisions after each. This won't be any less efficient, since the collision check is just 4 comparisons.<br><br>
**server.js**
```javascript
function move() {
  for (const id in inp) {
    ...
    pos[id].x += vel[id].x; // Move in x direction
    let cpos = { // Convert player coordinates into corner coordinates
      x1: pos[id].x - r,
      y1: pos[id].y - r,
      x2: pos[id].x + r,
      y2: pos[id].y + r,
    };
    for (const p in plat) {
      if (colCheck(cpos, plat[p])) { // Check for collision with every platform
        pos[id].x -= vel[id].x; // If there is collision, move back and stop x motion
        vel[id].x = 0;
        break;
      }
    }
    if (pos[id].x < r) {
      pos[id].x = r;
      vel[id].x = 0;
    } else if (pos[id].x > width - r) {
      pos[id].x = width - r;
      vel[id].x = 0;
    }

    pos[id].y += vel[id].y; // Move in y direction
    cpos = { // Convert coordinates to corner coordinates
      x1: pos[id].x - r,
      y1: pos[id].y - r,
      x2: pos[id].x + r,
      y2: pos[id].y + r,
    };
    for (const p in plat) {
      if (colCheck(cpos, plat[p])) { // Check for collision
        pos[id].y -= vel[id].y; // Move back if there is collision
        if (vel[id].y < 0) ground[id] = true; // If falling, the player has touched the ground and can jump again.
        vel[id].y = 0;
        break;
      }
    }
    if (pos[id].y < r) {
      pos[id].y = r;
      ground[id] = true;
      vel[id].y = 0;
    } else if (pos[id].y > height - r) {
      pos[id].y = height - r;
      vel[id].y = 0;
    }
    if (vel[id].y != 0) ground[id] = false;
  }
}
```
Finally, we have a working multiplayer platformer. Let's add a bit more functionality.
### Cutting platforms
Currently, the screen will only fill up more and more with platforms until it becomes impossible to move. Let's add some way to add a "negative" platform that sets a rectangular area to free space. <br>
In theory, cutting platforms seems simple: just split up the platform into smaller platforms that don't include the cut out part. However, the way this is done is important. Every platform has to remain a rectangle for the collision algorithm to work, and there are many ways that a platform can be cut. It would be best to find a way to do this in a way that works for every case.<br>

## In game chat
### XSS attacks
