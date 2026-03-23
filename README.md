# tmux-resurrect (Per-Pane ZSH History)

This fork specifically addresses individualized zshell histories (`HISTFILE`) for every single pane. 

In vanilla `tmux-resurrect`, windows and panes are restored *before* their names are applied. This creates a race condition where shells read default window names (like `zsh`) on startup, causing multiple panes to incorrectly dump their history into the same file. 

This fork modifies the restoration pipeline to extract and apply the exact window name and `automatic-rename` parameters *at the time of pane creation*.

## Installation

Via [TPM](https://github.com/tmux-plugins/tpm) (Tmux Plugin Manager), update your `.tmux.conf` to point to this fork:

```tmux
# HISTFILEs directory creation
if-shell "[ ! -d ~/.zsh_history.d ]" \
	"run-shell 'mkdir ~/.zsh_history.d/'"

set -g @plugin 'b-sct/tmux-resurrect-zsh-per-pane-hist'
set -g default-command 'HISTFILE="$HOME/.zsh_history.d/$(tmux display-message -p "#S_#W_#P" | sed "s/[^a-zA-Z0-9_-]/_/g")" exec zsh'
```

## TODO

*   **Window Naming vs. Window Index Drift:** 
    Since the `HISTFILE` path relies on `#W` (Window Name), remember that `zsh` only evaluates the `default-command` (or `.zshrc` logic) once upon pane creation. If a window is later renamed (either manually or via `automatic-rename`), the pane will continue writing to the history file named after the *original* window name. 
    *   *Workaround:* If you want completely immutable history files regardless of naming changes, swap `#W` for `#I` (Window Index) in your configuration strings (e.g., `#S_#I_#P`).

*   **Pane Index Shifting Conflicts:** 
    Tmux pane indices (`#P`) are dynamically numbered. If you close a pane in the middle of a layout (for example, deleting pane 2 in a window with 4 panes), the remaining panes shift their indices down (panes 3 and 4 instantly become 2 and 3 respectively). 
    *   *The Problem:* Because the `HISTFILE` environment variable is statically set when the shell boots, those shifted panes will continue writing to `..._3` and `..._4`. If you then split the window again, the *new* pane will be assigned index 4, causing it to read and write to the same `HISTFILE` as the still-running shifted pane. 
    *   *Future Implementation:* Investigate a more robust tracking method for `HISTFILE` allocation, as relying purely on Tmux's sequential `#P` index is fragile during active layout manipulations.

