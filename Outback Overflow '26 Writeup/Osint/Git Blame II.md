# CTF OSINT Writeup: 03 Git Blame II
<img width="482" height="572" alt="image" src="https://github.com/user-attachments/assets/6168de34-8a9d-4ce4-923b-5198b0f4cd20" />

> **Category:** OSINT | **Platform:** Outback Overflow '26   

---

## Challenge 03: Git Blame II

### 1. Challenge Overview
* **Category:** OSINT (Git Archaeology & Commit Diff Analysis)
* **Points:** 25 Points
* **Target Repository:** `https://github.com/jakeyb0i-qld/homelab`
* **Flag Captured:** `OO{h1st0ry_n3v3r_f0rg3ts}`

---

### 2. Clue Deconstruction & Technical Methodology
A common operational security (OpSec) failure among developers is accidentally committing sensitive files (such as `.env`, API keys, or private SSH keys) to a Git repository, and then attempting to "fix" the mistake by running `git rm .env` followed by a new commit.

Because Git is an immutable, content-addressable object store, deleting a file in a subsequent commit only records a new tree object where the file is absent; **the file and its entire historical content remain permanently stored in the commit history graph**. Anyone who clones the repository or views the commit diffs can recover the secret.

---

### 3. Step-by-Step Investigation Walkthrough
<img width="1190" height="672" alt="image" src="https://github.com/user-attachments/assets/6e4226dc-28f0-4bf9-bfb9-5fda18bc8c4d" />

#### Step 1: Repository Reconnaissance & Cloning
We cloned the target organization's repository (`jakeyb0i-qld/homelab`):

```bash
git clone https://github.com/jakeyb0i-qld/homelab.git
cd homelab
```

#### Step 2: Auditing the Full Commit History with Line Diffs
We ran `git log -p --all` to inspect every commit along with the exact added/deleted lines:

```bash
git log -p --all
```

#### Step 3: Discovering the Deleted Secret in Commit Diffs
Inspection of commit history revealed an early commit where configuration files were initially tracked, followed immediately by a cleanup commit titled `"remove secret files from tracking"`:

```diff
commit e7b249a0d81fbc65012a912d09b62f831a2c3401
Author: Jake <jakeyb0i-qld@users.noreply.github.com>
Date:   Wed Nov 18 14:10:22 2026 +1000

    initial homelab setup

diff --git a/.env b/.env
new file mode 100644
index 0000000..f982a1c
--- /dev/null
+++ b/.env
@@ -0,0 +1,5 @@
+POSTGRES_USER=admin
+POSTGRES_PASSWORD=supersecretpass
+GRAFANA_ADMIN_PASSWORD=admin
+FLAG=OO{h1st0ry_n3v3r_f0rg3ts}
+SECRET_KEY=c392f0918b92d832
```

The subsequent commit removed `.env` and added it to `.gitignore`, but the original commit retained the secret in plain text.

Flag= OO{h1st0ry_n3v3r_f0rg3ts}
* **Evidence File:** [`evidence/git_blame_2_commit_diff.txt`](evidence/git_blame_2_commit_diff.txt)

---
