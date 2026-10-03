# NEO Pavi — fork notes

This is a personal fork of [hughhowey/neo](https://github.com/hughhowey/neo), kept at
[anairofpavitas/neo](https://github.com/anairofpavitas/neo). It stays as close to upstream
as possible so Hugh's updates can be pulled in cleanly. Every fork change in code is
marked with a `// FORK:` (or `/* FORK: */`) comment — `grep -n "FORK:" *.js styles.css`
lists them all. The exception is `package.json`, which can't hold comments (see §2).

## What this fork changes

### 1. Beats — a third outline level

The Outline tab goes Chapter (1, 2, 3) → Section (A, B, C) → **Beat (i, ii, iii)**, for
Story Grid's five commandments per scene.

- **Data:** beats live on the existing section objects in `book.json`:
  `sectionNotes[chId][n].beats = [{ id, text }]`. The field only appears once a section has
  beats, so books without beats are byte-for-byte what they were.
- **Outline keys:**
  - Tab on a section → it becomes the last beat of the section above it (in the same
    chapter). Refused with a toast if that scene or any of its beats has written prose, or
    if it's the chapter's first section. The section's own beats come along after it.
  - Shift+Tab on a beat → it becomes a section right after its parent scene.
  - Shift+Tab on a section that has beats → it becomes a chapter; its beats become that
    chapter's sections. Refused if any of those beats has written prose.
  - Enter on a beat → new beat below (at the very start of a line with text: above).
  - Backspace on an empty beat deletes it. Backspace on an empty section that still has
    beats does nothing but toast — beats are never dropped silently.
  - Right-click a beat → Delete. Right-click a section with beats warns that its beats go too.
  - Arrow up/down move through beats like any other line.
  - Note: a promoted beat is its own scene now. If unwritten, its ghost moves to the end of
    the chapter with the other unstarted scenes; if already written, it gets no `***` before it.
  - Tab on a chapter line that has sections → it becomes a section of the chapter above, and
    its sections become that section's beats (any beats they had follow them, flattened).
    So Shift+Tab on a section and Tab back is a lossless round trip. Upstream, this drops the
    chapter's sections; the fork only changes it when there are sections to lose.
  - Backspace on an empty chapter line that still has sections does nothing but toast.
- **Manuscript:** a scene's beats appear as gray ghosts right after the scene's ghost, with
  no `***` between beats (`***` still separates scenes). Writing over a beat ghost turns
  just that ghost into prose. Once any part of a scene is written, the scene's `***` sticks
  to the prose, so it survives exports and re-syncs.
- **Exports:** beat ghosts carry `data-sec-id` exactly like section ghosts, so
  `parasFromHtml` strips them (and their planted `***`) from EPUB/DOCX/PDF/MD/TXT, and word
  counts skip them.

Code: the `FORK: BEATS` block after `syncGhosts` in `app.js` (`outlineBeatLine`,
`forkSectionKey`, `forkChapterKey`, `forkSectionMenu`, `forkEmitScenes`, `forkKeepSceneBreaks`), plus
one-line hooks in `renderOutline`, `outlineLine`, `syncGhosts` and the ghost `beforeinput`
handler. CSS is one block at the end of `styles.css`.

**Known limitation (inherited from upstream):** on re-sync, ghosts of scenes nobody has
started move to the end of the chapter. A started scene keeps its unwritten ghosts next
to its prose.

### 2. Build identity

`package.json` → `build`:

| Field | Upstream | Fork |
|---|---|---|
| `build.appId` | `com.hughhowey.neo` | `com.pavi.neo` |
| `build.productName` | `NEO` | `NEO Pavi` |
| `build.publish.owner` | `hughhowey` | `anairofpavitas` |

`main.js` `latestReleaseFromGitHub()` also points at `anairofpavitas/neo`.

The **top-level** `productName` stays `"NEO"` on purpose. Electron names the settings folder
(`~/Library/Application Support/NEO`) after it. Keeping it means the fork shares settings and
saved API keys with official NEO. It also shares the single-instance lock, so the two apps
can't have your library open at the same time and overwrite each other's saves. After a
packaged build, check that settings still live in `.../Application Support/NEO`.

The auto-updater only looks at this fork's GitHub releases. It finds nothing until you
publish a release, and a release needs a version number higher than upstream's.

### 3. A chapter's first ghost is its first line

Upstream, a chapter with nothing written starts with an empty line, and the outline's
ghosts land under it. The fork drops that empty line, so the first scene's ghost is the
chapter's first line and works like every later one: click it, it's selected, type over it.

- Only when the chapter has no written words. A line with anything in it (text, a sticky
  mark) is never removed.
- If the outline empties out, one blank line comes back so there's somewhere to type.
- Jumping to a chapter whose first line is a ghost selects that ghost, so typing replaces it
  instead of running into it.
- A ghost in first position never gets the drop cap; the prose that replaces it does.
- Existing chapters are fixed the next time they load.

Code: `forkFirstLine` after the beats block in `app.js`, called from `syncGhosts` and
`renderChapters`; one line in `focusChapterStart`; one CSS rule at the end of `styles.css`.
This is upstream behavior, so it's a candidate PR for Hugh.

## How official NEO treats a book with beats

- It opens fine and ignores beats. The data stays in `book.json`.
- The next time you edit that chapter's outline in official NEO, the unwritten beat ghosts
  vanish from the manuscript. The fork puts the ghosts back on its next outline edit. A
  `***` that official NEO deleted along with a beat ghost (a scene with empty section text,
  or one you started in official NEO by writing a later beat first) does not come back —
  re-add it by hand.
- **Shift+Tab on a section in official NEO drops that section's beats for good.**
- Tab on a chapter line in official NEO drops that chapter's sections (and their beats).
- Editing section text in official NEO is safe; beats stay.

## Pulling upstream updates

One-time setup (already done in this clone):

```sh
git remote add upstream https://github.com/hughhowey/neo.git
```

Each update:

```sh
git checkout main
git status                        # must be clean — commit or stash first
git fetch upstream
git log --oneline main..upstream/main   # what's new from Hugh
git rebase upstream/main
```

If the rebase stops on a conflict:

1. `git status` shows the files. Conflicts will almost always sit at `// FORK:` lines in
   `renderOutline`, `outlineLine`, `syncGhosts`, the ghost `beforeinput` handler, the end of
   `styles.css`, the `build` block of `package.json`, or `latestReleaseFromGitHub` in `main.js`.
2. Keep Hugh's new code, then re-apply the one-line FORK hook on top of it.
3. If Hugh rewrote the per-section loop in `syncGhosts`, compare his new loop with
   `forkEmitScenes` and carry his change over. For books without beats they must behave
   the same.
4. `git add <file>` then `git rebase --continue`.

Then publish your rebased branch: `git push --force-with-lease origin main`.

### Retest checklist after every upstream pull

- [ ] `npm install && npm start` launches.
- [ ] `grep -c "FORK:" app.js` shows the same count as before the pull. Nothing got
      dropped in a conflict.
- [ ] A book **without** beats: outline, ghosts and `***` look exactly as before.
- [ ] A book with beats: i/ii/iii show, Tab / Shift+Tab / Enter / Backspace / right-click all work.
- [ ] Write over a beat ghost; the other ghosts stay.
- [ ] A chapter with an outline but no prose: the first ghost is the first line, no blank above it.
- [ ] Export EPUB, DOCX, PDF, MD and TXT: no beat or section text leaks, and no stray `***`.
- [ ] `package.json` still has `com.pavi.neo`, `NEO Pavi`, `anairofpavitas`.
- [ ] Packaged build, first launch: saved API keys still work (macOS may ask once for
      Keychain access, since the build is signed as a different app).

### Rebuild

```sh
npm install
npm run package        # macOS; package:win / package:linux / package:all for others
```

The app lands in `dist/`. To publish an update the fork's auto-updater can see:

1. Bump `version` above upstream's.
2. Build with `GH_TOKEN` set, e.g. `npx electron-builder --mac --publish always`.
3. Or upload the files in `dist/` to a GitHub release on `anairofpavitas/neo` by hand.

Notarization (`mac.notarize: true`) needs your own Apple Developer credentials. Without them,
set it to `false` for local builds.

### NEO Pocket

`pocket/www` loads a copy of the desktop `app.js` and `styles.css`. The copy isn't committed
to git, so after any change or upstream pull, refresh it before building Pocket:

```sh
cd pocket
cp ../app.js ../covers.js ../styles.css ../i18n.js www/ && cp ../node_modules/jszip/dist/jszip.min.js www/ && cp -R ../fonts ../locales www/
npx cap copy ios      # and/or android
```
