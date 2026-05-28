# How Temporal and Kubernetes Were Used in This Project

A multiplayer Mafia game was built to explore distributed systems in a real, working application. The game has multiple phases: Night, Voting, Discussion, each with a countdown timer. Players take actions, the timer runs out, and the game moves to the next phase automatically.

Simple enough on the surface. But getting this to work reliably, without timers dying, without the game freezing, without data disappearing when a service restarts, that's where Temporal and Kubernetes came in.

## The Problem

The game runs across 5 separate services:

**Frontend** handles what players see in the browser. **Gateway** is the entry point that handles login and routes requests. **Game Engine** is the brain that handles game rules and phase logic. **Event Service** manages the countdown timers. **MongoDB** stores all game data.

Each service is its own program running in its own container.

### Problem 1: Timers that forget everything

The countdown timer for each phase was a simple background task running inside the Event Service. When the timer hit zero, it would call the Game Engine to move to the next phase.

This worked fine until the Event Service restarted due to a crash or a deployment. The moment it restarted, every active timer was gone. No record they ever existed. Players would be stuck in a phase forever with no way out.

### Problem 2: Who is the host?

The Gateway kept track of which player created each room in a simple in-memory variable:

```
_ROOM_HOSTS = { "room123": "Likhith" }
```

If the Gateway restarted, that variable reset to empty. Suddenly no one was the host. Clicking "Start Game" would fail with a cryptic error.

## Solution 1: Temporal

### What is Temporal?

Temporal is a tool that lets you write code that survives crashes.

Normally, if your program crashes mid-execution, whatever it was doing is lost. Temporal solves this by saving the progress of your code as it runs. If the program crashes, Temporal replays the code from where it left off when it restarts.

Think of it like a video game save point, except it saves automatically at every step.

### How it was used for the phase timer

Instead of a plain background task, the timer was written as a Temporal Workflow:

```
Start PhaseTimerWorkflow
  > Sleep for 30 seconds        (Temporal saves progress here)
  > Call AdvancePhase           (Temporal retries this if it fails)
```

The key difference is that `workflow.Sleep(30 seconds)` is not a normal sleep. It is a durable sleep. If the Event Service crashes at second 18, Temporal knows the timer had 12 seconds left. When the service restarts, it resumes the sleep with 12 seconds remaining. The players never notice anything happened.

If the call to advance the phase fails because the Game Engine is briefly down, Temporal automatically retries it with no manual intervention needed.

Every timer is also visible in the Temporal UI, a web dashboard where you can see every active timer, how long is left, and the full history of what happened.

Before Temporal, a service restart meant timers were lost and the game would freeze. An HTTP call failure meant the phase never advanced, silently. After Temporal, service restarts resume timers automatically and failed calls are retried until they succeed.

### How it was used for the host problem

The in-memory host variable was replaced with a Temporal Workflow, one per room. This workflow holds the room state including who the host is and what phase the game is in, and keeps it alive for the entire duration of the game.

Because Temporal saves its state to a database, the room information survives Gateway restarts. Any service can ask Temporal "who is the host of room 123?" at any time and get a correct answer.

## Solution 2: Kubernetes

### What is Kubernetes?

Kubernetes is a tool that manages your containers for you.

With Docker Compose, you start all your containers with one command. But if a container crashes, you have to notice it and restart it yourself. Docker Compose has no awareness of whether your app is healthy.

Kubernetes watches your containers 24/7. If one crashes, Kubernetes restarts it automatically. If you tell Kubernetes "I want 3 copies of the Gateway running", it will always maintain exactly 3, replacing any that die.

### How it was used

The entire application was described to Kubernetes as a set of configuration files. Each service has a file that says what container image to run, what environment variables it needs, how to check if it is healthy, and how many copies to run.

For example, the Gateway configuration tells Kubernetes to run 1 copy of the Gateway container, pass in the JWT secret from a secure store, and check the health endpoint every 5 seconds. If it stops responding, restart it.

Kubernetes also handles startup ordering. Temporal needs its database to be ready before it starts. The Event Service needs Temporal to be ready before it starts. Each service was configured to wait for its dependencies before starting:

```
Event Service startup:
  Step 1: Wait until Temporal is reachable on port 7233
  Step 2: Start the actual Event Service
```

This means even if everything starts at the same time, each service waits for what it needs.

### Secrets management

Before Kubernetes, sensitive config like database passwords and JWT secrets lived in a plain text file on the machine. Kubernetes has a built-in Secrets store for sensitive values. Containers read from Secrets at runtime, so credentials are never hardcoded or stored in source code.

Before Kubernetes, a container crash required a manual restart, sensitive config lived in plain files, and there was no way to check if services were healthy. After Kubernetes, crashes are handled automatically, config is stored securely, and health checks run on every service with automatic recovery.

## How They Work Together

Temporal and Kubernetes solve different problems.

Temporal manages the application logic, specifically the workflows and timers, and ensures they survive crashes and failed calls. Kubernetes manages the containers and ensures they are always running, healthy, and properly configured.

In the project, Kubernetes makes sure all 7 services are always running. Temporal makes sure the game logic, timers and room state, is always correct.

Neither one replaces the other. Kubernetes keeps the containers alive. Temporal keeps the application logic alive.

## Running It Locally

To run the full stack on a laptop, Minikube was used. Minikube creates a Kubernetes cluster inside Docker on your local machine. One command starts the cluster:

```
minikube start --driver=docker --memory=6144 --cpus=4
```

Then applying all the configuration files brings up the entire application:

```
kubectl apply -f ./k8s/manifests/
```

All 7 services including Temporal and its database spin up on the laptop. The Temporal UI is available at a local URL where every game's timer workflow can be watched in real time.

## What Changed

Temporal changed how background tasks are written. Before, a timer was written and left to chance. Now every long-running task is a workflow with checkpoints, something that can be paused, resumed, and retried without losing progress.

Kubernetes changed how deployment works. Before, deploying meant running docker-compose up and hoping nothing crashed. Now the infrastructure takes care of itself with health checks, restarts, and ordered startup.

Both tools have a learning curve. But once they click, it is hard to imagine going back.
