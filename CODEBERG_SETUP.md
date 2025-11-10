# Setting up Codeberg Mirror for Olocus

## Prerequisites

1. Create a Codeberg account at https://codeberg.org/user/sign_up (if you haven't already)
2. Create the organization or repository on Codeberg

## Option A: Create Repository on Codeberg First

1. Go to https://codeberg.org
2. Create a new repository:
   - Click "+" → "New Repository"
   - Name: `olocus-mvp` (or your preferred name)
   - Description: "Privacy-preserving location tracking and attestation protocol"
   - Visibility: Public
   - Initialize: DO NOT initialize with README (we'll push existing code)

## Option B: Mirror from GitHub to Codeberg

### Step 1: Add Codeberg Remote

```bash
# Remove the incorrect remote if it exists
git remote remove codeberg

# Add Codeberg remote with your username
# Replace YOUR_USERNAME with your Codeberg username
git remote add codeberg https://codeberg.org/YOUR_USERNAME/olocus-mvp.git

# Or if using an organization called 'olocus':
git remote add codeberg https://codeberg.org/olocus/olocus-mvp.git
```

### Step 2: Set up Authentication

#### Using Personal Access Token (Recommended)

1. Go to https://codeberg.org/user/settings/applications
2. Generate a new access token with `write:repository` scope
3. Save the token securely

Then push using:
```bash
# You'll be prompted for username and password
# Username: your-codeberg-username
# Password: your-access-token (not your account password)
git push codeberg main
```

#### Using SSH (Alternative)

1. Add your SSH key to Codeberg: https://codeberg.org/user/settings/keys
2. Change remote to SSH:
```bash
git remote set-url codeberg git@codeberg.org:olocus/olocus-mvp.git
```
3. Push:
```bash
git push codeberg main
```

### Step 3: Push All Branches and Tags

```bash
# Push main branch
git push codeberg main

# Push all tags (if any)
git push codeberg --tags

# To push all branches (if you have more than main):
git push codeberg --all
```

## Setting up Automatic Mirroring

### Option 1: GitHub Actions (GitHub → Codeberg)

Create `.github/workflows/mirror.yml`:

```yaml
name: Mirror to Codeberg

on:
  push:
    branches: [ main ]
  workflow_dispatch:

jobs:
  mirror:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
      with:
        fetch-depth: 0
    
    - name: Mirror to Codeberg
      env:
        CODEBERG_TOKEN: ${{ secrets.CODEBERG_TOKEN }}
      run: |
        git remote add codeberg https://$CODEBERG_TOKEN@codeberg.org/olocus/olocus-mvp.git
        git push codeberg main --force
```

Then add your Codeberg token as a GitHub secret:
1. Go to GitHub repo → Settings → Secrets → Actions
2. Add new secret: `CODEBERG_TOKEN` with your Codeberg access token

### Option 2: Git Hooks (Local)

Create `.git/hooks/post-push`:

```bash
#!/bin/bash
# Automatically push to Codeberg after pushing to GitHub
git push codeberg main
```

Make it executable:
```bash
chmod +x .git/hooks/post-push
```

## Updating README for Codeberg

Once set up, update your README.md to indicate both locations:

```markdown
## Repository Locations

- **GitHub**: https://github.com/svailsa/olocus-mvp (Development)
- **Codeberg**: https://codeberg.org/olocus/olocus-mvp (Primary/Mirror)

Development happens on GitHub, with automatic mirroring to Codeberg.
```

## Switching Primary to Codeberg (Future)

When ready to make Codeberg primary:

```bash
# Change origin to Codeberg
git remote set-url origin https://codeberg.org/olocus/olocus-mvp.git

# Keep GitHub as mirror
git remote set-url github https://github.com/svailsa/olocus-mvp.git

# Update push default
git config --global push.default current
```

## Troubleshooting

### Authentication Failed
- Ensure you're using an access token, not your password
- Check token has `write:repository` permissions
- Try SSH instead of HTTPS

### Repository Not Found
- Ensure the repository exists on Codeberg first
- Check the organization/username is correct
- Verify repository visibility settings

### Push Rejected
- Pull any changes from Codeberg first: `git pull codeberg main`
- Or force push (careful!): `git push codeberg main --force`

## Next Steps

1. Create the repository on Codeberg
2. Set up authentication (token or SSH)
3. Push the code
4. Set up automatic mirroring (optional)
5. Update documentation to reference both locations