# New GCP Environment Deployment Checklist

Living checklist for standing up BigLake in a new GCP project/environment
(e.g. `prod`, or any future environment). Update this file whenever the
process changes — treat it as the canonical runbook, not a one-off record.

**Deployment order is fixed by dependency:** `infra` → `catalog` → `etl` →
`knowledge` → `intelligence` → `api` → `ui`. Each phase below assumes the
previous ones are complete.

Each repo also has its own `SETUP.md` with the authoritative low-level
steps and current field/secret names — this checklist sequences them and
tracks cross-repo gotchas. If a repo's `SETUP.md` and this file disagree,
`SETUP.md` wins; update this file to match.

---

## Phase 0 — Terraform module parity check (do this first, every time)

`infra/environments/{env}/main.tf` files tend to drift — a new environment's
file (or an existing one mid-build) can lag behind the most complete/current
environment. Before doing anything else:

- [ ] Diff `infra/environments/{new-env}/main.tf` against the most
      up-to-date environment's `main.tf` (currently `test`)
- [ ] Confirm every module block present in the reference env also exists
      in the target env, with matching parameters:
      `core`, `network`, `storage`, `iam`, `metastore`, `auth_secrets`,
      `email`, `api`, `ui`, `prefect`, `openmetadata`, `knowledge_db`,
      `intelligence`, `cloud_armor`, `lb`, `dns`
- [ ] Confirm all `output` blocks match (`wif_provider`, `metastore_*`, etc.)
- [ ] Confirm `locals` are correctly scoped for the new env — especially
      `env`, `project_id`, `region`, `zone`, `dns_zone`, `subdomain_prefix`,
      `app_host`/`api_host`/`om_host`
- [ ] `variables.tf` for the new env — confirm it matches expectations
      (may legitimately be empty if all config lives in `locals`)
- [ ] Env-specific settings that should NOT be copied verbatim from another
      env: `force_destroy` on storage (test-only convenience),
      `developer_token_creator_members` (should be empty in prod — no dev
      impersonation), `geo_restrict_au` on `cloud_armor`

**Architecture decision to make explicitly for a new env sharing the
`biglake.au` domain:** does the new env get its own DNS zone (registrar
re-delegation) or a delegated subzone of the existing zone? Don't default
to copying the current env's `subdomain_prefix` pattern without deciding
this — it has real DNS/cert implications.

---

## Phase 1 — GCP project & GitHub prerequisites (`infra`)

1. [ ] Create GCP project, confirm billing linked
2. [ ] Enable APIs: `iam`, `compute`, `storage`, `secretmanager`,
       `cloudresourcemanager`, `iap`, `aiplatform` (Vertex AI, needed by
       `etl`)
3. [ ] Create `terraform-deploy` SA + grant bootstrap roles
       (`infra/SETUP.md` §2–3)
4. [ ] Create WIF pool + OIDC provider scoped to `big-lake/*` repos (§4)
5. [ ] Bind `terraform-deploy` SA to WIF for the `infra` repo (§5)
6. [ ] Create the GitHub environment (e.g. `prod`) in every repo: infra,
       api, ui, etl, catalog, knowledge, intelligence
7. [ ] Set `WIF_PROVIDER` secret (and `WIF_SERVICE_ACCOUNT` for infra) in
       each repo's new environment — value from
       `terraform output -raw wif_provider` after the first infra apply
8. [ ] Set `CONTINUOUS_DEPLOYMENT_ENABLED` variable per repo — recommend
       `false` initially for a new/prod-like env; flip on once stable
9. [ ] Trigger `Terraform Deploy` workflow (infra) targeting the new env
10. [ ] Import bootstrap WIF resources into TF state (§9)

## Phase 2 — Secrets & domain (`infra`)

11. [ ] Populate Secret Manager values via **GCP Console UI only** — never
        CLI (`gcloud secrets versions add`). Full table in
        `infra/SETUP.md` §10: GCS HMAC keys, OM MySQL passwords,
        `api-admin-password`, `api-jwt-secret`, `session-jwt-secret`,
        `magic-link-hmac-secret`, OAuth client id/secret ×3 providers
        (Google/LinkedIn/GitHub — **register new OAuth apps/redirect URIs
        for the new env's hostnames**, don't reuse another env's),
        `knowledge-pgvector-password`, `knowledge-neo4j-password`
        - Note: `dev-session-secret-test` is test-only by design
          (ADR-0017) — Terraform doesn't create this shell outside `test`
12. [ ] `knowledge_db` bootstrap order: populate passwords → apply infra →
        populate `knowledge-pgvector-dsn-{env}`/`knowledge-neo4j-auth-{env}`
        using the resolved internal DNS name
13. [ ] Configure Lakehouse runtime catalog (`gcloud beta biglake iceberg
        catalogs create`) for both lakehouse and raw buckets + grant
        `biglake.viewer`/`biglake.editor` IAM (`infra/SETUP.md` §11)
14. [ ] Resolve the domain/DNS decision from Phase 0, then delegate at the
        registrar (cheaperdomains.com.au) if a new zone is needed
15. [ ] Wait for the managed cert to go `ACTIVE`, then run the Phase 2
        cutover (`recreate_api_vm=true`) to drop public IPs/open firewalls
16. [ ] Smoke-test: `curl https://api.{host}/health`,
        `https://om.{host}/api/v1/system/version`, `https://app.{host}/`

## Phase 3 — Catalog (`catalog`, depends on infra)

17. [ ] First login to OM (via IAP tunnel to port 8585), change admin
        password immediately
18. [ ] Set `WIF_PROVIDER`, `WIF_SERVICE_ACCOUNT`,
        `OPENMETADATA_ADMIN_PASSWORD` GitHub secrets for the new env
19. [ ] Set `PROJECT_ID` env-level variable
20. [ ] Trigger `sync_catalog.yml` (custom properties, storage service,
        knowledge registry all self-provision)
21. [ ] Trigger the `Ingest` workflow, verify tables appear in OM UI

## Phase 4 — ETL (`etl`, depends on infra)

22. [ ] Confirm Vertex AI API enabled (Phase 1)
23. [ ] Set `WIF_PROVIDER`, `GCS_UPLOAD_SA_EMAIL` secrets + `PROJECT_ID`/
        `CONTINUOUS_DEPLOYMENT_ENABLED` variables
24. [ ] Trigger `Deploy Prefect` workflow
25. [ ] Verify `prefect-server`/`prefect-worker` systemd units active on
        `biglake-prefect-{env}`
26. [ ] **Run all etl pipelines against the new env** to seed gold-layer
        datasets — re-run from source/landing, do NOT bulk-copy Iceberg
        tables between environments (baked absolute GCS paths — see
        `/memories/repo/environment-copy-plan.md` for why)

## Phase 5 — Knowledge (`knowledge`, depends on infra + etl's Prefect VM)

27. [ ] Verify `knowledge_db` VM up, connection secrets populated
28. [ ] Verify Prefect deployments for knowledge flows registered on the
        shared VM
29. [ ] Run flows 1–4 (ingest → chunk → enrich → embed_and_index) against
        real sources for this env — check `pipeline.yaml` source configs
        are appropriate for the target env
30. [ ] Confirm manifest published at
        `gs://{bucket}/{manifest_prefix}/{version}/manifest.json`

## Phase 6 — Intelligence (`intelligence`, depends on knowledge_db + manifest)

31. [ ] Confirm `infra/modules/intelligence/startup.sh` env vars point at
        the correct manifest prefix/version for this env
32. [ ] Trigger `Deploy Intelligence` workflow
33. [ ] Verify `/health` via IAP tunnel returns `status: ok` with correct
        `manifest_version`/`embedding_model`
34. [ ] Smoke-test `/retrieve` and `/chat`

## Phase 7 — API (`api`, depends on infra + intelligence + catalog)

35. [ ] Confirm OAuth apps registered (Google/LinkedIn/GitHub) with this
        env's callback URLs, secrets populated
36. [ ] Confirm Workspace domain-wide delegation for `noreply@biglake.au`
        magic-link sending covers this env (one-time per domain, not
        per-env, if already configured)
37. [ ] Trigger `Deploy` workflow
38. [ ] Verify `curl https://api.{host}/health` →
        `{"status":"ok", "litestream": {...}}`
39. [ ] Verify OM soft-boot dependency: if OM unreachable at api boot,
        restart api after OM is healthy

## Phase 8 — UI (`ui`, depends on api deployed first)

40. [ ] Trigger `Deploy` workflow (bakes `VITE_API_BASE_URL` at build
        time — confirm this resolves to the LB/DNS hostname correctly,
        not a bare VM IP, once a global LB is in front)
41. [ ] Verify `https://app.{host}` loads the login page

## Phase 9 — End-to-end verification

42. [ ] Login flow (magic link / OAuth / passkey) works
43. [ ] Catalog sidebar populates from real OM data
44. [ ] Data query (`/query`) returns real rows from GCS/DuckDB
45. [ ] Chat/agent turn returns a real cited answer from intelligence
46. [ ] Notebook create/publish/share round-trip
47. [ ] Run the Newman CI suite against the new env — add a matching
        Postman environment file under `api/postman/environments/` if one
        doesn't exist yet

---

## Known outstanding gaps (check before relying on these in a new env)

- [ ] Litestream backup of API SQLite (ADR-0010) — not yet implemented in
      `infra/modules/api/startup.sh`
- [ ] Read-only pgvector role for `intelligence` — currently shares creds
      with other consumers
- [ ] Verify curation status of gold-layer datasets before wiring `api`
      to catalog in a new env

---

## Notes for future runs

- Keep this file's phase numbering stable across edits so notes/PRs can
  reference "Phase 5 step 29" etc. without going stale.
- When a phase's steps change (new secret, new module, new workflow),
  update this file in the same change — don't let it drift the way
  `infra/environments/prod/main.tf` drifted from `test/main.tf`.
- Record env-specific decisions made during a real rollout (e.g. the DNS
  subzone-vs-new-zone choice) in the relevant repo's ADR, not here — this
  file is the process checklist, not the decision log.
