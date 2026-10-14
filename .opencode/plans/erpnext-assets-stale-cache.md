# ERPNext asset 404s — diagnosis, fixes applied, prevention in repo

## Status: COMPLETE (pending commit approval)

| Item | State |
|---|---|
| Round 1 fix (2026-10-06): stale Redis `assets_json` cleared | Applied — then **recurred** |
| Round 2 diagnosis: recurrence root cause | Found — diverged per-container manifests |
| Round 2 fix: manifest sync + cache clear + backend restart | Applied & verified (**62/62 assets → 200**) |
| Prevention hook in repo (approved) | **Applied** — `deploy.yml` + `group_vars/all.yml`, syntax-check clean |
| `verify.yml` | **PASS** (9/9) |
| Commit | Not made — awaiting explicit permission |

## Recurrence root cause (round 2)

Each bench container owns a **private anonymous volume** at `sites/assets`
(image declares `VOLUME /home/frappe/frappe-bench/sites/assets`), so the
asset manifests diverged:

| Container | old `assets.json` | dist (own apps dir) |
|---|---|---|
| frontend (serves) | `017ab1…` GJUJFKCA | GJUJFKCA |
| backend (generates URLs) | `e8caff…` GJUJFKCA | 7HEULYOT (unused) |
| queue-short / queue-long / scheduler | `6968a50…` **MDFR7SDL (old build)** | QNF6VVCK |

Mechanism: any site-cache clear deletes the shared, no-TTL Redis key
`assets_json`; whichever container regenerates first wins. Queue/scheduler
held the **old** manifest → re-poisoned Redis → site broken again within hours.
`frappe.clear_cache()` fires on routine actions (settings saves, UI
"Clear Cache"), so clears recur naturally — the poison was the diverged files,
not the clear itself.

## Fixes applied (production, via Ansible ad-hoc)

1. **Synced the canonical manifest** (frontend's — verified: all 60 URLs in
   `assets.json` + `assets-rtl.json` resolve to existing files) into
   `frappe-backend-1`, `frappe-queue-short-1`, `frappe-queue-long-1`,
   `frappe-scheduler-1` via `docker cp`. All 5 now identical:
   `017ab169…` / `2a91c21c…`.
2. `docker exec frappe-backend-1 bench --site Jeyshawn clear-cache`.
3. `docker restart frappe-backend-1` — drops the workers' 10-minute
   in-process `ClientCache` (`frappe/utils/redis_wrapper.py`, `class ClientCache`)
   that still served the poisoned value.
4. RTL fixed as a side effect (synced `assets-rtl.json` → boot now points at
   real `css-rtl` files).

Evidence: Redis MONITOR captured `172.18.0.6 GET (miss) → SET assets_json
(GJUJFKCA content) → GET (hit)`; `EXISTS=1`, value `login.bundle.GJUJFKCA.css`.

## Prevention (repo changes, this change)

`ansible/playbooks/deploy.yml` — added hook tasks (pre / post / migrate,
`tags: [deploy, hooks]`, no-op unless vars set), matching `ansible/SKILL.md`
reference playbook.

`ansible/inventory/group_vars/all.yml`:

```yaml
app_post_deploy_cmd: "docker exec frappe-backend-1 bench --site Jeyshawn clear-cache"
```

Why this is sufficient for recreates: probed the image with `docker create`
(no start) — `custom:16` ships a **self-consistent** pair
(`login.bundle.QNF6VVCK.css` in both manifest and dist, RTL `47YD5KKH`).
After a stack recreate the only staleness risk is the no-TTL Redis key, which
the post-deploy hook clears.

## Verification results

- Login page renders; every referenced asset **200**: 62/62 (LTR + RTL + theme).
- `ansible/playbooks/verify.yml` → `ok=9 failed=0`.
- `ansible-playbook deploy.yml --syntax-check` → clean.

## Remaining recommendations (not done — need explicit approval)

1. **Commit** the two repo files (+ this plan doc if wanted).
2. **Structural hardening:** mount one shared named volume at `sites/assets`
   for all bench services in `overrides/compose.prod.yaml` so manifests can
   never diverge again (requires `docker compose up -d` recreate → short downtime).
3. Local tooling: `ansible/ansible.cfg` `stdout_callback = yaml`
   (community.general) crashes under local Python 3.14 — worked around with
   `ANSIBLE_STDOUT_CALLBACK=default`; consider switching to
   `ansible.builtin.default`.
