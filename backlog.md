# Ear Tuner — Backlog

<details>
<summary>Prefixes (for referencing items in other backlogs)</summary>

| Prefix | Project |
|--------|---------|
| P | Fiddle App Family (parent) |
| B | MicroBreaker |
| E | Ear Tuner (this project) |
| H | TuneHub |
| I | Intonio |
| L | TuneList |
| M | Media Markup |

</details>

| ID  | P   | S   | Description                                        | Notes                |
| --- | --- | --- | -------------------------------------------------- | -------------------- |
| B1 | P1 | . | Fix focus-loss audio silence bug | iOS only |
| E1 | P3 | . | Lower the smile arc status line so it appears in the correct facial position when animating | Currently the arc sits too high; should arc lower to look like a smile in the right place on the face |
| P1 | P2 | . | Restructure repo: rename ear-tuner to ear-tuner-src, create new ear-tuner repo for PWA build output served via GitHub Pages | Build script should copy src/renderer/ into deployment repo and push. Needed to prevent exposing backlog, research docs, config files publicly. |
