# How I Used Temporal and Kubernetes in The Project

I built a multiplayer Mafia game as part of a learning exercise. The game has multiple phases — Night, Voting, Discussion — each with a countdown timer. Players take actions, the timer runs out, and the game moves to the next phase automatically.

Simple enough on the surface. But getting this to work **reliably** — without timers dying, without the game freezing, without data disappearing when a service restarts — that's where Temporal and Kubernetes came in.

---

## The Problem

The game runs across 5 separate services:

- **Frontend** — what players see in the browser
- **Gateway** — the entry point that handles login and routes requests
- **Game Engine** — the brain, handles game rules and phase logic
- **Event Service** — manages the countdown timers
- **MongoDB** — stores all game data

Each service is its own program running in its own container.

### Problem 1: Timers that forget everything

The countdown timer for each phase was a simple background task running inside the Event Service. When the timer hit zero, it would call the Game Engine to move to the next phase.

This worked fine — until the Event Service restarted (due to a crash, a deployment, anything). The moment it restarted, **every active timer was gone**. No record they ever existed. Players would be stuck in a phase forever with no way out.

### Problem 2: Who is the host?

The Gateway kept track of which player created each room (the "host") in a simple in-memory variable:

```
_ROOM_HOSTS = { "room123": "Likhith" }
```

If the Gateway restarted, that variable reset to empty. Suddenly no one was the host. Clicking "Start Game" would fail with a cryptic error.

---

## Solution 1: Temporal

### What is Temporal?

Temporal is a tool that lets you write code that **survives crashes**.

Normally, if your program crashes mid-execution, whatever it was doing is lost. Temporal solves this by saving the progress of your code as it runs. If the program crashes, Temporal replays the code from where it left off when it restarts.

Think of it like a video game save point — except it saves automatically at every step.

### How I used it for the phase timer

Instead of a plain background task, I wrote the timer as a Temporal Workflow:

```
Start PhaseTimerWorkflow
  → Sleep for 30 seconds        ← Temporal saves progress here
  → Call AdvancePhase           ← Temporal retries this if it fails
```

The key difference: `workflow.Sleep(30 seconds)` is not a normal sleep. It is a **durable** sleep. If the Event Service crashes at second 18, Temporal knows the timer had 12 seconds left. When the service restarts, it resumes the sleep with 12 seconds remaining. The players never notice anything happened.

If the call to advance the phase fails (say the Game Engine is briefly down), Temporal automatically retries it — no manual intervention needed.

Every timer is also visible in the **Temporal UI** — a web dashboard where you can see every active timer, how long is left, and the full history of what happened.

**Before Temporal:**
- Service restarts → timers lost → game freezes
- HTTP call fails → phase never advances, silently

**After Temporal:**
- Service restarts → timers resume automatically
- HTTP call fails → Temporal retries until it succeeds

### How I used it for the host problem

I also replaced the in-memory host variable with a Temporal Workflow — one per room. This workflow holds the room state (who the host is, what phase the game is in) and keeps it alive for the entire duration of the game.

Because Temporal saves its state to a database (PostgreSQL), the room information survives Gateway restarts. Any service can ask Temporal "who is the host of room 123?" at any time and get a correct answer.

---

## Solution 2: Kubernetes

### What is Kubernetes?

Kubernetes is a tool that **manages your containers** for you.

With Docker Compose (what I used before), you start all your containers with one command. But if a container crashes, you have to notice it and restart it yourself. Docker Compose has no awareness of whether your app is healthy.

Kubernetes watches your containers 24/7. If one crashes, Kubernetes restarts it automatically. If you tell Kubernetes "I want 3 copies of the Gateway running", it will always maintain exactly 3 — replacing any that die.

### How I used it

I described my entire application to Kubernetes as a set of configuration files (called manifests). Each service has a manifest that says:

- What container image to run
- What environment variables it needs
- How to check if it's healthy
- How many copies to run

For example, the Gateway manifest tells Kubernetes:
- Run 1 copy of the Gateway container
- Pass in the JWT secret from a secure store (called a Secret)
- Check `/health` every 5 seconds — if it stops responding, restart it

Kubernetes also handles **startup ordering**. Temporal needs its database (PostgreSQL) to be ready before it starts. The Event Service needs Temporal to be ready before it starts. In Kubernetes, I configured each service to wait for its dependencies using `initContainers` — small tasks that run before the main container starts:

```
Event Service startup:
  1. Wait until Temporal is reachable on port 7233   ← initContainer
  2. Start the actual Event Service                   ← main container
```

This means even if everything starts at the same time, each service waits for what it needs.

### Secrets management

Before Kubernetes, sensitive config (database passwords, JWT secrets) lived in a `.env` file on my machine. Kubernetes has a built-in **Secrets** resource — a secure store for sensitive values. Containers read from Secrets at runtime, so credentials are never hardcoded or stored in source code.

**Before Kubernetes:**
- Container crashes → manually restart it
- Sensitive config in `.env` files
- No way to check if services are healthy

**After Kubernetes:**
- Container crashes → Kubernetes restarts it automatically
- Sensitive config in Kubernetes Secrets
- Health checks on every service, automatic recovery

---

## How They Work Together

Temporal and Kubernetes solve different but complementary problems:

| | Temporal | Kubernetes |
|---|---|---|
| **What it manages** | Your application logic (workflows, timers) | Your containers (infrastructure) |
| **What it survives** | Crashes mid-workflow, failed HTTP calls | Container crashes, restarts |
| **What it gives you** | Durable execution, automatic retries, visibility | Self-healing, health checks, secret management |

In the project:

- Kubernetes makes sure all 7 services are always running
- Temporal makes sure the game logic (timers, room state) is always correct

Neither one replaces the other. Kubernetes keeps the containers alive. Temporal keeps the application logic alive.

---

## The Setup Locally

To run the full stack on a laptop, I used **Minikube** — a tool that creates a Kubernetes cluster inside Docker on your local machine. One command:

```
minikube start --driver=docker --memory=6144 --cpus=4
```

Then I applied all the configuration files:

```
kubectl apply -f ./k8s/manifests/
```

And the entire application — all 7 services including Temporal and its database — spun up on my laptop. The Temporal UI was available at a local URL where I could watch every game's timer workflow in real time.

---

## What I Took Away

**Temporal** changed how I think about background tasks. Before, I would write a timer and just hope it doesn't crash. Now I think about every long-running task as a workflow with checkpoints — something that can be paused, resumed, and retried without losing progress.

**Kubernetes** changed how I think about deployment. Before, deploying meant "run docker-compose up and hope nothing crashes". Now the infrastructure takes care of itself — health checks, restarts, ordered startup — and I can focus on the application.

Both tools have a learning curve. But once they click, it's hard to imagine going back.

---

*Built while exploring distributed systems and cloud-native architecture.*
