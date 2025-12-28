import requests
from datetime import datetime, timezone, timedelta
import pytz
from pathlib import Path
import time

LAT = 33.0198
LNG = -96.6989
LOCAL_TZ = pytz.timezone("America/Chicago")
API_URL = "https://api.sunrise-sunset.org/json"
DAYS_AHEAD = 400  # generate plenty of future dates so the feed stays useful

def get_sun_times(date_str: str):
    params = {"lat": LAT, "lng": LNG, "date": date_str, "formatted": 0}
    r = requests.get(API_URL, params=params, timeout=20)
    r.raise_for_status()
    data = r.json()
    if data.get("status") != "OK":
        raise RuntimeError(f"API status={data.get('status')}: {data}")
    return data["results"]["sunrise"], data["results"]["sunset"]

def ics_escape(s: str) -> str:
    return s.replace("\\", "\\\\").replace(";", "\\;").replace(",", "\\,").replace("\n", "\\n")

def fmt_dt(dt: datetime) -> str:
    # Floating local time with TZID, format: YYYYMMDDTHHMMSS
    return dt.strftime("%Y%m%dT%H%M%S")

def build_ics():
    now_utc = datetime.now(timezone.utc)
    today = now_utc.date()
    lines = []
    
    lines.append("BEGIN:VCALENDAR")
    lines.append("VERSION:2.0")
    lines.append("PRODID:-//Alejandro//Sun Calendar//EN")
    lines.append("CALSCALE:GREGORIAN")
    lines.append("METHOD:PUBLISH")
    
    # Add VTIMEZONE for America/Chicago
    lines.append("BEGIN:VTIMEZONE")
    lines.append("TZID:America/Chicago")
    lines.append("BEGIN:STANDARD")
    lines.append("DTSTART:20231105T020000")
    lines.append("TZOFFSETFROM:-0500")
    lines.append("TZOFFSETTO:-0600")
    lines.append("RRULE:FREQ=YEARLY;BYMONTH=11;BYDAY=1SU")
    lines.append("END:STANDARD")
    lines.append("BEGIN:DAYLIGHT")
    lines.append("DTSTART:20230312T020000")
    lines.append("TZOFFSETFROM:-0600")
    lines.append("TZOFFSETTO:-0500")
    lines.append("RRULE:FREQ=YEARLY;BYMONTH=3;BYDAY=2SU")
    lines.append("END:DAYLIGHT")
    lines.append("END:VTIMEZONE")
    
    for i in range(DAYS_AHEAD):
        d = today + timedelta(days=i)
        date_str = d.strftime("%Y-%m-%d")
        
        try:
            sunrise_iso, sunset_iso = get_sun_times(date_str)
        except Exception as e:
            print(f"Failed to get sun times for {date_str}: {e}")
            continue
        
        sunrise_utc = datetime.fromisoformat(sunrise_iso).astimezone(pytz.utc)
        sunset_utc = datetime.fromisoformat(sunset_iso).astimezone(pytz.utc)
        sunrise_local = sunrise_utc.astimezone(LOCAL_TZ)
        sunset_local = sunset_utc.astimezone(LOCAL_TZ)
        
        created_utc = now_utc.strftime("%Y%m%dT%H%M%SZ")
        
        def add_event(title: str, start_local: datetime):
            uid = f"{date_str}-{title.replace(' ', '').lower()}@sun-calendar"
            lines.append("BEGIN:VEVENT")
            lines.append(f"UID:{uid}")
            lines.append(f"DTSTAMP:{created_utc}")
            lines.append(f"SUMMARY:{ics_escape(title)}")
            lines.append(f"DTSTART;TZID=America/Chicago:{fmt_dt(start_local)}")
            lines.append("TRANSP:TRANSPARENT")  # Mark as free time
            
            # Add reminder
            lines.append("BEGIN:VALARM")
            lines.append("ACTION:DISPLAY")
            lines.append("TRIGGER:-PT15M")  # 15 min before
            lines.append(f"DESCRIPTION:{title}")
            lines.append("END:VALARM")
            
            lines.append("END:VEVENT")
        
        add_event("🌅 Sunrise", sunrise_local)
        add_event("🌇 Sunset", sunset_local)
        
        # Rate limiting - be nice to the API
        if i > 0:
            time.sleep(0.1)
    
    lines.append("END:VCALENDAR")
    return "\n".join(lines) + "\n"

if __name__ == "__main__":
    out_dir = Path("docs")
    out_dir.mkdir(parents=True, exist_ok=True)
    ics_text = build_ics()
    (out_dir / "sun.ics").write_text(ics_text, encoding="utf-8")
    print("✅ Sun calendar generated: docs/sun.ics")
