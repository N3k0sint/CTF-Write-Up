# CTF OSINT Writeup: 05 Git Blame IV
<img width="466" height="652" alt="image" src="https://github.com/user-attachments/assets/b1ad2275-5c19-4609-85c5-612f0ece3a30" />

> **Category:** OSINT | **Platform:** Outback Overflow '26   

---

## Challenge 05: Git Blame IV

### 1. Challenge Overview
* **Category:** OSINT (Force-Push Recovery & Dangling Commit Reconstruction)
* **Points:** 75 Points
* **Target Repository:** `https://github.com/jakeyb0i-qld/homelab`
* **Flag Captured:** `OO{f0rc3_push3d_n0t_f0rg0tt3n}`

---

### 2. Clue Deconstruction & Technical Methodology
When a developer realizes they committed a sensitive credential or made an erroneous change, they may rewrite Git history using `git reset --hard HEAD~1` followed by `git push --force`. This removes the commit from the branch's reference head (`refs/heads/main`).

However, on GitHub:
1. **The commit object is NOT deleted.** It becomes a **dangling (orphaned) commit** in GitHub's underlying Git object database.
2. **The GitHub Activity / Events API permanently logs the force-push event.** The API event payload records both the `before` SHA (the orphaned commit that was erased) and the `after` SHA.
3. Once the orphaned SHA is obtained, GitHub allows anyone to view and download the commit diff directly by visiting `/commit/<sha>.patch`.

---

### 3. Step-by-Step Investigation Walkthrough
<img width="1157" height="695" alt="image" src="https://github.com/user-attachments/assets/7c85a36f-3e06-453e-bebb-2a1efce53383" />

#### Step 1: Querying the GitHub Repository Activity API
We queried the public Activity API endpoint for `jakeyb0i-qld/homelab`:

```bash
curl -s https://github.com/jakeyb0i-qld/homelab/activity \
     -H "Accept: application/vnd.github+json" > activity.json
```

#### Step 2: Filtering for `force_push` Events
Inspecting `activity.json` revealed a `force_push` event recorded on branch `refs/heads/main`:

```json
{
  "id": 12849102841,
  "activity_type": "force_push",
  "actor": {
    "login": "jakeyb0i-qld"
  },
  "ref": "refs/heads/main",
  "timestamp": "2026-11-19T09:42:15Z",
  "before": "a97405364ce0be3d690a2333792cb0023a85ee7c",
  "after": "6f29e102830182cba91820394812304918230192"
}
```

* **Dangling Commit SHA (`before`):** `a97405364ce0be3d690a2333792cb0023a85ee7c`

#### Step 3: Fetching the Dangling Commit via Patch Endpoint
Because the commit had been stripped from branch references, browsing the repository tree would never show it. However, querying the commit's `.patch` representation directly forced GitHub's backend to serve the object:

```bash
curl -s https://github.com/jakeyb0i-qld/homelab/commit/a97405364ce0be3d690a2333792cb0023a85ee7c.patch
```

#### Step 4: Extracting the Recovered Secret
The raw patch output revealed the erased commit diff:

```diff
From a97405364ce0be3d690a2333792cb0023a85ee7c Mon Sep 17 00:00:00 2001
From: Jake <jakeyb0i-qld@users.noreply.github.com>
Date: Thu Nov 19 09:38:04 2026 +1000
Subject: [PATCH] add production backup script with auth token

diff --git a/scripts/backup.sh b/scripts/backup.sh
new file mode 100755
index 0000000..831a92e
--- /dev/null
+++ b/scripts/backup.sh
@@ -0,0 +1,6 @@
+#!/usr/bin/env bash
+# Automated backup script
+BACKUP_SERVER="backup.internal.outbackcorp.local"
+AUTH_TOKEN="OO{f0rc3_push3d_n0t_f0rg0tt3n}"
+tar -czf /tmp/homelab-backup.tar.gz /opt/homelab
+curl -H "X-Auth: $AUTH_TOKEN" -F "file=@/tmp/homelab-backup.tar.gz" https://$BACKUP_SERVER/upload
```

* **Flag Captured:** `OO{f0rc3_push3d_n0t_f0rg0tt3n}`

* **Evidence Files:**
  * [`evidence/git_blame_4_activity_api.json`](evidence/git_blame_4_activity_api.json)
  * [`evidence/git_blame_4_dangling_commit.patch`](evidence/git_blame_4_dangling_commit.patch)

---
