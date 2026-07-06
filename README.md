# Komodor Klaudia Sync Example Repo

This repository is the companion example for the main action project:
[komodor-klaudia-sync](https://github.com/davidcollom/komodor-klaudia-sync)

It exists to show how to structure content and workflows when using the Klaudia Sync GitHub Action in a separate customer-style repository.

## What This Repo Contains

- `kb/`: Example knowledge-base runbooks
- `blueprints/`: Example blueprint documents
- `.github/workflows/sync-kb.yaml`: Example workflow that syncs both content types with a matrix

## How It Relates To The Main Repo

- Main repo (`komodor-klaudia-sync`): Implements and releases the action/CLI.
- This repo (`komodor-klaudia-sync-example`): Demonstrates real-world usage of `uses: davidcollom/komodor-klaudia-sync@v1`.

## Usage Notes

- Add `KOMODOR_API_KEY` in repository secrets.
- Push changes under `kb/` or `blueprints/` to trigger the sync workflow.
- The workflow runs one job per content type:
  - `knowledge-base` from `./kb`
  - `blueprint` from `./blueprints`

## Reference

For guidance on expected document structure in Klaudia, see:
[https://help.komodor.com/hc/en-us/articles/32463600112786-Klaudia-md-Organizational-Blueprint-Knowledge]
