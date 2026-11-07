# rex-tuicr

Review a coding agent's uncommitted changes in [tuicr](https://tuicr.dev) right beside it in
[Rex](https://www.superlogical.com/rex), and hand your comments back to that agent.

On a pane in a repo, one key opens `tuicr -w` in a pane on its right, in that pane's
folder. When the pane runs an agent (Claude Code, Codex, … anything that reports a Rex
program status), your comments go back to it. Save and quit tuicr
(`ZZ` or `:wq`) and the comments that are new or edited since the review opened go to the
agent as one numbered prompt:

```
Code review comments from tuicr (repo: app, branch: main):

1. src/a.rs:42
   the comment, continuation lines indented

Please address each comment. Reply with what you changed per item.
```

- **Paste** (`shift+cmd+d` below): pasted into the agent's input, unsent, so you can add words.
- **Submit** (palette only below): pasted and Enter pressed.

tuicr keeps old comments until a commit, so the script snapshots every tuicr session of the
checkout first and sends only what changed.

It works on any pane in a repo. On a pane with no agent (a shell, an editor) nothing is
typed into it, since Enter there would run the comments: they go to the clipboard instead.
If the agent's pane is gone by the time you quit tuicr, they go to the clipboard too. A port
of [herdr-app-tuicr](https://github.com/jmayaalv/herdr-app-tuicr).

## Needs

Rex for macOS, node 20+ (built-ins only), tuicr 0.28+ (`brew install tuicr`), git.

## Install

```sh
git clone https://github.com/jmayaalv/rex-tuicr ~/Developer/rex-tuicr
ln -s ~/Developer/rex-tuicr/tuicr-review ~/.config/rex/tuicr-review
```

Then add to `~/.config/rex/init.lua` and run `rex config reload`:

```lua
local terminal = "com.superlogical.terminal"

local function pane_folder(ctx)
  local p = rex.block.call(terminal, "process", { block_id = ctx.block_id })
  return p and p.foreground and p.foreground.cwd or os.getenv("HOME")
end

local function tuicr_review(mode)
  return function(ctx)
    -- Agents report a program status; any other pane only gets the clipboard.
    local st = rex.block.call(terminal, "program_status", { block_id = ctx.block_id })
    local how = (st and st.records and #st.records > 0) and mode or "clipboard"
    local _, err = rex.session.new_split{
      anchor_id = ctx.block_id,
      direction = "horizontal",
      side      = "after",
      focus     = true,
      layout    = rex.layout.block{
        flavor  = terminal .. ".shell",
        label   = "tuicr",
        options = {
          cwd          = pane_folder(ctx),
          command      = { os.getenv("HOME") .. "/.config/rex/tuicr-review", how, ctx.session_id, ctx.block_id },
          path_prepend = { "/opt/homebrew/bin" },  -- where tuicr and node are
        },
      },
    }
    if err ~= nil then error(err, 0) end
  end
end
rex.action{ name = "tuicr_review_paste", title = "Review Changes in tuicr (paste comments)", run = tuicr_review("paste") }
rex.action{ name = "tuicr_review_submit", title = "Review Changes in tuicr (submit comments)", run = tuicr_review("submit") }
rex.bind("shift+cmd+d", "tuicr_review_paste")  -- replaces Split Pane Down (shift+cmd+down still splits)
```

## License

MIT
