# IH-Plants 🌿

A React Native iOS app that helps you care for your houseplants. Photograph your plants, get automatic species identification, and never forget to water them again.

## What it does

- **Photo-based plant identification** — take or upload a photo and the app identifies the species via the Plant.id API, pulling real botanical watering data from MOBOT & RHS
- **Smart watering schedule** — watering frequency is derived automatically from the plant's moisture preference (dry = every 14 days, medium = every 7, wet = every 3) and is fully editable
- **Daily reminders** — push notifications at 9 AM tell you exactly which plants need water or spraying that day
- **Plant dashboard** — see all your plants at a glance with last watered date, plant age, next water date, and a fun celebration animation when everything's been taken care of
- **Works on iPhone** — distributed via TestFlight using Expo EAS Build (no Mac required to build)

## Tech stack

| Layer | Choice |
|---|---|
| App | React Native (Expo) + TypeScript |
| Storage | Expo SQLite (local, no backend) |
| Species AI | Plant.id API v3 |
| Notifications | Expo Notifications |
| Animation | Lottie |
| Distribution | TestFlight via EAS Build |

## Spec — what's ready

The full spec-driven design is in `.kiro/specs/plant-watering-tracker/`:

- `design.md` — full technical design including architecture diagrams, sequence flows, data models, algorithmic pseudocode, UI wireframes, error handling, and deployment plan
- `requirements.md` — 14 requirements with formal acceptance criteria covering every feature
- `tasks.md` — 16 implementation tasks with sub-tasks, ready to execute

## Status

> Spec complete. Implementation not yet started.

To resume: open this workspace in Kiro and say "run all tasks".
