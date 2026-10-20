# ECollect setup (Supabase + admin website)

There is one database, **Supabase**. The admin website (`web/`) and the mobile app (Expo, project root) both use it.

| Who | Uses | Signs in with |
|---|---|---|
| LGU Admin, Barangay Admin | Website (`web/`) | Email and password created in the system |
| Garbage collector | Mobile app | Login created by the Barangay Admin (website → Garbage collectors) |
| Resident | Mobile app | Self sign-up in the app |

Keys:
- **anon key** (Project Settings → API): public. It goes in `web/.env` and the app's `.env`.
- **service_role key**: secret. It only ever goes in `.env.seed` on your own computer and in Supabase itself. Never put it in `web/.env`, the app, GitHub or a chat.

---

## 1. Database

Open Supabase → **SQL Editor** → New query. Paste each file, click **Run**, and wait for it to finish before the next one.

1. `supabase/setup/01_schema.sql`
2. `supabase/setup/02_security.sql`
3. `supabase/setup/03_logic.sql`
4. **One** of:
   - `supabase/setup/04_upgrade_existing.sql`: your project **already has ECollect data** (the current project). It keeps every row; see the list at the top of the file. The last two result tables should show Zone 1 … Zone 10 with the official days and times, and `17` active fixed rows.
   - `supabase/setup/04_data.sql`: a **new, empty** project only. It loads the Barangay Poblacion data copied from the old MySQL database.

Files 01–03 are safe to run again later (for example after an update). Run 04 only once.

> **Back up first.** Free projects have no automatic backups. Before step 1, export each table from the Table Editor (… → Export to CSV).

If `04_upgrade_existing.sql` stops with "Zone records need attention", nothing was changed. Send the message to whoever maintains the system.

## 2. Staff-account function

Creating and deleting logins needs the service key, so it runs as a Supabase **Edge Function** rather than in the website. Install the Supabase CLI and deploy once (again after any change to `supabase/functions/`):

```
npx supabase login
npx supabase link --project-ref YOUR-PROJECT-REF      # the part before .supabase.co
npx supabase functions deploy manage-staff
```

Who can do what with it:
- the **LGU Admin** creates Barangay Admin accounts and deletes resident accounts;
- the **Barangay Admin** creates, edits and deletes garbage collector accounts.

The LGU Admin login already exists in the current project. For a new project, create it once with `npm run seed:lgu` (instructions at the top of `scripts/seed-lgu-admin.mjs`).

## 3. Website

```
cd web
cp .env.example .env        # then fill in VITE_SUPABASE_URL and VITE_SUPABASE_ANON_KEY
npm install
npm run dev                 # http://localhost:5173
```

Sign in on the home page with **Login**, choosing *Barangay Admin* or *LGU Admin*.

**Put it online (free):** on Vercel or Netlify, import the GitHub repository and set:
- **Root directory:** `web`
- **Build command:** `npm run build`
- **Output directory:** `dist`
- **Environment variables:** `VITE_SUPABASE_URL` and `VITE_SUPABASE_ANON_KEY`

`web/vercel.json` and `web/public/_redirects` make page links such as `/barangay/schedules` work when opened directly.

Then add the website's address in Supabase → Authentication → URL Configuration (Site URL).

## 4. After the upgrade, on the website

1. **Barangay Admin → Collection schedule → Truck crew:** choose the collector who drives the truck. Until then, no collector sees the fixed schedule in the app.
2. **Truck stops:** confirm each stop's exact spot with the barangay, then tick "confirmed". Consunji St. has no Google Maps link yet.
3. **Overlapping stops:** Back of Kim Guan and Sapporo–Araneta St. both run 5–6 PM Mon/Wed/Fri with one truck. Adjust the times once the barangay confirms the order.
4. **Zone areas:** the traced zone areas are provisional (dashed on the map) until the barangay confirms them.
5. **Collectors without a login:** collectors that appear with "No login yet" get one when you set a password on their Edit form.

## Notes

- **Free plan pausing:** Supabase pauses a free project after about a week without activity. Open the dashboard and restore it before a demo.
- **Zone lock:** the 10 zones are locked in the database: they cannot be added, renamed or deleted, even in the Table Editor. For an official re-zoning, see the comment at the top of the zone section in `03_logic.sql`.
- **Server clock:** times come from the server clock in Philippine time, not from phones.
