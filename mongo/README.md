# 🍃 MongoDB

A single-node MongoDB **replica set** (`rs0`) with authentication, for
[OurHomeWeb](https://github.com/EOSOClub/OurHome).

> [!IMPORTANT]
> OurHomeWeb **requires** a replica set, not a plain MongoDB. It uses Prisma
> transactions, and MongoDB only supports transactions on a replica set. A
> standalone `mongod` fails on the first write that uses one.

| Service | What it does | Address |
| --- | --- | --- |
| `mongo` | MongoDB 7, replica set `rs0`, auth on | `mongo:27017` on `ourhome_net`; host `127.0.0.1:27018` |
| `mongo-init` | One-shot: initiates the replica set and creates the app user, then exits 0 | n/a |

## 🚀 Setup

```bash
cd mongo
docker network create ourhome_net        # once per host; shared by every stack
cp .env.example .env                     # set both passwords
openssl rand -base64 756 > keyfile       # replica-set member auth (gitignored)
docker compose up -d
```

<details>
<summary>On Windows without <code>openssl</code></summary>
<br>

```powershell
$b = New-Object byte[] 756; [Security.Cryptography.RandomNumberGenerator]::Fill($b)
[Convert]::ToBase64String($b) | Set-Content -NoNewline keyfile
```

</details>

**Check it worked:**

```bash
docker compose ps               # mongo: healthy · mongo-init: exited (0)
docker compose logs mongo-init  # "replica set initiated" and "app user created"
```

## 🔗 Connect OurHomeWeb

In OurHomeWeb's `.env`:

```env
SERVER_DATABASE_URL=mongodb://household:<MONGO_APP_PASSWORD>@mongo:27017/household?replicaSet=rs0&authSource=admin
DOCKER_NETWORK=ourhome_net
```

The database (`household`) is created on first use. OurHomeWeb applies its
schema with `prisma db push` every time its container starts.

**From your PC** (Compass, or OurHomeWeb's `npm run dev`), go through the host
port and add `directConnection=true`. Without it, the driver tries to reach the
replica set's advertised name `mongo:27017`, which only exists inside Docker:

```
mongodb://household:<MONGO_APP_PASSWORD>@localhost:27018/household?directConnection=true&authSource=admin
```

## 🔑 Accounts

| User | Role | Created by |
| --- | --- | --- |
| `root` (`MONGO_ROOT_USERNAME`) | root | MongoDB's first boot (`MONGO_INITDB_ROOT_*`) |
| `household` (`MONGO_APP_USERNAME`) | `readWriteAnyDatabase` | `mongo-init` |

> [!WARNING]
> **Passwords are fixed on first boot.** Changing them in `.env` later does
> **not** change the existing users. Either change them as `root` with
> `db.changeUserPassword()`, or wipe and start over with
> `docker compose down -v`, which **deletes all data**.

## 💾 Backups

```bash
# Dump everything to ./backup-<date>.archive on the host
docker exec ourhome_mongo sh -c \
  'mongodump -u "$MONGO_INITDB_ROOT_USERNAME" -p "$MONGO_INITDB_ROOT_PASSWORD" --authenticationDatabase admin --archive' \
  > backup-$(date +%F).archive

# Restore
docker exec -i ourhome_mongo sh -c \
  'mongorestore -u "$MONGO_INITDB_ROOT_USERNAME" -p "$MONGO_INITDB_ROOT_PASSWORD" --authenticationDatabase admin --archive --drop' \
  < backup-YYYY-MM-DD.archive
```

Data lives in the `mongo_data` Docker volume. `docker compose down` keeps it,
and `docker compose down -v` deletes it.

## 🛠️ Troubleshooting

| Symptom | Fix |
| --- | --- |
| `mongo` restarts with a keyfile error | `keyfile` is missing, empty, or a folder (Docker creates a folder if the file didn't exist at first start). Delete it, regenerate it, and `docker compose up -d`. |
| OurHomeWeb: `Transactions are not supported` / `ReplicaSetNoPrimary` | `mongo-init` didn't finish. Check `docker compose logs mongo-init` and re-run `docker compose up mongo-init`. |
| `Authentication failed` | The password in OurHomeWeb's URL doesn't match `MONGO_APP_PASSWORD` *as it was on first boot* (see the warning above). |
| Compass hangs on `mongo:27017` | Add `directConnection=true` to the connection string. |
