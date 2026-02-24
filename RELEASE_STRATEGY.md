# Release Strategy

Dokumentasi aturan dan alur release untuk project ini.

---

## Daftar Isi

- [Konsep Dasar](#konsep-dasar)
- [Branch Strategy](#branch-strategy)
- [Versioning Rules](#versioning-rules)
- [Workflow Overview](#workflow-overview)
- [Alur Lengkap](#alur-lengkap)
- [Approval Gates](#approval-gates)
- [Hotfix Rules](#hotfix-rules)
- [Cherry-pick Convention](#cherry-pick-convention)
- [GitHub Secrets](#github-secrets)

---

## Konsep Dasar

Semua deploy **harus manual** — tidak ada auto-deploy saat push ke branch manapun.

Ada 3 jalur deploy:

| Jalur | Sumber | Tag | Deploy ke |
|---|---|---|---|
| **Preview** | `main` | `vX.X.X-preview` | Dev + Staging |
| **Official** | `release/vX.X.x` | `vX.X.X` | Dev + Staging + Production |
| **Hotfix** | `release/vX.X.x` | `vX.X.X` | Production only |

---

## Branch Strategy

### `main`
- Branch utama untuk development aktif
- Setiap deploy dari main menghasilkan tag **preview** (`-preview`)
- Tidak pernah deploy ke production langsung dari main
- Digunakan sebagai base saat membuat release branch

### `release/vX.X.x`
- Branch yang dibuat saat siap release ke production
- Format: `release/v1.6.x`, `release/v2.0.x`
- Dibuat dari `main` via workflow **Create Release Candidate**
- Versi patch (`.x`) di nama branch berarti bisa menerima beberapa patch/hotfix
- Setelah selesai, branch lama dibersihkan otomatis (menyisakan 2 terbaru)

### Aturan penamaan branch

```
main              → branch utama
release/v1.6.x    → release branch untuk minor v1.6
release/v2.0.x    → release branch untuk major v2.0
```

> Branch selain `main` dan `release/vX.X.x` tidak dikenali oleh workflow.

---

## Versioning Rules

### Format tag

```
vMAJOR.MINOR.PATCH           → official release   contoh: v1.6.0
vMAJOR.MINOR.PATCH-preview   → preview dari main  contoh: v1.6.3-preview
```

### Aturan bumping

#### Dari `main` (preview)
- `bump_type` dipilih manual: `patch` / `minor` / `major`
- Tag selalu mendapat suffix `-preview`
- Preview tags **tidak mengganggu** versi official
- Kalkulasi berdasarkan tag terbaru **termasuk** preview sebelumnya

```
v1.6.7-preview + patch → v1.6.8-preview
v1.6.7-preview + minor → v1.7.0-preview
```

#### Dari `release/vX.X.x` (official)
- `bump_type` **diabaikan**, versi auto dari nama branch
- Pertama kali: `release/v1.6.x` → `v1.6.0`
- Berikutnya: patch otomatis → `v1.6.1`, `v1.6.2`, dst

#### Hotfix
- `bump_type` **diabaikan**, selalu patch dari nama branch
- Sama dengan official, tapi jalur deploy berbeda

### Menentukan nama release branch

| Latest tag di main | Hasil branch |
|---|---|
| `v1.6.7-preview` | `release/v1.6.x` ← minor diambil dari preview, tidak di-bump |
| `v1.4.3` (real) + minor | `release/v1.5.x` |
| `v1.4.3` (real) + major | `release/v2.0.x` |

> Jika latest tag adalah preview, nama branch menggunakan minor yang **sama** (tidak di-bump).
> Jika latest tag adalah real (non-preview), nama branch di-bump sesuai `bump_type`.

---

## Workflow Overview

### 1. `create-release-candidate.yml` — Buat Release Branch

**Kapan dipakai:** Saat fitur di main sudah cukup matang dan siap masuk ke proses release ke production.

**Inputs:**
| Input | Wajib | Default | Keterangan |
|---|---|---|---|
| `base_branch` | Tidak | `main` | Branch asal |
| `bump_type` | Ya | `minor` | Hanya berlaku jika latest tag adalah real (non-preview) |

**Yang dilakukan:**
1. Ambil latest tag (termasuk preview)
2. Hitung nama branch (`release/vX.X.x`)
3. Validasi branch belum ada
4. Buat dan push branch baru

---

### 2. `create-release.yml` — Buat Tag & Trigger Deploy

**Kapan dipakai:** Setiap kali ingin deploy — baik dari main (preview), release branch (official), maupun hotfix.

**Inputs:**
| Input | Wajib | Default | Keterangan |
|---|---|---|---|
| `release_type` | Tidak | `regular` | `regular` atau `hotfix` |
| `release_branch` | Tidak | `main` | Kosongkan untuk main. Hotfix: kosongkan untuk auto-detect branch terbaru |
| `bump_type` | Ya | — | Hanya berlaku untuk main. Release branch & hotfix auto dari branch name |
| `notes` | Tidak | — | Catatan bebas yang muncul di release notes |

**Yang dilakukan:**
1. Resolve branch (auto-detect untuk hotfix)
2. Hitung versi berikutnya
3. Validasi tag belum ada
4. Generate changelog otomatis
5. Auto-detect cherry-pick source (via `git cherry-pick -x`)
6. Buat dan push tag
7. Buat GitHub Release
8. Trigger `release.yml` untuk deploy

---

### 3. `release.yml` — Deploy ke Environment

**Kapan dipakai:** Dipanggil otomatis oleh `create-release.yml`. Bisa juga dijalankan manual untuk re-deploy.

**Deploy matrix:**

| Source branch | Dev | Staging | Production |
|---|---|---|---|
| `main` + regular | ✅ Auto | ✅ Perlu approval | ❌ Tidak pernah |
| `release/vX.X.x` + regular | ✅ Auto | ✅ Perlu approval | ✅ Perlu approval |
| Hotfix | ❌ Skip | ✅ Perlu approval | ✅ Perlu approval |

---

### 4. `deploy.yml` — Reusable Deploy

Workflow internal yang dipanggil oleh `release.yml`. Tidak dijalankan langsung.

---

### 5. `cleanup.yml` — Auto Cleanup

Berjalan otomatis saat ada tag baru. Menyisakan **2 release branch terbaru**, sisanya dihapus.

---

## Alur Lengkap

### Skenario A — Preview cycle di main

```
Developer merge PR ke main
  ↓
Jalankan: Create Release
  release_type  : regular
  release_branch: (kosong / main)
  bump_type     : patch / minor / major
  notes         : (opsional)
  ↓
Tag dibuat: v1.6.8-preview
GitHub Release: pre-release ✓
  ↓
Deploy otomatis: Dev → Staging (perlu approval)
Production: SKIP
```

### Skenario B — Official release ke production

```
Step 1 — Buat release branch (sekali saja per minor)
  Jalankan: Create Release Candidate
    base_branch: main
    bump_type  : minor  ← diabaikan jika latest adalah preview
  ↓
  Branch dibuat: release/v1.6.x

Step 2 — Deploy official
  Jalankan: Create Release
    release_type  : regular
    release_branch: release/v1.6.x
    bump_type     : (diabaikan, auto)
    notes         : (opsional)
  ↓
  Tag: v1.6.0 (tanpa -preview)
  GitHub Release: full release ✓
  Deploy: Dev → Staging (approval) → Production (approval)
```

### Skenario C — Hotfix

```
Bug kritis ditemukan di production

Step 1 — Fix di main terlebih dahulu, lalu cherry-pick ke release branch
  git checkout release/v1.6.x
  git cherry-pick -x <sha-commit-fix>    ← wajib pakai -x
  git push origin release/v1.6.x

Step 2 — Deploy hotfix
  Jalankan: Create Release
    release_type  : hotfix
    release_branch: (kosong → auto-detect branch terbaru)
    bump_type     : (diabaikan, auto patch)
    notes         : "Critical fix: deskripsi bug"
  ↓
  Tag: v1.6.1
  GitHub Release: Cherry-picked from: `v1.6.8-preview` (auto-detect)
  Deploy: Staging (approval) → Production (approval)
```

---

## Approval Gates

Approval dikonfigurasi di **GitHub Settings → Environments → Required reviewers**.

| Environment | Approver | Keterangan |
|---|---|---|
| `development` | — | Auto, tidak perlu approval |
| `staging` | Any engineer | Perlu approval sebelum deploy |
| `production` | Tech lead / CTO | Perlu approval sebelum deploy |

> Optional: tambahkan **Wait timer** (misal 5 menit) di environment `production` sebagai buffer.

---

## Hotfix Rules

1. **Fix harus ada di `main` terlebih dahulu** sebelum cherry-pick ke release branch
2. **Wajib pakai `git cherry-pick -x`** agar source commit bisa di-trace otomatis
3. Hotfix **tidak deploy ke dev** — langsung staging (approval) → production (approval)
4. Hotfix branch: kosongkan `release_branch` untuk auto-detect branch terbaru, atau isi manual `release/vX.X.x`
5. Tag hotfix **tidak** mendapat suffix `-preview`

---

## Cherry-pick Convention

Workflow dapat **auto-detect** sumber cherry-pick jika menggunakan flag `-x`:

```bash
# ✅ Benar — workflow bisa trace source
git cherry-pick -x <sha>

# ❌ Salah — workflow tidak bisa trace
git cherry-pick <sha>
```

Dengan `-x`, commit message otomatis menjadi:
```
fix: resolve payment timeout

(cherry picked from commit a3f9b2c)
```

Workflow akan scan pola tersebut, map SHA ke tag, lalu tampilkan di release notes:
```
Cherry-picked from: `v1.6.5-preview, v1.6.7-preview`
```

Jika tidak ditemukan, baris tersebut tidak muncul di release notes.

---

## GitHub Secrets

| Secret | Digunakan di | Keterangan |
|---|---|---|
| `PAT_TOKEN` | `create-release.yml`, `create-release-candidate.yml` | Personal Access Token dengan scope `repo`. Dibutuhkan untuk push tag, push branch, dan trigger workflow lain. `GITHUB_TOKEN` default tidak bisa trigger cross-workflow. |
| `GITHUB_TOKEN` | `release.yml` (create GitHub Release) | Token default GitHub Actions, cukup untuk buat release. |

### Cara buat PAT_TOKEN
1. GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens
2. Permissions: **Contents** (read & write), **Actions** (read & write)
3. Simpan ke repo: Settings → Secrets and variables → Actions → `PAT_TOKEN`
