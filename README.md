# Cinema ticket booking system

A small Go service built around **one hard problem**: never sell the same
cinema seat twice, even when many users click "Book" on it at the same
instant. It's a gradual, teaching-oriented codebase where concurrency
correctness — not feature breadth — is the whole point.

## The Problem

Two users click "Book" on seat A1 at the same instant. Only one should win.

```bash
User A ──► read seat A1 → "free" ──► write booking ──► success
User B ──► read seat A1 → "free" ──► write booking ──► ???
```

Without any protection, both succeed. Now two people show up for the same
seat. This is a classic **check-then-act race**: the gap between reading
"free" and writing the booking is where the double-booking sneaks in.

## The Solution

Don't check-then-act — make the claim **atomic**. The seat hold is a single
Redis `SET` with the `NX` flag ("set only if the key does not exist"):

```go
// internal/booking/redis_store.go
res := s.rdb.SetArgs(ctx, key, val, redis.SetArgs{
    Mode: "NX",            // succeed only if seat:{movie}:{seat} is free
    TTL:  defaultHoldTTL,  // 2-minute hold, auto-released if never confirmed
})
```

Redis executes `SET NX` atomically, so out of any number of concurrent
requests for the same seat **exactly one** gets `OK`; everyone else gets
`ErrSeatAlreadyBooked`. The race is gone because there is no longer a
read-then-write window — the claim and the check are the same operation.

### Booking lifecycle

A booking moves through three states, mirroring a real "hold → pay → seated"
flow:

```
        POST .../hold                 PUT .../confirm
  free ───────────────► held ───────────────────────► confirmed
    ▲   (SET NX, TTL 2m)  │  (PERSIST — TTL removed)
    │                     │
    └─────────────────────┘
       TTL expires  OR  DELETE /sessions/{id}  (Release)
```

- **Hold** — `SET NX` with a 2-minute TTL. If the user walks away, the key
  expires and the seat frees itself automatically. No cleanup job needed.
- **Confirm** — `PERSIST` removes the TTL so the booking becomes permanent.
- **Release** — deletes the seat key and its session, freeing the seat now.

Two Redis keys back each hold:

```
seat:{movieID}:{seatID}  → booking JSON   (the seat claim; TTL while held)
session:{sessionID}      → seat key       (reverse lookup for confirm/release)
```

## API

| Method   | Path                                     | Description                         |
| -------- | ---------------------------------------- | ----------------------------------- |
| `GET`    | `/movies`                                | List movies (id, title, layout)     |
| `GET`    | `/movies/{movieID}/seats`                | List booked/confirmed seats         |
| `POST`   | `/movies/{movieID}/seats/{seatID}/hold`  | Hold a seat → returns a `session_id` |
| `PUT`    | `/sessions/{sessionID}/confirm`          | Confirm a held seat                 |
| `DELETE` | `/sessions/{sessionID}`                  | Release a held/confirmed seat       |
| `GET`    | `/`                                      | Static frontend (`static/`)         |

```bash
# hold seat A1 for a user
curl -X POST localhost:8080/movies/inception/seats/A1/hold \
  -d '{"user_id":"thiago"}'
# {"session_id":"...","movieID":"inception","seat_id":"A1","expires_at":"..."}

# confirm it (within the 2-minute window)
curl -X PUT localhost:8080/sessions/<session_id>/confirm \
  -d '{"user_id":"thiago"}'

# or release it
curl -X DELETE localhost:8080/sessions/<session_id> \
  -d '{"user_id":"thiago"}'
```

## Run

The store is Redis-backed, so start Redis first (compose also brings up
[redis-commander](http://localhost:8081) to inspect keys):

```bash
docker compose up -d          # Redis on :6379, redis-commander on :8081
go run ./cmd                  # API + frontend on :8080
```

Then open <http://localhost:8080>.

## Project layout

```
cmd/main.go                        # HTTP wiring: routes → handler → service → store
internal/booking/
├── domain.go                      # Booking type + BookingStore interface (the contract)
├── service.go                     # Business layer over any BookingStore
├── handler.go                     # HTTP handlers (hold / confirm / release / list)
├── redis_store.go                 # Redis SET NX + TTL implementation (the one wired up)
├── concurrent_store.go            # In-memory mutex store (didactic alternative)
└── memory_store.go                # Plain map store (shows the race without protection)
internal/adapters/redis/redis.go   # Redis client constructor
internal/utils/utils.go            # WriteJSON helper
static/index.html                  # Self-contained frontend
```

The `BookingStore` interface is the seam: the same `Service` works over the
Redis store, the mutex store, or the naive map store — swapping them is how
the project contrasts a safe implementation against a racy one.

## Stack

Go 1.25 · Redis 7 (`go-redis/v9`) · `google/uuid` · Docker Compose.
