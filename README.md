# skills

Claude Code skills and guides: council, cross-review, deep-code-review,
latex-build, review, rigorous-textual-review, textual-compression, and
WRITING_GOOD_RUST.md.

## Gemini seats in cross-review: the `agy` wrapper

The cross-review skill runs its Gemini seat with the Antigravity CLI, `agy`.

**Symptom.** When Gemini is queried through `agy`, a window pops up asking the
user to unlock their login keyring. A seat started from a script then blocks
with no output until someone answers the dialog. `agy` asks gnome-keyring for
its stored credentials; on many Linux desktops gnome-keyring is also the SSH
agent, and the request opens the unlock dialog. This is
[antigravity-cli issue #547](https://github.com/google-antigravity/antigravity-cli/issues/547),
open with no fix as of 2026-11-18. `unset SSH_AUTH_SOCK` and the gemini-cli
file-storage variables suggested there did not stop the dialog on Fedora 44.

**Fix.** Presenting the session as a remote login with no display makes `agy`
use its own token store. A wrapper named `agy`, in a directory ahead of the real
binary on `PATH`, does that for every call, so the skill runs `agy` plainly. The
real binary is in `~/.local/bin`, and agy's updater rewrites that file, so the
wrapper goes in `~/bin`:

```sh
#!/bin/sh
# ~/bin/agy: stops agy from opening the gnome-keyring unlock dialog
# (antigravity-cli issue #547).
export DISPLAY=""
export SSH_CLIENT="127.0.0.1 12345 22"
export SSH_TTY="/dev/pts/0"
exec "$HOME/.local/bin/agy" "$@"
```

```bash
chmod +x ~/bin/agy
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.bashrc     # and ~/.profile
```

Check with `command -v agy`, which must print `~/bin/agy`. Shells started before
the change still find the real binary; call `~/bin/agy` by full path there.
