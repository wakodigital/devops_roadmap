# Lab 02 — Editing Files

**Roadmap topics:** Vim · Nano
**Time:** about 2 hours (most of it is Vim repetition)
**Touches system state:** installs `vim-enhanced`. All edits are made to copies in `~/labwork/02`.

## What you will be able to do

- Make a quick, correct edit to a config file in nano.
- Open, navigate, edit, search, replace, save and quit in Vim without looking anything up.
- Edit root-owned files safely with `sudoedit`, and sudoers with `visudo`.

On a headless server every config change goes through a terminal editor. Nano is enough to get work done. Vim is worth the investment because `vi` is on every Unix machine you will ever SSH into, including minimal containers where nothing else is installed.

## Setup

```bash
mkdir -p ~/labwork/02 && cd ~/labwork/02
sudo dnf install -y vim-enhanced nano

# Working copies of real files, so there is something realistic to edit
cp /etc/ssh/sshd_config sshd_config
cp /etc/services services
cat > servers.txt <<'EOF'
web01 10.0.1.11 prod nginx
web02 10.0.1.12 prod nginx
web03 10.0.2.13 stage nginx
db01 10.0.1.21 prod postgres
db02 10.0.2.22 stage postgres
cache01 10.0.1.31 prod redis
EOF
```

A minimal Rocky install ships `vi` (the `vim-minimal` package) with no syntax highlighting and no `vimtutor`. `vim-enhanced` gives you the `vim` command.

---

## 1. Nano

Nano shows its shortcuts at the bottom of the screen. `^` means Ctrl and `M-` means Alt.

```bash
nano servers.txt
```

| Keys | Action |
|---|---|
| `Ctrl-O`, `Enter` | write (save) |
| `Ctrl-X` | exit (asks to save if modified) |
| `Ctrl-W` | search; `Alt-W` repeats |
| `Ctrl-\` | search and replace |
| `Ctrl-K` | cut the current line |
| `Ctrl-U` | paste |
| `Alt-6` | copy the current line |
| `Ctrl-_` | go to line number |
| `Alt-U` / `Alt-E` | undo / redo |
| `Ctrl-A` / `Ctrl-E` | start / end of line |
| `Alt-A` | start selecting text (then move, then cut or copy) |
| `Ctrl-G` | help |

Useful ways to start it:

```bash
nano -l sshd_config        # show line numbers
nano +25 sshd_config       # open at line 25
nano -B servers.txt        # keep a backup as servers.txt~ when saving
```

### Exercises 1 (nano)

Do all of these in `servers.txt`, in one session.

1. Add a line for `cache02 10.0.2.32 stage redis` at the end.
2. Move the `db01` line so it is the first line of the file.
3. Replace every `stage` with `staging`.
4. Save and exit. Verify with `cat servers.txt`.

<details><summary>Keystrokes</summary>

1. `Alt-/` jumps to the end of the file. Type the line.
2. Put the cursor on the `db01` line, `Ctrl-K` to cut. `Alt-\` jumps to the top. `Ctrl-U` to paste.
3. `Ctrl-\`, type `stage`, Enter, type `staging`, Enter, then `A` for all.
4. `Ctrl-O`, Enter, `Ctrl-X`.

</details>

---

## 2. Vim

### The one idea: modes

Vim is modal. The keyboard means different things depending on the mode you are in.

| Mode | How you get there | What keys do |
|---|---|---|
| **Normal** | `Esc` (it is where you start) | keys are commands: move, delete, copy, paste |
| **Insert** | `i`, `a`, `o`, and friends | keys type text |
| **Visual** | `v`, `V`, `Ctrl-v` | select text, then act on it |
| **Command-line** | `:` | save, quit, substitute, settings |

When lost, press `Esc` twice. You are now in Normal mode.

### Survival set

```bash
vim servers.txt
```

| Keys | Action |
|---|---|
| `i` | insert before the cursor |
| `Esc` | back to Normal |
| `:w` | write (save) |
| `:q` | quit |
| `:wq` or `ZZ` | save and quit |
| `:q!` | quit and throw away changes |
| `u` | undo |
| `Ctrl-r` | redo |

With only this table you can edit any file. Everything else makes you faster.

### Do vimtutor first

```bash
vimtutor
```

It takes 30 minutes and it is the most efficient Vim instruction that exists. It opens a copy of a lesson file that you edit as you read. Do it now, then come back. The rest of this section is reference and drills.

### Moving

| Keys | Motion |
|---|---|
| `h j k l` | left, down, up, right |
| `w` / `b` / `e` | next word start / previous word start / word end |
| `0` / `^` / `$` | start of line / first non-blank / end of line |
| `gg` / `G` | first line / last line |
| `42G` or `:42` | line 42 |
| `Ctrl-d` / `Ctrl-u` | half a page down / up |
| `Ctrl-f` / `Ctrl-b` | a page down / up |
| `%` | jump to the matching bracket |
| `f x` / `t x` | forward to / until the next `x` on this line; `;` repeats |
| `{` / `}` | previous / next blank line (paragraph) |
| `/text` / `?text` | search forward / backward; `n` and `N` for next and previous |
| `*` | search for the word under the cursor |

### Entering Insert mode

| Keys | Where typing begins |
|---|---|
| `i` / `a` | before / after the cursor |
| `I` / `A` | start / end of the line |
| `o` / `O` | a new line below / above |

### Editing: operator + motion

This is the part that makes Vim a language. An **operator** says what to do and a **motion** says to what. Learn a few of each and they combine.

Operators: `d` delete (which also cuts), `c` change (delete then enter Insert), `y` yank (copy), `>` and `<` indent.

| Command | Reads as | Effect |
|---|---|---|
| `dw` | delete word | deletes to the start of the next word |
| `d$` or `D` | delete to end of line | |
| `dd` | (doubled operator = whole line) | deletes the line |
| `3dd` | | deletes 3 lines |
| `dG` | delete to end of file | |
| `cw` | change word | deletes the word, leaves you in Insert |
| `ci"` | change inside quotes | replaces the contents of a quoted string |
| `ci(` | change inside parentheses | |
| `yy` | yank line | copies the line |
| `yw` | yank word | |
| `p` / `P` | put after / before the cursor | pastes whatever was last deleted or yanked |
| `x` | | delete one character |
| `r x` | | replace one character with `x` |
| `J` | | join the next line onto this one |
| `.` | | **repeat the last change** |
| `~` | | toggle case |

`.` is the most valuable single key in Vim. Make a change once, move, press `.`.

### Visual mode

| Keys | Selects |
|---|---|
| `v` | characters |
| `V` | whole lines |
| `Ctrl-v` | a rectangular block |

Select, then press an operator: `d`, `y`, `c`, `>`, `<`.

Commenting out several lines with block mode: put the cursor on the first line, `Ctrl-v`, move down with `j` to cover the lines, `I` (capital), type `#`, press `Esc`. The `#` appears on every selected line.

### Search and replace

```
:s/old/new/          first match on the current line
:s/old/new/g         every match on the current line
:%s/old/new/g        every match in the file
:%s/old/new/gc       same, asking for confirmation at each
:10,20s/old/new/g    lines 10 to 20 only
:%s/\<prod\>/production/g    whole word only
:g/pattern/d         delete every line that matches
:v/pattern/d         delete every line that does NOT match
```

If the pattern contains slashes, pick another delimiter: `:%s#/var/log#/srv/log#g`.

### Settings you will want

Type these in a running Vim to try them, or put them (without the colon) in `~/.vimrc` to keep them.

```
:set number            line numbers
:set relativenumber    distances from the cursor, good for 5dd / 8j
:set hlsearch          highlight search matches (:noh clears the highlight)
:set incsearch         jump to matches as you type
:set ignorecase smartcase   case-insensitive unless you type a capital
:set paste             turn off auto-indent before pasting from the clipboard
:syntax on
```

A reasonable starter `~/.vimrc`:

```vim
set number
set hlsearch incsearch
set ignorecase smartcase
set expandtab tabstop=2 shiftwidth=2
syntax on
```

`expandtab` with a width of 2 matters for you specifically: YAML does not allow tab characters, and Ansible playbooks, Compose files and Kubernetes manifests are all YAML.

### Working with more than one file

```
:e other.txt       open another file
:ls                list open buffers
:b2                switch to buffer 2
:sp file           horizontal split      :vsp file    vertical split
Ctrl-w w           cycle between splits
:r file            read a file in below the cursor
:r !date           read a command's output in below the cursor
:!ls -l            run a shell command without leaving Vim
```

### When Vim surprises you

| Symptom | Cause | Fix |
|---|---|---|
| Typing produces chaos | you are in Normal mode | `Esc`, `u` until sane, then `i` |
| Screen is frozen | you pressed `Ctrl-s` (terminal flow control) | `Ctrl-q` |
| "E45: 'readonly' option is set" | file is not writable by you | `:q!` and reopen with `sudoedit` |
| "Swap file already exists" | a previous session crashed, or the file is open elsewhere | `R` to recover, then delete the `.swp` file; or `Q` to quit and check |
| "recording @q" at the bottom | you pressed `q` and started a macro | press `q` again |
| Pasted text is staircase-indented | auto-indent applied to the paste | `u`, then `:set paste`, paste again, `:set nopaste` |

### Exercises 2 (Vim)

Reset the file first (re-run the `cat > servers.txt` heredoc from Setup). Then, in one Vim session:

1. Go to the last line and add `lb01 10.0.1.41 prod haproxy` below it.
2. Delete the `web03` line.
3. Copy the `db01` line and paste it under `db02`. Change the pasted line's name to `db03` and its address to end in `.23`.
4. Change every `prod` to `production` using one command.
5. Indent all lines containing `nginx` by one level, using visual line mode.
6. Undo the indent.
7. Save and quit, then check with `cat`.

<details><summary>Keystrokes</summary>

1. `G` `o` type the line, `Esc`
2. `/web03` Enter, `dd`
3. `/db01` Enter, `yy`, `/db02` Enter, `p`. The cursor is now on the pasted line: `cw` `db03` `Esc` changes the name, then `:s/\.21/.23/` changes the address.
4. `:%s/prod/production/g`
5. `gg`, `V`, `j` to extend over the web lines, `>`
6. `u`
7. `:wq`

</details>

### Exercises 3 (a realistic config edit)

Open your **copy** of `sshd_config` in Vim. This is the same edit you would make when hardening SSH (Lab 13 does it for real, with safety checks).

1. Turn on line numbers.
2. Find the line containing `PermitRootLogin`. It is probably commented out.
3. Duplicate that line, uncomment the copy, and set it to `PermitRootLogin no`.
4. Do the same for `PasswordAuthentication`, set to `no`.
5. Count how many lines are comments: use `:%s/^#//gn` (the `n` flag counts matches without changing anything).
6. Save as a new file with `:w sshd_config.hardened`, quit, and compare:

```bash
diff sshd_config sshd_config.hardened
```

<details><summary>Keystrokes</summary>

1. `:set number`
2. `/PermitRootLogin` Enter
3. `yy` `p` then `0` `x` (deletes the `#`), then `$` `cw` `no` `Esc` (adjust if the value is more than one word: `:s/prohibit-password/no/`)
4. `/PasswordAuthentication` Enter, then the same sequence
5. `:%s/^#//gn` prints something like "120 matches on 120 lines"
6. `:w sshd_config.hardened` then `:q!` (the original copy is left unmodified)

</details>

---

## 3. Editing files you do not own

Running an editor as root (`sudo vim /etc/…`) works, but it runs your whole editor, its plugins and any shell you launch from it as root.

`sudoedit` is the better habit. It copies the file to a temp location, opens it in your editor as **you**, and copies it back as root only if you saved changes.

```bash
export EDITOR=vim                 # put this in ~/.bashrc to make it stick (Lab 03)
sudoedit /etc/motd                # add a line, save, quit
cat /etc/motd
sudoedit /etc/motd                # remove the line again
```

Two files have their own safe editors that validate syntax before saving, because a syntax error in either can lock you out:

| File | Edit with | Why |
|---|---|---|
| `/etc/sudoers` and `/etc/sudoers.d/*` | `sudo visudo` / `sudo visudo -f /etc/sudoers.d/name` | a broken sudoers file disables sudo entirely |
| `/etc/passwd`, `/etc/group` | `sudo vipw`, `sudo vigr` | locks the file against concurrent edits |

`visudo` uses `$EDITOR`, so it will be Vim or nano depending on what you exported.

Whatever editor you use, the pattern for changing a service's config on a remote machine is always: **back up, edit, validate, reload, verify, and keep your existing session open until you have confirmed a new one works.**

```bash
sudo cp -a /etc/ssh/sshd_config /etc/ssh/sshd_config.bak.$(date +%F)   # back up
# ... edit ...
sudo sshd -t                                                           # validate: silence means OK
# ... reload the service, then test from a second terminal ...
```

Most daemons have an equivalent of `sshd -t`: `nginx -t`, `visudo -c`, `named-checkconf`, `systemd-analyze verify`.

---

## Check yourself

<details><summary>You opened a file in Vim, made a mess, and want out without saving. What do you type?</summary>
<code>Esc</code> then <code>:q!</code> and Enter.
</details>

<details><summary>What does <code>ci"</code> do, and why is it better than deleting characters one at a time?</summary>
"Change inside quotes": it deletes everything between the surrounding double quotes and puts you in Insert mode. It works from anywhere inside the quotes, it is one change (so <code>.</code> repeats it and <code>u</code> undoes it in one step).
</details>

<details><summary>What is the difference between <code>:s/a/b/</code>, <code>:s/a/b/g</code> and <code>:%s/a/b/g</code>?</summary>
First match on the current line; all matches on the current line; all matches on every line of the file.
</details>

<details><summary>Why use <code>visudo</code> and not <code>sudo vim /etc/sudoers</code>?</summary>
<code>visudo</code> checks the syntax when you save and refuses to install a broken file. A broken sudoers file means nobody can use sudo, and on a machine where root login is disabled that needs console access to fix.
</details>

<details><summary>Why does <code>expandtab</code> matter for YAML?</summary>
YAML forbids tab characters for indentation. <code>expandtab</code> makes the Tab key insert spaces.
</details>

## Keep practising

Vim only sticks through use. For the rest of this workbook, make every edit in Vim. It will be slower for about a week.

## Cleanup

```bash
rm -r ~/labwork/02
```

Keep `vim-enhanced` installed and keep your `~/.vimrc`.
