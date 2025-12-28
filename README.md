# Sun Calendar for Plano, Texas

Automatically generates and updates an iCalendar (`.ics`) file with sunrise and sunset times for Plano, Texas. Updates every 6 hours via GitHub Actions.

## Setup Instructions

### 1. Create a new GitHub repo
```bash
# Create new repo on GitHub, then clone it locally
git clone https://github.com/YOUR_USERNAME/YOUR_REPO_NAME.git
cd YOUR_REPO_NAME
```

### 2. Add these files to your repo:

**`sun_calendar.py`** - The main script (copy the full script provided)

**`.github/workflows/update-sun-calendar.yml`** - The automation workflow

Create the `.github/workflows/` directory structure:
```bash
mkdir -p .github/workflows
# Copy update-sun-calendar.yml into .github/workflows/
```

**`.gitignore`** - To ignore Python cache files (copy provided)

### 3. Create `docs/` directory
```bash
mkdir docs
```

### 4. Run locally once to test
```bash
pip install requests pytz
python sun_calendar.py
```

This should create `docs/sun.ics`.

### 5. Commit and push
```bash
git add .
git commit -m "Initial sun calendar setup"
git push origin main
```

### 6. Verify GitHub Actions is enabled
- Go to your repo → Settings → Actions → General
- Ensure "Actions permissions" is set to allow workflows

### 7. Manual trigger (optional)
Go to Actions tab → "Update Sun Calendar" → "Run workflow" to test immediately

## Using the Calendar

Once running, the `docs/sun.ics` file will be available at:
```
https://raw.githubusercontent.com/YOUR_USERNAME/YOUR_REPO_NAME/main/docs/sun.ics
```

### Subscribe in Calendar Apps:

**Apple Calendar / Outlook / Google Calendar:**
1. Open your calendar app
2. Add → Subscribe to calendar (or similar)
3. Paste the URL above
4. It will update every 6 hours

**Or just download:**
- Visit `https://github.com/YOUR_USERNAME/YOUR_REPO_NAME/raw/main/docs/sun.ics`
- Download and import into your calendar app

## Features

- ✅ Sunrise and sunset events for Plano, TX
- ✅ 400 days of future data (stays relevant)
- ✅ 15-minute reminders before each event
- ✅ Marked as "Free" time (won't block calendar)
- ✅ Auto-updates every 6 hours
- ✅ Proper timezone handling (America/Chicago)
- ✅ Minimal API calls with rate limiting

## Customization

Edit `sun_calendar.py` to change:
- `LAT` / `LNG` - Location coordinates
- `DAYS_AHEAD` - How many days to generate
- In `TRIGGER:-PT15M` - Reminder time (e.g., `-PT30M`, `-PT1H`)

## Notes

- The cron schedule `0 */6 * * *` runs at 12am, 6am, 12pm, 6pm UTC. Adjust if needed.
- GitHub Actions has a max 6-hour delay, so actual updates might be ±30 min.
- The script only commits if changes were made (won't spam with empty commits).
