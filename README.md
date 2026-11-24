# Rate Limiter Project

A simple, beginner-friendly **rate limiter** built with **HTML/CSS/JavaScript** (frontend), **Python + FastAPI** (backend), and **Redis** (database/cache). It uses the **Fixed Window Counter** algorithm.

This is a student project — the code is intentionally simple and easy to understand. No Docker, no React, no TypeScript, no databases other than Redis.

---

## What is a rate limiter?

A rate limiter controls how many requests a client can make to a server within a given time period. It protects the backend from being overwhelmed by too many requests (for example, from a buggy script, a brute-force attack, or a user clicking a button too fast).

In this project:
- A client may make **at most 5 requests**.
- The time window is **60 seconds**.
- The 6th request within the window is rejected with **HTTP 429**.
- After 60 seconds, the counter resets and the client can make 5 more requests.

---

## How the Fixed Window Counter algorithm works

The Fixed Window Counter divides time into fixed windows (here, 60-second windows). Each window has its own counter.

1. When a request arrives, the server looks up the counter for the current window and that client.
2. If there is no counter yet, it creates one starting at 0 and sets it to expire after 60 seconds.
3. If the counter is below the limit (5), the request is **allowed** and the counter is incremented by 1.
4. If the counter is already at the limit, the request is **rejected** with HTTP 429.
5. When the window expires (after 60 seconds), the counter is deleted automatically and a new window starts.

This project keeps a separate counter for every client (identified by `client_id`), so different users have independent limits.

---

## How Redis is used

Redis stores the request counters. Each client gets one Redis key:

```
ratelimit:<client_id>   -->   "5"     (with a 60-second TTL/expiry)
```

- The value is the number of requests made in the current window.
- `EXPIRE` makes the key disappear automatically after 60 seconds, which resets the counter.
- The increment-and-check happens inside a **Redis Lua script**, so it is **atomic**: no two requests can slip through between reading the count and updating it.

---

## Project structure

```
rate-limiter/
├── backend/
│   └── main.py              # FastAPI backend + Redis Lua script
├── frontend/
│   ├── index.html           # UI
│   ├── script.js            # fetch() calls to backend + UI updates
│   └── style.css            # Styles (unchanged from original design)
├── requirements.txt         # Python dependencies
└── README.md                # This file
```

---

## Install and run on Arch Linux

### 1. Install Redis

```bash
sudo pacman -S redis
```

### 2. Start Redis

```bash
# Start Redis in the background
redis-server --daemonize yes

# Check that Redis is running
redis-cli ping
# Expected reply: PONG
```

### 3. Set up the Python virtual environment

```bash
cd /home/badal/Desktop/projects/rate-limiter

# Create the virtual environment (if you have not already)
python3 -m venv .venv

# Activate it
source .venv/bin/activate

# Install dependencies
pip install -r requirements.txt
```

### 4. Run the backend (FastAPI)

```bash
source .venv/bin/activate
cd /home/badal/Desktop/projects/rate-limiter/backend
uvicorn main:app --reload --host 127.0.0.1 --port 8000
```

The backend is now running at `http://127.0.0.1:8000`.

### 5. Open the frontend

The frontend calls `http://127.0.0.1:8000/api/check`, so it must be served over HTTP (not opened as a bare `file://` page) for CORS to work.

Option A — with VS Code "Live Server": open `frontend/index.html` with Live Server (usually port 5500).

Option B — with Python's built-in server:

```bash
cd /home/badal/Desktop/projects/rate-limiter/frontend
python3 -m http.server 5500
```

Then open `http://127.0.0.1:5500` in your browser and click **Request**.

---

## How to test the rate limiter

### Quick backend test with curl

Allow 5 requests (HTTP 200):

```bash
curl -i http://127.0.0.1:8000/api/check
curl -i http://127.0.0.1:8000/api/check
curl -i http://127.0.0.1:8000/api/check
curl -i http://127.0.0.1:8000/api/check
curl -i http://127.0.0.1:8000/api/check
```

The 6th request should return HTTP 429:

```bash
curl -i http://127.0.0.1:8000/api/check
# Expected: HTTP/1.1 429 Too Many Requests
```

Further requests stay blocked until the 60-second window expires.

### Test separate clients

Counters are tracked per `client_id`:

```bash
curl -i "http://127.0.0.1:8000/api/check?client_id=alice"
curl -i "http://127.0.0.1:8000/api/check?client_id=bob"
```

Alice and Bob each get their own limit of 5.

### Test the frontend

1. Open `http://127.0.0.1:5500`.
2. Click **Request** five times — you should see "Request allowed!", the count go from 1 to 5, and the emoji change.
3. Click a 6th time — you should see the 🤡 emoji, "Rate limit exceeded...", and the button disabled.
4. Wait for the timer to reach 0 — the UI resets and you can start a new window.

### Verify Redis is enforcing the limit

While the server is running, inspect the counter directly in Redis:

```bash
redis-cli get ratelimit:default-client
# Returns the current count, e.g. "5"
redis-cli ttl ratelimit:default-client
# Returns seconds left until the window resets
```

---

## Notes

- The backend does **not** use a global Python variable for the rate limit — everything is stored in Redis.
- If Redis is not running, the backend returns HTTP 503 with an error message.
- CORS is configured for local frontend development (ports 5500 / 8000).
- No authentication or unnecessary endpoints are included.
