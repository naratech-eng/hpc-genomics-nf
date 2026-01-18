# GitBook Publishing Instructions

This document provides step-by-step instructions for publishing the HPC Genomics documentation to GitBook.

## Option 1: GitBook.com (Recommended - Cloud Hosting)

GitBook.com provides free hosting for open-source projects with GitHub integration.

### Step 1: Create a GitBook Account

1. Go to [https://www.gitbook.com](https://www.gitbook.com)
2. Click **"Sign up"**
3. Choose **"Sign up with GitHub"** for easy integration
4. Authorize GitBook to access your GitHub account

### Step 2: Create a New Space

1. In the GitBook dashboard, click **"New Space"**
2. Choose **"Import from GitHub"**
3. Select the repository: `naratech-eng/hpc-genomics-nf`
4. Configure the import:
   - **Branch:** `naratech` (or your default branch)
  - **Root directory:** Repository root
  - **Main file:** `README.md`

### Step 3: Configure GitHub Sync

1. In your GitBook space, go to **Settings → Integrations**
2. Enable **GitHub Sync**
3. Configure:
   - **Repository:** `naratech-eng/hpc-genomics-nf`
   - **Branch:** `naratech`
  - **Directory:** Repository root
   - **Sync mode:** Two-way (recommended)

### Step 4: Customize Your Space

1. Go to **Customize** in your space settings
2. Update:
   - **Name:** Genomic Nextflow HPC Design Guide
   - **Logo:** Upload a project logo (optional)
   - **Theme:** Choose your preferred theme
   - **Domain:** Configure custom domain (optional)

### Step 5: Publish

1. Click **"Publish"** in the top-right corner
2. Your documentation is now live!
3. Share the URL: `https://your-space.gitbook.io`

---

## Option 2: GitBook CLI (Self-Hosted/Static)

For self-hosted deployments or generating static HTML files.

### Prerequisites

```bash
# Install Node.js (if not installed)
brew install node  # macOS
# or
sudo apt install nodejs npm  # Ubuntu/Debian

# Install GitBook CLI globally
npm install -g gitbook-cli

# Verify installation
gitbook --version
```

### Build the Documentation

```bash
# Navigate to the repository root
cd /Users/nara/Documents/Development/HPC/hpc-genomics-nf

# Install GitBook plugins
gitbook install

# Build static HTML
gitbook build

# Output will be in _book/ directory
```

### Preview Locally

```bash
# Start local server
gitbook serve

# Open in browser: http://localhost:4000
```

### Deploy to GitHub Pages

```bash
# Build the book
gitbook build

# Create gh-pages branch (first time only)
git checkout --orphan gh-pages

# Remove all files except _book
git rm -rf .

# Copy built files
cp -r _book/* .
rm -rf _book

# Commit and push
git add .
git commit -m "Deploy GitBook documentation"
git push origin gh-pages

# Switch back to main branch
git checkout naratech
```

Then enable GitHub Pages in repository settings:
1. Go to **Settings → Pages**
2. Source: **Deploy from branch**
3. Branch: **gh-pages** / **(root)**
4. Save

Your documentation will be available at:
`https://naratech-eng.github.io/hpc-genomics-nf/`

---

## Option 3: GitBook with GitHub Actions (Automated)

Create automated deployments with GitHub Actions.

### Create Workflow File

Create `.github/workflows/gitbook.yml`:

```yaml
name: Build and Deploy GitBook

on:
  push:
    branches:
      - naratech
    paths:
      - 'README.md'
      - 'docs/**'
      - 'book.json'

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
      
      - name: Install GitBook CLI
        run: |
          npm install -g gitbook-cli
          gitbook fetch 3.2.3
      
      - name: Install plugins
        run: gitbook install
      
      - name: Build GitBook
        run: gitbook build
      
      - name: Deploy to GitHub Pages
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./_book
          publish_branch: gh-pages
```

This workflow will:
1. Trigger on pushes to `docs/` directory
2. Build the GitBook documentation
3. Deploy to GitHub Pages automatically

---

## Directory Structure Reference

```
hpc-genomics-nf/
├── README.md                   # Introduction (required)
├── book.json                    # GitBook configuration
├── docs/
│   ├── SUMMARY.md               # Table of contents (required)
│   ├── 01-project-overview.md
│   ├── 02-architecture.md
│   ├── 03-tech-stack.md
│   ├── 04-terraform.md
│   ├── 05-workflow.md
│   ├── 06-security-observability.md
│   ├── 07-cost-optimization.md
│   ├── 08-conclusion.md
│   ├── 09-references.md
│   ├── 10-developer-guidance.md
│   ├── 11-troubleshooting-gpu.md
│   ├── 12-validation-checklist.md
│   └── styles/
│       └── website.css          # Custom styles
└── GITBOOK_PUBLISHING.md        # This file
```

---

## Troubleshooting

### Common Issues

| Issue | Solution |
|-------|----------|
| `gitbook install` fails | Try `npm cache clean --force` then retry |
| CSS not loading | Ensure `book.json` paths are correct |
| Images not showing | Use relative paths from docs folder |
| TOC not rendering | Check `SUMMARY.md` formatting |

### GitBook.com Sync Issues

1. Check GitHub integration permissions
2. Verify branch and directory settings
3. Check for merge conflicts in synced content
4. Review GitBook activity log for errors

---

## Support

- **GitBook Documentation:** https://docs.gitbook.com
- **GitBook CLI GitHub:** https://github.com/GitbookIO/gitbook-cli
- **Project Issues:** https://github.com/naratech-eng/hpc-genomics-nf/issues
