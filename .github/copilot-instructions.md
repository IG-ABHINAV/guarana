# Copilot instructions

Read [AGENTS.md](../AGENTS.md) at the repository root before changing anything; it is the source of truth for this repository. The short version:

- Ship only a measured gain on a verified scene (`tools/bench` rows with `scene_ok: true`, baseline next to the change).
- New behaviour without a verified gain goes behind an `EDEN_NX2_*` environment switch that is off by default, and every switch gets a row in `docs/switches.md`.
- Never commit keys, firmware, game files or assets, dumps, shader caches, saves, logs with personal paths, or personal data.
- Commit subjects are `area: change` in the imperative, with measured numbers in the body.
- Check zortos293/Lichii for an existing fix first, port it with credit, and send our general fixes back upstream.
