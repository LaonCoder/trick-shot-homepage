# Patch Note Folder Structure Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Move v1.0.0 patch note content from flat `assets/patch/<id>.<lang>.md` files into `assets/patch/v1.0.0/{ko,en}/patch.md` + `assets/patch/v1.0.0/image/`, and update the one line in `js/main.js` that builds the fetch URL so future releases follow the same folder shape.

**Architecture:** Pure content/path reshuffle — no parser or rendering changes. `manifest.json` keeps listing release ids (now folder names); `loadPatchReleases()` changes its fetch URL template from `<id>.<lang>.md` to `<id>/<lang>/patch.md`.

**Tech Stack:** Vanilla JS (ES5-style), static files, git for moves.

## Global Constraints

- No change to `parsePatchMd`/`patchDetailHTML` grammar or output shape.
- `version:` frontmatter (drives the `#v1-0-0` URL hash via `patchSlug()`) stays `v1.0.0` in both files — unrelated to the folder rename.
- Cache-busting query strings (`?v=NN`) on `js/main.js` must be bumped in all 5 HTML files per existing site convention, since `js/main.js` changes.

---

### Task 1: Move v1.0.0 content into the new folder layout

**Files:**
- Move: `assets/patch/1.0.0_20260519.ko.md` → `assets/patch/v1.0.0/ko/patch.md`
- Move: `assets/patch/1.0.0_20260519.en.md` → `assets/patch/v1.0.0/en/patch.md`
- Move: `assets/patch/v1_0_0.jpg` → `assets/patch/v1.0.0/image/v1_0_0.jpg`
- Modify: `assets/patch/v1.0.0/ko/patch.md` (frontmatter `thumb:` line)
- Modify: `assets/patch/v1.0.0/en/patch.md` (frontmatter `thumb:` line)
- Modify: `assets/patch/manifest.json`

- [ ] **Step 1: Create the folder structure and move files with `git mv`**

```bash
mkdir -p assets/patch/v1.0.0/ko assets/patch/v1.0.0/en assets/patch/v1.0.0/image
git mv assets/patch/1.0.0_20260519.ko.md assets/patch/v1.0.0/ko/patch.md
git mv assets/patch/1.0.0_20260519.en.md assets/patch/v1.0.0/en/patch.md
git mv assets/patch/v1_0_0.jpg assets/patch/v1.0.0/image/v1_0_0.jpg
```

- [ ] **Step 2: Update the `thumb:` frontmatter path in both moved files**

In `assets/patch/v1.0.0/ko/patch.md` and `assets/patch/v1.0.0/en/patch.md`, change:

```
thumb: assets/patch/v1_0_0.jpg
```

to:

```
thumb: assets/patch/v1.0.0/image/v1_0_0.jpg
```

- [ ] **Step 3: Update `manifest.json`**

Replace the contents of `assets/patch/manifest.json` with:

```json
[
  "v1.0.0"
]
```

- [ ] **Step 4: Verify the moved files exist and old ones are gone**

```bash
ls assets/patch/v1.0.0/ko assets/patch/v1.0.0/en assets/patch/v1.0.0/image
ls assets/patch | grep -E '1\.0\.0_20260519|v1_0_0\.jpg' || echo "old files gone, as expected"
```

Expected: `patch.md` listed under `ko`/`en`, `v1_0_0.jpg` listed under `image`, and the grep line prints "old files gone, as expected".

- [ ] **Step 5: Commit**

```bash
git add assets/patch
git commit -m "$(cat <<'EOF'
v1.0.0 패치노트를 assets/patch/v1.0.0/{ko,en,image} 폴더 구조로 이동

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 2: Update `loadPatchReleases()` to fetch from the new path shape

**Files:**
- Modify: `js/main.js` (function `loadPatchReleases`, the `fetch(...)` call inside `versions.map(...)`)
- Modify: `index.html`, `patch-notes.html`, `privacy.html`, `terms.html`, `oss-license.html` (bump `js/main.js?v=NN` cache-bust query)

**Interfaces:**
- Consumes: `fetchPatchManifest()` output — array of strings (now `["v1.0.0"]` instead of `["1.0.0_20260519"]`); shape unchanged, only the string values differ.

- [ ] **Step 1: Change the fetch URL template**

Find in `js/main.js` (inside `loadPatchReleases`):

```javascript
        return fetch("assets/patch/" + id + "." + lang + ".md")
```

Replace with:

```javascript
        return fetch("assets/patch/" + id + "/" + lang + "/patch.md")
```

- [ ] **Step 2: Bump the `js/main.js` cache-bust version in all 5 HTML files**

```bash
grep -n "main.js?v=" index.html patch-notes.html privacy.html terms.html oss-license.html
```

Note the current number `N`, then:

```bash
sed -i '' "s/js\/main.js?v=$N/js\/main.js?v=$((N+1))/" index.html patch-notes.html privacy.html terms.html oss-license.html
grep -n "main.js?v=" index.html patch-notes.html privacy.html terms.html oss-license.html
```

Expected: all 5 files show the bumped version.

- [ ] **Step 3: Serve the site locally and verify the release still loads**

```bash
python3 devserver.py 8099 > /tmp/devserver2.log 2>&1 &
sleep 1
curl -s http://localhost:8099/assets/patch/manifest.json
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8099/assets/patch/v1.0.0/ko/patch.md
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8099/assets/patch/v1.0.0/en/patch.md
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8099/assets/patch/v1.0.0/image/v1_0_0.jpg
kill %1
```

Expected: manifest prints `["v1.0.0"]` (formatting aside), and all three curl status codes are `200`.

- [ ] **Step 4: Sanity-check `loadPatchReleases` end-to-end with Node against the real manifest + files**

```bash
node -e '
var fs = require("fs");
var src = fs.readFileSync("js/main.js", "utf8");
var m = src.match(/return fetch\("assets\/patch\/" \+ id \+ "\/" \+ lang \+ "\/patch\.md"\)/);
if (!m) { console.error("FAIL: fetch template not updated"); process.exit(1); }
var manifest = JSON.parse(fs.readFileSync("assets/patch/manifest.json", "utf8"));
manifest.forEach(function (id) {
  ["ko", "en"].forEach(function (lang) {
    var p = "assets/patch/" + id + "/" + lang + "/patch.md";
    if (!fs.existsSync(p)) { console.error("FAIL: missing " + p); process.exit(1); }
  });
});
console.log("OK: manifest ids resolve to existing patch.md files for ko and en");
'
```

Expected: prints `OK: manifest ids resolve to existing patch.md files for ko and en`.

- [ ] **Step 5: Commit**

```bash
git add js/main.js index.html patch-notes.html privacy.html terms.html oss-license.html
git commit -m "$(cat <<'EOF'
패치노트 로더가 버전 폴더 구조(<id>/<lang>/patch.md)를 읽도록 갱신

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```
