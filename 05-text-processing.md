# Lab 05 — Text Processing

**Roadmap topics:** stdout / stdin / stderr · pipe · tee · head · tail · wc · nl · cut · paste · sort · uniq · tr · expand · unexpand · split · join · grep · awk
**Time:** about 4 hours. This is the largest section of the roadmap; split it over two sittings (sections 1–6, then 7–9).
**Touches system state:** no. Everything happens in `~/labwork/05`.

## What you will be able to do

- Answer questions about a log file ("which IPs, how many errors, how many bytes") with a one-line pipeline.
- Pick the right small tool for each step and know when to reach for `awk`.
- Read other people's pipelines.

The reason this section is so big: on Linux, configuration is text, logs are text, and the output of nearly every command is text. Each tool below does one small job. The skill is combining them.

## Setup

```bash
mkdir -p ~/labwork/05 && cd ~/labwork/05

cat > access.log <<'EOF'
10.0.1.15 - - [08/Oct/2026:09:00:01 -0600] "GET /index.html HTTP/1.1" 200 5120
10.0.1.22 - - [08/Oct/2026:09:00:03 -0600] "GET /api/ports HTTP/1.1" 200 18230
203.0.113.9 - - [08/Oct/2026:09:00:04 -0600] "GET /wp-login.php HTTP/1.1" 404 312
10.0.1.15 - - [08/Oct/2026:09:00:07 -0600] "GET /css/main.css HTTP/1.1" 200 2048
198.51.100.7 - - [08/Oct/2026:09:00:09 -0600] "POST /api/login HTTP/1.1" 401 96
10.0.1.22 - - [08/Oct/2026:09:00:12 -0600] "GET /api/routes HTTP/1.1" 200 40112
203.0.113.9 - - [08/Oct/2026:09:00:13 -0600] "GET /.env HTTP/1.1" 404 312
198.51.100.7 - - [08/Oct/2026:09:00:15 -0600] "POST /api/login HTTP/1.1" 401 96
10.0.1.15 - - [08/Oct/2026:09:00:18 -0600] "GET /api/ports HTTP/1.1" 200 18230
203.0.113.9 - - [08/Oct/2026:09:00:19 -0600] "GET /admin HTTP/1.1" 403 150
10.0.1.31 - - [08/Oct/2026:09:00:21 -0600] "GET /index.html HTTP/1.1" 304 0
198.51.100.7 - - [08/Oct/2026:09:00:22 -0600] "POST /api/login HTTP/1.1" 200 512
10.0.1.22 - - [08/Oct/2026:09:00:25 -0600] "GET /api/ports HTTP/1.1" 500 87
10.0.1.15 - - [08/Oct/2026:09:00:28 -0600] "GET /js/app.js HTTP/1.1" 200 30500
203.0.113.9 - - [08/Oct/2026:09:00:30 -0600] "GET /phpmyadmin HTTP/1.1" 404 312
10.0.1.31 - - [08/Oct/2026:09:00:33 -0600] "GET /api/routes HTTP/1.1" 200 40112
10.0.1.22 - - [08/Oct/2026:09:00:36 -0600] "GET /api/ports HTTP/1.1" 200 18230
198.51.100.7 - - [08/Oct/2026:09:00:40 -0600] "GET /api/routes HTTP/1.1" 200 40112
10.0.1.15 - - [08/Oct/2026:09:00:44 -0600] "GET /favicon.ico HTTP/1.1" 404 312
10.0.1.31 - - [08/Oct/2026:09:00:47 -0600] "DELETE /api/routes/17 HTTP/1.1" 204 0
EOF

cat > inventory.csv <<'EOF'
host,role,env,cpu,mem_gb
web01,web,prod,4,8
web02,web,prod,4,8
web03,web,stage,2,4
db01,db,prod,16,64
db02,db,stage,8,32
cache01,cache,prod,2,16
lb01,lb,prod,2,4
ci01,build,dev,8,16
EOF

cat > owners.csv <<'EOF'
host,owner
cache01,platform
ci01,devex
db01,data
db02,data
lb01,platform
web01,frontend
web02,frontend
mon01,platform
EOF
```

`access.log` is in the standard web-server "common log format". Its space-separated fields:

| Field | 1 | 2–3 | 4–5 | 6 | 7 | 8 | 9 | 10 |
|---|---|---|---|---|---|---|---|---|
| Content | client IP | `-` `-` | `[timestamp` `zone]` | `"METHOD` | path | `PROTO"` | status | bytes |

---

## 1. stdout, stdin, stderr, pipe, tee

Lab 03 covered the three streams and redirecting them to files. A **pipe** connects one command's stdout directly to the next command's stdin.

```bash
cat access.log | wc -l          # works, but:
wc -l < access.log              # the same without the extra process
wc -l access.log                # most tools also take a filename
```

A pipeline is built left to right, one stage at a time. Run it after each addition and look at the output before adding the next stage. Nobody writes a five-stage pipeline in one go.

```bash
cut -d' ' -f1 access.log
cut -d' ' -f1 access.log | sort
cut -d' ' -f1 access.log | sort | uniq -c
cut -d' ' -f1 access.log | sort | uniq -c | sort -rn
```

Facts about pipes:

- **Only stdout goes through `|`.** stderr still goes to the terminal. Use `2>&1 |` or `|&` to send both.
- All stages run at the same time, each as its own process.
- The pipeline's exit status is the **last** command's. `false | true` succeeds. `set -o pipefail` in scripts changes that (Lab 14). `echo "${PIPESTATUS[@]}"` shows every stage's status.

```bash
ls /etc/hostname /nonexistent | wc -l            # count is 1; the error bypassed the pipe
ls /etc/hostname /nonexistent 2>&1 | wc -l       # count is 2
false | true; echo "status $? / each stage: ${PIPESTATUS[@]}"
```

### tee

`tee` copies stdin to a file **and** to stdout, like a T-junction in a pipe.

```bash
cut -d' ' -f1 access.log | sort -u | tee ips.txt | wc -l     # save the intermediate result, keep going
cat ips.txt
date | tee -a run.log                                         # -a appends
echo "setting" | sudo tee /etc/labfile > /dev/null            # writing as root (Lab 03)
sudo rm /etc/labfile
some_long_command 2>&1 | tee build.log                        # watch it AND keep a log (command is illustrative)
```

### Process substitution

`<(command)` lets a command's output stand in where a filename is expected. It is how you feed two pipelines into one tool.

```bash
diff <(cut -d, -f1 inventory.csv | sort) <(cut -d, -f1 owners.csv | sort)
```

### xargs: stdin to arguments

Some commands take their input as arguments, not on stdin (`rm`, `mkdir`, `ls`). `xargs` converts.

```bash
cut -d, -f1 inventory.csv | tail -n +2 | xargs -I{} echo "would ping {}"
find . -name '*.txt' -print0 | xargs -0 ls -l       # -print0 / -0: safe for any filename
```

---

## 2. head, tail, wc, nl

```bash
head access.log                 # first 10 lines
head -n 3 access.log
head -n -3 access.log           # everything EXCEPT the last 3
head -c 100 access.log          # first 100 bytes

tail -n 3 access.log            # last 3 lines
tail -n +2 inventory.csv        # from line 2 onward: the standard way to skip a header
tail -f /var/log/messages       # follow: print new lines as they arrive (Ctrl-C to stop; needs sudo)
tail -F app.log                 # like -f, but survives the file being rotated or recreated
```

Extract a range (lines 5 to 8):

```bash
head -n 8 access.log | tail -n 4
sed -n '5,8p' access.log        # the sed way
```

`tail -f` is how you watch a log while reproducing a problem in another terminal. On systemd systems `journalctl -f` does the same job for the journal (Lab 09).

```bash
wc access.log                   # lines  words  bytes  name
wc -l access.log                # lines
wc -w access.log                # words
wc -c access.log                # bytes
wc -m access.log                # characters (differs from bytes with UTF-8)
wc -L access.log                # length of the longest line
wc -l < access.log              # a bare number with no filename: what you want in scripts
ls /etc | wc -l                 # count anything
```

```bash
nl inventory.csv                # number the non-empty lines
nl -ba inventory.csv            # number all lines, including blank ones
nl -s': ' -w2 inventory.csv     # custom separator and width
cat -n inventory.csv            # the quick alternative
```

### Exercises 2

1. Print only line 13 of `access.log`.
2. Print `inventory.csv` without its header and without its last line.
3. How many files are in `/usr/bin`?
4. Show the three most recently modified entries in `/var/log`.

<details><summary>Solutions</summary>

```bash
head -n 13 access.log | tail -n 1          # 1   (or: sed -n '13p' access.log)
tail -n +2 inventory.csv | head -n -1      # 2
ls /usr/bin | wc -l                        # 3
ls -t /var/log | head -n 3                 # 4
```

</details>

---

## 3. cut and paste

### cut: select columns

```bash
cut -d, -f1 inventory.csv           # field 1, comma-delimited
cut -d, -f1,3 inventory.csv         # fields 1 and 3
cut -d, -f2- inventory.csv          # field 2 to the end
cut -d, -f-3 inventory.csv          # up to field 3
cut -c1-5 inventory.csv             # characters 1 to 5
cut -d: -f1,7 /etc/passwd           # usernames and their shells
cut -d, -f1,5 --output-delimiter=' has ' inventory.csv
```

The default delimiter is a tab. `cut` treats **every** delimiter character as a boundary, so it cannot handle columns padded with varying numbers of spaces:

```bash
ls -l | cut -d' ' -f5               # garbage: runs of spaces create empty fields
ls -l | awk '{print $5}'            # awk splits on runs of whitespace
ls -l | tr -s ' ' | cut -d' ' -f5   # or squeeze the spaces first
```

`cut` also cannot reorder: `-f3,1` prints field 1 then 3. Use `awk` for reordering.

### paste: glue lines side by side

```bash
cut -d, -f1 inventory.csv > hosts.txt
cut -d, -f3 inventory.csv > envs.txt
paste hosts.txt envs.txt                # joined with a tab
paste -d: hosts.txt envs.txt            # joined with a colon

tail -n +2 hosts.txt | paste -sd,       # -s: serialise all lines into ONE line
tail -n +2 hosts.txt | paste - - -      # three per row
```

Expected output of the last two:

```
web01,web02,web03,db01,db02,cache01,lb01,ci01
web01	web02	web03
db01	db02	cache01
lb01	ci01
```

`paste -sd,` is the standard way to turn a column into a comma-separated list. `paste -sd+ | bc` sums a column of numbers.

### Exercises 3

1. Print each user's name and home directory from `/etc/passwd`, separated by ` -> `.
2. Produce the single line `8,8,4,64,32,16,4,16` (the `mem_gb` column) from `inventory.csv`.
3. Sum that column using `paste` and `bc`.
4. Build a file `swapped.csv` with the columns of `inventory.csv` in the order `env,host`. (Hint: `cut` twice, `paste` once.)

<details><summary>Solutions</summary>

```bash
cut -d: -f1,6 --output-delimiter=' -> ' /etc/passwd                    # 1
tail -n +2 inventory.csv | cut -d, -f5 | paste -sd,                    # 2
tail -n +2 inventory.csv | cut -d, -f5 | paste -sd+ | bc               # 3 → 152
paste -d, <(cut -d, -f3 inventory.csv) <(cut -d, -f1 inventory.csv) > swapped.csv   # 4
```

</details>

---

## 4. sort and uniq

### sort

```bash
sort hosts.txt                      # alphabetical
sort -r hosts.txt                   # reversed
sort -u hosts.txt                   # sorted, duplicates removed
```

Without `-n`, numbers are compared as text, and `10` sorts before `9` because `1` < `9`. `-h` understands human-readable sizes (2K < 1M < 3G).

```bash
printf '10\n9\n100\n1\n' | sort
printf '10\n9\n100\n1\n' | sort -n
du -sh /var/log/* 2>/dev/null | sort -h | tail -5
```

Sorting by column: `-t` sets the delimiter and `-k` the key. A key is `start,end`, and **without the end the key runs to the end of the line**, so write `-k5,5` and not `-k5`.

```bash
tail -n +2 inventory.csv | sort -t, -k5,5nr               # by memory, numeric, descending
tail -n +2 inventory.csv | sort -t, -k3,3 -k4,4nr         # by env, then by cpu descending within each env
tail -n +2 inventory.csv | sort -t, -k2,2 -s              # -s stable: keep input order among equals
sort -t: -k3,3n /etc/passwd | head -5                     # passwd by UID
```

Expected output of the second command:

```
ci01,build,dev,8,16
db01,db,prod,16,64
web01,web,prod,4,8
web02,web,prod,4,8
cache01,cache,prod,2,16
lb01,lb,prod,2,4
db02,db,stage,8,32
web03,web,stage,2,4
```

Locale affects sort order. `LC_ALL=C sort` gives plain byte order, is faster, and is what you want when another tool (such as `join`) needs sorted input.

### uniq

`uniq` collapses **adjacent** identical lines. Unsorted duplicates are not adjacent, so `uniq` nearly always follows `sort`.

```bash
cut -d, -f3 inventory.csv | tail -n +2 | uniq             # wrong: prod appears more than once
cut -d, -f3 inventory.csv | tail -n +2 | sort | uniq      # right
cut -d, -f3 inventory.csv | tail -n +2 | sort | uniq -c   # with counts
cut -d, -f3 inventory.csv | tail -n +2 | sort | uniq -d   # only values that repeat
cut -d, -f3 inventory.csv | tail -n +2 | sort | uniq -u   # only values that appear once
```

### The frequency-count idiom

This is the single most useful pipeline in log analysis. Memorise its shape:

```
<extract one value per line> | sort | uniq -c | sort -rn | head
```

```bash
cut -d' ' -f1 access.log | sort | uniq -c | sort -rn
```

```
      5 10.0.1.15
      4 203.0.113.9
      4 198.51.100.7
      4 10.0.1.22
      3 10.0.1.31
```

(The order among lines with equal counts can differ with your locale.)

### Exercises 4

1. Which login shells are in use in `/etc/passwd`, and how many accounts have each? Most common first.
2. List the hosts in `inventory.csv` from most to least CPU, breaking ties by hostname.
3. Which request paths in `access.log` were hit more than once?
4. How many distinct client IPs are in `access.log`?
5. What are the five largest files or directories directly under `/usr/share`?

<details><summary>Solutions</summary>

```bash
cut -d: -f7 /etc/passwd | sort | uniq -c | sort -rn                   # 1
tail -n +2 inventory.csv | sort -t, -k4,4nr -k1,1                     # 2
cut -d' ' -f7 access.log | sort | uniq -d                             # 3 → /api/login /api/ports /api/routes /index.html
cut -d' ' -f1 access.log | sort -u | wc -l                            # 4 → 5
du -sh /usr/share/* 2>/dev/null | sort -h | tail -5                   # 5
```

</details>

---

## 5. tr, expand, unexpand

### tr: translate or delete characters

`tr` works on single characters, reads only stdin, and takes no filename.

```bash
echo "Hello World" | tr 'a-z' 'A-Z'           # map lowercase to uppercase
echo "Hello World" | tr '[:lower:]' '[:upper:]'   # the locale-safe spelling
echo "a,b,c" | tr ',' '\n'                    # one item per line
echo $PATH | tr ':' '\n'
echo "too    many   spaces" | tr -s ' '       # -s squeeze repeats into one
echo "a1b2c3" | tr -d '0-9'                   # -d delete
echo "phone: 555-1234" | tr -cd '0-9\n'       # -c complement: delete everything EXCEPT digits and newline
tr -d '\r' < windows.txt > unix.txt           # strip Windows carriage returns (illustrative filename)
head -c 32 /dev/urandom | tr -dc 'A-Za-z0-9' ; echo     # a random string
```

The carriage-return one fixes the classic `bad interpreter: /bin/bash^M` error from a script edited on Windows.

### expand and unexpand: tabs and spaces

```bash
head -3 inventory.csv | tr ',' '\t' > tabbed.txt
cat -A tabbed.txt                    # -A shows tabs as ^I and line ends as $
expand -t 10 tabbed.txt              # tabs → spaces, tab stops every 10 columns
expand -t 10 tabbed.txt | cat -A     # confirm: no ^I left
expand tabbed.txt > spaced.txt       # default tab stop is 8
unexpand -a spaced.txt | cat -A      # spaces → tabs (-a: all runs, not just leading ones)
```

Where this matters: YAML rejects tabs, and Makefiles require them. `cat -A` is how you see which one you have. `expand -t 2 broken.yml` repairs a YAML file that someone indented with tabs.

For aligning columns for reading, `column` is the usual tool:

```bash
column -s, -t inventory.csv
```

### Exercises 5

1. Convert `inventory.csv` to uppercase, with semicolons as separators.
2. Count how many times the letter `e` appears in `/etc/services`. (Hint: delete everything else.)
3. Print the words of `"the quick brown fox"` one per line, sorted.
4. Does `/etc/services` contain tab characters? Count the lines that do.

<details><summary>Solutions</summary>

```bash
tr 'a-z,' 'A-Z;' < inventory.csv                         # 1
tr -cd 'e' < /etc/services | wc -c                       # 2
echo "the quick brown fox" | tr ' ' '\n' | sort          # 3
grep -c $'\t' /etc/services                              # 4
```

</details>

---

## 6. split and join

### split: cut a file into pieces

```bash
split -l 6 -d access.log part-          # 6 lines each, numeric suffixes: part-00, part-01, ...
wc -l part-*
cat part-* | cmp - access.log && echo "reassembles exactly"

split -b 1M bigfile chunk-              # by size (illustrative)
split -n 4 access.log quarter-          # into exactly 4 pieces (may cut mid-line)
split -n l/4 access.log q-              # into 4 pieces without splitting lines
rm part-* quarter-* q-*
```

Typical uses: breaking a huge CSV into batches for a loader, or getting a file under an upload size limit and reassembling with `cat`.

### join: match lines of two files on a common field

`join` is a relational join for text files. **Both inputs must be sorted on the join field.**

```bash
tail -n +2 inventory.csv | sort -t, -k1,1 > inv.sorted
tail -n +2 owners.csv    | sort -t, -k1,1 > own.sorted

join -t, inv.sorted own.sorted
```

```
cache01,cache,prod,2,16,platform
ci01,build,dev,8,16,devex
db01,db,prod,16,64,data
db02,db,stage,8,32,data
lb01,lb,prod,2,4,platform
web01,web,prod,4,8,frontend
web02,web,prod,4,8,frontend
```

That is an inner join: `web03` (no owner) and `mon01` (not in inventory) are missing.

| Option | Meaning |
|---|---|
| `-t,` | field delimiter |
| `-1 N` / `-2 N` | join on field N of file 1 / file 2 (default: field 1) |
| `-a 1` / `-a 2` | also print unmatched lines from file 1 / 2 (left / right outer join) |
| `-v 1` / `-v 2` | print **only** unmatched lines from file 1 / 2 |
| `-e TEXT` | fill missing fields with TEXT (needs `-o`) |
| `-o 0,1.2,2.2` | output format: `0` is the join field, `1.2` is file 1 field 2 |

```bash
join -t, -a 1 -e NONE -o 0,1.2,1.3,2.2 inv.sorted own.sorted    # left join with a placeholder
join -t, -v 1 inv.sorted own.sorted                             # hosts with no owner → web03
join -t, -v 2 inv.sorted own.sorted                             # owners of unknown hosts → mon01
```

`comm` is the related tool for comparing two sorted lists line by line:

```bash
comm -12 <(cut -d, -f1 inv.sorted) <(cut -d, -f1 own.sorted)    # lines in both
comm -23 <(cut -d, -f1 inv.sorted) <(cut -d, -f1 own.sorted)    # only in the first
```

### Exercises 6

1. Produce a list of `host,env,owner` for every host that has an owner.
2. Produce `owner,host` sorted by owner.
3. Which hosts in `owners.csv` are missing from the inventory? Solve it once with `join` and once with `comm`.

<details><summary>Solutions</summary>

```bash
join -t, -o 0,1.3,2.2 inv.sorted own.sorted                                # 1
join -t, -o 2.2,0 inv.sorted own.sorted | sort                             # 2
join -t, -v 2 inv.sorted own.sorted | cut -d, -f1                          # 3a → mon01
comm -13 <(cut -d, -f1 inv.sorted) <(cut -d, -f1 own.sorted)               # 3b → mon01
```

</details>

---

## 7. grep

`grep` prints the lines that match a pattern.

```bash
grep 404 access.log                  # lines containing 404
grep -c 404 access.log               # count of matching lines
grep -n 404 access.log               # with line numbers
grep -v 200 access.log               # lines that do NOT match
grep -i "get" access.log             # case-insensitive
grep -w web inventory.csv            # whole word: matches ",web," but not "web01"
grep -o '/api/[a-z]*' access.log     # print only the matching part
grep -A2 -B1 ' 500 ' access.log      # 2 lines after, 1 before; -C3 for both sides
grep -l root /etc/* 2>/dev/null      # only the NAMES of files that match
grep -r "PermitRootLogin" /etc/ssh 2>/dev/null     # recurse into a directory
grep -rn --include='*.conf' listen /etc 2>/dev/null | head
grep -q prod inventory.csv && echo "has prod"      # quiet: exit status only, for scripts
grep -F '[08/Oct' access.log         # fixed string: no regex interpretation
grep -e 404 -e 500 access.log        # several patterns
```

`grep 404` also matches a response of 404 bytes, or an IP containing 404. Anchor to the field: `grep '" 404 '`.

### Regular expressions

Use `grep -E` (extended regex). Without `-E`, the characters `+ ? | ( ) { }` need backslashes, which is hard to read.

| Pattern | Matches |
|---|---|
| `.` | any one character |
| `^` / `$` | start / end of line |
| `[abc]` `[a-z]` `[^0-9]` | one of / range / none of |
| `*` | zero or more of the previous |
| `+` | one or more |
| `?` | zero or one |
| `{3}` `{2,4}` | exactly 3 / between 2 and 4 |
| `a\|b` | a or b |
| `( )` | grouping |
| `\.` | a literal dot |
| `\b` or `\<` `\>` | word boundary |
| `[[:space:]]` `[[:digit:]]` `[[:alpha:]]` | character classes |

```bash
grep -E '^10\.' access.log                         # requests from 10.x addresses
grep -vE '^10\.' access.log                        # everything else
grep -E '" (4|5)[0-9]{2} ' access.log              # any 4xx or 5xx status
grep -E 'POST|DELETE' access.log
grep -oE '^[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' access.log | sort -u     # extract the IPs
grep -E '^[^#]' /etc/ssh/sshd_config               # non-comment, non-empty lines
grep -vE '^\s*(#|$)' /etc/ssh/sshd_config          # the same, tolerating leading whitespace
```

The last one is worth an alias. It shows the effective content of any config file.

Quote patterns in **single quotes** so the shell does not expand `$`, `*` or `\` before grep sees them.

### Exercises 7

1. How many requests in `access.log` returned a 4xx or 5xx status? (Expected: 8)
2. Print only the paths (field 7) of requests from addresses that do not start with `10.`.
3. List the usernames in `/etc/passwd` whose shell is not `nologin` or `false`.
4. Find every file under `/etc` that mentions `wheel` (names only, errors hidden).
5. Print the non-comment, non-blank lines of `/etc/sudoers`.
6. Which lines of `inventory.csv` have a hostname ending in `01`?

<details><summary>Solutions</summary>

```bash
grep -cE '" [45][0-9]{2} ' access.log                               # 1 → 8
grep -vE '^10\.' access.log | cut -d' ' -f7                         # 2
grep -vE '(nologin|false)$' /etc/passwd | cut -d: -f1               # 3
sudo grep -rl wheel /etc 2>/dev/null                                # 4
sudo grep -vE '^\s*(#|$)' /etc/sudoers                              # 5
grep -E '^[a-z]+01,' inventory.csv                                  # 6
```

</details>

---

## 8. awk

`awk` is a small programming language for records and fields. A program is a list of:

```
pattern { action }
```

For each input line, awk splits it into fields, then runs the action of every pattern that matches. No pattern means every line. No action means print the line.

| Name | Meaning |
|---|---|
| `$0` | the whole line |
| `$1`, `$2`, … | the fields |
| `NF` | number of fields on this line; `$NF` is the last field |
| `NR` | current line number |
| `FS` / `OFS` | input / output field separator (`-F,` sets FS) |
| `BEGIN { }` | runs once before any input |
| `END { }` | runs once after all input |

### Printing and selecting

```bash
awk '{print $1}' access.log                       # first field; splits on runs of whitespace
awk '{print $1, $9}' access.log                   # a comma inserts OFS (a space)
awk '{print $9 " <- " $1}' access.log             # adjacent strings are concatenated
awk '{print NR": "$7}' access.log                 # line numbers
awk '{print $NF}' access.log                      # last field
awk -F, '{print $1, $5}' inventory.csv            # comma-separated input
awk -F, -v OFS=, '{print $3, $1}' inventory.csv   # reorder columns, keep commas
awk -F: '{print $1}' /etc/passwd
```

```bash
awk '$9 == 404' access.log                        # numeric comparison on a field
awk '$9 >= 400 {print $1, $7, $9}' access.log
awk '$10 > 20000 {print $7, $10}' access.log
awk '/login/' access.log                          # regex on the whole line (like grep)
awk '$7 ~ /^\/api/' access.log                    # regex on one field
awk '$7 !~ /^\/api/ {print $7}' access.log        # does not match
awk -F, 'NR > 1' inventory.csv                    # skip the header
awk -F, 'NR > 1 && $3 == "prod" && $4 >= 4' inventory.csv
awk 'NR >= 5 && NR <= 8' access.log               # a line range
```

Being able to test **one field** is awk's advantage over grep. `awk '$9 == 404'` cannot be fooled by a 404 that appears elsewhere on the line.

### Arithmetic and END

Variables start out as zero / empty and need no declaration.

```bash
awk '{sum += $10} END {print sum}' access.log                         # total bytes → 214883
awk '{sum += $10} END {print sum / NR}' access.log                    # average
awk '{sum += $10} END {printf "%.1f KB\n", sum / 1024}' access.log    # formatted
awk '$9 >= 400 {n++} END {print n " errors of " NR " requests"}' access.log
awk -F, 'NR > 1 {cpu += $4; mem += $5} END {print cpu " cores, " mem " GB"}' inventory.csv
awk 'length($0) > 80' access.log                                      # lines longer than 80 chars
```

`printf` works as in C: `%s` string, `%d` integer, `%.2f` float, `%-15s` left-justified in 15 columns, `%5d` right-justified in 5. It does not add a newline for you.

### Associative arrays: group by

An array indexed by a string gives you `GROUP BY` in one pass, with no sorting.

```bash
awk '{count[$1]++} END {for (ip in count) print count[ip], ip}' access.log | sort -rn
awk '{bytes[$1] += $10} END {for (ip in bytes) print bytes[ip], ip}' access.log | sort -rn
awk '{c[$9]++} END {for (s in c) print s, c[s]}' access.log | sort
awk -F, 'NR > 1 {m[$3] += $5} END {for (e in m) printf "%-6s %3d GB\n", e, m[e]}' inventory.csv | sort
```

Expected output of the second and fourth commands:

```
76659 10.0.1.22
56210 10.0.1.15
40816 198.51.100.7
40112 10.0.1.31
1086 203.0.113.9
```

```
dev     16 GB
prod   100 GB
stage   36 GB
```

`for (key in array)` visits keys in no particular order, which is why the output is piped to `sort`.

### BEGIN, and a full report

```bash
awk 'BEGIN {print "HOST\tMEM"} NR > 1 {print $1 "\t" $5}' FS=, inventory.csv
```

A longer program reads better across several lines. The shell lets a single-quoted string span lines:

```bash
awk '
  { reqs[$1]++; bytes[$1] += $10 }
  $9 >= 400 { errs[$1]++ }
  END {
    for (ip in reqs)
      printf "%-15s %5d %8d %6d\n", ip, reqs[ip], bytes[ip], errs[ip]
  }
' access.log | sort -k2,2nr
```

```
10.0.1.15           5    56210      1
10.0.1.22           4    76659      1
198.51.100.7        4    40816      2
203.0.113.9         4     1086      4
10.0.1.31           3    40112      0
```

Read the bottom-ranked-by-bytes row: `203.0.113.9` made 4 requests, all errors, for `/wp-login.php`, `/.env`, `/admin` and `/phpmyadmin`. That is a vulnerability scanner. Finding that in a real log is what this lab is for.

### awk on command output

```bash
df -h | awk 'NR > 1 && $5+0 > 50 {print $6, $5}'             # filesystems over 50% full ($5+0 turns "63%" into 63)
ps aux | awk 'NR > 1 {mem[$1] += $6} END {for (u in mem) printf "%-12s %8.1f MB\n", u, mem[u]/1024}' | sort -k2 -nr
free -m | awk '/^Mem:/ {printf "%.0f%% used\n", $3 / $2 * 100}'
ip -o -4 addr show | awk '{print $2, $4}'
```

### Exercises 8

Use awk for each.

1. Print the path and byte count for every successful (`200`) request larger than 10,000 bytes.
2. How many requests of each HTTP method (GET, POST, DELETE) are there? The method is in field 6 with a leading quote; strip it with `substr($6, 2)`.
3. Which path transferred the most total bytes?
4. From `inventory.csv`, print `host: N cores` for prod hosts only.
5. What is the average memory per host, by role?
6. Print the usernames in `/etc/passwd` with a UID of 1000 or more, with their UIDs.
7. Rewrite the frequency-count idiom from section 4 (`cut | sort | uniq -c | sort -rn`) with awk doing the counting.

<details><summary>Solutions</summary>

```bash
# 1
awk '$9 == 200 && $10 > 10000 {print $7, $10}' access.log

# 2
awk '{m[substr($6, 2)]++} END {for (k in m) print k, m[k]}' access.log
# GET 16, POST 3, DELETE 1

# 3
awk '{b[$7] += $10} END {for (p in b) print b[p], p}' access.log | sort -rn | head -1
# 120336 /api/routes

# 4
awk -F, 'NR > 1 && $3 == "prod" {print $1 ": " $4 " cores"}' inventory.csv

# 5
awk -F, 'NR > 1 {s[$2] += $5; n[$2]++} END {for (r in s) printf "%-6s %.1f\n", r, s[r] / n[r]}' inventory.csv | sort

# 6
awk -F: '$3 >= 1000 {print $1, $3}' /etc/passwd

# 7
awk '{c[$1]++} END {for (ip in c) print c[ip], ip}' access.log | sort -rn
```

</details>

---

## 9. Bonus: sed in five minutes

`sed` is not on the roadmap, but you cannot read other people's shell scripts without it. Its main job is substitution on a stream.

```bash
sed 's/prod/production/' inventory.csv          # first match per line
sed 's/web/WEB/g' inventory.csv                 # every match per line
sed -n '3,5p' inventory.csv                     # print only lines 3–5
sed '1d' inventory.csv                          # delete line 1
sed '/stage/d' inventory.csv                    # delete matching lines
sed -E 's/^([a-z]+)([0-9]+),/\2-\1,/' inventory.csv    # capture groups: web01 → 01-web
sed -i.bak 's/stage/staging/' inventory.csv     # edit the file IN PLACE, keeping inventory.csv.bak
mv inventory.csv.bak inventory.csv              # undo
```

`sed -i` is how scripts change config files. Always give it a backup suffix until you trust the expression.

---

## Which tool when

| Job | Tool |
|---|---|
| Lines that match a pattern | `grep` |
| A column, with a single-character delimiter | `cut` |
| A column, with messy whitespace; or a condition on a column; or arithmetic | `awk` |
| Count occurrences | `sort \| uniq -c \| sort -rn` or an awk array |
| Replace text | `sed 's/a/b/g'` |
| Replace or delete single characters | `tr` |
| Combine two files side by side | `paste` |
| Combine two files on a key | `join` |
| First or last lines, or follow a file | `head`, `tail` |
| JSON | none of these: use `jq` (`sudo dnf install -y jq`) |

## Challenge: incident questions

Answer each with a single pipeline against `access.log`.

1. Which IP had the most failed (`401`) login attempts, and did it eventually succeed?
2. List every distinct path requested by `203.0.113.9`, comma-separated on one line.
3. What percentage of requests were errors (status ≥ 400)?
4. Produce a table of status code, count, and percentage of all requests, sorted by count.
5. Print the busiest 10-second window: group requests by the timestamp truncated to tens of seconds (for example `09:00:0`, `09:00:1`) and count.
6. Now run question 1's approach against real data: failed SSH logins on your tower by source address.

<details><summary>Solutions</summary>

```bash
# 1
awk '$7 == "/api/login" {print $1, $9}' access.log | sort | uniq -c
#   1 198.51.100.7 200
#   2 198.51.100.7 401      → two failures, then a success

# 2
awk '$1 == "203.0.113.9" {print $7}' access.log | sort -u | paste -sd,
# /.env,/admin,/phpmyadmin,/wp-login.php

# 3
awk '$9 >= 400 {e++} END {printf "%.0f%%\n", e / NR * 100}' access.log
# 40%

# 4
awk '{c[$9]++} END {for (s in c) printf "%s %d %.0f%%\n", s, c[s], c[s] / NR * 100}' access.log | sort -k2,2nr | column -t

# 5
awk '{print substr($4, 14, 7)}' access.log | sort | uniq -c | sort -rn | head -3
# substr($4, 14, 7) takes "09:00:0" out of "[08/Oct/2026:09:00:01"

# 6
sudo grep 'Failed password' /var/log/secure | grep -oE 'from [0-9.]+' | sort | uniq -c | sort -rn | head
# An empty result is good news.
```

</details>

## Check yourself

<details><summary>Why does <code>uniq</code> need sorted input?</summary>
It only compares each line with the one before it. Duplicates that are not adjacent are not detected.
</details>

<details><summary>Why does <code>ls -l | cut -d' ' -f5</code> not give the size column?</summary>
<code>cut</code> treats each space as a separator, so the padding between columns produces empty fields. <code>awk '{print $5}'</code> splits on runs of whitespace.
</details>

<details><summary><code>sort -t, -k2</code> gives an unexpected order. What is wrong?</summary>
<code>-k2</code> means "from field 2 to the end of the line". To sort on field 2 alone, write <code>-k2,2</code>.
</details>

<details><summary>What does <code>cmd | tee file</code> do that <code>cmd > file</code> does not?</summary>
It writes to the file and also passes the data on to stdout, so you can watch it or pipe it further.
</details>

<details><summary>What must be true of both inputs to <code>join</code>?</summary>
Both must be sorted on the join field, using the same collation that <code>join</code> uses.
</details>

<details><summary>In awk, what is the difference between <code>/404/</code> and <code>$9 == 404</code>?</summary>
The first matches the text 404 anywhere on the line. The second compares field 9 only, as a number.
</details>

## Cleanup

```bash
rm -r ~/labwork/05
```
