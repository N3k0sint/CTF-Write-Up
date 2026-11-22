# CTF OSINT Writeup: 02 Git Blame I
<img width="488" height="487" alt="image" src="https://github.com/user-attachments/assets/049baf00-3062-4ae4-b43b-aae5743490e2" />

> **Category:** OSINT | **Platform:** Outback Overflow '26   

---

## Challenge 02: Git Blame I

### 1. Challenge Overview
* **Category:** OSINT (Source Control Intelligence & Profile Reconnaissance)
* **Points:** 25 Points
* **Challenge Context:** Initial commit author investigation leading to developer profile artifacts.
* **Flag Captured:** `OO{p4tch_f1l3s_t3ll_t4l3s}`

---

### 2. Clue Deconstruction & Technical Methodology
In Git repositories, commit metadata (`Author` and `Committer`) exposes developer names, email addresses, and GitHub usernames. When developers create a repository with the exact same name as their GitHub username (`<username>/<username>`), GitHub renders that repository's `README.md` as their **Special Profile README** on their public profile page.

Developers frequently leave draft notes, reminders, or secret tokens commented out using HTML comments (`<!-- comment -->`). Because standard Markdown renderers do not display HTML comments in the browser view, these notes remain hidden unless the raw Markdown source is audited.

(At first i thought just need github profiles names, so i tried so many and wrong HAHAHA)
---

### 3. Step-by-Step Investigation Walkthrough

#### Step 1: Identifying the Developer Account
From initial platform git repositories or challenge author metadata, the developer handle was traced:
<img width="1655" height="852" alt="image" src="https://github.com/user-attachments/assets/31c5fea1-c791-42c7-b91e-845c42bee45e" />

* **Target GitHub User:** `jakeyb0i-qld`
* **Profile URL:** `https://github.com/jakeyb0i-qld`

#### Step 2: Discovering the Special Profile Repository
Navigating to the user's public repositories revealed the special profile repository:
* **Repository:** `https://github.com/jakeyb0i-qld/jakeyb0i-qld`

#### Step 3: Auditing Raw Markdown Source for Hidden Comments
Viewing the profile page in a browser displays standard rendered text (bio, projects, tech stack). However, fetching the raw source of `README.md` exposed hidden developer comments:

```bash
https://raw.githubusercontent.com/jakeyb0i-qld/jakeyb0i-qld/main/README.md
or https://github.com/jakeyb0i-qld/jakeyb0i-qld/blob/main/README.md
```

#### Step 4: Extracting the Flag
Inside the raw markdown file, an embedded HTML comment was uncovered:
<img width="797" height="382" alt="image" src="https://github.com/user-attachments/assets/6290e384-c72e-41ff-93bd-d9c939440d4c" />

```markdown
# hey, i'm jake 👋

- 🦘 IT intern, Brisbane
- 🛠️ tinkering with my homelab, pi-hole and way too many raspberry pis
- 🎮 probably playing rocket league
- 📫 don't @ me about prod

<!-- note to self: OO{p4tch_f1l3s_t3ll_t4l3s} -->
```

Flag= OO{p4tch_f1l3s_t3ll_t4l3s}

* **Evidence Files:**
  * [`evidence/git_blame_1_profile_comment.txt`](evidence/git_blame_1_profile_comment.txt)
  * [`evidence/git_blame_1_commit_trace.log`](evidence/git_blame_1_commit_trace.log)

---
