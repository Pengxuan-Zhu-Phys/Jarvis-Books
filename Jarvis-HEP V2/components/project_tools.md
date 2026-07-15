# Component — Project tools (`project_scaffold`, `project_packager`, `official_project_library`, `project_crypto`)

**Role**: `Jarvis2 project …` — scaffold, pack, official catalog list/fetch/info, and
**restricted-archive encrypt/decrypt** (CLI only; users never run `openssl` by hand).
**Status**: **As-built** (D12.5 + D12.6, `jarvis2` ≥ `2056e3a`).
**Design refs**: closeout §5.3–5.4; Examples `catalog/README.md`.
**Modules**:
`jarvishep2/project_scaffold.py`, `project_packager.py`, `official_project_library.py`,
`project_crypto.py`, `client.py` (`dispatch_project`).

---

## 1. Commands (all crypto through this CLI)

| Command | Purpose |
|---------|---------|
| `Jarvis2 project create <name>` | Scaffold `bin/` `data/` `deps/` + markers |
| `Jarvis2 project pack [path] [--share\|--repro\|--full] [--man]` | Pack tarball |
| `Jarvis2 project pack [path] --encrypt --key KEY` | Pack **then** encrypt → `*.tar.gz.jenc` |
| `Jarvis2 project encrypt <archive.tar.gz> --key KEY` | Encrypt an existing pack |
| `Jarvis2 project list` / `browse` | Official library table (**Access** + **Key**) |
| `Jarvis2 project info <name>` | Details including key requirement / hint |
| `Jarvis2 project fetch <name> [--key KEY]` | Download; decrypt if restricted; unpack |

Key env (optional): `JARVIS_PROJECT_FETCH_KEY` (same value as `--key`).

Catalog override: `JARVIS_OFFICIAL_LIBRARY_INDEX_URL` (`https://…` or `file:///…`).

---

## 2. Catalog (no PyPI package)

True source is **one GitHub JSON** in Jarvis-Examples:

```text
https://raw.githubusercontent.com/Pengxuan-Zhu-Phys/Jarvis-Examples/main/catalog/official_project_library.json
```

Local mirror in the Examples repo: `catalog/official_project_library.json` (+ `catalog/README.md`).

Jarvis2 resolution order:

1. Remote index URL (default above, or env override)
2. User cache `~/.jarvis/cache/official_catalog.json` (last successful pull)
3. Packaged snapshot `jarvishep2/card/official_project_library.json`

### Catalog fields (schema_version 1)

| Field | Meaning |
|-------|---------|
| `name` | Id for `fetch` / `info` |
| `access` | `public` or `restricted` |
| `requires_key` | If true, `fetch` needs `--key` / env |
| `encryption.scheme` | `none` or `openssl-aes-256-cbc` |
| `encryption.hint` | Shown in `info` and fetch errors |
| `archive_url` | Plain `.tar.gz` or encrypted `.jenc` |
| `archive_root` | Subdir in archive; `.` = auto-detect |
| `entrypoint` | Suggested task YAML after fetch |

---

## 3. Public vs restricted (usage)

### End users

```bash
# Who needs a key?
Jarvis2 project list

# Public example
Jarvis2 project fetch Eggbox
cd Eggbox
Jarvis2 run bin/….yaml

# Restricted example (choose one style)
Jarvis2 project fetch SecretName --key 'YOUR_KEY'
export JARVIS_PROJECT_FETCH_KEY='YOUR_KEY'
Jarvis2 project fetch SecretName

Jarvis2 project info SecretName   # Access / Key required / hint
```

`list` / `browse` sample columns:

```text
Name        Access       Key         Category    Summary
Eggbox      public       no          sampling    …
SecretX     restricted   required    …           …
```

### Maintainers (restricted release)

```bash
# One-shot pack + encrypt
Jarvis2 project pack MyPrivate --repro --encrypt --key 'YOUR_KEY'
# → MyPrivate_repro_….tar.gz and MyPrivate_repro_….tar.gz.jenc

# Or encrypt after pack
Jarvis2 project pack MyPrivate --repro
Jarvis2 project encrypt MyPrivate_repro_….tar.gz --key 'YOUR_KEY'
```

Then:

1. Upload the **`.jenc`** (public ciphertext Release is OK; private URL is stricter).
2. **Do not** push the plaintext private tree to the public Examples repo.
3. Append a `restricted` entry to `Jarvis-Examples/catalog/official_project_library.json`.
4. Share the key out-of-band (not in git).

---

## 4. Encryption technology (implementation detail)

Users only use the CLI above. Internally:

| Item | Value |
|------|--------|
| Format | OpenSSL “Salted__” AES-256-CBC |
| KDF | PBKDF2-HMAC-SHA256 (default OpenSSL `-pbkdf2` iterations) |
| Scheme id in catalog | `openssl-aes-256-cbc` |
| Backend priority | (1) system `openssl` on PATH (2) optional `pip install cryptography` |

Interoperable with:

```bash
# Not required for normal use — CLI is preferred
openssl enc -aes-256-cbc -pbkdf2 -salt -in plain.tar.gz -out x.jenc -pass pass:KEY
openssl enc -aes-256-cbc -pbkdf2 -d -in x.jenc -out plain.tar.gz -pass pass:KEY
```

### What encryption does *not* hide

- The **catalog JSON** is public (names, summaries, URLs, “key required”).
- A **plaintext** project tree on a public git repo is still public.
- Encryption protects **archive payload content** given a shared passphrase model.

---

## 5. Layout markers (scaffold)

| Path | Role |
|------|------|
| `jarvis.project.yaml` | Descriptor (`&J`, defaults) |
| `.jarvis-project.json` | Machine marker |
| `bin/` `data/` `deps/` | Standard layout |
| Template defaults | `EnvReqs.V2` via `deps/environment_default.yaml` (no top-level `Runtime`) |

---

## 6. Tests

`tests/test_project_tools.py`: scaffold, pack modes, catalog normalize (public/restricted),
list table Key column, encrypt/decrypt round-trip, `pack --encrypt`, `encrypt`, fetch with key
on `file://` encrypted archive.

---

## 7. Related docs

- Examples catalog: `Jarvis-Examples/catalog/README.md`
- Install / CLI quickref: `Jarvis-HEP-v2/INSTALL.md` § Project tools
- Plan: D12.5 / D12.6 in [`../V2_DISTRIBUTED_PLAN.md`](../V2_DISTRIBUTED_PLAN.md)
