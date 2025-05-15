# Multisynq for LLMs

Multisynq is a multiplayer framework for web apps that works without server-side code. Instead, user code is executed client-side in synchronized virtual machines. This example will illustrate the usage and teach the main concepts. At the end you'll find a types definition file describing the API.

## Example: Multiblaster

```html
<html>
    <head>
        <meta charset="utf-8">
        <meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1,user-scalable=no">
        <title>Multiblaster</title>
        <style>
            html, body {
                margin: 0;
                height: 100%;
                overflow: hidden;
                touch-action: none;
                background: #999;
                -webkit-touch-callout: none;
                -webkit-user-select: none;
                user-select: none;
                touch-action: none;
            }
            #canvas {
                background: #000;
                object-fit: contain;
                max-width: 100%;
                max-height: 100%;
            }
            #joystick {
                position: absolute;
                right: 50px;
                bottom: 50px;
                width: 120px;
                height: 120px;
                border: 3px solid #FFF;
                border-radius: 60px;
                opacity: 0.5;
            }
            #knob {
                position: absolute;
                left: 20px;
                top: 20px;
                width: 80px;
                height: 80px;
                border-radius: 40px;
                background-color: #FFF;
            }
            #initials {
                position: fixed; bottom: 10px; right: 70px; padding: .5em; border-radius: 30px;
                opacity: 50%; background: lightgray; box-shadow: 1px 1px 5px black;
                border: none; color: #000; text-align: center; font-size: 1.2em;

            }
        </style>
        <script src="https://cdn.jsdelivr.net/npm/@multisynq/client@1.0.1/bundled/multisynq-client.min.js"></script>
    </head>
    <body>
        <canvas id="canvas" width="1000" height="1000"></canvas>
        <div id="joystick"><div id="knob"></div></div>
        <input id="initials" type="text" maxlength="10" size="10" placeholder="Initials">
        <script>
// We use a 2D canvas for rendering and Multisynq for multiplayer logic

// Multisynq works by splitting the code into model and view:
// * The model is the shared simulation state.
// * The view is the interface between the simulation and the local client.

// The model takes on the role of server code in traditional multiplayer
// approaches. There is no need to implement any networking code or server-side code.

// Model code execution is synchronized across all clients in the session
// via Multisynq's synchronizer network, which feeds the exact same time
// and sequence of user input events to all clients.
// This ensures that all clients are executing the exact same code at the
// exact same time, so the simulation is deterministic and consistent.
// It also means that no state needs to be transmitted between clients, or
// be sent from the server to the clients. Instead, you can run e.g. physics
// or NPC code in the model, and it will be exactly the same for all clients.

// All global constants used in the simulation must be defined as Multisynq.Constants
// so that they get hashed into the session ID, ensuring that all clients
// in the same session use the same constants
const C = Multisynq.Constants;
C.SHIP_ACCELERATION = 0.5;
C.SHIP_TURN_SPEED = 0.2;
C.SHIP_MAX_SPEED = 10;

// Any Multisynq app must be split into model and view.
// The model is the shared simulation state.
// The view is the interface between the simulation and the local client.

// The shared simulation model must be entirely self-contained
// and not depend on any state outside of it.
// This is to ensure that all clients have the same simulation state.

// Additionally, all code executed in the model must be identical on all
// clients. Multisynq ensures that by hashing the model code into the session ID.
// If the model uses any external code, it must be added to Constants
// to get hashed, too. For external libraries, it's ususally sufficient
// to add their version number to Constants so if two clients have different
// versions of the same library, they will be in different sessions.

// The root model class is the entry point for the simulation.
// It is created when the session is initially joined and is responsible for creating
// all other models in the simulation.
// It is also responsible for managing the simulation state and persisting it
// across sessions. The model is created once and then shared across all clients.
// The model is created with the `init` method, which is called when the session is joined.
// You must not implement any logic in the constructor, because that would interfere with the
// deserialization of the model state.

class Game extends Multisynq.Model {
    // No constructor, only init!
    init(_, persisted) {
        this.highscores = persisted?.highscores ?? {};
        this.ships = new Map();
        this.asteroids = new Set();
        this.blasts = new Set();
        // these events are published automatically when a view joins or exits
        // and are scoped to the session ID (a predefined property)
        this.subscribe(this.sessionId, "view-join", this.viewJoined);
        this.subscribe(this.sessionId, "view-exit", this.viewExited);
        // Create models using the `create` method, not "new"
        Asteroid.create({});
        // kick off the main loop
        this.mainLoop();
    }

    // create a new ship for every player that joins the session
    // the viewId is a unique identifier for each player in the session
    viewJoined(viewId) {
        const ship = Ship.create({ viewId });
        this.ships.set(viewId, ship);
    }

    viewExited(viewId) {
        const ship = this.ships.get(viewId);
        this.ships.delete(viewId);
        ship.destroy();
    }

    // Persistent highscore storage
    // the session is snapshotted automatically by Multisynq
    // so new players can join and leave at any time
    // But when the code changes, the session ID changes and only the
    // explicitly persisted state is used to initialize the new session
    // (and passed to the init method of the root model)
    setHighscore(initials, score) {
        if (this.highscores[initials] >= score) return;
        this.highscores[initials] = score;
        this.persistSession({ highscores: this.highscores });
    }

    // The main loop calls itself every 50ms using the `future` mechanism
    // This is a special Multisynq function that schedules a function to be called
    // in the future, under control of Multisynq rather than the browser.
    // Future calls are synchronized across all clients in the session
    // and are executed in the same order on all clients. They are stored as
    // part of the snapshot just like subscriptions.
    mainLoop() {
        for (const ship of this.ships.values()) ship.move();
        for (const asteroid of this.asteroids) asteroid.move();
        for (const blast of this.blasts) blast.move();
        this.checkCollisions();
        this.future(50).mainLoop(); // move & check every 50 ms
    }

    // check for collisions between asteroids, ships and blasts
    // note that this is a naive O(n^2) algorithm which is fine for this game
    checkCollisions() {
        for (const asteroid of this.asteroids) {
            if (asteroid.wasHit) continue;
            const minx = asteroid.x - asteroid.size;
            const maxx = asteroid.x + asteroid.size;
            const miny = asteroid.y - asteroid.size;
            const maxy = asteroid.y + asteroid.size;
            for (const blast of this.blasts) {
                if (blast.x > minx && blast.x < maxx && blast.y > miny && blast.y < maxy) {
                    asteroid.hitBy(blast);
                    break;
                }
            }
            for (const ship of this.ships.values()) {
                if (!ship.wasHit && ship.x + 10 > minx && ship.x - 10 < maxx && ship.y + 10 > miny && ship.y - 10 < maxy) {
                    if (!ship.score && Math.abs(ship.x-500) + Math.abs(ship.y-500) < 40) continue; // no hit if just spawned
                    ship.hitBy(asteroid);
                    break;
                }
            }
        }
    }
}
Game.register("Game");

// SpaceObject is a base class for all objects in the game.
// It provides basic properties and methods for moving and rotating objects.

class SpaceObject extends Multisynq.Model {
    // the root model is the game itself and well-known as "modelRoot"
    get game() { return this.wellKnownModel("modelRoot"); }

    init() {
        this.x = 0;
        this.y = 0;
        this.a = 0;
        this.dx = 0;
        this.dy = 0;
        this.da = 0;
    }

    move() {
        // drift through space
        this.x += this.dx;
        this.y += this.dy;
        this.a += this.da;
        if (this.x < 0) this.x += 1000;
        if (this.x > 1000) this.x -= 1000;
        if (this.y < 0) this.y += 1000;
        if (this.y > 1000) this.y -= 1000;
        if (this.a < 0) this.a += 2 * Math.PI;
        if (this.a > 2 * Math.PI) this.a -= 2 * Math.PI;
    }
}

// each player has a ship and can control it via published events
// with the scope of the player's viewId. Each ship subscribes to
// the events published by a different viewId, so the ship can be controlled
// by the player that owns it. The ship is created when the player joins
// the session and destroyed when the player leaves.
class Ship extends SpaceObject {
    init({ viewId }) {
        super.init();
        this.viewId = viewId;
        this.initials = '';
        this.subscribe(viewId, "left-thruster", this.leftThruster);
        this.subscribe(viewId, "right-thruster", this.rightThruster);
        this.subscribe(viewId, "forward-thruster", this.forwardThruster);
        this.subscribe(viewId, "fire-blaster", this.fireBlaster);
        this.subscribe(viewId, "set-initials", this.setInitials);
        this.reset();
    }

    reset() {
        // Multisynq patches Math.random() to be deterministic if executed in the model
        // (i.e. the same sequence of random numbers is generated on all clients)
        this.x = 480 + 40 * Math.random();
        this.y = 480 + 40 * Math.random();
        this.a = -Math.PI / 2;
        this.dx = 0;
        this.dy = 0;
        this.left = false;
        this.right = false;
        this.forward = false;
        this.score = 0;
        this.wasHit = 0;
    }

    leftThruster(active) {
        this.left = active;
    }

    rightThruster(active) {
        this.right = active;
    }

    forwardThruster(active) {
        this.forward = active;
    }

    fireBlaster() {
        if (this.wasHit) return;
        // create blast moving at speed 20 in direction of ship
        const dx = Math.cos(this.a) * 20;
        const dy = Math.sin(this.a) * 20;
        const x = this.x + dx;
        const y = this.y + dy;
        Blast.create({ x, y, dx, dy, ship: this });
        // kick ship back a bit
        this.accelerate(-C.SHIP_ACCELERATION);
    }

    move() {
        if (this.wasHit) {
            // keep drifting as debris for 3 seconds
            if (++this.wasHit > 60) this.reset();
        } else {
            // process thruster controls
            if (this.forward) this.accelerate(C.SHIP_ACCELERATION);
            if (this.left) this.a -= C.SHIP_TURN_SPEED;
            if (this.right) this.a += C.SHIP_TURN_SPEED;
        }
       super.move();
    }

    accelerate(acceleration) {
        this.dx += Math.cos(this.a) * acceleration;
        this.dy += Math.sin(this.a) * acceleration;
        if (this.dx > 10) this.dx = 10;
        if (this.dx < -10) this.dx = -10;
        if (this.dy > 10) this.dy = 10;
        if (this.dy < -10) this.dy = -10;
    }

    setInitials(initials) {
        if (!initials) return;
        for (const ship of this.game.ships.values()) {
            if (ship.initials === initials) return;
        }
        const highscore = this.game.highscores[this.initials];
        if (highscore !== undefined) delete this.game.highscores[this.initials];
        this.initials = initials;
        this.game.setHighscore(this.initials, Math.max(this.score, highscore || 0));
    }

    scored() {
        this.score++;
        if (this.initials) this.game.setHighscore(this.initials, this.score);
    }

    hitBy(asteroid) {
        // turn both into debris
        this.wasHit = 1;
        asteroid.wasHit = 1;
    }
}
Ship.register("Ship");

// Asteroids are created by the game model and move around randomly.
// They are destroyed when they are hit by a blast or when they
// are hit by a ship. When they are hit, they split into two smaller
// asteroids that move faster and are harder to hit. When they are
// small enough, they turn into debris and are destroyed.

class Asteroid extends SpaceObject {
    init({ size, x, y, a, dx, dy, da }) {
        super.init();
        if (size) {
            // init second asteroid after spliting
            this.size = size;
            this.x = x;
            this.y = y;
            this.a = a;
            this.dx = dx;
            this.dy = dy;
            this.da = da;
        } else {
            // init new large asteroid
            this.size = 40;
            this.x = Math.random() * 400 - 200;
            this.y = Math.random() * 400 - 200;
            this.a = Math.random() * Math.PI * 2;
            const speed = Math.random() * 4 + 1;
            this.dx = Math.cos(this.a) * speed;
            this.dy = Math.sin(this.a) * speed;
            this.da = (0.02 + Math.random() * 0.03) * (Math.random() < 0.5 ? 1 : -1);
            this.wasHit = 0;
            this.move(); // random (x,y) is around top-left corner, wrap to other corners
        }
        this.game.asteroids.add(this);
    }

    move() {
        if (this.wasHit) {
            // keep drifting as debris, larger pieces drift longer
            if (++this.wasHit > this.size) return this.destroy();
        }
        super.move();
    }

    hitBy(blast) {
        blast.ship.scored();
        if (this.size > 20) {
            // split into two smaller faster asteroids
            // by changing this asteroid's size, speed and direction
            // and creating a new one moving in the opposite direction
            this.size *= 0.7;
            this.da *= 1.5;
            this.dx = -blast.dy * 10 / this.size;
            this.dy = blast.dx * 10 / this.size;
            Asteroid.create({ size: this.size, x: this.x, y: this.y, a: this.a, dx: -this.dx, dy: -this.dy, da: this.da });
        } else {
            // turn into debris
            this.wasHit = 1;
            this.ship = blast.ship;
        }
        blast.destroy();
    }

    destroy() {
        this.game.asteroids.delete(this);
        super.destroy();
        // keep at least 5 asteroids around
        if (this.game.asteroids.size < 5) Asteroid.create({});
    }
}
Asteroid.register("Asteroid");

// Blasts are created by the ship when it fires its blaster.
// They move in the direction of the ship and disappear after 1.5 seconds.
// They are also destroyed when they hit an asteroid.
class Blast extends SpaceObject {
    init({x, y, dx, dy, ship}) {
        super.init();
        this.ship = ship;
        this.x = x;
        this.y = y;
        this.dx = dx;
        this.dy = dy;
        this.t = 0;
        this.game.blasts.add(this);
    }

    move() {
        // move for 1.5 second before disappearing
        this.t++;
        if (this.t > 30) {
            this.destroy();
            return;
        }
        super.move();
    }

    destroy() {
        this.game.blasts.delete(this);
        super.destroy();
    }
}
Blast.register("Blast");


/////////// Code below is executed outside of synced VM ///////////

// We use a Multisynq View as the Interface between shared simulation and local
// renderer. We can read state directly from the simulation model, but
// must not modify it. Any manipulation of the simulation state must be via
// `publish` operations to ensure it stays synchronized across all clients

class Display extends Multisynq.View {
    constructor(model) {
        super(model);
        this.model = model;

        const joystick = document.getElementById("joystick");
        const knob = document.getElementById("knob");

        // we use a very economical control scheme that only publishes
        // events when the user presses or releases a key to toggle thrusters
        // or shoot the blaster.
        // We use this view's auto-assigned viewId to scope the events to this player
        // and "our" ship model uses the viewId to subscribe to "our" events
        document.onkeydown = (e) => {
            joystick.style.display = "none";
            if (e.repeat) return;
            switch (e.key) {
                case "a": case "A": case "ArrowLeft":  this.publish(this.viewId, "left-thruster", true); break;
                case "d": case "D": case "ArrowRight": this.publish(this.viewId, "right-thruster", true); break;
                case "w": case "W": case "ArrowUp":    this.publish(this.viewId, "forward-thruster", true); break;
                case "s": case "S": case " ":          this.publish(this.viewId, "fire-blaster"); break;
            }
        };
        document.onkeyup = (e) => {
            if (e.repeat) return;
            switch (e.key) {
                case "a": case "A": case "ArrowLeft":  this.publish(this.viewId, "left-thruster", false); break;
                case "d": case "D": case "ArrowRight": this.publish(this.viewId, "right-thruster", false); break;
                case "w": case "W": case "ArrowUp":    this.publish(this.viewId, "forward-thruster", false); break;
            }
        };

        // on-screen joystick generates the same events as the keyboard
        let x = 0, y = 0, id = 0, right = false, left = false, forward = false;
        document.onpointerdown = (e) => {
            if (!id) {
                id = e.pointerId;
                x = e.clientX;
                y = e.clientY;
                joystick.style.left = `${x - 60}px`;
                joystick.style.top = `${y - 60}px`;
                joystick.style.display = "block";
            }
        };
        document.onpointermove = (e) => {
            e.preventDefault();
            if (id === e.pointerId) {
                let dx = e.clientX - x;
                let dy = e.clientY - y;
                if (dx > 30) {
                    dx = 30;
                    if (!right) { this.publish(this.viewId, "right-thruster", true); right = true; }
                } else if (right) { this.publish(this.viewId, "right-thruster", false); right = false; }
                if (dx < -30) {
                    dx = -30;
                    if (!left) { this.publish(this.viewId, "left-thruster", true); left = true; }
                } else if (left) { this.publish(this.viewId, "left-thruster", false); left = false; }
                if (dy < -30) {
                    dy = -30;
                    if (!forward) { this.publish(this.viewId, "forward-thruster", true); forward = true; }
                } else if (forward) { this.publish(this.viewId, "forward-thruster", false); forward = false; }
                if (dy > 0) dy = 0;
                knob.style.left = `${20 + dx}px`;
                knob.style.top = `${20 + dy}px`;
            }
        }
        document.onpointerup = (e) => {
            e.preventDefault();
            if (id === e.pointerId) {
                id = 0;
                if (!right && !left && !forward) {
                    this.publish(this.viewId, "fire-blaster");
                }
                if (right) { this.publish(this.viewId, "right-thruster", false); right = false; }
                if (left) { this.publish(this.viewId, "left-thruster", false); left = false;  }
                if (forward) { this.publish(this.viewId, "forward-thruster", false); forward = false; }
                knob.style.left = `20px`;
                knob.style.top = `20px`;
            } else {
                this.publish(this.viewId, "fire-blaster");
            }
        }
        document.onpointercancel = document.onpointerup;
        document.oncontextmenu = e => { e.preventDefault();  this.publish(this.viewId, "fire-blaster"); }
        document.ontouchend = e => e.preventDefault(); // prevent double-tap zoom on iOS

        // the initials input field is used to set the player's initials
        // which we keep in local storage and use to identify the player
        // the initials are used to display the player's score in the highscore list
        initials.ontouchend = () => initials.focus(); // and allow input ¯\_(ツ)_/¯
        initials.onchange = () => {
            localStorage.setItem("io.multisynq.multiblaster.initials", initials.value);
            this.publish(this.viewId, "set-initials", initials.value);
        }
        if (localStorage.getItem("io.multisynq.multiblaster.initials")) {
            initials.value = localStorage.getItem("io.multisynq.multiblaster.initials");
            this.publish(this.viewId, "set-initials", initials.value);
            // after reloading, our previous ship with initials is still there, so just try again once
            setTimeout(() => this.publish(this.viewId, "set-initials", initials.value), 1000);
        }
        initials.onkeydown = (e) => {
            if (e.key === "Enter") {
                initials.blur();
                e.preventDefault();
            }
        }

        this.smoothing = new WeakMap(); // position cache for interpolated rendering

        this.context = canvas.getContext("2d"); // our rendering context

        // in many apps we would subscribe to events published by the model here, e.g.
        // this.subscribe(this.model.id, "score-changed", this.updateScore);
        // but in this app we render the whole scene every frame, so we don't need to
        // subscribe to anything. The model is only used to get the current state of the game.
    }

    // detach is called when the session is interrupted before the view is destroyed
    // we need to manually release all resources this class created and
    // unsubscribe from DOM events etc. because on reconnect, the whole view will be recreated
    detach() {
        super.detach();
        document.onkeydown = null;
        document.onkeyup = null;
        document.onpointerdown = null;
        document.onpointermove = null;
        document.onpointerup = null;
        document.onpointercancel = null;
        document.oncontextmenu = null;
        // document.ontouchend = null; // leave this one alone, it prevents double-tap zoom on iOS
        initials.onchange = null;
        initials.onkeydown = null;
        initials.ontouchend = null;
    }

    // update is called by Multisynq once per render frame for the root view
    // it reads from the shared model, interpolates between frames, and renders
    // we use a 1000x1000 2D canvas to render the game in a simple line drawing style
    // the canvas is cleared and redrawn every frame
    update() {
        this.context.clearRect(0, 0, 1000, 1000);
        this.context.fillStyle = "rgba(255, 255, 255, 0.5)";
        this.context.lineWidth = 3;
        this.context.strokeStyle = "white";
        this.context.font = "30px sans-serif";
        this.context.textAlign = "left";
        this.context.textBaseline = "middle";
        // model highscore only keeps players with initials, merge with unnamed players
        const highscore = Object.entries(this.model.highscores);
        const labels = new Map();
        for (const ship of this.model.ships.values()) {
            let label = ship.initials;
            if (!label) {
                label = `Player ${labels.size + 1}`;
                highscore.push([label, ship.score]);
            }
            labels.set(ship, label);
        }
        // draw sorted highscore
        for (const [i, [label, score]] of highscore.sort((a, b) => b[1] - a[1]).entries()) {
            this.context.fillText(`${i + 1}. ${label}: ${score}`, 10, 30 + i * 35);
        }
        // draw ships, asteroids, and blasts
        this.context.font = "40px sans-serif";
        for (const ship of this.model.ships.values()) {
            const { x, y, a } = this.smoothPosAndAngle(ship);
            this.drawWrapped(x, y, 300, () => {
                this.context.textAlign = "right";
                this.context.fillText(labels.get(ship), -30 + ship.wasHit * 2, 0);
                this.context.textAlign = "left";
                this.context.fillText(ship.score, 30 - ship.wasHit * 2, 0);
                this.context.rotate(a);
                if (ship.wasHit) this.drawShipDebris(ship.wasHit);
                else this.drawShip(ship, ship.viewId === this.viewId);
            });
        }
        for (const asteroid of this.model.asteroids) {
            const { x, y, a } = this.smoothPosAndAngle(asteroid);
            this.drawWrapped(x, y, 60, () => {
                this.context.rotate(a);
                if (asteroid.wasHit) this.drawAsteroidDebris(asteroid.size, asteroid.wasHit * 2);
                else this.drawAsteroid(asteroid.size);
            });
        }
        for (const blast of this.model.blasts) {
            const { x, y } = this.smoothPos(blast);
            this.drawWrapped(x, y, 5, () => {
                this.drawBlast();
            });
        }
    }

    // the view remembers the last position an object was rendered at
    // and uses linear interpolation to smoothly animate the object
    // if the distance is too large, it snaps the object instead
    smoothPos(obj) {
        if (!this.smoothing.has(obj)) {
            this.smoothing.set(obj, { x: obj.x, y: obj.y, a: obj.a });
        }
        const smoothed = this.smoothing.get(obj);
        const dx = obj.x - smoothed.x;
        const dy = obj.y - smoothed.y;
        if (Math.abs(dx) < 50) smoothed.x += dx * 0.3; else smoothed.x = obj.x;
        if (Math.abs(dy) < 50) smoothed.y += dy * 0.3; else smoothed.y = obj.y;
        return smoothed;
    }

    // same for the angle
    smoothPosAndAngle(obj) {
        const smoothed = this.smoothPos(obj);
        const da = obj.a - smoothed.a;
        if (Math.abs(da) < 1) smoothed.a += da * 0.3; else smoothed.a = obj.a;
        return smoothed;
    }

    // draw an object on both sides of a screen edge if necessary
    drawWrapped(x, y, size, draw) {
        const drawIt = (x, y) => {
            this.context.save();
            this.context.translate(x, y);
            draw();
            this.context.restore();
        }
        drawIt(x, y);
        // draw again on opposite sides if object is near edge
        if (x - size < 0) drawIt(x + 1000, y);
        if (x + size > 1000) drawIt(x - 1000, y);
        if (y - size < 0) drawIt(x, y + 1000);
        if (y + size > 1000) drawIt(x, y - 1000);
        if (x - size < 0 && y - size < 0) drawIt(x + 1000, y + 1000);
        if (x + size > 1000 && y + size > 1000) drawIt(x - 1000, y - 1000);
        if (x - size < 0 && y + size > 1000) drawIt(x + 1000, y - 1000);
        if (x + size > 1000 && y - size < 0) drawIt(x - 1000, y + 1000);
    }

    drawShip(ship, highlight) {
        this.context.beginPath();
        this.context.moveTo(+20,   0);
        this.context.lineTo(-20, +10);
        this.context.lineTo(-20, -10);
        this.context.closePath();
        this.context.stroke();
        if (highlight) {
            this.context.fill();
        }
        if (ship.forward) {
            this.context.moveTo(-20, +5);
            this.context.lineTo(-30,  0);
            this.context.lineTo(-20, -5);
            this.context.stroke();
        }
        if (ship.left) {
            this.context.moveTo(-18,  -9);
            this.context.lineTo(-13, -15);
            this.context.lineTo(-10,  -7);
            this.context.stroke();
        }
        if (ship.right) {
            this.context.moveTo(-18,  +9);
            this.context.lineTo(-13, +15);
            this.context.lineTo(-10,  +7);
            this.context.stroke();
        }
    }

    drawShipDebris(t) {
        this.context.beginPath();
        this.context.moveTo(+20 + t,   0 + t);
        this.context.lineTo(-20 + t, +10 + t);
        this.context.moveTo(-20 - t * 1.4, +10);
        this.context.lineTo(-20 - t * 1.4, -10);
        this.context.moveTo(-20 + t, -10 - t);
        this.context.lineTo(+20 + t,   0 - t);
        this.context.stroke();
    }

    drawAsteroid(size) {
        this.context.beginPath();
        this.context.moveTo(+size,  0);
        this.context.lineTo( 0, +size);
        this.context.lineTo(-size,  0);
        this.context.lineTo( 0, -size);
        this.context.closePath();
        this.context.stroke();
    }

    drawAsteroidDebris(size, t) {
        this.context.beginPath();
        this.context.moveTo(+size + t,  0 + t);
        this.context.lineTo( 0 + t, +size + t);
        this.context.moveTo(-size - t,  0 - t);
        this.context.lineTo( 0 - t, -size - t);
        this.context.moveTo(-size - t,  0 + t);
        this.context.lineTo( 0 - t, +size + t);
        this.context.moveTo(+size + t,  0 - t);
        this.context.lineTo( 0 + t, -size - t);
        this.context.stroke();
    }

    drawBlast() {
        this.context.beginPath();
        this.context.ellipse(0, 0, 2, 2, 0, 0, 2 * Math.PI);
        this.context.closePath();
        this.context.stroke();
    }
}

// The widget dock shows a QR code for the session
// URL so other players can join easily. Also, on shift+click
// it shows a stats page with debug info.
Multisynq.App.makeWidgetDock(); // creates a div with id="croquet_dock"

// This is the main entry point for the application
// it joins the session, resumes the simulation, and attaches the view to it
// The Multisynq API key here is a placeholder that will work on the local network
// but not on the public internet. Before deploying your own game, please create
// your own API key at multisynq.io/coder and use that instead.
Multisynq.Session.join({
    apiKey: '234567_Paste_Your_Own_API_Key_Here_7654321', // from multisynq.io/coder
    appId: 'io.multisynq.multiblaster-tutorial',          // choose an appId. No need to version it since code hashes are used for versioning
    model: Game, // the root model class
    view: Display, // the root view class
    // normally, a random session is created and the URL + QR code reflects that
    // uncomment the following two lines to force a specific named session without changing the URL
    // name: "my-session-name",             // e.g. "public"
    // password: "my-session-password",     // e.g. "none"
});
        </script>
    </body>
</html>
```

## Multisynq Types

Here are TypeScript definitions for the Multisynq API

```ts
declare module "@multisynq/client" {

    export type ClassId = string;

    export interface Class<T> extends Function {
        new (...args: any[]): T;
    }

    export type InstanceSerializer<T, IS> = {
        cls: Class<T>;
        write: (value: T) => IS;
        read: (state: IS) => T;
    }

    export type StaticSerializer<S> = {
        writeStatic: () => S;
        readStatic: (state: S) => void;
    }

    export type InstAndStaticSerializer<T, IS, S> = {
        cls: Class<T>;
        write: (value: T) => IS;
        read: (state: IS) => T;
        writeStatic: () => S;
        readStatic: (state: S) => void;
    }

    export type Serializer = InstanceSerializer<any, any> | StaticSerializer<any> | InstAndStaticSerializer<any, any, any>;

    export type SubscriptionHandler<T> = ((e: T) => void) | string;

    export abstract class PubSubParticipant<SubOptions> {
        publish<T>(scope: string, event: string, data?: T): void;
        subscribe<T>(scope: string, event: string | {event: string} | {event: string} & SubOptions, handler: SubscriptionHandler<T>): void;
        unsubscribe<T>(scope: string, event: string, handler?: SubscriptionHandler<T>): void;
        unsubscribeAll(): void;
    }

    export type FutureHandler<T extends any[]> = ((...args: T) => void) | string;

    export type QFuncEnv = Record<string, any>;

    export type EventType = {
        scope: string;
        event: string;
        source: "model" | "view";
    }

    /**
     * Models are synchronized objects in Multisynq.
     *
     * They are automatically kept in sync for each user in the same [session]{@link Session.join}.
     * Models receive input by [subscribing]{@link Model#subscribe} to events published in a {@link View}.
     * Their output is handled by views subscribing to events [published]{@link Model#publish} by a model.
     * Models advance time by sending messages into their [future]{@link Model#future}.
     *
     * ## Instance Creation and Initialization
     *
     * ### Do __NOT__ create a {@link Model} instance using `new` and<br>do __NOT__ override the `constructor`!
     *
     * To __create__ a new instance, use [create()]{@link Model.create}, for example:
     * ```
     * this.foo = FooModel.create({answer: 123});
     * ```
     * To __initialize__ an instance, override [init()]{@link Model#init}, for example:
     * ```
     * class FooModel extends Multisynq.Model {
     *     init(options={}) {
     *         this.answer = options.answer || 42;
     *     }
     * }
     * ```
     * The **reason** for this is that Models are only initialized by calling `init()`
     * the first time the object comes into existence in the session.
     * After that, when joining a session, the models are deserialized from the snapshot, which
     * restores all properties automatically without calling `init()`. A constructor would
     * be called all the time, not just when starting a session.
     *
     * @hideconstructor
     * @public
     */
    export class Model extends PubSubParticipant<{}> {
        id: string;

        /**
         * __Create an instance of a Model subclass.__
         *
         * The instance will be registered for automatical snapshotting, and is assigned an [id]{@link Model#id}.
         *
         * Then it will call the user-defined [init()]{@link Model#init} method to initialize the instance,
         * passing the {@link options}.
         *
         * **Note:** When your model instance is no longer needed, you must [destroy]{@link Model#destroy} it.
         * Otherwise it will be kept in the snapshot forever.
         *
         * **Warning**: never create a Model instance using `new`, or override its constructor. See [above]{@link Model}.
         *
         * Example:
         * ```
         * this.foo = FooModel.create({answer: 123});
         * ```
         *
         * @public
         * @param options - option object to be passed to [init()]{@link Model#init}.
         *     There are no system-defined options as of now, you're free to define your own.
         */
        static create<T extends typeof Model>(this: T, options?: any): InstanceType<T>;

        /**
         * __Registers this model subclass with Multisynq__
         *
         * It is necessary to register all Model subclasses so the serializer can recreate their instances from a snapshot.
         * Also, the [session id]{@link Session.join} is derived by hashing the source code of all registered classes.
         *
         * **Important**: for the hashing to work reliably across browsers, be sure to specify `charset="utf-8"` for your `<html>` or all `<script>` tags.
         *
         * Example
         * ```
         * class MyModel extends Multisynq.Model {
         *   ...
         * }
         * MyModel.register("MyModel")
         * ```
         *
         * @param classId Id for this model class. Must be unique. If you use the same class name in two files, use e.g. `"file1/MyModel"` and `"file2/MyModel"`.
         * @public
         */
        static register(classId:ClassId): void;

        /** Static version of [wellKnownModel()]{@link Model#wellKnownModel} for currently executing model.
         *
         * This can be used to emulate static accessors, e.g. for lazy initialization.
         *
         * __WARNING!__ Do not store the result in a static variable.
         * Like any global state, that can lead to divergence.
         *
         * Will throw an error if called from outside model code.
         *
         * Example:
         * ```
         * static get Default() {
         *     let default = this.wellKnownModel("DefaultModel");
         *     if (!default) {
         *         console.log("Creating default")
         *         default = MyModel.create();
         *         default.beWellKnownAs("DefaultModel");
         *     }
         *     return default;
         * }
         * ```
         */
        static wellKnownModel<M extends Model>(name: string): Model | undefined;

        /**
         * __Static declaration of how to serialize non-model classes.__
         *
         * The Multisynq snapshot mechanism only knows about {@link Model} subclasses.
         * If you want to store instances of non-model classes in your model, override this method.
         *
         * `types()` needs to return an Object that maps _names_ to _class descriptions_:
         * - the name can be any string, it just has to be unique within your app
         * - the class description can either be just the class itself (if the serializer should
         *   snapshot all its fields, see first example below), or an object with `write()` and `read()` methods to
         *   convert instances from and to their serializable form (see second example below).
         * - the serialized form answered by `write()` can be almost anything. E.g. if it answers an Array of objects
         *   then the serializer will be called for each of those objects. Conversely, these objects will be deserialized
         *   before passing the Array to `read()`.
         *
         * The types only need to be declared once, even if several different Model subclasses are using them.
         *
         * __NOTE:__ This is currently the only way to customize serialization (for example to keep snapshots fast and small).
         * The serialization of Model subclasses themselves can not be customized.
         *
         * Example: To use the default serializer just declare the class:</caption>
         * ```
         * class MyModel extends Multisynq.Model {
         *   static types() {
         *     return {
         *       "SomeUniqueName": MyNonModelClass,
         *       "THREE.Vector3": THREE.Vector3,        // serialized as '{"x":...,"y":...,"z":...}'
         *       "THREE.Quaternion": THREE.Quaternion,
         *     };
         *   }
         * }
         * ```
         *
         * Example: To define your own serializer, declare read and write functions:
         * ```
         * class MyModel extends Multisynq.Model {
         *   static types() {
         *     return {
         *       "THREE.Vector3": {
         *         cls: THREE.Vector3,
         *         write: v => [v.x, v.y, v.z],        // serialized as '[...,...,...]' which is shorter than the default above
         *         read: a => new THREE.Vector3(a[0], a[1], a[2]),
         *       },
         *       "THREE.Color": {
         *         cls: THREE.Color,
         *         write: color => '#' + color.getHexString(),
         *         read: state => new THREE.Color(state),
         *       },
         *     };
         *   }
         * }
         * ```
         * @public
         */
        static types(): Record<ClassId, Class<any> | Serializer>;


        /** Find classes inside an external library
         *
         * This recursivley traverses a dummy object and gathers all object classes found.
         * Returns a mapping that can be returned from a Model's static `types()` method.
         *
         * This can be used to gather all internal class types of a third party library
         * that otherwise would not be accessible to the serializer
         *
         * Example: If `Foo` is a class from a third party library
         *   that internally create a `Bar` instance,
         *   this would find both classes
         * ```
         * class Bar {}
         * class Foo { bar = new Bar(); }
         * static types() {
         *    const sample = new Foo();
         *    return this.gatherClassTypes(sample, "MyLib");
         *    // returns { "MyLib.Foo": Foo, "MyLib.Bar": Bar }
         * }
         * ```
         * @param {Object} dummyObject - an instance of a class from the library
         * @param {String} prefix - a prefix to add to the class names
         * @since 2.0
         */
        static gatherClassTypes<T extends Object>(dummyObject: T, prefix: string): Record<ClassId, Class<any>>;

        /**
         * This is called by [create()]{@link Model.create} to initialize a model instance.
         *
         * In your Model subclass this is the place to [subscribe]{@link Model#subscribe} to events,
         * or start a [future]{@link Model#future} message chain.
         *
         * **Note:** When your model instance is no longer needed, you must [destroy]{@link Model#destroy} it.
         *
         * @param options - there are no system-defined options, you're free to define your own
         * @public
         */
        init(_options: any, persistentData?: any): void;

        /**
         * Unsubscribes all [subscriptions]{@link Model#subscribe} this model has,
         * unschedules all [future]{@link Model#future} messages,
         * and removes it from future snapshots.
         *
         * Example:
         * ```
         * removeChild(child) {
         *    const index = this.children.indexOf(child);
         *    this.children.splice(index, 1);
         *    child.destroy();
         * }
         * ```
         * @public
         */
        destroy(): void;

        /**
         * **Publish an event to a scope.**
         *
         * Events are the main form of communication between models and views in Multisynq.
         * Both models and views can publish events, and subscribe to each other's events.
         * Model-to-model and view-to-view subscriptions are possible, too.
         *
         * See [Model.subscribe]{@link Model#subscribe}() for a discussion of **scopes** and **event names**.
         * Refer to [View.subscribe]{@link View#subscribe}() for invoking event handlers *asynchronously* or *immediately*.
         *
         * Optionally, you can pass some **data** along with the event.
         * For events published by a model, this can be any arbitrary value or object.
         * See View's [publish]{@link View#publish} method for restrictions in passing data from a view to a model.
         *
         * If you subscribe inside the model to an event that is published by the model,
         * the handler will be called immediately, before the publish method returns.
         * If you want to have it handled asynchronously, you can use a future message:
         * `this.future(0).publish("scope", "event", data)`.
         *
         * Note that there is no way of testing whether subscriptions exist or not (because models can exist independent of views).
         * Publishing an event that has no subscriptions is about as cheap as that test would be, so feel free to always publish,
         * there is very little overhead.
         *
         * Example:
         * ```
         * this.publish("something", "changed");
         * this.publish(this.id, "moved", this.pos);
         * ```
         * @param {String} scope see [subscribe]{@link Model#subscribe}()
         * @param {String} event see [subscribe]{@link Model#subscribe}()
         * @param {*=} data can be any value or object
         * @public
         */
        publish<T>(scope: string, event: string, data?: T): void;

        /**
         * **Register an event handler for an event published to a scope.**
         *
         * Both `scope` and `event` can be arbitrary strings.
         * Typically, the scope would select the object (or groups of objects) to respond to the event,
         * and the event name would select which operation to perform.
         *
         * A commonly used scope is `this.id` (in a model) and `model.id` (in a view) to establish
         * a communication channel between a model and its corresponding view.
         *
         * You can use any literal string as a global scope, or use [`this.sessionId`]{@link Model#sessionId} for a
         * session-global scope (if your application supports multipe sessions at the same time).
         * The predefined events [`"view-join"`]{@link event:view-join} and [`"view-exit"`]{@link event:view-exit}
         * use this session scope.
         *
         * The handler must be a method of `this`, e.g. `subscribe("scope", "event", this.methodName)` will schedule the
         * invocation of `this["methodName"](data)` whenever `publish("scope", "event", data)` is executed.
         *
         * If `data` was passed to the [publish]{@link Model#publish} call, it will be passed as an argument to the handler method.
         * You can have at most one argument. To pass multiple values, pass an Object or Array containing those values.
         * Note that views can only pass serializable data to models, because those events are routed via a reflector server
         * (see [View.publish]{@link View#publish}).
         *
         * Example:
         * ```
         * this.subscribe("something", "changed", this.update);
         * this.subscribe(this.id, "moved", this.handleMove);
         * ```
         * Example:
         * ```
         * class MyModel extends Multisynq.Model {
         *   init() {
         *     this.subscribe(this.id, "moved", this.handleMove);
         *   }
         *   handleMove({x,y}) {
         *     this.x = x;
         *     this.y = y;
         *   }
         * }
         * class MyView extends Multisynq.View {
         *   constructor(model) {
         *     this.modelId = model.id;
         *   }
         *   onpointermove(evt) {
         *      const x = evt.x;
         *      const y = evt.y;
         *      this.publish(this.modelId, "moved", {x,y});
         *   }
         * }
         * ```
         * @param {String} scope - the event scope (to distinguish between events of the same name used by different objects)
         * @param {String} event - the event name (user-defined or system-defined)
         * @param {Function|String} handler - the event handler (must be a method of `this`, or the method name as string)
         * @return {this}
         * @public
         */
        subscribe<T>(scope: string, event: string, handler: SubscriptionHandler<T>): void;

        /**
         * Unsubscribes this model's handler for the given event in the given scope.
         * @param {String} scope see [subscribe]{@link Model#subscribe}
         * @param {String} event see [subscribe]{@link Model#subscribe}
         * @param {Function=} handler - the event handler (if not given, all handlers for the event are removed)
         * @public
         */
        unsubscribe<T>(scope: string, event: string, handler?: SubscriptionHandler<T>): void;

        /**
         * Unsubscribes all of this model's handlers for any event in any scope.
         * @public
         */
        unsubscribeAll(): void;

        /**
         * Scope, event, and source of the currently executing subscription handler.
         *
         * The `source' is either `"model"` or `"view"`.
         *
         * ```
         * // this.subscribe("*", "*", this.logEvents)
         * logEvents(data: any) {
         *     const {scope, event, source} = this.activeSubscription!;
         *     console.log(`${this.now()} Event in model from ${source} ${scope}:${event} with`, data);
         * }
         * ```
         * @returns {Object} `{scope, event, source}` or `undefined` if not in a subscription handler.
         * @since 2.0
         * @public
         */
        get activeSubscription(): EventType | undefined;

        /**
         * **Schedule a message for future execution**
         *
         * Use a future message to automatically advance time in a model,
         * for example for animations.
         * The execution will be scheduled `tOffset` milliseconds into the future.
         * It will run at precisely `this.now() + tOffset`.
         *
         * Use the form `this.future(100).methodName(args)` to schedule the execution
         * of `this.methodName(args)` at time `this.now() + tOffset`.
         *
         * **Hint**: This would be an unusual use of `future()`, but the `tOffset` given may be `0`,
         * in which case the execution will happen asynchronously before advancing time.
         * This is the only way for asynchronous execution in the model since you must not
         * use Promises or async functions in model code (because a snapshot may happen at any time
         * and it would not capture those executions).
         *
         * **Note:** the recommended form given above is equivalent to `this.future(100, "methodName", arg1, arg2)`
         * but makes it more clear that "methodName" is not just a string but the name of a method of this object.
         * Also, this will survive minification.
         * Technically, it answers a [Proxy]{@link https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/Proxy}
         * that captures the name and arguments of `.methodName(args)` for later execution.
         *
         * See this [tutorial]{@tutorial 1_1_hello_world} for a complete example.
         *
         * Example: single invocation with two arguments
         * ```
         * this.future(3000).say("hello", "world");
         * ```
         * Example: repeated invocation with no arguments
         * ```
         * tick() {
         *     this.n++;
         *     this.publish(this.id, "count", {time: this.now(), count: this.n)});
         *     this.future(100).tick();
         * }
         * ```
         * @param {Number} tOffset - time offset in milliseconds, must be >= 0
         * @returns {this}
         * @public
         */
        future<T extends any[]>(tOffset?:number, method?: FutureHandler<T>, ...args: T): this;

        /**
         * **Cancel a previously scheduled future message**
         *
         * This unschedules the invocation of a message that was scheduled with [future]{@link Model#future}.
         * It is okay to call this method even if the message was already executed or if it was never scheduled.
         *
         * **Note:** as with [future]{@link Model#future}, the recommended form is to pass the method itself,
         * but you can also pass the name of the method as a string.
         *
         * @example
         * this.future(3000).say("hello", "world");
         * ...
         * this.cancelFuture(this.say);
         * @param {Function} method - the method (must be a method of `this`)
         * @returns {Boolean} true if the message was found and canceled, false otherwise
         * @since 1.1.0-16
         * @public
        */
        cancelFuture<T extends any[]>(method: FutureHandler<T>): boolean;

        /** **Generate a synchronized pseudo-random number**
         *
         * This returns a floating-point, pseudo-random number in the range 0–1 (inclusive of 0, but not 1)
         * with approximately uniform distribution over that range
         * (just like [Math.random](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/random)).
         *
         * Since the model computation is synchronized for every user on their device, the sequence of random numbers
         * generated must also be exactly the same for everyone. This method provides access to such a random number generator.
        */
        random(): number;

        /** **The model's current time**
         *
         * Every [event handler]{@link Model#subscribe} and [future message]{@link Model#future} is run at a precisely defined moment
         * in virtual model time, and time stands still while this execution is happening. That means if you were to access this.now()
         * in a loop, it would never answer a different value.
         *
         * The unit of now is milliseconds (1/1000 second) but the value can be fractional, it is a floating-point value.
         */
        now(): number;

        /** Make this model globally accessible under the given name. It can be retrieved from any other model in the same session
         * using [wellKnownModel()]{@link Model#wellKnownModel}.
         *
         * Hint: Another way to make a model well-known is to pass a name as second argument to [Model.create()]{@link Model#create}.
         *
         * Example:
         * ```
         *  class FooManager extends Multisynq.Model {
         *      init() {
         *          this.beWellKnownAs("UberFoo");
         *      }
         *  }
         *  class Underlings extends Multisynq.Model {
         *      reportToManager(something) {
         *          this.wellKnownModel("UberFoo").report(something);
         *      }
         *  }
         * ```*/
        beWellKnownAs(name: string): void;

        /** Access a model that was registered previously using beWellKnownAs().
         *
         * Note: The instance of your root Model class is automatically made well-known as `"modelRoot"`
         * and passed to the [constructor]{@link View#constructor} of your root View during [Session.join]{@link Session.join}.
         *
         * Example:
         * ```
         * const topModel = this.wellKnownModel("modelRoot");
         * ```
         */
        wellKnownModel<M extends Model>(name: string): Model | undefined;


        /** Look up a model in the current session given its `id`.
         *
         * Example:
         * ```
         * const otherModel = this.getModel(otherId);
         * ```
         */
        getModel<M extends Model>(id: string): M | undefined;

        /** This methods checks if it is being called from a model, and throws an Error otherwise.
         *
         * Use this to protect some model code against accidentally being called from a view.
         *
         * Example:
         * ```
         * get foo() { return this._foo; }
         * set foo(value) { this.modelOnly(); this._foo = value; }
         * ```*/
        modelOnly(errorMessage?: string): boolean;

        /**
         * **Create a serializable function that can be stored in the model.**
         *
         * Plain functions can not be serialized because they may contain closures that can
         * not be introspected by the snapshot mechanism. This method creates a serializable
         * "QFunc" from a regular function. It can be stored in the model and called like
         * the original function.
         *
         * The function can only access global references (like classes), *all local
         * references must be passed in the `env` object*. They are captured
         * as constants at the time the QFunc is created. Since they are constants,
         * re-assignments will throw an error.
         *
         * In a fat-arrow function, `this` is bound to the model that called `createQFunc`,
         * even in a different lexical scope. It is okay to call a model's `createQFunc` from
         * anywhere, e.g. from a view. QFuncs can be passed from view to model as arguments
         * in `publish()` (provided their environment is serializable).
         *
         * **Warning:** Minification can change the names of local variables and functions,
         * but the env will still use the unminified names. You need to disable
         * minification for source code that creates QFuncs with env. Alternatively, you can
         * pass the function's source code as a string, which will not be minified.
         *
         * Behind the scenes, the function is stored as a string and compiled when needed.
         * The env needs to be constant because the serializer would not able to capture
         * the values if they were allowed to change.
         *
         * @example
         * const template = { greeting: "Hi there," };
         * this.greet = this.createQFunc({template}, (name) => console.log(template.greeting, name));
         * this.greet(this, "friend"); // logs "Hi there, friend"
         * template.greeting = "Bye now,";
         * this.greet(this, "friend"); // logs "Bye now, friend"
         *
         * @param env - an object with references used by the function
         * @param func - the function to be wrapped, or a string with the function's source code
         * @returns a serializable function bound to the given environment
         * @public
         * @since 2.0
         */
        createQFunc<T extends Function>(env: QFuncEnv, func: T|string): T;
        createQFunc<T extends Function>(func: T|string): T;

        persistSession(func: () => any): void;

        /** **Identifies the shared session of all users**
         *
         * (as opposed to the [viewId]{@link View#viewId} which identifies the non-shared views of each user).
         *
         * The session id is used as "global" scope for events like [`"view-join"`]{@link event:view-join}.
         *
         * See {@link Session.join} for how the session id is generated.
         *
         * If your app has several sessions at the same time, each session id will be different.
         *
         * Example
         * ```
         * this.subscribe(this.sessionId, "view-join", this.addUser);
         * ```*/
        sessionId: string;

        /** **The number of users currently in this session.**
         *
         * All users in a session share the same Model (meaning all model objects) but each user has a different View
         * (meaning all the non-model state). This is the number of views currently sharing this model.
         * It increases by 1 for every [`"view-join"`]{@link event:view-join}
         * and decreases by 1 for every [`"view-exit"`]{@link event:view-exit} event.
         *
         * Example
         * ```
         * this.subscribe(this.sessionId, "view-join", this.showUsers);
         * this.subscribe(this.sessionId, "view-exit", this.showUsers);
         * showUsers() { this.publish(this.sessionId, "view-count", this.viewCount); }
         * ```*/
        viewCount: number;

        /** make module exports accessible via any subclass */
        static Multisynq: Multisynq;
    }

    export type ViewLocation = {
        country: string;
        region: string;
        city?: {
            name: string;
            lat: number;
            lng: number;
        }
    }

    /** payload of view-join and view-exit if viewData was passed in Session.join */
    export type ViewInfo<T> = {
        viewId: string              // set by reflector
        viewData: T                 // passed in Session.join
        location?: ViewLocation     // set by reflector if enabled in Session.join
    }

    export type ViewSubOptions = {
        handling?: "queued" | "oncePerFrame" | "immediate"
    }

    export class View extends PubSubParticipant<ViewSubOptions> {
        /**
         * A View instance is created in {@link Session.join}, and the root model is passed into its constructor.
         *
         * This inherited constructor does not use the model in any way.
         * Your constructor should recreate the view state to exactly match what is in the model.
         * It should also [subscribe]{@link View#subscribe} to any changes published by the model.
         * Typically, a view would also subscribe to the browser's or framework's input events,
         * and in response [publish]{@link View#publish} events for the model to consume.
         *
         * The constructor will, however, register the view and assign it an [id]{@link View#id}.
         *
         * **Note:** When your view instance is no longer needed, you must [detach]{@link View#detach} it.
         * Otherwise it will be kept in memory forever.
         *
         * @param {Model} model - the view's model
         * @public
         */
        constructor(model: Model);

        /**
         * **Unsubscribes all [subscriptions]{@link View#subscribe} this model has,
         * and removes it from the list of views**
         *
         * This needs to be called when a view is no longer needed to prevent memory leaks.
         *
         * Example:
         * ```
         * removeChild(child) {
         *    const index = this.children.indexOf(child);
         *    this.children.splice(index, 1);
         *    child.detach();
         * }
         * ```
         * @public
         */
        detach(): void;

        /**
         * **Publish an event to a scope.**
         *
         * Events are the main form of communication between models and views in Multisynq.
         * Both models and views can publish events, and subscribe to each other's events.
         * Model-to-model and view-to-view subscriptions are possible, too.
         *
         * See [Model.subscribe]{@link Model#subscribe} for a discussion of **scopes** and **event names**.
         *
         * Optionally, you can pass some **data** along with the event.
         * For events published by a view and received by a model,
         * the data needs to be serializable, because it will be sent via the reflector to all users.
         * For view-to-view events it can be any value or object.
         *
         * Note that there is no way of testing whether subscriptions exist or not (because models can exist independent of views).
         * Publishing an event that has no subscriptions is about as cheap as that test would be, so feel free to always publish,
         * there is very little overhead.
         *
         * Example:
         * ```
         * this.publish("input", "keypressed", {key: 'A'});
         * this.publish(this.model.id, "move-to", this.pos);
         * ```
         * @param {String} scope see [subscribe]{@link Model#subscribe}()
         * @param {String} event see [subscribe]{@link Model#subscribe}()
         * @param {*=} data can be any value or object (for view-to-model, must be serializable)
         * @public
         */
        publish<T>(scope: string, event: string, data?: T): void;

        /**
         * **Register an event handler for an event published to a scope.**
         *
         * Both `scope` and `event` can be arbitrary strings.
         * Typically, the scope would select the object (or groups of objects) to respond to the event,
         * and the event name would select which operation to perform.
         *
         * A commonly used scope is `this.id` (in a model) and `model.id` (in a view) to establish
         * a communication channel between a model and its corresponding view.
         *
         * Unlike in a model's [subscribe]{@link Model#subscribe} method, you can specify when the event should be handled:
         * - **Queued:** The handler will be called on the next run of the [main loop]{@link Session.join},
         *   the same number of times this event was published.
         *   This is useful if you need each piece of data that was passed in each [publish]{@link Model#publish} call.
         *
         *   An example would be log entries generated in the model that the view is supposed to print.
         *   Even if more than one log event is published in one render frame, the view needs to receive each one.
         *
         *   **`{ event: "name", handling: "queued" }` is the default.  Simply specify `"name"` instead.**
         *
         * - **Once Per Frame:** The handler will be called only _once_ during the next run of the [main loop]{@link Session.join}.
         *   If [publish]{@link Model#publish} was called multiple times, the handler will only be invoked once,
         *   passing the data of only the last `publish` call.
         *
         *   For example, a view typically would only be interested in the current position of a model to render it.
         *   Since rendering only happens once per frame, it should subscribe using the `oncePerFrame` option.
         *   The event typically would be published only once per frame anyways, however,
         *   while the model is catching up when joining a session, this would be fired rapidly.
         *
         *   **`{ event: "name", handling: "oncePerFrame" }` is the most efficient option, you should use it whenever possible.**
         *
         * - **Immediate:** The handler will be invoked _synchronously_ during the [publish]{@link Model#publish} call.
         *   This will tie the view code very closely to the model simulation, which in general is undesirable.
         *   However, if the view needs to know the exact state of the model at the time the event was published,
         *   before execution in the model proceeds, then this is the facility to allow this without having to copy model state.
         *
         *   Pass `{event: "name", handling: "immediate"}` to enforce this behavior.
         *
         * The `handler` can be any callback function.
         * Unlike a model's [handler]{@link Model#subscribe} which must be a method of that model,
         * a view's handler can be any function, including fat-arrow functions declared in-line.
         * Passing a method like in the model is allowed too, it will be bound to `this` in the subscribe call.
         *
         * Example:
         * ```
         * this.subscribe("something", "changed", this.update);
         * this.subscribe(this.id, {event: "moved", handling: "oncePerFrame"}, pos => this.sceneObject.setPosition(pos.x, pos.y, pos.z));
         * ```
         * @tutorial 1_4_view_smoothing
         * @param {String} scope - the event scope (to distinguish between events of the same name used by different objects)
         * @param {String|Object} eventSpec - the event name (user-defined or system-defined), or an event handling spec object
         * @param {String} eventSpec.event - the event name (user-defined or system-defined)
         * @param {String} eventSpec.handling - `"queued"` (default), `"oncePerFrame"`, or `"immediate"`
         * @param {Function} handler - the event handler (can be any function)
         * @return {this}
         * @public
         */
        subscribe(scope: string, eventSpec: string | {event: string, handling: "queued" | "oncePerFrame" | "immediate"}, callback: (e: any) => void): void;

        /**
         * Unsubscribes this view's handler for the given event in the given scope.
         * @param {String} scope see [subscribe]{@link View#subscribe}
         * @param {String} event see [subscribe]{@link View#subscribe}
         * @param {Function=} handler - the event handler (if not given, all handlers for the event are removed)
         * @public
         */
        unsubscribe<T>(scope: string, event: string, handler?: SubscriptionHandler<T>): void;

        /**
         * Unsubscribes all of this views's handlers for any event in any scope.
         * @public
         */
        unsubscribeAll(): void;

        /**
         * Scope, event, and source of the currently executing subscription handler.
         *
         * The `source' is either `"model"` or `"view"`.
         *`
         * ```
         * // this.subscribe("*", "*", this.logEvents)
         * logEvents(data: any) {
         *     const {scope, event, source} = this.activeSubscription;
         *     console.log(`Event in view from ${source} ${scope}:${event} with`, data);
         * }
         * ```
         * @returns {Object} `{scope, event, source}` or `undefined` if not in a subscription handler.
         * @since 2.0
         * @public
         */
        get activeSubscription(): EventType | undefined;

        /**
         * The ID of the view.
         * @public
         */
        viewId: string;

        /** **Schedule a message for future execution**
         * This method is here for symmetry with [Model.future]{@link Model#future}.
         *
         * It simply schedules the execution using `window.setTimeout`.
         * The only advantage to using this over setTimeout() is consistent style.
         */
        future(tOffset: number): this;

        /** **Answers `Math.random()`**
         *
         * This method is here purely for symmetry with [Model.random]{@link Model#random}.
         */
        random(): number;

        /** **The model's current time**
         *
         * This is the time of how far the model has been simulated.
         * Normally this corresponds roughly to real-world time, since the reflector is generating
         * time stamps based on real-world time.
         *
         * If there is [backlog]{@link View#externalNow} however (e.g while a newly joined user is catching up),
         * this time will advance much faster than real time.
         *
         * The unit is milliseconds (1/1000 second) but the value can be fractional, it is a floating-point value.
         *
         * Returns: the model's time in milliseconds since the first user created the session.
        */
        now(): number;

        /** **The latest timestamp received from reflector**
         *
         * Timestamps are received asynchronously from the reflector at the specified tick rate.
         * [Model time]{@View#now} however only advances synchronously on every iteration of the [main loop]{@link Session.join}.
         * Usually `now == externalNow`, but if the model has not caught up yet, then `now < externalNow`.
         *
         * We call the difference "backlog". If the backlog is too large, Multisynq will put an overlay on the scene,
         * and remove it once the model simulation has caught up. The `"synced"` event is sent when that happens.
         *
         * The `externalNow` value is rarely used by apps but may be useful if you need to synchronize views to real-time.
         *
         * Example:
         * ```
         * const backlog = this.externalNow() - this.now();
         * ```
        */
        externalNow(): number;

        /**
         * **The model time extrapolated beyond latest timestamp received from reflector**
         *
         * Timestamps are received asynchronously from the reflector at the specified tick rate.
         * In-between ticks or messages, neither [now()]{@link View#now} nor [externalNow()]{@link View#externalNow} advances.
         * `extrapolatedNow` is `externalNow` plus the local time elapsed since that timestamp was received,
         * so it always advances.
         *
         * `extrapolatedNow()` will always be >= `now()` and `externalNow()`.
         * However, it is only guaranteed to be monotonous in-between time stamps received from the reflector
         * (there is no "smoothing" to reconcile local time with reflector time).
         */
        extrapolatedNow(): number;

        /** Called on the root view from [main loop]{@link Session.join} once per frame. Default implementation does nothing.
         *
         * Override to add your own view-side input polling, rendering, etc.
         *
         * If you want this to be called for other views than the root view,
         * you will have to call those methods from the root view's `update()`.
         *
         * The time received is related to the local real-world time. If you need to access the model's time, use [this.now()]{@link View#now}.
        */
        update(time: number): void;

        /** Access a model that was registered previously using beWellKnownAs().
         *
         * Note: The instance of your root Model class is automatically made well-known as `"modelRoot"`
         * and passed to the [constructor]{@link View#constructor} of your root View during [Session.join]{@link Session.join}.
         *
         * Example:
         * ```
         * const topModel = this.wellKnownModel("modelRoot");
         * ```
         */
        wellKnownModel<M extends Model>(name: string): Model | undefined;

        /** Access the session object.
         *
         * Note: The view instance may be taken down and reconstructed during the lifetime of a session. the `view` property of the session may differ from `this`, when you store the view instance in our data structure outside of Multisynq and access it sometime later.
         * @public
         */

        get session(): MultisynqSession<View>;

        /** make module exports accessible via any subclass */
        static Multisynq: Multisynq;
    }

    export type MultisynqSession<V extends View> = {
        id: string,
        view: V,
        step: (time: number) => void,
        leave: () => Promise<void>,
        data: {
            fetch: (handle: DataHandle) => Promise<ArrayBuffer>,
            store: (data: ArrayBuffer, options?: { shareable?: boolean, keep?: boolean }) => Promise<DataHandle>
            toId: (handle: DataHandle) => string,
            fromId: (id: string) => DataHandle,
        }
    }

    export type MultisynqModelOptions = object;
    export type MultisynqViewOptions = object;

    export type MultisynqDebugOption =
        "session" | "messages" | "sends" | "snapshot" |
        "data" | "hashing" | "subscribe" | "publish" | "events" | "classes" | "ticks" |
        "write" | "offline";

    type ClassOf<M> = new (...args: any[]) => M;

    export type MultisynqSessionParameters<M extends Model, V extends View, T> = {
        apiKey?: string,
        appId: string,
        name?: string|Promise<string>,
        password?: string|Promise<string>,
        model: ClassOf<M>,
        view?: ClassOf<V>,
        options?: MultisynqModelOptions,
        viewOptions?: MultisynqViewOptions,
        viewData?: T,
        location?: boolean,
        step?: "auto" | "manual",
        tps?: number|string,
        autoSleep?: number|boolean,
        rejoinLimit?: number,
        eventRateLimit?: number,
        reflector?: string,
        files?: string,
        box?: string,
        debug?: MultisynqDebugOption | Array<MultisynqDebugOption>
    }

    /**
     * The Session is the entry point for a Multisynq App.
     *
     * @hideconstructor
     * @public
     */
    export class Session {

        /**
         * **Join a Multisynq session.**
         *
         */
        static join<M extends Model, V extends View, T>(
            parameters: MultisynqSessionParameters<M, V, T>
        ): Promise<MultisynqSession<V>>;
    }

    export var Constants: object;

    export const VERSION: string;

    interface IApp {
	sessionURL:string;
	root:HTMLElement|null;
	sync:boolean;
	messages:boolean;
	badge:boolean;
	stats:boolean;
	qrcode:boolean;
	makeWidgetDock(options?:{debug?:boolean, iframe?:boolean, badge?:boolean, qrcode?:boolean, stats?:boolean, alwaysPinned?:boolean, fixedSize?:boolean}):void;
	makeSessionWidgets(sessionId:string):void;
	makeQRCanvas(options?:{text?:string, width?:number, height?:number, colorDark?:string, colorLight?:string, correctLevel?:("L"|"M"|"Q"|"H")}):any;
	clearSessionMoniker():void;
	showSyncWait(bool:boolean):void;
	messageFunction(msg:string, options?:{
	    duration?:number,
	    gravity?:("bottom"|"top"),
	    position?:("right"|"left"|"center"|"bottom"),
	    backgroundColor?:string,
	    stopOnFocus?:boolean
	}):void;
	showMessage(msg:string, options?:any):void;
	isMultisynqHost(hostname:string):boolean;
	referrerURL():string;
        autoSession:(name:string) => Promise<string>;
        autoPassword:(options?:{key?:string, scrub:boolean, keyless:boolean}) => Promise<string>;
        randomSession:(len?:number) => string;
        randomPassword:(len?:number) => string;
    }

    /**
     * The App API is under construction.
     *
     * @public
     */

    export var App:IApp;


    interface DataHandle {
	store(sessionId:string, data:(string|ArrayBuffer), keep?:boolean):Promise<DataHandle>;
	fetch(sessionid:string, handle:DataHandle):string|ArrayBuffer;
	hash(data:((...arg:any) => void|string|DataView|ArrayBuffer), output?:string):string;
    }

    /**
     * The Data API is under construction.
     *
     * @public
     */

    export var Data: DataHandle;

    type Multisynq = {
        Model: typeof Model,
        View: typeof View,
        Session: typeof Session,
        Data: DataHandle,
        Constants: typeof Constants,
        App: IApp,
    }

}
```

