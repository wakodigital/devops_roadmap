# Lab 14 — Shell Programming

**Roadmap topics:** Literals · Variables · Loops · Conditionals · Debugging
**Time:** about 4 hours
**Touches system state:** no. Everything happens in `~/labwork/14`. The finished script can optionally be installed in `/usr/local/bin`.

## What you will be able to do

- Quote correctly, every time, and explain why.
- Use variables, parameter expansion, arrays and command substitution.
- Write `if`, `case`, `for` and `while` constructs with the right test syntax.
- Write functions, parse options, handle errors and clean up after yourself.
- Debug a script that misbehaves, using tracing, strict mode and a linter.
- Build a command-line tool that behaves like one.

Bash is the glue of systems work: CI pipelines, container entrypoints, cron jobs, installers and the `command:` lines in your Ansible playbooks are all shell. It is also a language full of traps, and most of this lab is about the handful of habits that avoid them.

## Setup

```bash
mkdir -p ~/labwork/14 && cd ~/labwork/14
bash --version | head -1
```

### Your first script

```bash
cat > hello.sh <<'EOF'
#!/usr/bin/env bash
# The first line (the "shebang") names the interpreter that runs this file.
echo "Hello from $0, running as $(whoami) in $PWD"
EOF
chmod +x hello.sh
./hello.sh               # run via the shebang (needs the x bit)
bash hello.sh            # run by naming the interpreter (x bit not needed)
source hello.sh          # run inside the CURRENT shell: no new process (Lab 03)
```

`#!/usr/bin/env bash` finds `bash` on the PATH. `#!/bin/bash` is equally common. Use `#!/bin/sh` only if you restrict yourself to POSIX features, because on Debian and Ubuntu `sh` is `dash`, which lacks arrays, `[[ ]]` and much else used below.

Every command returns an **exit status**: 0 for success, 1–255 for failure. That number is the shell's notion of true and false, and everything in the conditionals section is built on it.

```bash
true;  echo $?           # 0
false; echo $?           # 1
ls /nonexistent; echo $? # 2
```

---

## 1. Literals

A literal is a value written directly in the source: a string or a number. In shell, almost everything is a string, and **quoting** decides how much the shell interprets before your command sees it.

### The three kinds of quoting

| Form | The shell expands | Use for |
|---|---|---|
| `'single'` | nothing. Every character is literal | fixed text, regexes, `awk` and `sed` programs |
| `"double"` | `$variables`, `$(commands)`, `$((arithmetic))`, and backslash before `$ " \` and backtick | **almost everything that contains a variable** |
| unquoted | all of the above, **then** splits the result on whitespace and expands `* ? [ ]` as filename patterns | only when you want that splitting |
| `$'ansi'` | backslash escapes: `\n`, `\t`, `\'`, `\x41` | strings needing a real tab or newline |

```bash
name="world"
echo 'single: $name'           # single: $name
echo "double: $name"           # double: world
echo "cost: \$5"               # cost: $5
echo $'col1\tcol2\nnext line'  # a real tab and newline
echo "today has $(date +%A | wc -c) characters, and 6 x 7 = $((6 * 7))"
echo 'it'\''s'                 # a single quote inside single quotes: close, escaped quote, reopen
echo "it's"                    # easier
```

### Why unquoted variables are bugs

```bash
file="my report.txt"
touch "$file"
ls -l $file                    # two arguments: "my" and "report.txt". Both fail.
ls -l "$file"                  # one argument

pattern="*"
echo $pattern                  # expands to every filename in the directory
echo "$pattern"                # *
rm "$file"
```

**Rule: put double quotes around every `$variable` and every `$(command)`.** The exceptions are rare and deliberate. This single habit prevents most shell bugs, including the destructive ones.

### Numbers

The shell stores numbers as strings and treats them as integers inside arithmetic contexts.

```bash
echo $((7 * 6))                # 42
echo $((7 / 2))                # 3      integer division
echo $((7 % 2))                # 1      remainder
echo $((2 ** 10))              # 1024
echo $((0x1f))                 # 31     hexadecimal
echo $((010))                  # 8      a leading zero means OCTAL
echo $((10#010))               # 10     force base 10
echo "scale=2; 7 / 2" | bc     # 3.50   bash has no floating point; use bc or awk
awk 'BEGIN {printf "%.2f\n", 7 / 2}'
```

The octal rule bites with dates: `$(( $(date +%m) + 1 ))` fails in August and September, because `08` and `09` are invalid octal. Write `$(( 10#$(date +%m) + 1 ))`.

### Booleans

There is no boolean type. `true` and `false` are commands that exit 0 and 1. Conventions you will see:

```bash
verbose=0                      # an integer flag, tested with (( verbose ))
debug=true                     # a command name, tested with: if $debug; then
enabled="yes"                  # a string, tested with [[ $enabled == yes ]]
```

Inside `(( ))`, zero is false and non-zero is true, the reverse of exit statuses. `(( 0 ))` *exits* with status 1.

### Other literal forms

```bash
echo {a,b,c}.txt               # brace expansion: a.txt b.txt c.txt
echo {1..5} {01..03} {a..e}    # sequences
echo ~ ~root                   # tilde expansion (only unquoted, at the start of a word)
cat <<'EOF'                    # here-document: a multi-line literal (Lab 03)
  $not_expanded
EOF
```

### Exercises 1

Predict the output of each line before running it.

```bash
x=5
echo '$x'
echo "$x"
echo "\$x is $x"
echo $((x * 2))
echo "$((x * 2))"
echo '$((x * 2))'
echo "$(echo "$x")"
echo $x$x
echo "${x}0"
echo "$x0"
echo *
echo "*"
```

Then:

1. Print exactly: `He said "it's $5"` using one `echo`.
2. Store the text `a   b   c` (three spaces between letters) in a variable and print it once with the spaces preserved and once collapsed. Explain the difference.
3. Compute 17 divided by 4 as an integer, as a remainder, and to three decimal places.

<details><summary>Solutions</summary>

```
$x
5
$x is 5
10
10
$((x * 2))
5
55
50
(empty line: the variable is named x0, which is unset)
(every filename in the directory)
*
```

```bash
echo "He said \"it's \$5\""                                    # 1
v="a   b   c"; echo "$v"; echo $v                              # 2: unquoted, the shell splits into three words
                                                               #    and echo joins them with single spaces
echo $((17 / 4)) $((17 % 4)); echo "scale=3; 17 / 4" | bc      # 3
```

</details>

---

## 2. Variables

```bash
name="tower"                   # NO spaces around =.   name = "tower" runs a command called "name"
count=3
echo "$name has $count disks"
echo "${name}_backup"          # braces mark where the name ends
readonly PI=3.14159            # cannot be changed or unset afterwards
unset count
```

Convention: lowercase for your own variables, UPPERCASE for environment variables and constants. That avoids overwriting `PATH`, `USER` or `HOME` by accident.

### Command substitution

```bash
today=$(date +%F)              # capture a command's stdout (trailing newlines are removed)
kernel="$(uname -r)"
files=$(ls | wc -l)
echo "On $today, kernel $kernel, $files files here"
user_count=$(getent passwd | wc -l)
```

Backticks `` `cmd` `` are the old spelling. Use `$( )`: it nests and is readable.

### Special variables

| Variable | Value |
|---|---|
| `$0` | the script's name as invoked |
| `$1` … `$9`, `${10}` | positional arguments |
| `$#` | number of arguments |
| `"$@"` | **all arguments, each as a separate word**. The correct way to pass them on |
| `"$*"` | all arguments joined into one word |
| `$?` | exit status of the last command |
| `$$` | PID of the shell |
| `$!` | PID of the last background command |
| `$_` | last argument of the previous command |
| `$LINENO`, `$FUNCNAME`, `$BASH_SOURCE` | where you are: for error messages |
| `$RANDOM`, `$SECONDS` | a random integer; seconds since the shell started |
| `$IFS` | the characters used for word splitting (space, tab, newline) |

```bash
cat > args.sh <<'EOF'
#!/usr/bin/env bash
echo "script: $0"
echo "count:  $#"
echo "first:  ${1:-<none>}"
echo 'looping over "$@":'; for a in "$@"; do echo "  [$a]"; done
echo 'looping over $* (unquoted):'; for a in $*; do echo "  [$a]"; done
EOF
chmod +x args.sh
./args.sh one "two words" three
```

`"$@"` preserved `two words` as one argument. Unquoted `$*` split it. Always use `"$@"`.

`shift` discards `$1` and moves the rest down, which is how option-parsing loops consume arguments.

### Parameter expansion

String operations without calling another program.

```bash
f="/var/log/app/server.log.gz"
```

| Expansion | Result | Meaning |
|---|---|---|
| `${#f}` | `26` | length |
| `${f##*/}` | `server.log.gz` | remove the **longest** match of `*/` from the front: the basename |
| `${f%/*}` | `/var/log/app` | remove the **shortest** match of `/*` from the end: the dirname |
| `${f%.gz}` | `/var/log/app/server.log` | strip a suffix |
| `${f%%.*}` | `/var/log/app/server` | remove the longest match of `.*` from the end |
| `${f#*.}` | `log.gz` | remove the shortest match of `*.` from the front |
| `${f/log/LOG}` | `/var/LOG/app/server.log.gz` | replace the first match |
| `${f//log/LOG}` | `/var/LOG/app/server.LOG.gz` | replace all matches |
| `${f:0:8}` | `/var/log` | substring: offset, length |
| `${f^^}` / `${f,,}` | upper / lower case | |

Memory aid: `#` is to the left of `$` on the keyboard and removes from the left; `%` is to the right and removes from the right. Doubled means greedy.

Defaults and guards:

| Expansion | Meaning |
|---|---|
| `${var:-default}` | the value, or `default` if unset or empty |
| `${var:=default}` | the same, and also assign it |
| `${var:+alt}` | `alt` if set and non-empty, else nothing |
| `${var:?message}` | the value, or **abort the script** with `message` if unset or empty |

```bash
echo "Deploying to ${DEPLOY_ENV:-staging}"
port="${1:-8080}"
```

`${var:?}` is the guard for anything destructive. Compare (these use `echo`, so nothing is removed):

```bash
DIR=""
echo rm -rf "$DIR"/*                   # would expand to the entire root directory
echo rm -rf "${DIR:?DIR is not set}"/* # refuses, with an error
```

### Arrays

```bash
hosts=(web01 web02 db01)       # create
hosts+=(cache01)               # append
echo "${hosts[0]}"             # first element (indexing starts at 0)
echo "${hosts[-1]}"            # last
echo "${hosts[@]}"             # all elements
echo "${#hosts[@]}"            # how many
echo "${!hosts[@]}"            # the indexes
echo "${hosts[@]:1:2}"         # a slice: 2 elements starting at index 1
for h in "${hosts[@]}"; do echo "host: $h"; done
unset 'hosts[1]'               # remove one (leaves a gap in the indexes)

mapfile -t lines < /etc/hostname           # read a file into an array, one line per element
files=(*.sh)                               # an array from a glob: safe with spaces in names
```

Always `"${array[@]}"`, quoted, with the `@`. It expands to one word per element, whatever the elements contain.

Building a command in an array is the safe way to assemble arguments conditionally:

```bash
opts=(-a --delete)
[[ ${DRY_RUN:-} ]] && opts+=(-n)
echo rsync "${opts[@]}" src/ dest/
```

Associative arrays (maps):

```bash
declare -A port=([ssh]=22 [https]=443)
port[dns]=53
echo "${port[ssh]}"
for name in "${!port[@]}"; do echo "$name -> ${port[$name]}"; done
[[ -v port[smtp] ]] || echo "no smtp entry"
```

### Reading input

```bash
read -r -p "Name: " answer; echo "Hello, $answer"
read -r -s -p "Secret: " secret; echo                 # -s: do not echo what is typed
read -r first rest <<< "alpha beta gamma"; echo "$first | $rest"
IFS=: read -r user _ uid _ <<< "$(getent passwd root)"; echo "$user has uid $uid"
```

Always `read -r`. Without it, backslashes in the input are interpreted.

### Scope

Variables are global to the script unless declared `local` inside a function. A child process gets a copy of **exported** variables and can never change the parent's (Lab 03).

### Exercises 2

1. Write `info.sh` that takes a file path as its argument and prints the directory, the filename, the filename without its last extension, and the extension, using only parameter expansion. Test with `/etc/ssh/sshd_config.rpmnew` and `/tmp/archive.tar.gz`.
2. Make it print a usage message and exit with status 2 when no argument is given.
3. Store the names of all `.sh` files in the current directory in an array. Print how many there are and the last one.
4. Write a line that sets `LOG_LEVEL` to `info` only if it is not already set, then prints it. Run it with and without `LOG_LEVEL=debug` in front.
5. Split the string `2026-10-08` into three variables `y`, `m`, `d` using `read` and `IFS`.
6. Build an associative array mapping three service names to `active` or `inactive`, by asking `systemctl is-active` for each.

<details><summary>Solutions</summary>

```bash
# 1 and 2
cat > info.sh <<'EOF'
#!/usr/bin/env bash
if [[ $# -ne 1 ]]; then echo "usage: ${0##*/} PATH" >&2; exit 2; fi
p=$1
base=${p##*/}
echo "directory: ${p%/*}"
echo "filename:  $base"
echo "stem:      ${base%.*}"
echo "extension: ${base##*.}"
EOF
chmod +x info.sh; ./info.sh /etc/ssh/sshd_config.rpmnew; ./info.sh /tmp/archive.tar.gz; ./info.sh; echo "exit $?"

# 3
scripts=(*.sh); echo "${#scripts[@]} scripts, last is ${scripts[-1]}"

# 4
echo 'echo "${LOG_LEVEL:=info}"' > lvl.sh; bash lvl.sh; LOG_LEVEL=debug bash lvl.sh

# 5
IFS=- read -r y m d <<< "2026-10-08"; echo "$y / $m / $d"

# 6
declare -A state
for s in sshd crond docker; do state[$s]=$(systemctl is-active "$s"); done
for s in "${!state[@]}"; do echo "$s: ${state[$s]}"; done
```

</details>

---

## 3. Conditionals

`if` runs a command and branches on its exit status. Nothing more.

```bash
if grep -q "^$USER:" /etc/passwd; then
  echo "you have a local account"
elif getent passwd "$USER" > /dev/null; then
  echo "you exist in a directory service"
else
  echo "who are you?"
fi
```

Any command can be the condition: `if systemctl is-active --quiet docker; then …`, `if ping -c1 -W1 host >/dev/null; then …`, `if command -v jq >/dev/null; then …`.

### Test commands

To compare strings, numbers and files you need a command that exits 0 or 1 accordingly.

| Form | What it is | Use |
|---|---|---|
| `[[ … ]]` | bash keyword | **the default in bash scripts**: no word splitting inside, pattern and regex matching, `&&` and `\|\|` |
| `[ … ]` | the `test` command (POSIX) | scripts that must run under `sh`. Every variable inside must be quoted |
| `(( … ))` | arithmetic | integer comparisons with normal operators |

**Strings**

| Test | True when |
|---|---|
| `[[ $a == "$b" ]]` | equal |
| `[[ $a != "$b" ]]` | not equal |
| `[[ -z $a ]]` | empty |
| `[[ -n $a ]]` | not empty |
| `[[ $a == web* ]]` | matches a glob pattern (right side **unquoted**) |
| `[[ $a =~ ^web[0-9]+$ ]]` | matches a regex (unquoted); captures land in `${BASH_REMATCH[1]}` |
| `[[ $a < $b ]]` | sorts before, as text |

**Integers**

| `[[ ]]` / `[ ]` | `(( ))` | Meaning |
|---|---|---|
| `-eq` `-ne` | `==` `!=` | equal, not equal |
| `-lt` `-le` | `<` `<=` | less than, or equal |
| `-gt` `-ge` | `>` `>=` | greater than, or equal |

**Files**

| Test | True when the path… |
|---|---|
| `-e` | exists |
| `-f` | is a regular file |
| `-d` | is a directory |
| `-L` | is a symlink |
| `-r` `-w` `-x` | is readable, writable, executable by you |
| `-s` | exists and is not empty |
| `a -nt b` / `a -ot b` | a is newer / older than b |

**Combining**

```bash
[[ -f $f && -r $f ]]           # and
[[ $x == a || $x == b ]]       # or
[[ ! -d $dir ]]                # not
```

```bash
n=10
[[ $n -gt 9 ]]  && echo "numeric comparison: 10 is greater than 9"
(( n > 9 ))     && echo "the same, in arithmetic form"
[[ $n > 9 ]]    || echo "string comparison: the text 10 sorts before the text 9"

host=web01
[[ $host == web* ]] && echo "a web host"
if [[ $host =~ ^([a-z]+)([0-9]+)$ ]]; then echo "role=${BASH_REMATCH[1]} number=${BASH_REMATCH[2]}"; fi

[[ -d /etc && -r /etc/hostname ]] && echo "readable"
[[ -s /etc/hostname ]] && echo "not empty"
```

The classic errors, all of which run without a syntax error:

```bash
x="two words"
[ $x = "two words" ]                       # error: too many arguments. The unquoted variable became two words.
[[ $x = "two words" ]] && echo "fine: no word splitting inside [[ ]]"
[ 5 > 3 ] && ls -l 3; rm -f 3              # ">" is a redirect inside [ ]: it created a file called "3"
n=010
[[ $n = 10 ]] || echo "= compares text: 010 is not 10. Use -eq for numbers."
n=""
[ $n -gt 5 ]                               # error: unary operator expected. n is empty and unquoted, so it vanished.
```

### Short-circuit forms

```bash
mkdir -p out && cd out && cd ..           # each step runs only if the previous one succeeded
command -v jq > /dev/null || echo "jq is not installed"
[[ -f config.env ]] && source config.env
```

`a && b || c` is **not** if-then-else: `c` also runs when `a` succeeds and `b` fails. Use a real `if` whenever there are two branches.

### case

For matching one value against several patterns, `case` is clearer than a chain of `elif`.

```bash
cat > kind.sh <<'EOF'
#!/usr/bin/env bash
case "${1:-}" in
  *.tar.gz|*.tgz) echo "gzip tarball" ;;
  *.tar.xz)       echo "xz tarball" ;;
  *.zip)          echo "zip archive" ;;
  start|stop|restart)
                  echo "a service action: $1" ;;
  [0-9]*)         echo "starts with a digit" ;;
  "")             echo "usage: $0 NAME" >&2; exit 2 ;;
  *)              echo "no idea" ;;
esac
EOF
chmod +x kind.sh
./kind.sh backup.tgz; ./kind.sh restart; ./kind.sh 42abc; ./kind.sh what; ./kind.sh
```

Patterns are globs, `|` separates alternatives, `;;` ends a branch, `*)` is the default.

### Exit codes of your own

```bash
exit 0      # success
exit 1      # general failure
exit 2      # by convention: wrong usage
```

A script's exit status is that of its last command unless you `exit N`. Callers (`if`, `&&`, CI systems, systemd, cron) act on it, so make it truthful.

### Exercises 3

1. Write `filetype.sh PATH` that prints whether the path is a directory, a symlink, an executable file, an empty file, a regular file, or missing. Check symlink before the others. Why?
2. Write `grade.sh N` that prints `invalid` unless N is an integer from 0 to 100, otherwise `fail` (<50), `pass` (50–79) or `distinction` (80+). Validate with a regex.
3. Write `svc.sh NAME` that prints `running`, `stopped but enabled`, `stopped and disabled`, or `no such unit`, using only the exit statuses of `systemctl` commands.
4. Write a `case` that classifies the current hour (`date +%H`) as night (00–05), morning, afternoon or evening. Mind the octal trap.
5. Explain why `[[ $a == $b ]]` and `[[ $a == "$b" ]]` can give different results. Demonstrate with `b='*'`.

<details><summary>Solutions</summary>

```bash
# 1
cat > filetype.sh <<'EOF'
#!/usr/bin/env bash
p=${1:?usage: filetype.sh PATH}
if   [[ -L $p ]]; then echo "symlink -> $(readlink "$p")"
elif [[ ! -e $p ]]; then echo "missing"
elif [[ -d $p ]]; then echo "directory"
elif [[ ! -s $p ]]; then echo "empty file"
elif [[ -x $p ]]; then echo "executable file"
else echo "regular file"
fi
EOF
# -L first: every other test follows symlinks, so a link to a directory would report "directory",
# and a dangling link would report "missing".

# 2
cat > grade.sh <<'EOF'
#!/usr/bin/env bash
n=${1:-}
if [[ ! $n =~ ^[0-9]+$ ]] || (( 10#$n > 100 )); then echo invalid; exit 2; fi
if   (( 10#$n < 50 )); then echo fail
elif (( 10#$n < 80 )); then echo pass
else echo distinction
fi
EOF

# 3
cat > svc.sh <<'EOF'
#!/usr/bin/env bash
u=${1:?usage: svc.sh UNIT}
if ! systemctl cat "$u" > /dev/null 2>&1; then echo "no such unit"; exit 1; fi
if   systemctl is-active  --quiet "$u"; then echo "running"
elif systemctl is-enabled --quiet "$u"; then echo "stopped but enabled"
else echo "stopped and disabled"
fi
EOF

# 4
h=$((10#$(date +%H)))
case $h in
  [0-5])        echo night ;;
  [6-9]|1[01])  echo morning ;;
  1[2-7])       echo afternoon ;;
  *)            echo evening ;;
esac

# 5
a=hello; b='*'
[[ $a == $b ]]   && echo "unquoted: b is a PATTERN, and * matches anything"
[[ $a == "$b" ]] || echo "quoted: b is a literal string, and hello is not *"
```

</details>

---

## 4. Loops

### for

```bash
for h in web01 web02 db01; do echo "host $h"; done            # a literal list
for f in /etc/*.conf; do echo "${f##*/}"; done | head -5      # a glob: the right way to loop over files
for i in {1..5}; do printf '%d ' "$i"; done; echo             # a sequence
for i in $(seq 1 2 9); do printf '%d ' "$i"; done; echo       # with a step, or with variables as bounds
for (( i = 0; i < 5; i++ )); do printf '%d ' "$i"; done; echo # C style
for arg in "$@"; do echo "$arg"; done                         # the script's arguments
for h in "${hosts[@]}"; do echo "$h"; done                    # an array
```

**Never loop over `$(ls)`.** It splits filenames on spaces and expands any glob characters in them. A glob gives you each name intact:

```bash
touch "a file.txt" b.txt
for f in $(ls *.txt); do echo "[$f]"; done     # [a] [file.txt] [b.txt]   wrong
for f in *.txt;       do echo "[$f]"; done     # [a file.txt] [b.txt]     right
rm "a file.txt" b.txt
```

When a glob matches nothing, the loop runs once with the literal pattern. Guard with `[[ -e $f ]] || continue`, or set `shopt -s nullglob`.

### while and until

```bash
n=3
while (( n > 0 )); do echo "countdown $n"; n=$((n - 1)); done

until ping -c1 -W1 127.0.0.1 > /dev/null 2>&1; do echo "waiting..."; sleep 1; done; echo "reachable"

while true; do date +%T; sleep 1; done          # an endless loop: Ctrl-C to stop
```

`while COMMAND` loops as long as the command succeeds. `until` is the opposite. "Wait until a port answers" is a pattern you will write in entrypoints and deploy scripts constantly; give it a limit:

```bash
tries=0
until nc -z 127.0.0.1 22 2>/dev/null; do
  tries=$((tries + 1))
  if (( tries >= 10 )); then echo "gave up after $tries attempts" >&2; break; fi
  sleep 1
done
```

### Reading lines

The correct idiom for processing a file line by line:

```bash
while IFS= read -r line; do
  echo "line: $line"
done < /etc/hostname
```

`IFS=` keeps leading and trailing whitespace; `-r` keeps backslashes. Split into fields by setting `IFS` and naming several variables:

```bash
while IFS=: read -r user _ uid _ _ home shell; do
  (( uid >= 1000 && uid < 60000 )) && echo "$user ($uid) $home $shell"
done < /etc/passwd
```

Reading from a command: pipe it in, **or** use process substitution. The difference matters.

```bash
count=0
printf '1\n2\n3\n' | while read -r l; do count=$((count + 1)); done
echo "after a pipe: $count"                 # 0: the loop ran in a subshell (Lab 07), its variable is gone

count=0
while read -r l; do count=$((count + 1)); done < <(printf '1\n2\n3\n')
echo "after process substitution: $count"   # 3: the loop ran in this shell
```

A command inside a `while read` loop that itself reads stdin (most often `ssh`) swallows the rest of your input, and the loop ends after one pass. Give it `< /dev/null`, or for ssh use `-n`.

### break and continue

```bash
for f in /etc/*; do
  [[ -f $f ]] || continue                   # skip anything that is not a regular file
  [[ $f == /etc/h* ]] || continue
  echo "$f"
  [[ $f == /etc/hosts ]] && break           # stop the loop entirely
done
```

### Loops and pipelines

A loop is a command: you can redirect its output or pipe it as a whole.

```bash
for u in root "$USER" nobody; do
  printf '%s\t%s\n' "$u" "$(id -u "$u")"
done | sort -k2 -n | column -t

for i in {1..3}; do echo "entry $i"; done > entries.txt
```

Parallel work with background jobs and `wait`:

```bash
for h in 127.0.0.1 127.0.0.2 127.0.0.3; do
  ( ping -c1 -W1 "$h" > /dev/null 2>&1 && echo "$h up" || echo "$h down" ) &
done
wait
```

For anything beyond a handful, `xargs -P 8` does bounded parallelism.

Prefer a pipeline to a loop when a tool can process the whole stream: `awk` over 100,000 lines takes milliseconds, while a `while read` loop running `cut` on each line takes minutes.

### Exercises 4

1. Print the 7 times table from 1 to 12, aligned in columns with `printf`.
2. For every regular file in `/etc` ending in `.conf`, print its line count and name, sorted by line count, largest five only.
3. Read `/etc/passwd` and print the accounts whose shell is not `nologin` or `false`, as `user -> shell`.
4. Write `waitfor.sh HOST PORT [TIMEOUT]` that waits until a TCP port accepts connections, printing a dot per second, and exits 0 on success or 1 on timeout (default 30 seconds).
5. Create 20 files `data-01.txt` … `data-20.txt`, then rename them in a loop to `2026-data-01.txt` and so on, using `echo` first to preview.
6. Sum the sizes in bytes of all files in `/var/log` you can read, with a loop. Then do it with one `find … -printf` piped to `awk`. Compare how long each takes with `time`.

<details><summary>Solutions</summary>

```bash
# 1
for i in {1..12}; do printf '%2d x 7 = %3d\n' "$i" $((i * 7)); done

# 2
for f in /etc/*.conf; do [[ -f $f && -r $f ]] && printf '%d %s\n' "$(wc -l < "$f")" "$f"; done | sort -rn | head -5

# 3
while IFS=: read -r user _ _ _ _ _ shell; do
  [[ $shell == */nologin || $shell == */false ]] || echo "$user -> $shell"
done < /etc/passwd

# 4
cat > waitfor.sh <<'EOF'
#!/usr/bin/env bash
host=${1:?usage: waitfor.sh HOST PORT [TIMEOUT]}; port=${2:?missing PORT}; timeout=${3:-30}
for (( i = 0; i < timeout; i++ )); do
  if nc -z "$host" "$port" 2>/dev/null; then echo " $host:$port is up"; exit 0; fi
  printf .; sleep 1
done
echo " timed out after ${timeout}s" >&2; exit 1
EOF
chmod +x waitfor.sh; ./waitfor.sh 127.0.0.1 22 5; ./waitfor.sh 127.0.0.1 1 3; echo "exit $?"

# 5
touch data-{01..20}.txt
for f in data-*.txt; do echo mv -- "$f" "2026-$f"; done      # preview
for f in data-*.txt; do mv -- "$f" "2026-$f"; done
rm 2026-data-*.txt

# 6
time ( t=0; for f in /var/log/*; do [[ -f $f && -r $f ]] && t=$(( t + $(stat -c %s "$f") )); done; echo "$t" )
time ( find /var/log -maxdepth 1 -type f -readable -printf '%s\n' | awk '{s += $1} END {print s}' )
```

</details>

---

## 5. Functions

Functions are not a roadmap node, but no real script is written without them.

```bash
greet() {
  local who=${1:-world}        # arguments are $1, $2 ... just like a script's
  echo "hello, $who"
}
greet; greet tower

add() { echo $(( $1 + $2 )); } # "return" a value by printing it...
sum=$(add 2 3); echo "sum=$sum"  # ...and capture it with $( )

is_listening() { ss -tln | grep -q ":$1 "; }     # return true or false through the exit status
if is_listening 22; then echo "sshd is listening"; fi
```

- `return N` sets the function's **exit status** (0–255). It does not return data. Data comes back on stdout.
- Declare every variable inside a function `local`. Without it, the variable is global and silently overwrites the caller's.
- Define functions before the line that calls them.

```bash
x=global
f() { x=changed; local y=inside; }
f; echo "$x ${y:-unset}"       # changed unset
```

Three functions worth having at the top of every script:

```bash
log()  { printf '%s %s\n' "$(date +%T)" "$*" >&2; }
die()  { printf 'error: %s\n' "$*" >&2; exit 1; }
need() { command -v "$1" > /dev/null || die "required command not found: $1"; }
```

Diagnostics go to **stderr** (`>&2`), so that a script's real output can be piped or captured cleanly.

A retry helper shows `"$@"` being used to run a command passed as arguments:

```bash
retry() {
  local attempts=$1 i; shift
  for (( i = 1; i <= attempts; i++ )); do
    "$@" && return 0
    echo "attempt $i of $attempts failed: $*" >&2
    sleep 1
  done
  return 1
}
retry 3 false; echo "exit $?"
retry 3 curl -fsS -o /dev/null https://example.com && echo "reachable"
```

---

## 6. Debugging

### Check syntax without running

```bash
bash -n hello.sh && echo "syntax ok"
```

It catches unclosed quotes, missing `fi` or `done`. It does not catch logic errors, and most shell bugs are valid syntax.

### Trace execution

```bash
bash -x args.sh one "two words"
```

Each command is printed, **after expansion**, prefixed with `+`, before it runs. You see what the shell is really executing, with the quoting made visible. This is the most useful single debugging tool in shell.

Trace only part of a script, and add file and line numbers through `PS4`:

```bash
cat > trace.sh <<'EOF'
#!/usr/bin/env bash
PS4='+ ${BASH_SOURCE##*/}:${LINENO}: '
name="a b"
set -x                         # tracing on
[ -n "$name" ] && echo "$name"
set +x                         # tracing off
echo "not traced"
EOF
bash trace.sh
```

`bash -v script` prints each line as it is read, before expansion. Other quick tools: `echo "DEBUG: x=[$x]" >&2` with brackets so that empty values and stray spaces show, and `declare -p var` to dump any variable, arrays included, with its exact contents.

### Strict mode

Bash's defaults are forgiving in dangerous ways: a failed command is ignored and the script carries on; a misspelt variable is silently empty. Three options change that. Put this at the top of every script:

```bash
set -euo pipefail
```

| Option | Effect |
|---|---|
| `-e` (errexit) | exit as soon as a command fails |
| `-u` (nounset) | treat an unset variable as an error |
| `-o pipefail` | a pipeline fails if **any** stage fails, not only the last |

See each one work:

```bash
printf 'set -e\necho one\nfalse\necho two\n' > e.sh; bash e.sh; echo "exit $?"        # stops after "one"

printf 'set -u\ntarget=/tmp/x\necho "removing $taget/*"\n' > u.sh; bash u.sh           # taget: unbound variable

bash -c 'false | true; echo "without pipefail: $?"'
bash -c 'set -o pipefail; false | true; echo "with pipefail: $?"'
```

The `-u` example is the bug that deletes things: a typo in a variable name inside an `rm` path.

`set -e` has exceptions you must know. A failing command does **not** stop the script when it is:

- the condition of `if`, `while` or `until`
- anywhere but last in an `&&` / `||` list
- negated with `!`
- inside a function that is itself being called in one of those positions

```bash
cat > e2.sh <<'EOF'
set -e
if false; then :; fi                 # fine: a test
false || echo "handled"              # fine: part of a || list
f() { false; echo "f carried on after false"; }
if f; then echo "f succeeded"; fi    # errexit is switched off inside f because f is being tested
EOF
bash e2.sh
```

So `set -e` is a safety net with holes. Check explicitly what matters: `cd "$dir" || die "cannot cd to $dir"`.

To let a command fail on purpose under strict mode: `grep -q pattern file || true`, or capture the status: `if ! output=$(command); then …`.

Under `-u`, optional arguments need a default: `${1:-}` in place of `$1`.

### Traps

`trap` runs code when the script receives a signal or reaches certain events.

```bash
cat > t.sh <<'EOF'
#!/usr/bin/env bash
set -Eeuo pipefail
trap 'echo "error: line $LINENO: \"$BASH_COMMAND\" exited with status $?" >&2' ERR
tmp=$(mktemp)
trap 'rm -f "$tmp"; echo "cleaned up $tmp" >&2' EXIT

echo "working in $tmp"
ls /nonexistent-directory
echo "never reached"
EOF
bash t.sh; echo "exit $?"
```

| Trap | Fires |
|---|---|
| `EXIT` | whenever the script exits, for any reason. **The place for cleanup** |
| `ERR` | when a command fails (same exceptions as `set -e`). `-E` makes functions inherit it |
| `INT`, `TERM`, `HUP` | on those signals (Lab 07) |
| `DEBUG` | before every command |

`mktemp` plus an `EXIT` trap is the standard pattern for temporary files: a unique, private file that is removed however the script ends.

### ShellCheck

ShellCheck is a static analyser that knows the traps in this lab and several hundred more. Run it on everything you write. It is in EPEL as `ShellCheck`; since you have Docker, no install is needed:

```bash
sc() { docker run --rm -v "$PWD:/mnt:ro,Z" koalaman/shellcheck:stable "$@"; }
sc args.sh
```

(The `Z` relabels the mount for SELinux so that the container may read it. Use it only on directories of your own like this one, never on system directories.)

Each finding has a code (`SC2086: Double quote to prevent globbing and word splitting`) with an explanation at `https://www.shellcheck.net/wiki/SC2086`. Add it to your editor and to CI, and treat its warnings as errors.

### Exercise 6: debug this script

It is meant to back up a directory into a dated tarball. It has five bugs and no syntax errors.

```bash
cat > buggy.sh <<'EOF'
#!/usr/bin/env bash
# Backs up a directory into a dated tarball. It has five bugs.

src_dir=$1
backup_dir="$HOME/labwork/14/backups"
stamp=$(date +%F)

if [ $# = 0 ]
then
  echo "usage: $0 DIRECTORY"
fi

mkdir -p $backup_dir
cd $src_dir
count=`ls | wc -l`

if [ $count > 0 ]; then
  tar -czf $backup_dir/backup-$stamp.tar.gz .
  echo "backed up $count items from $src_dir to $backupdir"
fi
EOF
mkdir -p "dir with space" && touch "dir with space/a" "dir with space/b"
```

1. Run `bash -n buggy.sh`. What does it tell you?
2. Run `bash buggy.sh "dir with space"`. List everything wrong with the result. Look at what it backed up, and at the files in your current directory afterwards.
3. Run it with `bash -x` and find the line where it first goes wrong.
4. Run ShellCheck on it. Which bugs does it find, and which does it miss?
5. Add `set -euo pipefail` at the top and run it again, with and without an argument. Which bugs now stop the script?
6. Fix all five.

<details><summary>Solution</summary>

1. Nothing: the syntax is valid.
2. `cd: too many arguments`, then it carried on **in the wrong directory**, counted and archived your whole lab directory, created a stray file named `0`, and printed an empty destination.

| # | Bug | Consequence | Fix |
|---|---|---|---|
| 1 | no `exit` after the usage message | carries on with an empty `src_dir`; `cd` with no argument goes to `$HOME` and would archive your home directory | `exit 2` |
| 2 | unquoted `$src_dir`, `$backup_dir` | a path with a space becomes several arguments | quote every expansion |
| 3 | `cd` not checked | when `cd` fails, everything after it runs in the wrong place | `cd "$src_dir" \|\| exit 1` |
| 4 | `[ $count > 0 ]` | `>` is a redirect: creates a file `0`, and the test is always true | `(( count > 0 ))` or `-gt` |
| 5 | `$backupdir` (typo) | empty string in the message | `$backup_dir`; `set -u` catches it |

ShellCheck reports the quoting, the unchecked `cd`, the backticks, the `>` in `[ ]`, and the unassigned `backupdir`. It cannot know that you meant to exit after the usage message.

With `set -euo pipefail`: no argument stops at `src_dir=$1` (unbound variable); the failing `cd` stops the script before it archives the wrong directory; the typo stops at the last line. Strict mode turned three silent misbehaviours into immediate errors.

```bash
cat > fixed.sh <<'EOF'
#!/usr/bin/env bash
# Backs up a directory into a dated tarball.
set -euo pipefail

if (( $# != 1 )); then
  echo "usage: ${0##*/} DIRECTORY" >&2
  exit 2
fi

src_dir=$1
backup_dir="$HOME/labwork/14/backups"
stamp=$(date +%F)

[[ -d $src_dir ]] || { echo "error: not a directory: $src_dir" >&2; exit 1; }

mkdir -p "$backup_dir"
items=("$src_dir"/*)
if [[ -e ${items[0]} ]]; then
  tar -czf "$backup_dir/backup-$stamp.tar.gz" -C "$src_dir" .
  echo "backed up ${#items[@]} items from $src_dir to $backup_dir"
else
  echo "nothing to back up in $src_dir"
fi
EOF
rm -f 0; rm -rf backups
bash fixed.sh "dir with space"; tar -tzf backups/backup-*.tar.gz
bash fixed.sh; echo "exit $?"
```

The fixed version also stops changing directory at all (`tar -C` does it), which removes a whole class of error.

</details>

---

## 7. Putting it together: a real tool

`logsum` summarises web access logs in the format from Lab 05. Read it top to bottom; every construct in it has appeared above.

```bash
cat > logsum <<'EOF'
#!/usr/bin/env bash
#
# logsum - summarise web access logs in common log format
#
set -euo pipefail

readonly PROG="${0##*/}"
top=5
min_status=400
out=""
verbose=0

usage() {
  cat <<USAGE
Usage: $PROG [-n TOP] [-s MIN_STATUS] [-o FILE] [-v] LOGFILE...

  -n TOP         list the TOP busiest clients (default: $top)
  -s MIN_STATUS  treat statuses >= MIN_STATUS as errors (default: $min_status)
  -o FILE        write the report to FILE instead of the terminal
  -v             verbose: progress messages on stderr
  -h             show this help
USAGE
}

log() { if (( verbose )); then printf '%s: %s\n' "$PROG" "$*" >&2; fi; }
die() { printf '%s: error: %s\n' "$PROG" "$*" >&2; exit 1; }
is_number() { [[ $1 =~ ^[0-9]+$ ]]; }

while getopts ':n:s:o:vh' opt; do
  case "$opt" in
    n) is_number "$OPTARG" || die "-n needs a number, got '$OPTARG'"; top=$OPTARG ;;
    s) is_number "$OPTARG" || die "-s needs a number, got '$OPTARG'"; min_status=$OPTARG ;;
    o) out=$OPTARG ;;
    v) verbose=1 ;;
    h) usage; exit 0 ;;
    :) die "option -$OPTARG needs an argument" ;;
    \?) die "unknown option -$OPTARG (try -h)" ;;
  esac
done
shift $((OPTIND - 1))

if (( $# == 0 )); then usage >&2; exit 2; fi

tmp=$(mktemp)
trap 'rm -f "$tmp"' EXIT

files=0
for f in "$@"; do
  [[ -f $f && -r $f ]] || die "cannot read '$f'"
  log "reading $f"
  case "$f" in
    *.gz) zcat -- "$f" >> "$tmp" ;;
    *)    cat  -- "$f" >> "$tmp" ;;
  esac
  files=$((files + 1))
done

total=$(wc -l < "$tmp")
(( total > 0 )) || die "no log lines found"

report() {
  local errors bytes pct
  errors=$(awk -v m="$min_status" '$9 >= m {n++} END {print n + 0}' "$tmp")
  bytes=$(awk '{s += $10} END {print s + 0}' "$tmp")
  pct=$(( errors * 100 / total ))

  printf 'Files read:      %d\n' "$files"
  printf 'Requests:        %d\n' "$total"
  printf 'Errors (>=%d):  %d (%d%%)\n' "$min_status" "$errors" "$pct"
  printf 'Bytes sent:      %d\n' "$bytes"
  printf '\nTop %d clients:\n' "$top"
  awk '{c[$1]++} END {for (ip in c) printf "%6d  %s\n", c[ip], ip}' "$tmp" | sort -k1,1nr -k2,2 | head -n "$top"
  printf '\nStatus codes:\n'
  awk '{c[$9]++} END {for (s in c) printf "%6d  %s\n", c[s], s}' "$tmp" | sort -k2,2n
}

if [[ -n $out ]]; then
  report > "$out"
  log "report written to $out"
else
  report
fi
EOF
chmod +x logsum
```

What to notice:

| Lines | Technique |
|---|---|
| `set -euo pipefail` | strict mode |
| `readonly PROG="${0##*/}"` | the script's own name, for messages, without its path |
| `usage`, `log`, `die` | small functions; all diagnostics on stderr |
| `getopts ':n:s:o:vh'` | option parsing. A letter followed by `:` takes an argument (in `$OPTARG`). The leading `:` lets you handle errors yourself: `:` for a missing argument, `?` for an unknown option |
| `shift $((OPTIND - 1))` | drop the parsed options, leaving the file arguments in `"$@"` |
| exit 2 for wrong usage, 1 for runtime errors | meaningful exit codes |
| `mktemp` and `trap … EXIT` | a private temp file, always removed |
| `-- "$f"` | a filename starting with `-` is not mistaken for an option |
| `case "$f" in *.gz)` | transparently handle rotated, compressed logs |
| `local` in `report` | no leaked variables |
| `report > "$out"` | a function's output redirected as a unit |

Recreate the sample log from Lab 05 (the `cat > access.log <<'EOF'` block in its Setup section) in this directory, then:

```bash
./logsum access.log
```

```
Files read:      1
Requests:        20
Errors (>=400):  8 (40%)
Bytes sent:      214883

Top 5 clients:
     5  10.0.1.15
     4  10.0.1.22
     4  198.51.100.7
     4  203.0.113.9
     3  10.0.1.31

Status codes:
    10  200
     1  204
     1  304
     2  401
     1  403
     4  404
     1  500
```

```bash
./logsum -n 2 -s 500 -v access.log access.log
./logsum; echo "exit $?"                       # usage, exit 2
./logsum -n abc access.log; echo "exit $?"     # validation, exit 1
./logsum -x access.log; echo "exit $?"
./logsum nosuch.log; echo "exit $?"
gzip -k access.log && ./logsum access.log.gz | head -3
./logsum -o report.txt -v access.log && cat report.txt
sc logsum                                      # ShellCheck should have nothing to say
```

### Exercises 7

Extend `logsum`. After each change, run ShellCheck and re-test the error cases.

1. Add `-p`, which adds a "Top paths" section (field 7) alongside the top clients.
2. Add `-e`, which makes the script exit with status 3 when the error percentage exceeds 25, so that it can be used as a check in a pipeline or a timer.
3. Accept `-` as a filename meaning standard input, so that `zcat old.log.gz | ./logsum -` works.
4. Add a `--help` long option. (`getopts` does not support long options. Handle it before the `getopts` loop.)
5. Install it: `sudo install -m 755 logsum /usr/local/bin/logsum`, and run it from another directory.

<details><summary>Sketches</summary>

```bash
# 1  add "p" to the getopts string, paths=0 with the defaults, "p) paths=1 ;;" in the case, and in report():
if (( paths )); then
  printf '\nTop %d paths:\n' "$top"
  awk '{c[$7]++} END {for (p in c) printf "%6d  %s\n", c[p], p}' "$tmp" | sort -k1,1nr -k2,2 | head -n "$top"
fi

# 2  add "e" and strict=0; compute the percentage outside report(), and at the very end:
if (( strict && errors * 100 / total > 25 )); then exit 3; fi

# 3  in the file loop, before the readability test:
if [[ $f == - ]]; then cat >> "$tmp"; files=$((files + 1)); continue; fi

# 4  before the getopts loop:
if [[ ${1:-} == --help ]]; then usage; exit 0; fi
```

</details>

---

## When not to use shell

Shell is right for gluing commands together: run this, check that, move those. Move to Python (or Go) when you find yourself needing data structures beyond a flat array, real arithmetic, JSON or YAML handling beyond a `jq` one-liner, careful error handling across many steps, or tests. A rough signal is a script passing a hundred lines with more logic than commands. For configuring machines, an Ansible module is nearly always better than a shell script, because the module is idempotent and the script is not.

## Check yourself

<details><summary>What is the difference between <code>"$@"</code> and <code>$*</code>?</summary>
<code>"$@"</code> expands to each argument as its own word, preserving spaces inside arguments. Unquoted <code>$*</code> joins and then re-splits everything on whitespace, breaking arguments that contain spaces.
</details>

<details><summary>Why quote <code>"$var"</code>?</summary>
Unquoted, the value is split into words on whitespace and each word is expanded as a filename pattern. A value with a space becomes several arguments, an empty value disappears, and a <code>*</code> becomes a file list.
</details>

<details><summary>What do <code>${f##*/}</code> and <code>${f%.*}</code> do?</summary>
The first removes everything up to the last slash, leaving the filename. The second removes the last dot and what follows, stripping one extension.
</details>

<details><summary>Why does <code>[ $n > 5 ]</code> not compare numbers?</summary>
Inside <code>[ ]</code>, <code>&gt;</code> is output redirection. It creates a file named <code>5</code> and tests whether <code>$n</code> is a non-empty string. Use <code>-gt</code>, or <code>(( n &gt; 5 ))</code>.
</details>

<details><summary>What does <code>set -euo pipefail</code> do, and name one case where <code>-e</code> does not stop the script.</summary>
Exit on any failing command, treat unset variables as errors, and make a pipeline fail if any stage fails. <code>-e</code> does not act on a command used as an <code>if</code> or <code>while</code> condition, or one followed by <code>&amp;&amp;</code> or <code>||</code>.
</details>

<details><summary>A counter incremented inside <code>cmd | while read …</code> is zero after the loop. Why?</summary>
Each stage of a pipeline runs in a subshell. The loop changed a copy of the variable. Feed the loop with <code>done &lt; &lt;(cmd)</code> so that it runs in the current shell.
</details>

<details><summary>How should a script create and clean up a temporary file?</summary>
<code>tmp=$(mktemp)</code> followed by <code>trap 'rm -f "$tmp"' EXIT</code>. The name is unique and unpredictable, and the trap removes it however the script ends.
</details>

<details><summary>How does a function return a string? A success or failure?</summary>
A string: print it and let the caller capture it with <code>$( )</code>. Success or failure: through its exit status, with <code>return 0</code> or <code>return 1</code>.
</details>

<details><summary>Which option shows you each command after expansion, as it runs?</summary>
<code>bash -x script</code>, or <code>set -x</code> inside it.
</details>

## Cleanup

```bash
rm -rf ~/labwork/14
```

Keep `/usr/local/bin/logsum` if you installed it, and keep the `sc` function in your `~/.bashrc`.
