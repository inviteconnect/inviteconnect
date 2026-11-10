# File Share (GitHub + Render + PostgreSQL)

A private file-sharing dashboard. You log in, upload files (up to 50 MB, any type), generate download links, and create URL redirect links. Each link uses one of five encryption types (A to E), and every click is counted in the statistics.

Everything runs on **Render**:
- the web service runs the app from your **GitHub** repository
- a **Render PostgreSQL** database stores the files, links, and statistics

No Cloudflare account is needed.

## What you need

- A GitHub account: https://github.com/signup
- A Render account: https://render.com. Check at signup whether it asks for a card.

## Part 1: Create the PostgreSQL database on Render

1. In Render, click **New → PostgreSQL**.
2. Fill in:
   - **Name:** `file-share-db`
   - **Database:** leave the default
   - **Region:** choose the region closest to you. **Use the same region for the web service later.**
   - **Plan:** **Free**
3. Click **Create Database** and wait until its status shows **Available**.
4. Open the database's page and find **Connections**. Copy the **Internal Database URL**. It starts with `postgresql://` and includes a host such as `dpg-xxxxxxxx-a`.

**Save the Internal Database URL somewhere safe now.** You'll paste it into the web service in Part 3. It contains the database password, so keep it private.

## Part 2: Put the code on GitHub

1. On GitHub, click **New repository**. Name it `file-share` and choose **Private**.
2. Upload the contents of the `file-share` folder (not the folder itself). Use **Add file → Upload files** and drag in `package.json`, `render.yaml`, `.gitignore`, `README.md`, and the `src` and `test` folders. Then click **Commit changes**.

With Git:
```bash
cd file-share
git init && git add . && git commit -m "File share"
git branch -M main
git remote add origin https://github.com/<your-name>/file-share.git
git push -u origin main
```

## Part 3: Create the web service on Render

1. In Render, click **New → Web Service** and connect your GitHub account. Choose the `file-share` repository.
2. Fill in:
   - **Name:** `file-share`
   - **Region:** **the same region as your database**
   - **Runtime:** **Node**
   - **Build command:** `npm install`
   - **Start command:** `npm start`
   - **Plan:** **Free**
3. Scroll to **Environment Variables** and click **Add Environment Variable**:
   - **Key:** `DATABASE_URL`
   - **Value:** paste the **Internal Database URL** you saved in Part 1
4. Optional: add `MAX_UPLOAD_BYTES` with the value `52428800`. This is the 50 MB limit and is already the default.
5. Click **Create Web Service** and wait for the first deploy to finish.

If you use a Blueprint instead (**New → Blueprint**), Render shows a field for `DATABASE_URL`. Paste the same Internal Database URL there.

Render shows your address, such as `https://file-share-xxxx.onrender.com`. Open it. The first visit shows **Create your login**. Create a username and password, log in, and start using the dashboard.

Each time you push a change to GitHub, Render redeploys automatically.

## Using the site

- **Create** dropdown: **File upload** (the default) or **URL redirect**.
- **File upload:** upload a file and generate a download link (`/d/<token>`). Opening the link downloads the file.
- **URL redirect:** enter a destination URL and generate a redirect link (`/r/<token>.<6-character suffix>`). Only http and https addresses are accepted.
- Every link can use one of the encryption types A to E. E is the default.
- **Your files** and **Your links** list everything you've created. Each link has Copy, Test, and Delete buttons.
- **Statistics** shows totals, clicks by type, and every link with its URL, destination, and click times. **Reset statistics** clears the counts.
- The gear icon opens **Settings**, where you can change your username or password, or reset all data.
- The sun/moon icon switches between dark and light mode. Dark is the default.
- Logging in on a new browser or device signs out the old one. You're also logged out after 6 hours of inactivity, when you log out, or when you clear cookies.

## Important: the free database expires after 30 days

Render's free PostgreSQL database is deleted **30 days after it's created**. Once it's deleted, its data can't be recovered unless you upgrade it. To keep your files, links, and statistics:

1. **Back up every 25 days or so.** Install the PostgreSQL client tools on your computer. Then, on the database's page in Render, copy the **External Database URL** and run:

   ```bash
   pg_dump --no-owner --no-acl "<External Database URL>" > backup.sql
   ```

   The external URL is the one that ends in `render.com`. Keep `backup.sql` in a safe place.

2. **Restore into a new database when the old one expires.**
   - Create a new PostgreSQL database in Render as in Part 1. Copy its **Internal** and **External** URLs.
   - Load your backup into the new database:
     ```bash
     psql "<New External Database URL>" < backup.sql
     ```
   - In the web service, open **Environment**, change `DATABASE_URL` to the new **Internal** URL, and save. Render redeploys.

If you skip the backup, the database is deleted with everything in it. You'd see **Create your login** again, and existing links would stop working.

## Free-plan limits

These limits come from Render's free plan and may change. Check Render's pricing page before relying on them.

- **Database size:** the free database has a fixed size limit. Check the current amount on Render's pricing page before uploading large files. Deleting files frees space.
- **Connections:** the free database allows a small number of connections. The app uses at most 5.
- **Sleep:** the web service sleeps after about 15 minutes without visitors. The first visit after that takes about 30 to 60 seconds. Your data is in the database, so it's not affected.
- **Sessions:** logins are kept in memory, so a web service restart signs everyone out. Files, links, and statistics are not affected.
- **Click counts** are saved to the database every 15 minutes, and again when the service stops. A crash could lose the last few minutes of clicks.
- **Memory:** an upload is held in memory while it's stored. Render's free web service has 512 MB, which is enough for 50 MB files.

## Changing the settings

- **Upload limit:** `MAX_UPLOAD_BYTES` in the web service's environment. It's in bytes. Also change the `MAX_BYTES` value in `src/index.html` so the page checks the same limit.
- **Forgotten login:** a reset deletes the `system/config.json` row in the database. Use **psql** with the External Database URL: `DELETE FROM kv WHERE k = 'system/config.json';`. Then open the site and create a new login. Don't delete `system/secret.json`, or encrypted (type C) links will stop working.

## Testing before you deploy (optional)

```bash
npm install
npm test
```

The tests run the server against an in-memory stand-in for the database, so they need no Render account. They check setup, login, sessions, uploads, the five link types, redirects, statistics, settings, and the database adapter.

To run the app on your own computer against a real database, set `DATABASE_URL` to the External Database URL and run `npm start`. Then open `http://localhost:10000`.

## Troubleshooting

| Problem | Fix |
|---|---|
| Logs say "Missing environment variable DATABASE_URL" | Add `DATABASE_URL` under **Environment** in the web service, then redeploy. |
| "Could not connect to the database" | Check that `DATABASE_URL` is the **Internal** URL, and that the web service and database are in the same region. |
| The site shows errors after about 30 days | The free database has expired. Follow the restore steps above. |
| The site takes a minute to load | The service was asleep. This is normal on the free plan. |
| Everyone is logged out after the service restarts | Expected. Logins are kept in memory. Log in again. |
| Changes to the code don't appear | Check that the push reached GitHub, then look at the deploy log in Render. |
