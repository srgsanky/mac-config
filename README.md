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
