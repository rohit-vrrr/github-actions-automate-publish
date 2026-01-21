# Introduction

**GitHub Actions** = CI/CD pipelines that run inside GitHub when something happens

`Pipeline → Build → create .npmrc → publish to Azure Artifacts`

## What we are going to automate (your manual steps)

Current manual process:
1. Build Angular library
2. Go to `dist/<project-name>`
3. Add `.npmrc`
4. Authenticate to Azure Artifacts
5. Increment version
6. Run `npm publish`

After automation (GitHub Actions):

🚀 All of the above runs automatically when you push to `master`

## Prerequisites (one-time setup)

**🔐 A. Create Azure DevOps PAT (Personal Access Token)**

Azure DevOps → User settings → Personal access tokens

Scopes:
✅ Packaging → Read & write

Copy the token

**🔐 B. Store PAT in GitHub Secrets**

GitHub repo → Settings → Secrets and variables → Actions

Add New repository secret

| Name               | Value        |
| ------------------ | ------------ |
| `AZURE_DEVOPS_PAT` | `<your PAT>` |

This is **mandatory** - never hardcode tokens.

## How authentication will work (important)

Instead of `vsts-npm-auth` (which is aging), we’ll use a token-based `.npmrc`.

GitHub Actions will dynamically create:

```ini
registry=https://pkgs.dev.azure.com/time-payroll-kmddk/Delta/_packaging/hcm-angular/npm/registry/
always-auth=true
//pkgs.dev.azure.com/time-payroll-kmddk/Delta/_packaging/hcm-angular/npm/registry/:_authToken=${AZURE_DEVOPS_PAT}
```

✔ Works perfectly from GitHub<br/>
✔ No interactive auth<br/>
✔ Recommended by Microsoft

## GitHub Actions workflow (copy–paste ready)

Create this file in your repo:

📄 `.github/workflows/publish.yml`

```yaml
name: Build and Publish to Azure Artifacts

on:
  push:
    branches:
      - master

jobs:
  publish:
    runs-on: ubuntu-latest

    steps:
      # 1️⃣ Checkout code
      - name: Checkout repository
        uses: actions/checkout@v4

      # 2️⃣ Setup Node.js
      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: 18

      # 3️⃣ Install dependencies
      - name: Install dependencies
        run: npm ci

      # 4️⃣ Build Angular library (storybook lib)
      - name: Build library
        run: npm run build

      # 5️⃣ Create .npmrc inside dist folder
      - name: Configure npm for Azure Artifacts
        run: |
          cat <<EOF > dist/<project-name>/.npmrc
          registry=https://pkgs.dev.azure.com/time-payroll-kmddk/Delta/_packaging/hcm-angular/npm/registry/
          always-auth=true
          //pkgs.dev.azure.com/time-payroll-kmddk/Delta/_packaging/hcm-angular/npm/registry/:_authToken=${AZURE_DEVOPS_PAT}
          EOF
        env:
          AZURE_DEVOPS_PAT: ${{ secrets.AZURE_DEVOPS_PAT }}

      # 6️⃣ Bump version (patch)
      - name: Bump version
        working-directory: dist/<project-name>
        run: npm version patch --no-git-tag-version

      # 7️⃣ Publish to Azure Artifacts
      - name: Publish package
        working-directory: dist/<project-name>
        run: npm publish
```

🔁 Replace:
`<project-name>`

## Where will the package appear?

**Azure DevOps → Artifacts → hcm-angular feed**

Exactly the same as before.

## How versioning works here (important)

This line:
`npm version patch --no-git-tag-version`

Automatically:
- `1.2.3` → `1.2.4`
- Updates `package.json`
- Does NOT commit back to GitHub

Optional improvements later:
- Use `minor` or `major`
- Use commit messages (semantic versioning)
- Push version bump back to repo
- Publish only on GitHub Releases

## Safety checks (recommended)

You may want to prevent:<br/>
❌ duplicate publishes<br/>
❌ publishing on failed builds

Add this before publish:
```yaml
      - name: Check if version already exists
        working-directory: dist/<project-name>
        run: npm view <package-name> version || echo "New version"
```

## Summary
```vbnet
GitHub repo (code)
        |
        | push to master
        v
GitHub Actions
        |
        | build Angular lib
        | create .npmrc
        | authenticate to Azure
        | npm publish
        v
Azure Artifacts (hcm-angular feed)
```
