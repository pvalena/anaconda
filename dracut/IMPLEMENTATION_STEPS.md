# Implementation Steps: Auto-detect SSL Certificates for Python Dependencies

## Overview
Replace the hardcoded certificate inclusion with dynamic detection similar to dracut's `url-lib` module, but adapted for Python files instead of shared libraries.

## Current State (module-setup.sh:95-128)
- TODO placeholder with reference implementation from `url-lib`
- Reference scans `libcurl.so.*` and `libcrypto.so.*` for embedded cert paths
- Need to adapt for Python `.py` files installed by the `python-deps` loop

## Known Facts
- **Target file**: `/etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem`
  - Found in `/usr/lib/python3.14/site-packages/requests/certs.py:19`
  - Returned by `requests.certs.where()` function
- **Scope**: Only scan `.py` files from the python-deps loop (lines 87-93)
- **Pattern**: Look for `\.pem` or `\.crt` file paths in Python source

## Implementation Steps

### 1. Collect Python files during installation
**Location**: After the python-deps loop (after line 93)

```bash
# Track Python files as they're installed
_pydeps=()
PYTHONHASHSEED=42 "$moddir/python-deps" "$moddir/parse-kickstart" "$moddir/driver_updates.py" | while read -r dep; do
    case "$dep" in
        *.so) inst "$dep" ;;
        *.py)
            inst_simple "$dep"
            _pydeps+=("$dep")  # Track installed .py files
            ;;
        *) inst "$dep" ;;
    esac
done
```

**Alternative approach** (simpler): Scan files after the loop completes by re-running grep on the same file list.

### 2. Scan Python files for certificate paths
**Location**: Right after line 93, before the existing TODO comment

**Approach A** (following url-lib pattern closely):
```bash
# Scan installed Python files for SSL certificate bundles
_crt=""
_crts=""
_found=""

for _pyfile in *.py files from python-deps*; do
    [[ -e $_pyfile ]] || continue
    # Look for .pem or .crt paths in Python source code
    read -r -d '' _crt < <(grep -E --binary-files=text -z -o "(['\"])/etc/[^'\"]*\.(pem|crt)(['\"])" "$_pyfile" | sed "s/['\"]//g; s/\x0//g")
    [[ $_crt ]] || continue
    [[ $_crt == /*/* ]] || continue  # Must be absolute path with directory
    if [[ -e $_crt ]]; then
        _crts="$_crts $_crt"
        _found=1
    fi
done

if [[ $_found ]] && [[ -n $_crts ]]; then
    for _crt in $_crts; do
        if ! inst "${_crt#"${dracutsysrootdir-}"}"; then
            dwarn "Couldn't install '$_crt' SSL CA cert bundle; HTTPS might not work."
            continue
        fi
    done
fi
```

**Approach B** (simpler, more targeted):
```bash
# Scan Python files for certificate references
_crts=""
_found=""

# Scan all .py files that were installed via python-deps
PYTHONHASHSEED=42 "$moddir/python-deps" "$moddir/parse-kickstart" "$moddir/driver_updates.py" | while read -r dep; do
    [[ $dep == *.py ]] || continue
    [[ -f $dep ]] || continue

    # Extract certificate paths from Python source
    while IFS= read -r _crt; do
        # Validate it's an absolute path and file exists
        [[ $_crt == /* ]] || continue
        [[ -e $_crt ]] || continue
        _crts="$_crts $_crt"
        _found=1
    done < <(grep -oE "([\'\"])/etc/[^'\"]*\.(pem|crt)([\'\"])" "$dep" | sed "s/['\"]//g" | sort -u)
done

# Install discovered certificate bundles
if [[ $_found ]] && [[ -n $_crts ]]; then
    for _crt in $_crts; do
        if ! inst "$_crt"; then
            dwarn "Couldn't install '$_crt' SSL CA cert bundle; HTTPS might not work."
        fi
    done
fi
```

### 3. Test the implementation
**Local validation steps** (no full test runs):

```bash
# 1. Verify grep pattern finds the cert path
grep -oE "([\'\"])/etc/[^'\"]*\.(pem|crt)([\'\"])" \
    /usr/lib/python3.14/site-packages/requests/certs.py | sed "s/['\"]//g"
# Expected: /etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem

# 2. Check that cert file exists
ls -lh /etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem

# 3. Verify python-deps output contains requests module
cd /home/pvalena/rhel/packages/dracut/anaconda/dracut
PYTHONHASHSEED=42 ./python-deps parse-kickstart driver_updates.py | grep -i requests

# 4. Syntax check modified module-setup.sh
bash -n module-setup.sh

# 5. Shellcheck validation
shellcheck module-setup.sh
```

### 4. Edge cases to consider
- Multiple certificate paths in different files → deduplicate
- Certificate file doesn't exist → warn but don't fail
- No certificate paths found → optional warning (like url-lib does)
- Binary .pyc files → skip (only scan .py source)
- Relative paths → reject (only absolute paths)

### 5. Final cleanup
- Remove the TODO comment block (lines 95-128)
- Keep implementation concise (10-20 lines max)
- Add brief comment explaining what it does
- Follow the url-lib pattern for consistency

## Recommended Approach
**Approach B (simpler)** is recommended because:
- Re-uses the python-deps command output
- More direct and readable
- Easier to debug
- Less risk of scope issues with variables in while loops
- Naturally deduplicates with `sort -u`

## Success Criteria
✓ Certificate bundle auto-detected from Python dependencies
✓ `/etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem` installed in initramfs
✓ Code passes shellcheck without new warnings
✓ Pattern matches url-lib style for consistency
✓ No hardcoded certificate paths
