# AuctionSystem
_A real-time, salary-cap auction platform for running live player drafts._

> **Note:** The source code for this project is hosted in a private repository for security/proprietary reasons. This repository serves as a technical overview and documentation hub.

> **Note:** Public registration is currently disabled — accounts are admin-provisioned only. If you'd like to see the app in action, check out the **[Demo](#demo)** section below for a video walkthrough and screenshots.

**Live site:** [hellotulio.com](https://hellotulio.com)
---

## Status

This project is **actively developed** — features, polish, and infrastructure work are all still landing regularly. The public site at hellotulio.com is live and in use; this repo exists as a public-facing overview while the source remains private.

---

## Demo

**Video demo:** [Youtube](https://www.youtube.com/watch?v=rHsBM6HdnCo)

<!-- Example embed once uploaded:
https://github.com/<user>/<repo>/assets/<id>/demo.mp4
-->

### Screenshots

| View | Preview |
| --- | --- |
| Landing page | [Landing](docs/images/Landing_page.PNG) <!-- ![Landing](docs/images/landing.png) --> |
| Auctions list | [Auctions](docs/images/auction_list.PNG) <!-- ![Auctions](docs/images/auctions-list.png) --> |
| Live auction room | [Auction Room 1](docs/images/Auction_Room.PNG) [Auction Room 2](docs/images/Auction_Room_2.PNG) <!-- ![Auction room](docs/images/auction-room.png) --> |
| Management dashboard | [Management 1](docs/images/management_1.PNG) [Management 2](docs/images/management_2.PNG) [Management 3](docs/images/management_3.PNG) <!-- ![Management](docs/images/management.png) --> |

---

## What is it?

AuctionSystem is a web application for hosting live "player auctions" — events where a group of team captains bid on players to assemble rosters, with every captain working against a fixed salary cap.

Each auction is tied to a specific draft event and is run by an admin. Every captain has deduction and this is subtracted from the auction total. When a player is put up for auction, captains bid against each other in real time. Whoever wins the current player, that player is then assigned to their roster, and no other team can bid for or take that player afterward. By the time the auction ends, every captain has a finalized roster of the players they won, along with whatever balance they have left.

The flow is **turn-based nomination → open bidding → sold**. One captain nominates a player on their turn, then every captain can bid on that player during a short timed window. When the bid timer expires, the highest bidder wins, the player is assigned to their roster, and the next captain in the rotation gets to nominate. An admin can pause, resume, or complete the auction at any point.

---

## Key features

- **Real-time bidding** over WebSockets with live updates, current-leader tracking, and auto-reconnect (exponential backoff + jitter).
- **Auction lifecycle:** lobby → running → paused → completed, with admin-controlled state transitions.
- **Turn-based nomination** with configurable nomination and bid timers; expired turns roll over automatically.
- **Salary-cap enforcement** — per-team balance tracking, per-bid validation, and configurable roster size limits.
- **Admin management dashboard** for creating auctions, bulk-importing players from CSV/JSON, assigning teams and captains, and overriding picks manually when needed.
- **Secure auth layer:** JWT access + refresh tokens, bcrypt password hashing, CSRF protection on state-changing requests, and per-endpoint rate limiting.
- **Resilient client:** singleton token refresh, WebSocket reconnect on tab visibility, SWR-backed data fetching with automatic revalidation.

---

## Tech stack

| Layer | Technology |
| --- | --- |
| **Frontend** | Next.js 16 (App Router), React 19, TypeScript, Tailwind CSS 4, SWR, native WebSocket API |
| **Backend** | FastAPI, Python 3.11, SQLAlchemy 2 (async), Alembic, asyncpg |
| **Database & cache** | PostgreSQL 16, Redis 7 (pub/sub + FIFO task queue) |
| **Auth & security** | JWT (python-jose), bcrypt, SlowAPI rate limiting, CSRF tokens, security headers (HSTS, X-Frame-Options, etc.) |
| **Infrastructure** | Docker, Docker-Compose, NGINX (TLS termination + WebSocket upgrade), Docker Secrets |

---

## Architecture

The Next.js frontend is served behind an NGINX reverse proxy that handles TLS termination and WebSocket upgrade, and forwards API traffic to the FastAPI backend. The backend reads and writes to PostgreSQL via async SQLAlchemy, and uses Redis for both a pub/sub event bus (pushing auction events out to connected WebSocket clients) and a FIFO task queue. A separate async worker container consumes that queue to handle timed nomination and bid expiry, publishing the resulting state changes back through Redis so every connected client sees them live.

---

## Technical hurdles

- **Stale UI when more than 5 captains were bidding at once**
  - The auction room was running behind and out of date once a bunch of captains started bidding on the same player. The UI just wasn't keeping up with what the server already knew.
  - Fixed by adding SWR with different `refreshInterval` values per query so the most important data refreshes the fastest, active item every 2 seconds, auction + teams every 3 seconds, unsold list every 5 seconds. This allowed for more up to date information without over loading the endpoint.
  - stil priotizes the socker connection, this is used as safety net if the update does not go through.
  - relies on mutate, that is triggered when the server returns a message that something has changed through the WebSocket.
- **"Invalid token" errors from refresh intervals**
  - users were getting invalid token errors from long sessions, and during bidding.
  - added refresh at 10 minutes 5 minutes before the expiration of the 15 min access token.
  - wrapped it in a promise to help to make sure they were not invalidated.
- **Race conditions / duplicate bids in the auction engine**
  - When two captains clicked bid at basically the same instant, it was possible for both to succeed and the second one would overwrite or duplicate the first.
  - Used `SELECT ... FOR UPDATE` (SQLAlchemy `with_for_update()`) on the team row and item row inside the bid transaction, so only one request at a time can be can be handled.
  - Added a Postgres **partial unique index** (`uq_bid_one_active_per_item`) that enforces at most one active bid per item at the database level, so even if the app-level logic had a gap, the DB would reject the duplicate.
  - Wrapped the commit in a try/except on `IntegrityError`: if the same bid somehow lands twice (e.g. a client retry), we roll back and return the existing bid instead.
- **WebSocket reconnection**
  - Rewrote the socket hook to auto-reconnect with **exponential backoff** (1s, 2s, 4s, ... capped at 30s)
  - Added a `visibilitychange` listener, when the tab comes back to focus, if the socket isn't actually open we close the stale one and reconnect immediately instead of waiting for the next backoff tick.
  - On a successful reconnect we call an `onReconnect()` callback that re-fetches SWR data, so the UI state catches up to whatever happened while we were offline.

<!--
Possible topics to expand on:
- Real-time state synchronization across multiple connected clients
- WebSocket authentication alongside CSRF-protected HTTP requests
- Bulk import performance (up to 10,000 items) and validation
-->

---
<!--
## Contact

Feel free to reach out if you'd like a live walkthrough or access to the demo environment.
-->

