# 🧾 Paperless-ngx

[Paperless-ngx](https://docs.paperless-ngx.com) archives your bills and
receipts with full-text OCR. In Our Home it is where bills arrive: emailed
bills come in from Proton Mail through the [Bridge](../proton-bridge), and paper
ones come from a phone photo or upload.

| | |
| --- | --- |
| URL | `PAPERLESS_URL` (default `http://localhost:8200`) |
| Login | `PAPERLESS_ADMIN_USER` / `PAPERLESS_ADMIN_PASSWORD` from `.env` |
| Data | `./data/`: database, originals, archive, search index |
| Inbox folder | `./consume/`: any file dropped here is imported, then removed |
| Backups | `./export/` (see [Backups](#backups)) |

Containers: `paperless` (web + worker), `paperless_db` (Postgres 17),
`paperless_redis`, plus `paperless_tika` and `paperless_gotenberg`, which turn
emails into PDFs.

## 🚀 Setup

```bash
cd paperless
docker network create ourhome_net    # skip if it already exists
cp .env.example .env                 # set the secrets and PAPERLESS_URL
docker compose up -d
docker compose logs -f webserver     # first start takes a minute (migrations)
```

Open `PAPERLESS_URL` and sign in as the admin user.

> [!TIP]
> **Outside access:** keep `PAPERLESS_BIND=127.0.0.1` and put a reverse proxy
> or tunnel in front (Cloudflare Tunnel, Caddy, nginx). Then set `PAPERLESS_URL`
> to the public `https://` address. For **LAN-only** phone uploads, set
> `PAPERLESS_BIND=0.0.0.0` and `PAPERLESS_URL=http://<server-ip>:8200` instead.

## 📧 Bills from email (Proton via the Bridge)

Set up the [Bridge](../proton-bridge) first and keep its `info` credentials handy.

**1. In Proton Mail,** create a folder called `Bills` and a filter that files
bill emails into it, for example by sender (your utilities, card issuers,
insurers) or by subject containing "bill", "statement" or "invoice".

**2. In Paperless, go to *Mail → Mail accounts → Add*:**

| Field | Value |
| --- | --- |
| IMAP server | `protonmail-bridge` |
| IMAP port | `143` |
| IMAP security | **None** ([why](../proton-bridge/README.md#use-from-another-container)) |
| Username / password | The Bridge's `info` credentials |

Press **Test**. It should succeed in a second or two.

**3. Add two mail rules** on that account:

| Rule | Folder | Consumption scope | Attachment type | Action | Assign tag |
| --- | --- | --- | --- | --- | --- |
| Bill attachments | `Folders/Bills` | Attachments only | Attachments only | Mark as read | `bill` |
| Bill emails | `Folders/Bills` | Only process the email body (as .eml) | n/a | Mark as read | `bill` |

The first catches bills sent as PDF attachments. The second catches bills that
are only an HTML email, saved as a PDF via Gotenberg.

Proton folders appear as `Folders/<name>`, and labels as `Labels/<name>`. Mail is
checked every 10 minutes; to test a rule straight away, use *Mail → Process mail*.

<details>
<summary><b>Mail troubleshooting</b></summary>
<br>

| Symptom | Fix |
| --- | --- |
| Test fails: "connection reset by peer" (Bridge log) or `SSL: WRONG_VERSION_NUMBER` (`data/data/log/paperless.log`) | IMAP security is set to SSL/STARTTLS. Set it to **None**. |
| "resolves to a non-public address" | `PAPERLESS_EMAIL_ALLOW_INTERNAL_HOSTS` is missing from the compose file. |
| "Name or service not known" | The Bridge isn't running, or Paperless isn't on `ourhome_net`. Check `docker network inspect ourhome_net`. |
| Login fails | The Bridge password changed (after a Bridge reset). Copy the new one from `info`. |

</details>

## 📱 Bills on paper

- **Phone app:** *Swift Paperless* (iOS), or *Paperless Mobile* / *Paperless
  Share* (Android). Point it at `PAPERLESS_URL` and use its scanner.
- **Browser:** open Paperless on the phone, *Documents → Upload → Take Photo*.
- **Folder:** drop files into `./consume/` on the server.

For better OCR, photograph the page flat on a dark background in good light.
Paperless straightens and rotates it automatically.

## 🗂️ Recommended organisation

| Create | Settings | Why |
| --- | --- | --- |
| Tag `bill` | *Matching: None* | Only the mail rules and you assign it, never auto-guessing. |
| Tag `receipt` | *Matching: None* | Same, for receipts. |
| Custom field `Amount` | Type *Monetary* | The amount due or paid. |

Paperless learns **correspondents** (who sent it) and **document types** from
your corrections. Fix the first 10–20 documents by hand and its auto-matching
gets good after that.

> [!WARNING]
> **Keep tags, correspondents and document types unowned.** Anything created
> in the web UI is owned by whoever created it, and is then hidden from every
> other user and every API token, including the one OurHomeWeb will use. After
> creating one, set *Edit → Permissions → Owner* to **none**. To find any that
> drifted:
>
> ```bash
> docker exec paperless python3 /usr/src/paperless/src/manage.py shell -c \
>   "from documents.models import Tag,Correspondent,DocumentType,StoragePath; \
>    [print(M.__name__, M.objects.exclude(owner__isnull=True).count()) for M in (Tag,Correspondent,DocumentType,StoragePath)]"
> ```
>
> Replace `.count()` with `.update(owner=None)` to fix them.

<a id="ourhomeweb"></a>

## 🔗 OurHomeWeb

OurHomeWeb imports documents tagged **`bill`** (becomes an unpaid bill) and
**`bill-payment`** (recorded as a payment on the matching bill) every 15
minutes, from the moment it's switched on. It reads the **Amount**, **Due date**
and **Account number** custom fields and never changes anything in Paperless.

What it needs here:

1. The tags `bill` and `bill-payment` (*Matching: None*, *Owner: none*). Point
   the mail rules above at `bill` for bills; payment confirmations get
   `bill-payment` (a second Proton folder plus rule, or tag them by hand).
2. The custom fields **Amount** (Monetary) and **Due date** (Date), and ideally
   **Account number** (Text), filled on those documents.
3. A **read-only user** (e.g. `ourhome`) with *view* on Document, Tag,
   Correspondent and Custom field, and its **API token**.

Then in OurHomeWeb's `.env`:

```env
SERVER_PAPERLESS_URL=http://paperless:8000
SERVER_PAPERLESS_TOKEN=<token of the read-only user>
PAPERLESS_PUBLIC_URL=<your PAPERLESS_URL, for "Open in Paperless" links>
```

Full guide, dry-run preview and troubleshooting:
[OurHomeWeb → docs/paperless-import.md](https://github.com/EOSOClub/OurHome/blob/main/docs/paperless-import.md).

<a id="backups"></a>

## 💾 Backups

```bash
docker compose exec webserver document_exporter ../export --zip    # → ./export/
```

`./data/` holds live Postgres files, so back up the exporter's output, or stop
the stack before copying `./data/`. To restore on a fresh install, use
`document_importer` ([docs](https://docs.paperless-ngx.com/administration/#importer)).

## ⬆️ Updating

```bash
docker compose pull && docker compose up -d
```

Tika stays pinned to 3.x; see the comment in `docker-compose.yml`.
