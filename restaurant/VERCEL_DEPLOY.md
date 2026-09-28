# Deploy BAANRAO on Vercel

The app uses local JSON files when run on a developer machine. On Vercel it uses PostgreSQL for restaurant data and Vercel Blob for menu images; it will report a configuration error rather than silently saving production data to Vercel's temporary filesystem.

## Connect persistent storage

1. Import this repository as a Python project in Vercel. The project uses the Python 3.12 runtime; Vercel detects the Flask app, and the build step copies `static/` assets to `public/static/` for CDN delivery.
2. In the Vercel Marketplace, add a PostgreSQL provider such as Neon and connect it to this project. Make sure the provider supplies `DATABASE_URL` to Production and Preview.
3. Create a **public** Vercel Blob store and connect it to the project. Vercel adds `BLOB_READ_WRITE_TOKEN` to the selected environments. Menu images are public assets.
4. Add `SECRET_KEY` as a Vercel environment variable. Generate a value with `python -c "import secrets; print(secrets.token_hex(32))"`. Keep the same value for Production and Preview, or use a different stable value for each environment.
5. Redeploy after adding the environment variables. On first start, the app creates its PostgreSQL state table and seeds the starter data if the database is empty.

PostgreSQL is stored as one JSONB state row so the existing business logic can use its current transaction interface. Writes lock that row for the duration of the operation, preventing concurrent Vercel instances from overwriting each other's changes. This is suitable for the current small restaurant app; a high traffic deployment should move entities to separate relational tables.

## Move existing local data

Install the packages from `requirements.txt`, set the target `DATABASE_URL`, and set `BLOB_READ_WRITE_TOKEN` if the local `data/uploads` folder contains menu images. Then run:

```powershell
python migrate_to_postgres.py --source data/db.json
```

The script moves `data/db.json`, `data/audit.log`, and valid menu images from `data/uploads` into PostgreSQL and Blob. It refuses to replace an initialized database unless `--replace` is passed. `--replace` overwrites the entire current restaurant state, so use it only for the initial import into a newly seeded target.

If you want a fresh restaurant instead, do not run the import; the app creates the starter data automatically.

## Local development

Without `DATABASE_URL`, the app continues using `data/db.json` and `data/uploads`. Set `DATABASE_URL` to a local or hosted PostgreSQL URL to exercise the PostgreSQL backend locally. When `BLOB_READ_WRITE_TOKEN` is present, image uploads use Blob; otherwise local development stores images on disk.
