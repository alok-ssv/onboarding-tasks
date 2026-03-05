# Repository Context Snapshot

Use this file as the version anchor for all onboarding docs in this repository.

Snapshot date (UTC): `2026-03-05`

## Source repositories

| Repository | Local path | Commit |
| --- | --- | --- |
| `ssvlabs/ssv` | `../ssv` | `ac765ac67` |
| `ssvlabs/ssv-spec` | `../ssv-spec` | `45153e4e` |
| `ssvlabs/charts` | `.sources/charts` | `3cc70ed` |
| `ssvlabs/gitops-stage` | `.sources/gitops-stage` | `ea9471be` |
| `ssvlabs/gitops-production` (`gitops-prod`) | `.sources/gitops-production` | `8c388b9f` |

## Refresh procedure

Run from `/Users/Alok/dev/onboarding-tasks`:

```bash
git -C ../ssv rev-parse --short HEAD
git -C ../ssv-spec rev-parse --short HEAD
git -C .sources/charts rev-parse --short HEAD
git -C .sources/gitops-stage rev-parse --short HEAD
git -C .sources/gitops-production rev-parse --short HEAD
date -u +%Y-%m-%d
```
