# TAMALE Production Roadmap

This repository hosts the static TAMALE Production Roadmap dashboard.

## Live Dashboard
🔗 **[View Dashboard](https://aarixu.github.io/tamale-todo/)**

## How it works
This is a 100% static React app running entirely in the browser using Babel standalone. 
There is NO build step. The data source is the `tasks.json` file in this repository.

## How to update tasks
1. Go to the live dashboard and make your edits (which are saved instantly to your browser's `localStorage`).
2. Click the **"Export JSON"** (or "Copy JSON") button to get the updated JSON.
3. Come back to this repository, edit `tasks.json` on the GitHub web UI, and paste the new JSON.
4. Commit your changes. GitHub Pages will deploy the update in ~30 seconds.
5. In your next session (or on a different device), the dashboard will fetch the latest `tasks.json` from the repo.

## Security Notice
🚨 **NO SECRETS IN THIS REPO** 🚨
This is a PUBLIC repository to allow AI tools to fetch the state via web requests. 
The `context.md` file has been fully sanitized. Do not commit any `secrets.md` or `.env` files.
