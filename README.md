# 🎬 Ostrich Studios

A single-file web app for filmmakers to build professional shot lists directly from screenplays. No backend, no installation required — just open in your browser and start annotating.

## 📋 What It Does

Transform your screenplay into a detailed, production-ready shot list with cinematography metadata for every moment.

### Core Workflow

1. **Upload or paste** your script (plain text or `.fdx`)
2. **Click any word** in the script to assign shot details
3. **Build your shot list** automatically in the right panel
4. **Review, flag, and export** for production

## 🎥 Shot Metadata

Assign comprehensive cinematography details to each shot:

| Field | Options |
|-------|---------|
| **Shot Type** | Extreme Wide, Long, Medium Long, Mid, Close Up, Extreme Close Up |
| **Angle** | Bird's Eye, High, Eye Level, Low, Worm's Eye, Over-the-Shoulder |
| **Movement** | Static, Panning, Tilt, Tracking/Dolly, Handheld, Steadicam, Crane/Jib |
| **Motion** | Static / Moving toggle |
| **Notes** | Free text for director/DP notes |

## ✨ Key Features

- **Color-coded workflow** — Green for annotated shots, red for flagged revisions
- **Live shot review** — Print list with checkboxes; mark shots for revision in real-time
- **Real-time sync** — Changes in the main script instantly update the review tab
- **Automatic persistence** — All work saved to browser localStorage; survives page refresh
- **Multiple export formats** — Print-ready PDF with branding, or CSV for spreadsheets
- **Offline compatible** — Works completely offline via `file://` protocol

## ⚡ Performance

Built for speed and efficiency:

- **DocumentFragment rendering** — Handles 10,000+ word scripts smoothly
- **O(1) DOM updates** — No full re-renders; only changed elements update
- **Map-based lookups** — 10-50x faster than array searches
- **Optimized animations** — GPU-accelerated transitions for buttery-smooth UX

## 🛠️ Tech Stack

- **Pure HTML/CSS/JavaScript** — Zero dependencies, zero build tools
- **Single-file architecture** — Entire app in one `index.html` file
- **Google Fonts** — Cormorant Garamond (UI), Courier New (script text)
- **localStorage API** — Client-side persistence

## 🎨 Branding

**Ostrich Cinemas** · [@ostrich.cinemas](https://instagram.com/ostrich.cinemas)

- Color scheme: Cream (`#f5f2e8`) on black
- SVG ostrich logo in header and print outputs

## 🚀 Getting Started

### Option 1: GitHub Pages (Live Demo)
Visit: **[https://anshul3102.github.io/OstrichS-S/](https://anshul3102.github.io/OstrichS-S/)**

### Option 2: Local Usage
```bash
# Clone the repo
git clone https://github.com/Anshul3102/OstrichS-S.git

# Open index.html in your browser
cd OstrichS-S
open index.html  # macOS
start index.html # Windows
```

### Option 3: Offline
Download `index.html` and open it directly — works completely offline!

## 📦 Repository

**GitHub:** [github.com/Anshul3102/OstrichS-S](https://github.com/Anshul3102/OstrichS-S)

## 📄 License

[Add your license here - e.g., MIT, Apache 2.0, etc.]

## 🤝 Contributing

Contributions welcome! Please open an issue or submit a pull request.

---

Built with ❤️ by [Ostrich Cinemas](https://instagram.com/ostrich.cinemas)
