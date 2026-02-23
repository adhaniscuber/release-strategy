# Release Strategy

GitHub Flow + Release Branch dengan SemVer per change.

## Struktur Workflow

```
.github/workflows/
├── deploy.yml          # Reusable — dipanggil workflow lain
├── continuous.yml      # Push ke main → dev + staging otomatis
├── release.yml         # Push tag → dev → staging → prod
├── create-release.yml  # Manual — buat tag dari release branch
└── cleanup.yml         # Auto — hapus release branch lama
```

## Setup GitHub Environments

Buat 3 environments di **Settings → Environments**:

| Environment | Required Reviewers | Notes |
|---|---|---|
| `development` | — | Tidak perlu approval |
| `staging` | Tim engineer (semua) | Sadar kapan deploy staging |
| `production` | Tech lead / CTO saja | Wait timer 5 menit (opsional) |

## Flow Normal (Sprint)

```
1. Developer merge PR ke main
   → continuous.yml otomatis deploy ke dev + staging

2. Code freeze → buat release branch
   git checkout main
   git checkout -b release/v1.5.x
   git push origin release/v1.5.x

3. QA final di release branch

4. Siap rilis → Actions → Create Release
   release_branch : release/v1.5.x
   bump_type      : minor
   → otomatis buat tag v1.5.0
   → otomatis trigger release.yml
   → dev → staging (any engineer approve) → prod (tech lead approve)
```

## Flow Hotfix

```
1. Fix di main dulu (PR seperti biasa)

2. Cherry-pick ke release branch aktif
   git checkout release/v1.5.x
   git cherry-pick <commit-hash>
   git push origin release/v1.5.x

3. Actions → Create Release
   release_branch : release/v1.5.x
   bump_type      : patch
   → buat tag v1.5.3
   → trigger release.yml (staging → prod, skip dev bisa via workflow_dispatch)
```

## SemVer

| Bump | Kapan | Contoh |
|---|---|---|
| `patch` | Bug fix / hotfix | v1.5.2 → v1.5.3 |
| `minor` | Fitur baru | v1.5.x → v1.6.0 |
| `major` | Breaking change | v1.x.x → v2.0.0 |

Tag hanya dibuat dari **release branch**, bukan dari main langsung.
Setelah sprint release, main lanjut ke minor berikutnya (v1.5.x → v1.6.x) agar v1.5.x reserved untuk hotfix di release branch.

## Branch Lifecycle

- Simpan **2 release branch terakhir** (untuk rollback & hotfix)
- `cleanup.yml` otomatis hapus yang lama setiap kali tag baru di-push
