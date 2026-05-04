---
name: gsas
description: Open GSAS-II project files (.gpx) on this machine and extract refinement values (wR, lattice parameters, phase info, atom params, histogram data). Use whenever the user asks to read, inspect, compare, or batch-extract values from .gpx files.
---

# GSAS-II scripting

## Configuration (per-machine)

Machine-specific paths live in `config.json` next to this `SKILL.md` — **never hardcode `/Users/...` paths in this file**. All snippets below use `<gsas_root>` and `<python>` as placeholders that you substitute at runtime from `config.json`.

Schema (see `config.example.json`):

```json
{
  "gsas_root": "<absolute path to the GSAS-II tree (parent of the inner GSASII/ package)>",
  "python":    "<absolute path to a python with GSAS-II deps installed>"
}
```

### First-time setup (auto-bootstrap)

On first invocation, check whether `config.json` exists alongside this file. If not, run the bootstrap below — it probes common install locations, verifies the import works, and writes `config.json`. Re-run if GSAS-II moves.

```bash
SKILL_DIR="$(cd -- "$(dirname -- "${BASH_SOURCE[0]:-$HOME/.claude/skills/gsas/SKILL.md}")" && pwd)"

# 1. Find GSAS-II root (parent of the inner GSASII/ package)
GSAS_ROOT=""
for cand in "$HOME/g2main/GSAS-II" "$HOME/GSAS-II" "/opt/GSAS-II"; do
  [ -d "$cand/GSASII" ] && GSAS_ROOT="$cand" && break
done
[ -z "$GSAS_ROOT" ] && { echo "GSAS-II tree not found — set gsas_root manually in config.json"; exit 1; }

# 2. Find a python with GSAS-II deps
PY=""
for cand in "$HOME/g2main/bin/python3.13" "$HOME/g2main/bin/python3"; do
  [ -x "$cand" ] && PY="$cand" && break
done
[ -z "$PY" ] && { echo "python3.13 not found — set python manually in config.json"; exit 1; }

# 3. Verify the import actually works
"$PY" -c "import sys; sys.path.insert(0, '$GSAS_ROOT'); from GSASII import GSASIIscriptable" \
  || { echo "GSASIIscriptable import failed under $PY — check installation"; exit 1; }

# 4. Write config
cat > "$SKILL_DIR/config.json" <<EOF
{
  "gsas_root": "$GSAS_ROOT",
  "python":    "$PY"
}
EOF
echo "Wrote $SKILL_DIR/config.json"
```

After bootstrap, read the values once per session before generating any snippet:

```bash
GSAS_ROOT=$(jq -r .gsas_root ~/.claude/skills/gsas/config.json)
PY=$(jq -r .python           ~/.claude/skills/gsas/config.json)
```

Substitute `$GSAS_ROOT` for `<gsas_root>` and `$PY` for `<python>` in the snippets below.

## Importing the scriptable API

`GSASIIscriptable` uses **relative imports**. The old `sys.path.insert(...,'GSASII')` + `import GSASIIscriptable` pattern fails with `ImportError: attempted relative import with no known parent package`.

Correct pattern:

```python
import sys
sys.path.insert(0, '<gsas_root>')   # parent of the GSASII package
from GSASII import GSASIIscriptable as G2sc
```

Run with:
```bash
<python> script.py
```

## Opening a project

```python
gpx = G2sc.G2Project(gpxfile='/path/to/project.gpx')
```

## Common values to extract

```python
# Histograms (powder patterns)
for h in gpx.histograms():
    h.name                  # e.g. "PWDR RuO_v2_80.csv"
    h.get_wR()              # weighted profile R, e.g. 12.638
    # h.getdata('X'/'Yobs'/'Ycalc'/'Background'/'Residual'/'Weight') -> numpy arrays

# Phases
for p in gpx.phases():
    p.name
    p.get_cell()                  # dict: length_a/b/c, angle_alpha/beta/gamma, volume
    p.get_cell_and_esd()          # (cell_dict, esd_dict) — esds keyed the same way
    p.data['General']['SGData']['SpGrp']  # space group string, e.g. "P 4_2/m n m"
    # p.atoms()                   # atoms with .label, .type, .coordinates, .occupancy, .uiso
```

### Rwp, GOF, χ² (project-level refinement stats)

These are NOT on the histogram object — they live in the Covariance block:

```python
rv = gpx.data['Covariance']['data']['Rvals']
rv['Rwp']    # weighted profile R, percent (= histogram.get_wR())
rv['GOF']    # goodness of fit S = sqrt(reduced chi^2)
rv['chisq']  # raw chi^2 sum (NOT the χ² people usually report)
rv['Nobs']   # observations
rv['Nvars']  # refined variables
# Reduced chi-squared (what GSAS-II manual / IUCr call χ²) = GOF**2
```

Confirm with the user whether they want **GOF** or **GOF² (= reduced χ²)** when they say "chi-square" — both are common; default to GOF² (IUCr).

`histogram.residuals` also has `wR`, `R`, `Rb`, per-phase `Rf`, `Rf^2`, `Nref`, `sumInt` — but not GOF/chi².

### Formatting values with esds

GSAS-II ships a helper that formats `value(esd)` like papers do (`5.6226(1)` etc.):

```python
from GSASII import GSASIImath as G2mth
G2mth.ValEsd(cell['length_a'], esd['length_a'])   # -> "4.5296(4)"
```

For batch extraction across many gpx files, glob the folder and call the same methods; results are plain Python floats / numpy scalars.

## Notes

- `get_wR()` returns wR as a percentage-like float (matches GSAS-II GUI display).
- `get_cell()` values come back as `np.float64`; cast with `float(...)` before writing to CSV/JSON if needed.
- `gpx.save()` writes back; only call after explicit refinement steps.
- Don't try to refine inside read-only inspection scripts — just instantiate `G2Project` and read.

## Pitfalls

- Don't add `<gsas_root>/GSASII` to sys.path — the inner directory is the package, not a script root. Always add the parent (`<gsas_root>`).
- Don't use the system Python; numpy import will fail. Always use `<python>` from `config.json`.
- Each histogram may print `5 values read from .../config.ini` and a binary-dir line on import — that's normal stdout, not an error.
- python-docx is not installed in the GSAS-II env by default. Install once: `<python> -m pip install python-docx -q`.

## Editing Word docs with extracted values

When updating an existing `.docx` paragraph in place, **don't** assign to `paragraph.text` (read-only) and **don't** clear all runs (loses inline formatting like italic / subscript / superscript on adjacent words).

Workflow:

1. Inspect runs first: `for r in paragraph.runs: print(r.font.italic, r.font.subscript, r.font.superscript, r.text)`.
2. Identify which run holds the bulk numeric text (usually one big run between formatted runs like italic *R*<sub>wp</sub> or italic space-group symbol).
3. Replace only that run's `.text`, preserving leading/trailing context the adjacent runs expect (e.g. keep ` = ` prefix and ` The space group is ` suffix).
4. Always `shutil.copy(path, path + '.bak')` before saving — Word docs are not git-tracked in user dirs.

χ² and Å³ in these docs are typically plain digits (`χ2`, `Å3`), not real superscripts — match the existing style rather than inventing superscripts.
