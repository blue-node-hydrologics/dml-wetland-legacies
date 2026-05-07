# GitHub Repository Setup & Upload Guide

*For: dml-wetland-legacies — blue-node-hydrologics organization*

---

## Part 1: Create the GitHub Repository

### Prerequisites

1. **Install Git** (if not already installed):
   ```bash
   # macOS — Git ships with Xcode Command Line Tools
   xcode-select --install
   # Verify
   git --version
   ```

2. **Install GitHub CLI** (makes everything easier):
   ```bash
   brew install gh
   gh auth login   # follow the prompts; choose HTTPS, then paste a browser token
   ```
   If you prefer not to use the CLI, the web steps are noted in italics below each CLI command.

---

### Step 1 — Create the repo on GitHub

```bash
gh repo create blue-node-hydrologics/dml-wetland-legacies \
  --public \
  --description "Analysis code for 'Drained Prairie Pothole Wetlands Provide Blueprints for Restoration in the US Corn Belt'" \
  --homepage "https://github.com/blue-node-hydrologics/dml-wetland-legacies" \
  --disable-wiki
```

*Web alternative: go to github.com → New repository → Owner = blue-node-hydrologics, Name = dml-wetland-legacies, Public, no README (you already have one), MIT license (already have one). Create repository.*

---

### Step 2 — Initialize the local folder as a Git repository

Open Terminal, then:

```bash
# Navigate to your project folder
cd "/Users/kimberlyvanmeter/Library/CloudStorage/GoogleDrive-vanmeterlab@gmail.com/My Drive/Research/Projects/NASA - UMRB Legacy Wetlands/DML_2025/notebooks/blue-node-hydrologics-paper"

# Initialize git
git init -b main

# Point to the GitHub remote
git remote add origin https://github.com/blue-node-hydrologics/dml-wetland-legacies.git
```

---

### Step 3 — First commit and push

```bash
# Stage everything (your .gitignore will automatically exclude data files, TIFFs, etc.)
git add .

# Review what will be committed — should be only notebooks, docs, environment files
git status

# Commit
git commit -m "Initial commit: analysis pipeline for DML legacy wetlands paper"

# Push
git push -u origin main
```

That's it — your repository is live.

---

## Part 2: Automate Future Uploads

Once the repo exists, keeping it updated is just three commands. The script below wraps them for convenience.

### Option A — One-liner you can run any time

```bash
cd "/Users/kimberlyvanmeter/Library/CloudStorage/GoogleDrive-vanmeterlab@gmail.com/My Drive/Research/Projects/NASA - UMRB Legacy Wetlands/DML_2025/notebooks/blue-node-hydrologics-paper" \
  && git add . \
  && git commit -m "Update: $(date '+%Y-%m-%d')" \
  && git push
```

### Option B — Save as a shell script

Save the file below as `push_to_github.sh` in the project folder, then run it with `bash push_to_github.sh` whenever you want to sync.

```bash
#!/usr/bin/env bash
# push_to_github.sh
# Run from any directory — always operates on the dml-wetland-legacies repo.

set -euo pipefail

REPO_DIR="/Users/kimberlyvanmeter/Library/CloudStorage/GoogleDrive-vanmeterlab@gmail.com/My Drive/Research/Projects/NASA - UMRB Legacy Wetlands/DML_2025/notebooks/blue-node-hydrologics-paper"

echo "→ Changing to repo directory..."
cd "$REPO_DIR"

echo "→ Staging all changes..."
git add .

# Only commit if there is actually something staged
if git diff --cached --quiet; then
  echo "  Nothing new to commit — already up to date."
else
  MSG="${1:-Update: $(date '+%Y-%m-%d %H:%M')}"
  echo "→ Committing: $MSG"
  git commit -m "$MSG"
  echo "→ Pushing to origin/main..."
  git push
  echo "✓ Done."
fi
```

**Usage:**

```bash
# Simple sync (auto-generates a timestamp message)
bash push_to_github.sh

# Custom commit message
bash push_to_github.sh "Add figure notebooks for UAI and lollipop plots"
```

---

## Part 3: Best Practices for Research Code on GitHub

### What your .gitignore already handles correctly ✓
- All data files (`.tif`, `.shp`, `.gpkg`, `.csv`, `.parquet`) — correct; data stays local
- OS junk (`.DS_Store`, `Thumbs.db`)
- Python caches (`__pycache__`, `*.pyc`)
- Secrets (`.env`, `*.key`)
- Jupyter checkpoints (`.ipynb_checkpoints/`)

### Before making the repo public, check:

1. **Fill in author affiliations** in `README.md` (lines 149–152 have placeholders like `[Affiliation 1 — fill in]`).
2. **Strip notebook outputs** that contain absolute paths from your development machine — these can leak machine-specific paths and look messy. Run:
   ```bash
   pip install nbstripout
   nbstripout --install   # installs a git filter that auto-strips outputs on commit
   ```
   Or strip manually before the first push:
   ```bash
   jupyter nbconvert --clear-output --inplace notebooks/**/*.ipynb
   ```
3. **Add a GitHub Release** once the paper is accepted, tagging the exact version of the code used for the published results:
   ```bash
   gh release create v1.0.0 --title "Published version (Nature Sustainability)" --notes "Code as used in the accepted manuscript."
   ```

### Connecting to Zenodo for a citable DOI

Once your repo is public:
1. Go to [zenodo.org](https://zenodo.org) → Log in with GitHub → "GitHub" tab
2. Flip the toggle next to `dml-wetland-legacies`
3. Create a GitHub Release (step above) — Zenodo will automatically archive it and mint a DOI
4. Add the DOI badge to your README

This gives reviewers and readers a permanently citable snapshot of the code.
