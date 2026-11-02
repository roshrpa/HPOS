# Hollywood POS server

PostgreSQL database and a small API that the tablets and counter computers sync with.

What it does today:

- keeps one copy of every bill, cancellation (with reason), discount and day figure
- accepts bills from many devices and ignores repeats, so a retry after a bad connection is safe
- hands each device the events the other devices made
- keeps the menu and settings in one place, with a version number so two people cannot overwrite each other by accident
- gives a day report straight from the database (cash, card, discounts, cancelled bills, sales by category)

What is **not** done yet: the tablet app and the POS do not call this server yet. That is the next step. Until then they keep working on their own, in the browser.

## Files

| File | Purpose |
| --- | --- |
| `db/schema.sql` | Tables and the SQL functions that do the work. Loaded automatically the first time the database starts. |
| `api/src/server.js` | The HTTP API (Node.js, one dependency: `pg`). |
| `docker-compose.yml` | Starts PostgreSQL, the API and Caddy (HTTPS) together. |
| `Caddyfile` | Gets and renews the HTTPS certificate. |
| `.env.example` | The settings you fill in. |
| `scripts/backup.sh` | Daily backup. |

## Set it up on a server

1. Rent a small Linux server (Ubuntu 24.04, 2 CPU, 4 GB RAM, 50 GB disk is plenty). Pick a data centre near you.
2. Point a domain name at it (an `A` record, for example `pos.yourshop.com`).
3. Log in and install Docker: https://docs.docker.com/engine/install/ubuntu/
4. Allow only the ports you need: `ufw allow 22 && ufw allow 80 && ufw allow 443 && ufw enable`. Use SSH keys, not passwords.
5. Get the code and fill in the settings:

   ```sh
   git clone https://github.com/roshrpa/HPOS.git /opt/hpos
   cd /opt/hpos/server
   cp .env.example .env
   nano .env        # set POSTGRES_PASSWORD, ADMIN_TOKEN, DOMAIN
   docker compose up -d
   ```

6. Check it: `curl https://pos.yourshop.com/health` should answer `{"ok":true}`.
7. Register each device once. The token is shown only this one time, so copy it into the device straight away:

   ```sh
   curl -X POST https://pos.yourshop.com/admin/devices \
     -H "X-Admin-Token: YOUR_ADMIN_TOKEN" -H "Content-Type: application/json" \
     -d '{"name":"Tablet 1","kind":"tablet"}'
   ```

8. Back up every day: add `30 3 * * * cd /opt/hpos/server && ./scripts/backup.sh` to `crontab -e`, and copy the `backups` folder somewhere else now and then.

To stop a lost tablet from syncing: `docker compose exec db psql -U hpos hpos -c "update devices set active=false where name='Tablet 1'"`.

## API

All device calls send `Authorization: Bearer <device token>`.

| Call | What it does |
| --- | --- |
| `GET /health` | Answers `{"ok":true}` when the database is reachable. |
| `POST /sync/push` | Body `{"events":[{id, type, t, payload}]}`. Up to 500 events. Returns how many were new and how many repeats. |
| `GET /sync/pull?after=N` | Events made by the other devices after sequence number `N`. Keep the returned `last_seq` for next time. |
| `GET /config/{key}` | Reads the menu or settings, with its version. |
| `PUT /config/{key}` | Body `{"value":{...},"version":N}`. Answers `409` with the current copy if somebody changed it first. |
| `POST /admin/devices` | Registers a device (needs `X-Admin-Token`). |
| `GET /admin/report/day?date=YYYY-MM-DD` | Day report (needs `X-Admin-Token`). |

Event types: `sale` and `sale_cancel` build the sales tables. Other types (`purchase`, `waste`, `stock_count`, `cash_out`, `day_close`) are stored and handed to the other devices as they are.

A `sale` payload:

```json
{ "id": "uuid", "no": "T1-0001", "day": "2026-11-02", "t": 1762050000000, "pay": "cash",
  "subtotal": 50, "discount": 5, "discReason": "Loyal customer", "total": 45,
  "vat": 0, "vatRate": 0, "tendered": 50,
  "lines": [ { "cat": "Milk Shakes", "name": "Chocolate", "size": "", "price": 25, "qty": 2 } ] }
```

A `sale_cancel` payload: `{ "id": "<sale uuid>", "reason": "Wrong item entered", "note": "" }`.

## Good to know

- The database is never opened to the internet. Only the API container can reach it.
- Changing `db/schema.sql` later does not change a database that already exists. Add a new migration file when the time comes.
- This is a first version. It has no user logins yet (only device tokens), so it cannot say which person gave a discount.
