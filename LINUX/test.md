The issue is that `registry.internal` is resolving to the wrong IP (`10.255.255.1`) instead of `127.0.0.1`.

### Step 1: Check where the resolution is coming from

```bash
cat /etc/hosts
grep registry.internal /etc/hosts

cat /etc/nsswitch.conf
```

If `/etc/hosts` contains:

```text
10.255.255.1 registry.internal
```

change it to:

```text
127.0.0.1 registry.internal
```

### Step 2: Edit `/etc/hosts`

Using `vi`:

```bash
sudo vi /etc/hosts
```

or with `sed`:

```bash
sudo sed -i 's/10\.255\.255\.1/127.0.0.1/' /etc/hosts
```

If the entry doesn't exist, append it:

```bash
echo "127.0.0.1 registry.internal" | sudo tee -a /etc/hosts
```

### Step 3: Verify

```bash
getent hosts registry.internal
```

Expected:

```text
127.0.0.1 registry.internal
```

Then:

```bash
ping -c1 registry.internal
```

Expected:

```text
PING registry.internal (127.0.0.1) ...
64 bytes from localhost (127.0.0.1): icmp_seq=1 ttl=64 time=...
```

### If it still resolves to `10.255.255.1`

Check for other name sources:

```bash
grep '^hosts:' /etc/nsswitch.conf
```

Ensure `files` comes before `dns`, e.g.:

```text
hosts: files dns
```

Also check:

```bash
grep registry.internal /etc/hosts
```

For this Crucible/Linux lab, the fix is almost always **correcting the `/etc/hosts` entry** from `10.255.255.1` to `127.0.0.1`.

Your `/etc/hosts` file shows the problem:

```text
10.255.255.1 registry.internal
```

Replace it with:

```text
127.0.0.1 registry.internal
```

The quickest command is:

```bash
sed -i 's/^10\.255\.255\.1 registry\.internal/127.0.0.1 registry.internal/' /etc/hosts
```

Then verify:

```bash
getent hosts registry.internal
```

It should output:

```text
127.0.0.1 registry.internal
```

Finally:

```bash
ping -c1 registry.internal
```

You should see:

```text
64 bytes from localhost (127.0.0.1)
```

If `sed` doesn't work, open the file with:

```bash
vi /etc/hosts
```

Change:

```text
10.255.255.1 registry.internal
```

to:

```text
127.0.0.1 registry.internal
```

Save with `Esc`, then `:wq` and press **Enter**.
The error:

```text
sed: cannot rename ... Device or resource busy
```

usually means **`/etc/hosts` is mounted in a way that prevents `sed -i` from replacing the file** (common in Docker/Kubernetes labs).

Instead, overwrite the file contents directly.

### Option 1 (simplest)

Run:

```bash
cat >/etc/hosts <<'EOF'
# Kubernetes-managed hosts file.
127.0.0.1 localhost
::1 localhost ip6-localhost ip6-loopback
fe00::0 ip6-localnet
fe00::0 ip6-mcastprefix
fe00::1 ip6-allnodes
fe00::2 ip6-allrouters
10.80.66.72 lab-bhageerathareddykupp-dns-broken-165b4f9c
127.0.0.1 registry.internal
EOF
```

Then verify:

```bash
getent hosts registry.internal
ping -c1 registry.internal
```

### Option 2 (if the above fails)

Use:

```bash
echo "127.0.0.1 registry.internal" >> /etc/hosts
```

Then check:

```bash
getent hosts registry.internal
```

If it **still** returns `10.255.255.1`, tell me the output of:

```bash
mount | grep hosts
```

That will show how `/etc/hosts` is mounted so we can use the correct fix.
You're very close.

`getent` now returns **two entries**:

```text
10.255.255.1 registry.internal
127.0.0.1 registry.internal
```

The old incorrect entry is still present. You need to **remove**:

```text
10.255.255.1 registry.internal
```

Since `sed -i` doesn't work, use `vi`:

```bash
vi /etc/hosts
```

Navigate to:

```text
10.255.255.1 registry.internal
```

Delete that line:

1. Press `Esc`
2. Move the cursor onto the line.
3. Press `dd` (deletes the whole line).
4. Press `:wq` and Enter to save.

Then verify:

```bash
getent hosts registry.internal
```

It should show only:

```text
127.0.0.1 registry.internal
```

Finally:

```bash
ping -c1 registry.internal
```

If `vi` also won't let you edit the file, run:

```bash
mount | grep /etc/hosts
```

and share the output. That will determine the correct workaround.
The goal is to **remove the runaway log file**.

Run:

```bash
rm -f /workspace/app.log
```

Then verify:

```bash
ls -lh /workspace/app.log
du -sh /workspace
```

Expected output:

```text
ls: cannot access '/workspace/app.log': No such file or directory
```

and

```text
4.0K    /workspace
```

If `rm` fails because the file is in use, truncate it first:

```bash
truncate -s 0 /workspace/app.log
rm -f /workspace/app.log
```

For this Crucible lab, simply deleting `/workspace/app.log` is the expected fix.

The file exists but has the **wrong permissions**.

Expected permissions:

```text
-r--r--r-- 1 root root 20 ... /srv/app/config.conf
```

That corresponds to **`444`**.

Run:

```bash
chmod 444 /srv/app/config.conf
```

Or equivalently:

```bash
chmod a+r /srv/app/config.conf
chmod u-w /srv/app/config.conf
```

Then verify:

```bash
ls -l /srv/app/config.conf
su appuser -s /bin/sh -c 'cat /srv/app/config.conf'
```

Expected output:

```text
-r--r--r-- 1 root root 20 ... /srv/app/config.conf
```

and

```text
db_password: s3cr3t
```

**Tip for Crucible:** If the challenge says "wrong ownership/mode" and the owner is already `root root`, you usually only need to fix the **mode** (`chmod`), not the ownership (`chown`).

This is a classic **deleted file still held open** problem.

The output shows:

```text
python3  16  root  3r  ... /var/lib/crux/ghost.dat (deleted)
```

The file has been deleted, but **PID 16** (`python3`) still has it open, so the disk space isn't released.

### Fix

Restart or terminate the process holding the file:

```bash
kill 16
```

or, if it doesn't exit:

```bash
kill -9 16
```

Then verify:

```bash
lsof +L1 2>/dev/null | head
```

It should produce **no output**.

Also check:

```bash
du -sh /var/lib/crux 2>/dev/null
```

It should still show:

```text
4.0K    /var/lib/crux
```

**Why this works:** On Linux, deleting a file only removes its directory entry. The disk space is not actually freed until **all processes close the file**. Killing (or restarting) the process releases the file descriptor, freeing the space.

`kill` without a signal sends **SIGTERM (15)**. This process is ignoring SIGTERM, so you need **SIGKILL (9)**.

Run:

```bash
kill -9 $(pgrep crux-immortal)
```

or:

```bash
pkill -9 crux-immortal
```

Verify:

```bash
pgrep -a crux-immortal
```

There should be **no output**.

If it's still running, check whether it's being restarted by a parent process:

```bash
ps -fp 14
ps -ef | grep crux-immortal
```

But for this Crucible lab, the expected solution is usually:

```bash
kill -9 $(pgrep crux-immortal)
```

This is almost always caused by a **stale lock file**.

### 1. Find the lock file

Run:

```bash
find / -name "*.lock" 2>/dev/null
```

or

```bash
find /var /tmp /run -name "*.lock" 2>/dev/null
```

### 2. Check whether a build is actually running

```bash
pgrep -af crux-build
```

If **no build process is running**, remove the stale lock:

```bash
rm -f /path/to/the/lockfile
```

Typical locations are:

```bash
rm -f /tmp/crux-build.lock
rm -f /var/run/crux-build.lock
rm -f /run/crux-build.lock
rm -f /var/lock/crux-build.lock
```

### 3. Verify

```bash
/ usr/local/bin/crux-build
```

(use the correct command without the space:)

```bash
/usr/local/bin/crux-build
```

Expected:

```text
Build completed successfully.
```

### Faster method for Crucible

If you don't know the lock location, inspect the script:

```bash
cat /usr/local/bin/crux-build
```

or

```bash
grep -i lock /usr/local/bin/crux-build
```

It usually reveals the exact lock file path to delete.

Since:

* `pgrep -af crux-build` returns **nothing** (no build is running), and
* You found the lock file:

```text
/var/lib/crux/build.lock
```

it's a **stale lock file**.

Run:

```bash
rm -f /var/lib/crux/build.lock
```

Then verify:

```bash
/usr/local/bin/crux-build
```

It should output:

```text
Build completed successfully.
```

If it still says "another build is already running", inspect the script to confirm the lock path:

```bash
grep -n lock /usr/local/bin/crux-build
```

or

```bash
cat /usr/local/bin/crux-build
```

But in this lab, removing `/var/lib/crux/build.lock` is almost certainly the correct fix.

This lab is about a **broken symbolic link**.

From your output:

```text
X -> /workspace/treasure-map/lost-treasure
```

The symlink itself exists, but the target is broken or missing.

### Step 1: Read the clue

```bash
cat /workspace/treasure-map/clue.txt
```

The clue will tell you where the real treasure is (often under `/cave/...`).

### Step 2: Find the treasure

Search for it:

```bash
find / -name "lost-treasure" 2>/dev/null
```

or

```bash
find / -type f 2>/dev/null | grep treasure
```

### Step 3: Copy the treasure contents

Once you find the real `lost-treasure` file, copy its contents to the required location:

```bash
cat /path/to/real/lost-treasure > /home/admin/solution
```

### If you're stuck

Please run these two commands and paste their output:

```bash
cat /workspace/treasure-map/clue.txt
find / -name "lost-treasure" 2>/dev/null
```

Those outputs will identify the exact treasure path and the final command to solve the lab.

The clue says:

> **"The treasure is not lost, but the path to it may be. Look closely at everything in this cave."**

This means the **symlink is pointing to the wrong location**.

Run these commands:

```bash
cd /workspace/treasure-map
find cave -ls
```

or

```bash
find cave -type f -o -type l -o -type d
```

Also check what's inside the cave:

```bash
ls -lR cave
```

You're looking for the **actual treasure file** (it may have a slightly different name or location).

Once you find it, copy its contents to the required file:

```bash
cat <actual_treasure_file> > /home/admin/solution
```

If `X` is the broken symlink, you can also fix it by recreating it:

```bash
rm cave/X
ln -s <correct_target> cave/X
```

**Please paste the output of:**

```bash
find cave -ls
```

That output will reveal the correct target, and I can give you the exact command to finish the lab.

You're copying the **clue**, not the treasure.

The clue says:

> "Look closely at everything in this cave."

Also, your error:

```bash
bash: /home/admin/solution: No such file or directory
```

means the correct destination may not exist yet.

Run these commands:

```bash
pwd
ls -la
ls -la cave
ls -la /home
find /workspace/treasure-map -maxdepth 2 -ls
```

From your screenshot I also see:

```text
README.txt
cave
forest
island
treasure
```

The treasure is likely in one of those directories/files, not `clue.txt`.

A quick search:

```bash
find /workspace/treasure-map -type f -exec grep -l "treasure" {} \; 2>/dev/null
```

or simply:

```bash
find /workspace/treasure-map -type f
```

Then copy the **contents of the real treasure file** to the required location. If `/home/admin` doesn't exist, check the expected user:

```bash
ls /home
```

**Please paste the output of:**

```bash
find /workspace/treasure-map -type f
ls /home
```

That will let me give you the exact final command.

The important clue is the broken symlink:

```text
cave/X -> /workspace/treasure-map/lost-treasure
```

But there is **no `lost-treasure`**. The treasure is probably elsewhere.

Run these commands:

```bash
cat /workspace/treasure-map/README.txt
cat /workspace/treasure-map/forest/note.txt
cat /workspace/treasure-map/island/sign.txt
cat /workspace/treasure-map/treasure/flag.txt
```

(If it's actually named `flog.txt` as in your output, use:)

```bash
cat /workspace/treasure-map/treasure/flog.txt
```

One of these files will tell you the correct path or contain the treasure.

**Please paste the output of `README.txt` first**:

```bash
cat /workspace/treasure-map/README.txt
```

That will reveal the exact final command to solve the lab.

Now follow the clues in order.

Run these commands:

```bash
cat /workspace/treasure-map/island/sign.txt
cat /workspace/treasure-map/forest/note.txt
cat /workspace/treasure-map/treasure/flag.txt
```

The contents will reveal where the treasure is or the treasure message itself.

Then write the treasure message to the required file:

```bash
mkdir -p /home/admin
cat /workspace/treasure-map/treasure/flag.txt > /home/admin/solution
```

If the clue tells you to use another file instead, replace `flag.txt` with that file.

**Please paste the outputs of:**

```bash
cat /workspace/treasure-map/island/sign.txt
cat /workspace/treasure-map/forest/note.txt
cat /workspace/treasure-map/treasure/flag.txt
```

Those three files contain the remaining clues needed to finish the lab.

You have found the treasure message:

```text
Congratulations! You found the hidden treasure.
```

Now save **that exact message** into `/home/admin/solution`.

Since `/home/admin` may not exist, create it first:

```bash
mkdir -p /home/admin
cat /workspace/treasure-map/treasure/flag.txt > /home/admin/solution
```

Then verify:

```bash
cat /home/admin/solution
```

Expected output:

```text
Congratulations! You found the hidden treasure.
```

The verifier is telling you the real problem:

```text
/workspace/treasure-map/cave/X: No such file or directory
```

The challenge is **not just to read the treasure**. You must **repair the broken symlink**.

Currently:

```text
X -> /workspace/treasure-map/lost-treasure
```

but `/workspace/treasure-map/lost-treasure` does **not** exist.

Delete the broken symlink and recreate it so it points to the real treasure file:

```bash
cd /workspace/treasure-map/cave
rm X
ln -s ../treasure/flag.txt X
```

Then verify:

```bash
cat /workspace/treasure-map/cave/X
```

It should print:

```text
Congratulations! You found the hidden treasure.
```

Finally, save it to the solution file:

```bash
mkdir -p /home/admin
cat /workspace/treasure-map/cave/X > /home/admin/solution
```

Then click **Verify**.

If `cat /workspace/treasure-map/cave/X` still fails, run:

```bash
ls -l /workspace/treasure-map/treasure
```

and paste the output so I can confirm the exact filename (`flag.txt` vs another name).

You're almost there. The mistake is that you tried to **`cd` into `X`**.

`X` is a **symbolic link to a file**, not a directory, so:

```bash
cd /workspace/treasure-map/cave/X
```

will always give:

```text
Not a directory
```

Instead, verify the symlink with:

```bash
ls -l /workspace/treasure-map/cave/X
readlink /workspace/treasure-map/cave/X
cat /workspace/treasure-map/cave/X
```

If `cat` prints:

```text
Congratulations! You found the hidden treasure.
```

then create the solution:

```bash
mkdir -p /home/admin
cat /workspace/treasure-map/cave/X > /home/admin/solution
```

If `cat /workspace/treasure-map/cave/X` says **"No such file or directory"**, then the symlink target is still wrong.

Please run **only this command** and paste the output:

```bash
ls -l /workspace/treasure-map/treasure
```

That will show the exact filename so I can tell you the correct `ln -s` command.

The symlink is now pointing correctly. The issue is your commands were mistyped (`ls -1`, `readlink` got combined).

From `/workspace/treasure-map/cave`, run exactly:

```bash
cat X
```

If it prints:

```text
Congratulations! You found the hidden treasure.
```

then run:

```bash
mkdir -p /home/admin
cat X > /home/admin/solution
```

If `cat X` fails, verify the target:

```bash
readlink X
ls -l ../treasure
```

From your screenshot, I can already see `X` points to:

```text
../treasure/flag.txt
```

which is the correct target. After creating `/home/admin/solution`

You have **50 files**, where **49 are identical** and **1 file is different**.

Find the odd file:

```bash
md5sum /home/admin/data/* | sort
```

The file with the **different MD5 hash** is the modified one.

Then compare it with one of the normal files:

```bash
diff /home/admin/data/<normal_file> /home/admin/data/<different_file>
```

or use:

```bash
grep -o -w -f /home/admin/data/<different_file> /home/admin/data/<different_file>
```

The easiest method is:

```bash
diff /home/admin/data/file1 /home/admin/data/<different_file>
```

The output will show the **extra word**.

Finally, write **only that word** to the solution:

```bash
echo "<extra_word>" > /home/admin/solution
```

### Faster (recommended)

Run this command to identify the odd file:

```bash
md5sum /home/admin/data/* | sort
```
The odd file is already visible:

```text
/home/admin/data/file37.txt
```

Now compare it with any normal file, for example `file1.txt`:

```bash
diff /home/admin/data/file1.txt /home/admin/data/file37.txt
```

or just find the extra word directly:

```bash
grep -o -w -F -x -v -f /home/admin/data/file1.txt /home/admin/data/file37.txt
```

The `diff` output will show the **extra word**. Then write **only that word** to the solution:

```bash
echo "<extra_word>" > /home/admin/solution
```

If you want the exact answer, paste the output of:

```bash
diff /home/admin/data/file1.txt /home/admin/data/file37.txt
```

I'll identify the extra word immediately.

The `diff` output shows the added word:

```text
< the quick brown fox jumps over the lazy dog
> BANANA the quick brown fox jumps over the lazy dog
```

The extra word is:

```text
BANANA
```

Write **only** that word to the solution file:

```bash
echo "BANANA" > /home/admin/solution
```

Verify:

```bash
cat /home/admin/solution
```

Expected output:

```text
BANANA
```

You need to identify the IP address responsible for all the **401 Unauthorized** responses and add it to the block list.

Run:

```bash
awk '$9==401 {print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr
```

This prints the IPs that generated 401 responses, for example:

```text
13 192.168.1.100
```

Then add **only the IP address** to the config:

```bash
echo "192.168.1.100" >> /etc/nginx/blocked_ips.conf
```

Verify:

```bash
cat /etc/nginx/blocked_ips.conf
```

It should look like:

```text
# Add abusive IP addresses here
192.168.1.100
```

If you want me to tell you the exact IP, paste the output of:

```bash
awk '$9==401 {print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr
```

You just need to **start a simple HTTP server** serving `/srv/http` on **port 8080**.

First check if Python is available:

```bash
python3 --version
```

If it is, run:

```bash
cd /srv/http
python3 -m http.server 8080
```

Leave that terminal running, then in another terminal (or background it):

```bash
curl -s http://127.0.0.1:8080
```

You should get:

```html
<html>
<body>
<h1>Crux Internal Service</h1>
<p>Server is responding correctly.</p>
</body>
</html>
```

### If you only have one terminal

Start the server in the background:

```bash
cd /srv/http
python3 -m http.server 8080 >/dev/null 2>&1 &
```

Then verify:

```bash
curl -s http://127.0.0.1:8080
```

If `python3` is not installed, check for alternatives:

```bash
busybox httpd -f -p 8080 -h /srv/http
```

or

```bash
python -m SimpleHTTPServer 8080
```

For this Crucible lab, `python3 -m http.server 8080` is usually the expected solution.

You need to **generate a new certificate** and **restart the HTTPS service**.

### 1. Generate a new self-signed certificate

```bash
openssl req -x509 -newkey rsa:2048 -nodes \
-keyout /opt/crux/certs/server.key \
-out /opt/crux/certs/server.crt \
-days 365 \
-subj "/CN=crux.internal"
```

### 2. Verify the new certificate

```bash
openssl x509 -in /opt/crux/certs/server.crt -noout -enddate
openssl x509 -checkend 86400 -noout -in /opt/crux/certs/server.crt
echo $?
```

Expected:

```text
Certificate will not expire
0
```

### 3. Restart the service

First identify it:

```bash
systemctl | grep crux
```

or

```bash
ps -ef | grep 8443
```

Then restart it, for example:

```bash
systemctl restart crux
```

or

```bash
systemctl restart crux-https
```

or kill the old process if it's a standalone server:

```bash
pkill -f 8443
```

and start it again.

### 4. Verify what the server presents

```bash
echo | openssl s_client -connect 127.0.0.1:8443 2>/dev/null | openssl x509 -noout -enddate
```

The `notAfter` date should now be **in the future**.

### If restarting doesn't work

Inspect how the service is started:

```bash
grep -R "/opt/crux/certs" /etc/systemd /opt/crux 2>/dev/null
```

or

```bash
systemctl list-units | grep crux
```

This will reveal the exact service name to restart.

Your certificate file and the server are both still using the **expired certificate**.

Run these commands exactly:

```bash
openssl req -x509 -newkey rsa:2048 -nodes \
-keyout /opt/crux/certs/server.key \
-out /opt/crux/certs/server.crt \
-days 365 \
-subj "/CN=crux.internal"
```

Then verify:

```bash
openssl x509 -checkend 86400 -noout -in /opt/crux/certs/server.crt
echo $?
```

You should see:

```text
Certificate will not expire
0
```

Now **restart the HTTPS service**. First find its name:

```bash
systemctl --type=service | grep crux
```

If nothing appears:

```bash
ps -ef | grep 8443
```

or

```bash
ss -ltnp | grep 8443
```

Then restart the service (using the service name you found), or kill the process and start it again.

### If you're unsure, paste the output of these two commands:

```bash
systemctl --type=service | grep crux
ss -ltnp | grep 8443
```

Yes. You can write it as a **single line**:

```bash
openssl req -x509 -newkey rsa:2048 -nodes -keyout /opt/crux/certs/server.key -out /opt/crux/certs/server.crt -days 365 -subj "/CN=crux.internal"
```

The new certificate is generated successfully:

```text
Certificate will not expire
```

Now the server is **still using the old certificate** because it was started before you replaced the files.

You found the process:

```text
PID 23
openssl s_server -cert /opt/crux/certs/server.crt -key /opt/crux/certs/server.key
```

Kill it:

```bash
kill 23
```

Then start it again:

```bash
openssl s_server -cert /opt/crux/certs/server.crt -key /opt/crux/certs/server.key -accept 8443 -www &
```

Finally verify:

```bash
echo | openssl s_client -connect 127.0.0.1:8443 2>/dev/null | openssl x509 -noout -enddate
```

The `notAfter` date should now be **about one year in the future**, and the Crucible lab should pass.

This is another **deleted file descriptor** problem.

First identify which process has the deleted file open:

```bash
lsof +L1
```

or

```bash
ls -l /proc/*/fd/* 2>/dev/null | grep deleted
```

If the deleted file is held by PID `<pid>`, stop that process:

```bash
kill <pid>
```

If it doesn't exit:

```bash
kill -9 <pid>
```

Then verify:

```bash
ls -l /proc/<pid>/fd | grep deleted
```

It should return either:

```text
No such process
```

or **no output**.

### Quickest method

If only one process is holding a deleted file:

```bash
kill $(lsof +L1 | awk 'NR==2{print $2}')
```

This lab is straightforward. The leaking process is already identified:

```text
63 /bin/sh /opt/crux/leak.sh
```

Stop it:

```bash
kill 63
```

If it doesn't stop within a second:

```bash
kill -9 63
```

Or directly by name:

```bash
pkill -f leak.sh
```

Verify:

```bash
pgrep -af leak.sh
```

Expected output:

```text
(no output)
```

The process is likely **already stopped**, but the **TIME_WAIT** sockets haven't drained yet.

Do this:

1. Make sure the process is really gone:

```bash
pgrep -af leak.sh
```

If it still shows a PID:

```bash
kill -9 $(pgrep -f leak.sh)
```

2. Check the remaining TIME_WAIT connections:

```bash
ss -tan state time-wait | wc -l
```

3. Wait a few seconds (TIME_WAIT expires automatically):

```bash
sleep 30
```

Then click **Verify** again.

### If it still fails after 30 seconds

There may be another process recreating `leak.sh`. Check:

```bash
ps -ef | grep leak
pgrep -af sh
```

If you see `/opt/crux/leak.sh` again, kill its parent too:

```bash
pstree -p
```

or

```bash
ps -o pid,ppid,cmd -p $(pgrep -f leak.sh)
```

and kill the parent process (but **never kill PID 1**).

Please paste the outputs of:

```bash
pgrep -af leak.sh
ss -tan state time-wait | wc -l
```

You've already killed the leaking process. `pgrep -af leak.sh` shows **no output**, so that's correct.

The failure:

> **FAIL: too many TCP connections remain (26)**

means the **TIME_WAIT** sockets haven't expired yet.

Run:

```bash
ss -tan state time-wait | wc -l
```

If it still shows around 26, simply wait:

```bash
sleep 60
```

Then check again:

```bash
ss -tan state time-wait | wc -l
```

Click **Verify** once the count drops.

If it **doesn't** drop after 60 seconds, there may be another process creating connections. Check:

```bash
ss -tan state time-wait
```

and

```bash
ps -ef
```

Paste the output of:

```bash
ss -tan state time-wait | wc -l
```

If `ss -tan state time-wait | wc -l` returns **1**, then only **one TIME_WAIT connection** remains.

Wait a few more seconds:

```bash
sleep 10
```

Then verify again:

```bash
ss -tan state time-wait | wc -l
```

If it becomes **0**, click **Verify**.

If Crucible still fails even when the count is 0 or 1, paste the output of:

```bash
ss -tan state time-wait
```

This lab just requires you to **empty the directory**.

Run:

```bash
rm -rf /var/lib/crux/container-layer/*
```

If hidden files exist, also run:

```bash
find /var/lib/crux/container-layer -mindepth 1 -delete
```

Verify:

```bash
find /var/lib/crux/container-layer -type f | wc -l
```

Expected:

```text
0
```

And:

```bash
ls /var/lib/crux/container-layer | head -3
```

Expected:

```text
(no output)
```

The error:

```text
Argument list too long
```

happens because there are **50,000 files**.

Use `find` instead of `rm *`:

```bash
find /var/lib/crux/container-layer -type f -delete
```

If that doesn't work, use:

```bash
find /var/lib/crux/container-layer -mindepth 1 -exec rm -f {} \;
```

Then verify:

```bash
find /var/lib/crux/container-layer -type f | wc -l
```

It should return:

```text
0
```

And:

```bash
ls /var/lib/crux/container-layer | head
```

From the screenshots you shared, here are the Crucible SRE lab scenarios and their typical solutions. These are suitable for learning Linux troubleshooting. (For an actual assessment, expect variations rather than identical tasks.)

| Lab                              | Problem                                  | Solution                                                                                                                                                                                       |
| -------------------------------- | ---------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Name Resolution Failing**      | `registry.internal` resolves to wrong IP | Edit `/etc/hosts` so it contains `127.0.0.1 registry.internal`. If `/etc/hosts` is bind-mounted, append a correct entry (`echo "127.0.0.1 registry.internal" >> /etc/hosts`).                  |
| **Disk Full – Runaway Log**      | Huge log file filling disk               | `rm /workspace/app.log` (or `truncate -s 0 /workspace/app.log` if deletion isn't expected).                                                                                                    |
| **Permission Denied**            | Config unreadable by service user        | Fix permissions, e.g. `chmod 644 /srv/app/config.conf` (ownership stays `root:root`).                                                                                                          |
| **Disk Full, Nothing to Delete** | Deleted file still open                  | `lsof +L1` → identify PID → `kill <pid>` (or restart the process) to release space.                                                                                                            |
| **It Won't Die**                 | Process ignores normal `kill`            | `kill -9 <pid>` (SIGKILL).                                                                                                                                                                     |
| **Build Blocked by Stale Lock**  | Stale lock file                          | Verify no build process is running, then remove `/var/lib/crux/build.lock`.                                                                                                                    |
| **Treasure Map**                 | Broken symlink                           | Replace the broken symlink so `cave/X` points to the real file (e.g. `ln -sf ../treasure/flag.txt /workspace/treasure-map/cave/X`), then copy the treasure contents to `/home/admin/solution`. |
| **Needle in a Haystack**         | One file differs                         | `md5sum` to find odd file → `diff` with a normal file → write the extra word to `/home/admin/solution`.                                                                                        |
| **Block Abusive IP**             | Find IP causing HTTP 401                 | `awk '$9==401{print $1}' access.log \| sort \| uniq -c \| sort -nr` → append offending IP to `/etc/nginx/blocked_ips.conf`.                                                                    |
| **HTTP Service Down**            | No web server on port 8080               | `cd /srv/http && python3 -m http.server 8080` (or equivalent) to serve `index.html`.                                                                                                           |
| **Expired TLS Certificate**      | Expired certificate                      | Generate new certificate with `openssl req -x509 ...`, replace `server.crt`/`server.key`, restart HTTPS service.                                                                               |
| **Deleted File Descriptor**      | Deleted log still held open              | `lsof +L1` → kill/restart process holding deleted file.                                                                                                                                        |
| **TCP Connections Leaking**      | `leak.sh` creating TIME_WAIT sockets     | `pkill -f leak.sh`; wait for TIME_WAIT entries to drain.                                                                                                                                       |
| **Container Layer Full**         | 50,000 files                             | `find /var/lib/crux/container-layer -type f -delete` (don't use `rm *`, which hits "Argument list too long").                                                                                  |

### Other common Crucible SRE/Linux labs you may encounter

* Disk 100% full because of large files (`df`, `du`, `find`)
* High CPU usage (`top`, `ps`, `kill`)
* High memory usage (`free`, `ps aux --sort=-%mem`)
* Service won't start (`systemctl`, `journalctl`)
* Port already in use (`ss -tulpn`, `lsof -i`)
* SSH service down
* DNS resolution broken
* Incorrect file ownership (`chown`)
* Incorrect permissions (`chmod`)
* Cron job not running
* Read-only filesystem
* Log rotation issues
* Zombie process investigation
* Inode exhaustion (`df -i`)
* Disk I/O bottleneck (`iostat`, `iotop`)
* Nginx/Apache configuration issues
* Broken symbolic links
* File descriptor exhaustion (`ulimit`, `lsof`)
* Process restart loops
* Certificate expiration
* Stale PID files
* Network connectivity debugging (`ping`, `ip`, `ss`, `curl`, `nc`)
* Firewall blocking ports
* Listening service on wrong port
* Socket leak investigation
* Mount point missing
* Filesystem not mounted
* Corrupted `/etc/hosts`
* Log analysis with `grep`, `awk`, `sed`, `cut`, `sort`, `uniq`

### Commands worth memorizing

```bash
df -h
df -i
du -sh *
find
grep
awk
sed
sort
uniq
head
tail
less
cat
ps
top
htop
kill
kill -9
pkill
pgrep
systemctl
journalctl
ss -tulpn
lsof
lsof +L1
ip addr
ip route
ping
curl
nc
chmod
chown
ln -s
readlink
md5sum
diff
truncate
rm
find -delete
tar
gzip
openssl
python3 -m http.server
```
