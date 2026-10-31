# Cookbook

Real flows, shortest path first. Every command here is the same code path the test
suite exercises; nothing is illustrative-only.

---

## 1. Rebrand a docker-compose app, start to finish

```bash
relabeld scan ./app --json > scan.json        # read-only: what would change
relabeld init ./app --name "Nova Status" --domain status.nova.example \
  --primary "#2563EB" --logo ./nova.svg --favicon ./nova.ico
relabeld apply ./app --dry-run                 # the plan, nothing written
relabeld apply ./app                           # patch strings, assets, env, compose
docker compose -f app/docker-compose.yml up -d
relabeld verify --url http://localhost:3001 --brand app/brand.yaml --json
relabeld report --input verify.json --out report.html
```

`apply` is deterministic: the same `brand.yaml` and the same input produce the same
bytes. Run it twice and the second run reports `changed: 0`.

---

## 2. Gate a deploy in CI

```yaml
name: rebrand
on: [push]

jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker compose up -d && sleep 10
      - uses: horizonbymuneeb/relabel@v1.1.0
        with:
          url: http://localhost:3001
          brand: brand.yaml
          report: relabel-report.html
          fail-on: brand,console,overflow
```

The Action runs `verify`, applies the `fail-on` policy, writes the screenshots and
the JSON proof into `relabel-evidence`, and posts the failed checks to the job
summary.

`fail-on` takes groups: `brand`, `console`, `overflow`, `screenshot(s)`, `favicon`.
Anything outside the list is reported as a warning and does not fail the build —
useful while a rebrand is still settling.

---

## 3. Verify a dev server while you iterate

```bash
# terminal 1
npm run dev

# terminal 2 — rerun the gate after every change
while true; do
  relabeld verify --url http://localhost:3000 --brand brand.yaml --json \
    --shots .relabel/verify \
    | node -e 'let s="";process.stdin.on("data",d=>s+=d).on("end",()=>{const r=JSON.parse(s);console.log(r.summary, r.checks.filter(c=>c.status==="fail").map(c=>c.id))})'
  sleep 3
done
```

The screenshots land in `.relabel/verify` on every run, so you can watch the layout
tighten rather than guess.

---

## 4. Add a recipe for an app that is not in the library

```bash
relabeld recipe list                          # what ships
relabeld scan ./new-app --json | head -40     # which strings and assets exist
cp -r recipes/uptime-kuma recipes/new-app     # start from the closest one
$EDITOR recipes/new-app/recipe.yaml           # match globs, strings, assets, renames
relabeld recipe add recipes/new-app           # install it into the library
relabeld apply ./new-app --recipe new-app --dry-run
```

A recipe is data, not code: `match` (which files prove this is that app), `strings`
(old names), `assets` (logo/favicon targets), `renames` (files whose names contain
the brand), `compose` (service/env mapping). See [recipe-spec.md](recipe-spec.md).

---

## 5. Two brands, one codebase (staging + production)

Keep one `brand.yaml` per target and let `apply` take the file:

```bash
relabeld apply ./app --brand brands/staging.yaml
relabeld apply ./app --brand brands/prod.yaml
```

Because the patching is deterministic and every asset copy is hash-checked, you can
rebuild either brand from a clean checkout at any time — the output is the input.

---

## 6. When verification fails

| Failed check | What it actually means | First thing to try |
|---|---|---|
| `brand-title` / `brand-strings` | the old name is still rendered somewhere | `relabeld scan ./app --json` — the string lives in a place the recipe does not patch |
| `console-errors` | the page throws after the rename (often a hard-coded asset path) | open the screenshot in `.relabel/verify`, then the browser console at the same URL |
| `overflow-desktop` / `overflow-mobile` | the new name is longer than the layout expects | shorten the brand, or raise the container width in the theme |
| `favicon` | the served favicon hash is not the one `brand.yaml` declares | `apply` copies assets; check the file was included in the image build |
| `screenshots` | the capture itself failed (browser, timeout, unreachable URL) | `relabeld doctor` |

Every check id is stable — matching on it in `fail-on` is supported.

---

## 7. Roll back a rebrand

`apply` writes backups beside the files it replaces (`.bak.<timestamp>`) and refuses
to write when a rewritten reference would dangle. To go back:

```bash
git checkout -- ./app          # if the app is in git, this is the honest rollback
relabeld apply ./app --brand brand.yaml --dry-run   # confirm what would change now
```

`--allow-dangling` exists for the rare case where you *want* the rename even though a
reference in the repo points at the old path — read the warning before using it.
