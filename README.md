# Mac configuration

## Install xcode

```bash
xcode-select --install
```

## Remove default Mac apps from the dock

```bash
python3 -c "
import subprocess, plistlib, tempfile, os

result = subprocess.run(['defaults', 'export', 'com.apple.dock', '-'], capture_output=True)
apps = plistlib.loads(result.stdout)['persistent-apps']

keep = {'Apps', 'Mail', 'Calendar'}
filtered = [a for a in apps if a.get('tile-data', {}).get('file-label', '') in keep]

full_plist = plistlib.loads(result.stdout)
full_plist['persistent-apps'] = filtered

with tempfile.NamedTemporaryFile(suffix='.plist', delete=False) as f:
    plistlib.dump(full_plist, f)
    tmp = f.name

subprocess.run(['defaults', 'import', 'com.apple.dock', tmp])
os.unlink(tmp)
" && killall Dock
```

See the current list of apps in the dock

```bash
defaults read com.apple.dock persistent-apps | grep "file-label" | sed 's/.*= "\(.*\)";/\1/'
```

## Setup keys for github access

```bash
# GitHub identity and dedicated key location.
GITHUB_EMAIL="you@example.com"
GITHUB_KEY="$HOME/.ssh/id_ed25519_github"

# Ensure SSH has a private place to store keys.
mkdir -p "$HOME/.ssh"
chmod 700 "$HOME/.ssh"

# Create a GitHub-only SSH key.
ssh-keygen -t ed25519 -C "$GITHUB_EMAIL" -f "$GITHUB_KEY"

# Make the key available to your Mac login session.
ssh-add --apple-use-keychain "$GITHUB_KEY" 2>/dev/null || ssh-add "$GITHUB_KEY"

# Copy and print the public key so it can be added to GitHub.
pbcopy < "$GITHUB_KEY.pub"
cat "$GITHUB_KEY.pub"
```

### Configure individual repo to use github identity

```bash
github_identity() {
  # Identity to use for GitHub commits from this repo.
  local name="Your Name"
  local email="you@example.com"

  # Dedicated SSH key reserved for GitHub access.
  local key="$HOME/.ssh/id_ed25519_github"

  # Repo to configure; defaults to the current directory.
  local repo="${1:-.}"

  # Stop early if the GitHub SSH key has not been created yet.
  if [[ ! -f "$key" ]]; then
    echo "Missing key: $key"
    echo "Create it first with:"
    echo "ssh-keygen -t ed25519 -C \"$email\" -f \"$key\""
    return 1
  fi

  # Make sure this is being applied only to a real Git repository.
  git -C "$repo" rev-parse --is-inside-work-tree >/dev/null 2>&1 || {
    echo "Not a git repo: $repo"
    return 1
  }

  # Make the key available for GitHub SSH authentication.
  ssh-add --apple-use-keychain "$key" 2>/dev/null || ssh-add "$key"

  # Scope the GitHub commit identity to this repo only.
  git -C "$repo" config user.name "$name"
  git -C "$repo" config user.email "$email"

  # Force this repo to use the dedicated GitHub SSH key.
  git -C "$repo" config core.sshCommand "ssh -i $key -o IdentitiesOnly=yes"

  # Confirm what was configured.
  echo "Configured GitHub identity for:"
  git -C "$repo" rev-parse --show-toplevel
  echo "$name <$email>"
  echo "SSH key: $key"
}
```
