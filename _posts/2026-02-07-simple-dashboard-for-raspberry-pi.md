---
title: "SimpleDashboard: A Zero-Dependency Weather Dashboard for Raspberry Pi"
date: 2026-02-07 20:00:00 -0500
categories: [Automation]
tags: [raspberry-pi, javascript, dashboard, weather, kiosk, pirateweather, home-automation]
image:
  path: /assets/img/posts/simple-dashboard.png
  alt: SimpleDashboard showing time, weather, forecast, and calendar on a dark bokeh background
---

I have a vertically-mounted monitor in my kitchen that I wanted to use as a simple information display. I tried [DAKboard](https://dakboard.com/) and [MagicMirror](https://magicmirror.builders/), and while both are impressive projects with tons of extensibility, neither did exactly what I wanted without either paying for a subscription or wrestling with module configurations.

What I wanted was simple: date, time, weather with a short forecast, and a calendar. No personal data, no integrations, no build step. Just a single HTML file I could throw on a Raspberry Pi and forget about.

So I built one.

![SimpleDashboard](/assets/img/posts/simple-dashboard.png){: w="400" }
_SimpleDashboard running on a Saturday evening — 17°F with a "feels like" of 1°F. Winter in Virginia._

## What It Does

- **Clock** — large, readable from across the room, updates every second
- **Date** — day of week and full date on one line
- **Current weather** — emoji icon, temperature, and "feels like" when it differs significantly
- **3-day forecast** — icons with high/low temperatures
- **Monthly calendar** — today highlighted
- **Background image** — full-bleed with a dark overlay for text contrast

Everything auto-scales to fit the browser window, so it works on anything from a phone screen to a 50" TV.

## The Stack

The entire dashboard is a single `index.html` file with embedded CSS and vanilla JavaScript. No frameworks, no build tools, no npm. Configuration lives in a separate `config.js` so the main file never needs editing.

Weather data comes from [PirateWeather](https://pirateweather.net/), a free API that's a drop-in replacement for the old Dark Sky API. You get the current conditions and an 8-day forecast with a single API call.

```javascript
const CONFIG = {
  weatherApiKey: 'your-pirateweather-api-key',
  weatherLat: '38.8183',
  weatherLon: '-77.4173',
  weatherUnits: 'us',        // 'us' for Fahrenheit, 'si' for Celsius
  weatherRefreshMin: 15,     // refresh weather every N minutes
  backgroundImage: 'background.jpg',
  timeFormat: 12,            // 12 or 24
};
```

## Auto-Scaling

The trickiest part was making everything scale proportionally regardless of screen size. The approach: all sizing uses a CSS custom property `--s` as a base unit. On load, the content is measured at `--s: 1px` to get its natural dimensions in "scale units", then `--s` is recalculated so the content fills the viewport.

```javascript
function measureContent() {
  const el = document.getElementById('content');
  el.style.height = 'auto';
  document.documentElement.style.setProperty('--s', '1px');
  contentUnitsW = el.offsetWidth;
  contentUnitsH = el.offsetHeight;
  el.style.height = '100%';
}

function updateScale() {
  document.body.style.height = window.innerHeight + 'px';
  const s = Math.min(
    window.innerWidth / contentUnitsW,
    window.innerHeight / (contentUnitsH * 1.05)
  );
  document.documentElement.style.setProperty('--s', s + 'px');
}
```

This means every font size, spacing, and element dimension is defined as a multiple of `--s`, and the whole layout scales as a unit.

## Raspberry Pi Kiosk Mode

The project includes a `setup-pi.sh` script that handles the full kiosk setup on a Raspberry Pi:

1. Installs nginx and emoji fonts (`fonts-noto-color-emoji` is required for weather emoji on Linux)
2. Deploys the dashboard files to `/var/www/html`
3. Configures Chromium to launch in full-screen kiosk mode on boot
4. Disables screen blanking and hides the mouse cursor

```bash
git clone https://github.com/tksunw/SimpleDashboard.git
cd SimpleDashboard
cp config.js.default config.js
# edit config.js with your API key and coordinates
# add your background.jpg
./setup-pi.sh
```

Reboot, and the dashboard starts automatically. To update later, just copy the changed files to `/var/www/html/`.

## Source

The project is on GitHub: [tksunw/SimpleDashboard](https://github.com/tksunw/SimpleDashboard)
