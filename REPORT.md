# tmux contribution report

Upstream: https://github.com/tmux/tmux (default branch `master`)
Base for every branch below: upstream `master` at
`a73616d781f2af13085691c741cfb0780d365328`.

Nothing was posted, pushed or opened on the upstream repository. Both fixes
are pushed to this fork only, each on its own branch.

| | Branch | Commit | Files |
|---|---|---|---|
| Fix 1 | `fix-choose-tree-info-flag` | `1c9288e9f632efab40b09c8b6fda801b4ab9154e` | cmd-choose-tree.c, window-tree.c, tmux.1, regress/choose-tree.sh |
| Fix 2 | `fix-usage-strings` | `3bed3bbda56a8721197cc4821cabf930ed1e5f95` | cmd-choose-tree.c, cmd-list-{buffers,clients,keys,panes,windows}.c, cmd-load-buffer.c, tmux.1, regress/list-commands.sh |

Both commits are authored by `b84330808 <b84330808@gmail.com>`, and each
message ends with `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.

**The two branches conflict on one line**: the `choose-tree` `.usage` string
in cmd-choose-tree.c. Fix 1 adds `i` and fix 2 adds `y`. Whichever lands
second needs `[-GhikNrswyZ]`. Nothing else overlaps. tmux.1 merges cleanly.

## Policy notes

How tmux takes contributions:

- README and .github/README.md: "Bug reports, feature suggestions and
  especially code contributions are most welcome. Please send by email to:
  tmux-users@googlegroups.com. Or open a GitHub issue or pull request."
- Wiki https://github.com/tmux/tmux/wiki/Contributing says code can come by
  email or as a GitHub PR. OpenBSD CVS is the primary repository. A PR is
  reviewed on GitHub, then applied as a patch to OpenBSD and synced back. For
  example, the similar usage-string fix tmux/tmux#5751 was closed with "Applied
  to OpenBSD now, will be in GitHub later." So expect the PR to be *closed*
  (not merged) once it is applied.
- Code style: OpenBSD style(9). That means 8-column tabs, lines under 80
  columns, no C++ comments, and declarations at the top of a function. Both
  diffs follow it.
- .github/CONTRIBUTING.md only covers opening issues (logs, `$TERM` and so
  on). There is no PR template.
- .github/copilot-instructions.md only tells Copilot not to leave review
  comments. It is not a contributor policy.

AI policy, from the wiki Contributing page, section "AI-Assisted
Contributions":

- "Code written with the assistance of AI can be acceptable."
- "PRs consisting of large dumps of unreviewed AI-generated code are not
  acceptable." Contributors must review and understand the change and be
  prepared to explain it.
- AI output must "either be trivial enough to be not copyrightable (basic
  refactoring, one line bug fixes), or there must be a public statement
  available from the AI publisher showing they do not assert copyright over
  the work". The page itself lists Anthropic's commercial terms: "Anthropic
  hereby assigns to Customer its right, title and interest (if any) in and to
  Outputs."

Conclusion: GitHub PRs are accepted and AI-assisted contributions are allowed
with disclosure and review. Both changes are small (fix 1 is +20/-5 including
the test; fix 2 is +27/-13 and almost all string edits). The PR bodies below
disclose Claude Code use. Before opening either PR, Wei-Lun should read the
diff and be ready to explain it.

---

## Fix 1: `choose-tree -i` is documented but rejected

### How it was found

I found this myself; there is no upstream issue for it. I compared each
command's accepted flags (the `.args` template in `cmd-*.c`) with its synopsis
in tmux.1. `choose-tree` was the only command whose manual lists a flag that
the command rejects.

History:

- `bf187170` ("Extend client mode so the preview can be changed to a view with
  a summary of the client terminal...") added `-i` to **choose-client** in
  code. Its tmux.1 hunks went into the **choose-tree** section instead: `-i` in
  the choose-tree synopsis, and "`-i` starts showing client information instead
  of the preview" in the choose-tree description. The choose-client synopsis
  stayed `[-hkNryZ]`.
- `c0f213b0` ("Add info modes to tree mode like for clients.") gave tree mode
  its own information view on the `i` key. It fixed the key tables but left
  the `-i` flag text under choose-tree, and choose-tree still does not accept
  `-i`.

Result on master: `choose-tree -i` (documented) fails with "unknown flag -i".
`choose-client -i` (undocumented) works.

### Change

- cmd-choose-tree.c: add `i` to choose-tree's args template and usage.
- window-tree.c: in `window_tree_init()`, set `data->preview_is_info` from
  `-i` and set the view name to "info" or "preview" to match, exactly as
  `window_client_init()` does.
- tmux.1: add `-i` to the choose-client synopsis and description. Change the
  choose-tree `-i` text to "starts showing session, window and pane
  information instead of the preview".
- regress/choose-tree.sh: open `choose-tree -i`, wait for `(view: info)`,
  press `i`, wait for `(view: preview)`.

### PR title

    Accept -i for choose-tree to start with the information view

### PR body draft

    choose-client -i was added in bf187170 to start client mode with the
    client information view, but the manual hunks for it went into the
    choose-tree section: choose-tree's synopsis lists -i and says it "starts
    showing client information instead of the preview", while choose-client
    does not mention it. c0f213b0 later gave tree mode its own information
    view on the i key, but choose-tree still rejects the flag:

        $ tmux choose-tree -i
        command choose-tree: unknown flag -i

    This makes choose-tree -i start with the session, window and pane
    information view (the same view the i key shows), documents -i for
    choose-client, and corrects the choose-tree description.
    regress/choose-tree.sh now checks that choose-tree -i opens the
    information view and that i switches back to the preview.

    I used Claude Code to write this change.

### Reproduction and before/after

Build (same for every binary below):

```sh
git remote add upstream https://github.com/tmux/tmux.git && git fetch upstream
git checkout upstream/master        # or fix-choose-tree-info-flag
sh autogen.sh && ./configure && make
```

Manual reproduction:

```sh
./tmux -Ltest -f/dev/null new -d -x80 -y24
./tmux -Ltest choose-tree -i; echo $?
./tmux -Ltest choose-client -i; echo $?
./tmux -Ltest kill-server
```

Before (upstream master a73616d7):

```
$ tmux choose-tree -i; echo $?
command choose-tree: unknown flag -i
1
$ tmux choose-client -i; echo $?
0
```

After (fix-choose-tree-info-flag 1c9288e9):

```
$ tmux choose-tree -i; echo $?
0
$ tmux choose-client -i; echo $?
0
```

Regress test. These are the fix branch's regress/choose-tree.sh, run against
each binary through `TEST_TMUX`:

```sh
cd regress
env -i LC_CTYPE=C.UTF-8 SHELL=/bin/sh TEST_TMUX=/path/to/master/tmux sh choose-tree.sh; echo "exit=$?"
env -i LC_CTYPE=C.UTF-8 SHELL=/bin/sh TEST_TMUX=/path/to/fix1/tmux   sh choose-tree.sh; echo "exit=$?"
```

```
--- TEST_TMUX=upstream-master
command choose-tree: unknown flag -i
choose-tree -i failed
exit=1
--- TEST_TMUX=fix-choose-tree-info-flag
exit=0
```

Mutation check: I made `-i` parse but left it unused (`preview_is_info = 0`
in `window_tree_init`). The new test then fails with `timed out waiting for
'(view: info)'`, so it checks the behaviour and not only flag parsing.

choose-client.sh still passes on the fix branch.

### Competing PR and issue search (all states)

I searched with `gh api -X GET search/issues` on `repo:tmux/tmux`, covering
open and closed results:

- `is:pr choose-tree -i` (16 results), `is:pr choose-tree info` (2),
  `is:pr choose-client info` (2), `is:pr choose-client -i` (8),
  `is:pr info view tree mode` (1). None touches choose-tree `-i` or the
  information view flag. The closest are tmux/tmux#5177 (adds `-h`, closed) and
  tmux/tmux#5521 (preview memory leak, closed).
- `is:issue choose-tree -i` (147 results, the newest pages reviewed),
  `is:issue "unknown flag -i"` (1, unrelated: tmux/tmux#3462),
  `is:issue choose-tree info view` (6, unrelated). The open feature request
  tmux/tmux#5823 ("choose-tree UI improvements") asks for inline help, new
  sessions and moving tagged items. It does not cover `-i`.
- All 20 open upstream PRs were listed (newest tmux/tmux#5833). None touches
  cmd-choose-tree.c or window-tree.c init.

---

## Fix 2: usage strings and synopses that do not match command flags

### How it was found

I found this myself, with the same cross-check plus `tmux list-commands`. The
problems:

- Missing space between options. `"... [-O order]" CMD_TARGET_*_USAGE`
  concatenates to `[-O order][-t target-session]` (list-clients, list-windows)
  and `[-O order][-t target-window]` (list-panes). These came from `3c3d9ce3`,
  which added `-O`. list-keys has `[-P prefix-string][-T key-table]`.
- Usage strings leave out flags the command accepts and the manual documents:
  `-r` for list-buffers and list-clients (from `3c3d9ce3`), `-w` for
  load-buffer (from `37b1600d`), and `-y` for choose-buffer, choose-client and
  choose-tree. `5a68730e` added `-y` to the args templates and tmux.1 but not
  to the usage strings.
- customize-mode accepts `-y`, and window-customize.c uses it for
  `PROMPT_ACCEPT`, but neither its usage nor tmux.1 mentions it. tmux.1 also
  shows `[template]` for customize-mode, which takes no arguments:
  `customize-mode x` gives "too many arguments (need at most 0)".
- The capture-pane synopsis in tmux.1 lacks `-T`, although the text below it
  describes `-T`.

Not changed on purpose: flags that are accepted but undocumented for
compatibility (`display-message -F`, `join-pane -p`, `refresh-client -F`,
`select-pane -P/-g`, `switch-client -F`, `run-shell -s`). These look like
deliberately hidden legacy options.

### PR title

    Fix usage strings and synopses that do not match command flags

### PR body draft

    Some usage strings shown by list-commands and in usage errors join two
    options without a space:

        list-clients (lsc) [-F format] [-f filter] [-O order][-t target-session]
        list-keys (lsk) [-1aNr] [-F format] [-O order] [-P prefix-string][-T key-table] [key]
        list-panes (lsp) [-asr] [-F format] [-f filter] [-O order][-t target-window]
        list-windows (lsw) [-ar] [-F format] [-f filter] [-O order][-t target-session]

    Others leave out flags the commands accept and the manual documents: -r
    for list-buffers and list-clients, -w for load-buffer, and -y for
    choose-buffer, choose-client and choose-tree. customize-mode accepts -y
    too but neither its usage nor the manual mention it, and the manual shows
    a [template] argument it does not take. The capture-pane synopsis is
    missing -T, which is described below it.

    regress/list-commands.sh now checks that no usage string has "][" and
    that the flags above appear.

    I used Claude Code to write this change.

### Reproduction and before/after

```sh
./tmux -Ltest -f/dev/null new -d
./tmux -Ltest list-commands | grep -E "^(list-buffers|list-clients|list-keys|list-panes|list-windows|load-buffer|choose-buffer|choose-client|choose-tree|customize-mode) "
./tmux -Ltest kill-server
```

Before (upstream master a73616d7):

```
choose-buffer [-kNrZ] [-F format] [-f filter] [-K key-format] [-O sort-order] [-t target-pane] [template]
choose-client [-hikNrZ] [-F format] [-f filter] [-K key-format] [-O sort-order] [-t target-pane] [template]
choose-tree [-GhkNrswZ] [-F format] [-f filter] [-K key-format] [-O sort-order] [-t target-pane] [template]
customize-mode [-kNZ] [-F format] [-f filter] [-t target-pane]
list-buffers (lsb) [-F format] [-f filter] [-O order]
list-clients (lsc) [-F format] [-f filter] [-O order][-t target-session]
list-keys (lsk) [-1aNr] [-F format] [-O order] [-P prefix-string][-T key-table] [key]
list-panes (lsp) [-asr] [-F format] [-f filter] [-O order][-t target-window]
list-windows (lsw) [-ar] [-F format] [-f filter] [-O order][-t target-session]
load-buffer (loadb) [-b buffer-name] [-t target-client] path
```

After (fix-usage-strings 3bed3bbd):

```
choose-buffer [-kNryZ] [-F format] [-f filter] [-K key-format] [-O sort-order] [-t target-pane] [template]
choose-client [-hikNryZ] [-F format] [-f filter] [-K key-format] [-O sort-order] [-t target-pane] [template]
choose-tree [-GhkNrswyZ] [-F format] [-f filter] [-K key-format] [-O sort-order] [-t target-pane] [template]
customize-mode [-kNyZ] [-F format] [-f filter] [-t target-pane]
list-buffers (lsb) [-r] [-F format] [-f filter] [-O order]
list-clients (lsc) [-r] [-F format] [-f filter] [-O order] [-t target-session]
list-keys (lsk) [-1aNr] [-F format] [-O order] [-P prefix-string] [-T key-table] [key]
list-panes (lsp) [-asr] [-F format] [-f filter] [-O order] [-t target-window]
list-windows (lsw) [-ar] [-F format] [-f filter] [-O order] [-t target-session]
load-buffer (loadb) [-w] [-b buffer-name] [-t target-client] path
```

Regress test. These are the fix branch's regress/list-commands.sh, run
against each binary:

```sh
cd regress
env -i LC_CTYPE=C.UTF-8 SHELL=/bin/sh TEST_TMUX=/path/to/master/tmux sh list-commands.sh; echo "exit=$?"
env -i LC_CTYPE=C.UTF-8 SHELL=/bin/sh TEST_TMUX=/path/to/fix2/tmux   sh list-commands.sh; echo "exit=$?"
```

```
--- TEST_TMUX=upstream-master
list-clients (lsc) [-F format] [-f filter] [-O order][-t target-session]
list-keys (lsk) [-1aNr] [-F format] [-O order] [-P prefix-string][-T key-table] [key]
list-panes (lsp) [-asr] [-F format] [-f filter] [-O order][-t target-window]
list-windows (lsw) [-ar] [-F format] [-f filter] [-O order][-t target-session]
exit=1
--- TEST_TMUX=fix-usage-strings
exit=0
```

These related tests also pass on the fix branch: choose-tree.sh,
choose-client.sh, choose-buffer.sh, customize-mode.sh, buffers.sh,
list-keys-sort-name.sh and capture-pane-*.sh.

### Competing PR and issue search (all states)

- `is:pr usage string` (20), `is:pr usage in:title` (6),
  `is:pr list-clients usage` (9), `is:pr load-buffer -w` (4),
  `is:pr customize-mode -y` (1), `is:pr capture-pane -T` (31),
  `is:pr synopsis` (1), `is:pr manual flags missing` (7),
  `is:issue usage missing space` (5), `is:issue list-commands usage` (7).
- None touches these strings. Earlier fixes of the same kind, all closed and
  applied: tmux/tmux#5751 (split-window/new-pane summaries, Oct 2026),
  tmux/tmux#4445 ("Fix missing space in usage"), tmux/tmux#4351 (missing
  spaces after `CMD_TARGET_PANE_USAGE`), tmux/tmux#1628 (capture-pane usage
  space). They show the maintainer accepts this kind of fix.
- No open upstream PR changes cmd-list-*.c, cmd-load-buffer.c or the
  cmd-choose-tree.c usage strings.

---

## Testing summary

- Full regress suite (`cd regress && make -j8`) on **unmodified upstream
  master** built with `./configure --enable-asan`: 281 passed, 45 skipped,
  0 failed. The skipped tests are the image-* tests, because the build had no
  `--enable-images`.
- Full regress suite (`make -j8`) on the **fix 2** build, plain
  `./configure`: 280 passed, 45 skipped, 1 failed. The failure was
  `utf8-test.sh`, which runs `cat UTF-8-test.txt`, sleeps 1 second and then
  compares a capture. Run alone it passed 3/3 on the master build, the fix 1
  build and the fix 2 build, so it is a timing flake under parallel load.
  Neither change touches UTF-8 or capture code.
- Fix 1: choose-tree.sh and choose-client.sh pass. choose-tree.sh fails on
  master and passes with the fix.
- Fix 2: list-commands.sh fails on master and passes with the fix. The
  related tests listed above also pass.
- Both fixes merged onto master, with the one conflicting line resolved to
  `[-GhikNrswyZ]` and built with `--enable-asan`: choose-tree.sh,
  choose-client.sh, list-commands.sh and customize-mode.sh all pass with no
  ASAN reports.

## Upstream issues considered and rejected

- tmux/tmux#5828 (sscanf parsers accept trailing junk): the reporter offered
  to send a patch, so a PR would compete with theirs.
- tmux/tmux#5814 (`:@` / `:{current}` targets): the maintainer invited a PR
  and the reporter said they are starting on it.
- tmux/tmux#5820 (colon in session name): allowing `:` and `.` was a
  deliberate maintainer choice (`65a032b2` "Allow :. in names again,
  forbidding them is overly pernickety"). The maintainer replied with a
  workaround.
- tmux/tmux#5223 (show-pane on minimised panes): the commands only exist on
  the hidden-panes branch, not master.
- tmux/tmux#5822 (`protocol version mismatch (client 8, server 8)`): macOS
  only, no logs, and I could not reproduce it on Linux.
- tmux/tmux#5829 (search prompt cursor colour): a design and feature request
  about the 3.8 prompt changes, not a contained bug.
