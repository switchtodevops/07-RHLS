# Editing Text Files

This chapter focuses on Vim, the text editor you can use directly in a terminal. For the exam, make sure you can enter text, save it, quit safely, and recognise Vim’s modes.

## 3.1 Editing text files from the command line

### Open or create a file

```bash
vim filename
```

If the file exists, Vim opens it. If it doesn’t, Vim creates it when you save. On systems with only Vim’s minimal package, use:

```bash
vi filename
```

RHEL provides `vim-minimal` for core `vi` features and `vim-enhanced` for more features, help, and `vimtutor`.

### Vim modes

Vim starts in command mode. The same key can do different things depending on the current mode.

| Mode | What it’s for | How to enter or leave |
| --- | --- | --- |
| Command | Navigate and edit with keystrokes | Vim starts here; press `Esc` to return here |
| Insert | Type text into the file | Press `i`; press `Esc` to leave |
| Visual | Select text to manipulate | Press `v`, `V`, or `Ctrl+v` |
| Extended command | Save, quit, and run editor commands | Press `:` from command mode |

If you’re unsure which mode you’re in, press `Esc` a couple of times. You’ll be back in command mode.

### Basic editing and saving

From command mode:

- `i` — start inserting text before the cursor.
- `x` — delete the character under the cursor.
- `u` — undo the most recent edit.

Use `:` commands to save or quit:

```vim
:w      " save and stay in Vim
:wq     " save and quit
:q!     " quit and discard changes since the last save
```

The capitalization of Vim commands matters: for example, `x` and `X` do different things.

### Select and copy text

Visual mode lets you select text before acting on it:

- `v` — select characters.
- `V` — select whole lines.
- `Ctrl+v` — select a rectangular block.

After selecting text, press `y` to copy it (yank), move to where you want the copy, and press `p` to paste it.

### Learn Vim using its tutorial

Run:

```bash
vimtutor
```

The chapter’s exercise practises the introductory lessons: navigation, quitting without saving, deleting text, entering insert mode, and saving with `:wq`. The `h`, `j`, `k`, and `l` keys move left, down, up, and right.

### Vim configuration

Vim settings can be configured system-wide or for a specific user:

- `/etc/vimrc` — system-wide settings.
- `~/.vimrc` — settings for the current user.

For example:

```vim
autocmd FileType yaml setlocal ts=2
set number
```

This sets a tab stop of 2 for YAML files and displays line numbers.

## 3.2 Guided exercise: Edit text files from the command line

The exercise has you use `vimtutor` to practise the basic workflow:

1. Open the tutorial.
2. Navigate and edit text.
3. Practise quitting without saving unwanted changes.
4. Enter insert mode and make changes.
5. Save and quit using `:wq`.

The important practical lesson is to return to command mode with `Esc` before entering a save or quit command.

## 3.3 Summary

Remember the core workflow:

```text
vim filename → i → type text → Esc → :wq
```

If you need to exit without saving:

```text
Esc → :q!
```

Exam focus: know the difference between command, insert, visual, and extended command modes—and choose the correct save or quit command.
