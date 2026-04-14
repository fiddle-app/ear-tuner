# Ear Tuner

See parent [fiddle/CLAUDE.md](../CLAUDE.md) for project context.

## Purpose

Ear training and pitch discrimination tool. Plays two tones and asks the user
to identify which is higher/lower or how they differ. Designed for fiddle
players developing relative pitch.

## Status

Built (PWA); design review pending.

## Target Platform

PWA today, served via GitHub Pages (`fiddle-app.github.io/ear`).
Future: Capacitor iOS wrap for iPhone/iPad + App Store.

## Key Files

- `index.html` — single-file app (HTML + CSS + JS)
- `sounds/` — audio sample files
- `specs/ear-tuner-spec.md` — design spec (fonts, colors, UI decisions)

## Known Issues

Known issues are tracked in [backlog.md](backlog.md)
