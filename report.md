# PES-VCS Lab Report
**Name:** Rakshan R  
**SRN:** PES1UG24CS365  
**Repository:** https://github.com/rakshan-2006/PES1UG24CS365-PES-VCS

---

## Phase 1: Object Storage

### Screenshot 1A — test_objects output
![Screenshot 1A](screenshots/1A.png)

### Screenshot 1B — find .pes/objects -type f
![Screenshot 1B](screenshots/1B.png)

---

## Phase 2: Tree Objects

### Screenshot 2A — test_tree output
![Screenshot 2A](screenshots/2A.png)

### Screenshot 2B — xxd of raw tree object
![Screenshot 2B](screenshots/2B.png)

---

## Phase 3: Index / Staging Area

### Screenshot 3A — pes init → pes add → pes status
![Screenshot 3A](screenshots/3A.png)

### Screenshot 3B — cat .pes/index
![Screenshot 3B](screenshots/3B.png)

---

## Phase 4: Commits and History

### Screenshot 4A — pes log with three commits
![Screenshot 4A](screenshots/4A.png)

### Screenshot 4B — find .pes -type f | sort
![Screenshot 4B](screenshots/4B.png)

### Screenshot 4C — cat .pes/refs/heads/main and cat .pes/HEAD
![Screenshot 4C](screenshots/4C.png)

### Screenshot Final — make test-integration
![Screenshot Final](screenshots/final.png)

---

## Phase 5: Branching and Checkout (Analysis)

### Q5.1 — How would you implement `pes checkout <branch>`?

To implement `pes checkout <branch>`, the following files need to change in `.pes/`:

- `.pes/HEAD` must be updated to contain `ref: refs/heads/<branchname>`
- The working directory files must be replaced with files from the target branch's commit tree

The steps would be:
1. Read the target branch file at `.pes/refs/heads/<branch>` to get the commit hash
2. Read that commit object to get its tree hash
3. Recursively walk the tree, writing each blob's contents to the correct file path in the working directory
4. Delete any files that exist in the current tree but not in the target tree
5. Update `.pes/HEAD` to point to the new branch

What makes this complex:
- You must handle files that exist in one branch but not the other (create or delete)
- You must handle subdirectories recursively
- If the user has uncommitted changes to a file that differs between branches, you must refuse and warn the user — otherwise their work is lost
- Directory creation and deletion must be handled carefully to avoid leaving empty directories

---

### Q5.2 — How to detect "dirty working directory" conflict during branch switch?

Using only the index and object store:

1. For each entry in the index, compare `mtime_sec` and `size` against the actual file on disk using `stat()`. If either differs, the file has been modified since it was staged — it is "dirty".

2. For each dirty file, look up its blob hash in the current branch's tree (by walking the HEAD commit's tree). Then look up the same file's blob hash in the target branch's tree.

3. If the two blob hashes differ AND the file is dirty on disk, that means:
   - The user has local changes
   - The two branches have different versions of that file
   - A checkout would overwrite the user's changes

4. In this case, refuse the checkout and print an error like: `error: your changes to 'file.txt' would be overwritten by checkout`

This approach requires no re-hashing of file contents — just metadata comparison for the dirty check, and hash comparison between two tree objects for the conflict check.

---

### Q5.3 — What is "detached HEAD" and how do you recover?

**Detached HEAD** means `.pes/HEAD` contains a raw commit hash directly instead of a branch reference like `ref: refs/heads/main`.

This happens when you check out a specific commit rather than a branch name.

**What happens if you commit in this state:**
- New commits are created and chained correctly
- But no branch file is updated — only HEAD itself moves
- If you then switch to another branch, HEAD stops pointing at those commits
- Those commits become unreachable — no branch file references them
- They will eventually be deleted by garbage collection

**How to recover:**
1. Before switching away, note the hash of your last detached commit:
```bash
   cat .pes/HEAD   # shows the raw hash
```
2. Create a new branch pointing to that hash:
```bash
   echo "<hash>" > .pes/refs/heads/recovery
```
3. Update HEAD to point to that branch:
```bash
   echo "ref: refs/heads/recovery" > .pes/HEAD
```

Your commits are now safe on the `recovery` branch.

---

## Phase 6: Garbage Collection (Analysis)

### Q6.1 — Algorithm to find and delete unreachable objects

**Algorithm (Mark and Sweep):**

**Mark phase:**
1. Start with all branch files in `.pes/refs/heads/` — read each commit hash
2. For each commit hash, add it to a `reachable` set
3. Read the commit object, extract its tree hash — add to reachable set
4. Recursively walk the tree: for each entry, add the blob or subtree hash to reachable set, recurse into subtrees
5. Follow the parent pointer to the previous commit, repeat until no parent

**Sweep phase:**
1. Walk every file under `.pes/objects/`
2. Reconstruct the hash from the directory name + filename
3. If the hash is NOT in the reachable set, delete the file
4. Remove any empty shard directories

**Data structure:** A hash set (implemented as a sorted array of `ObjectID` structs with binary search, or a hash table) to store reachable hashes. Lookup must be O(1) or O(log n).

**Estimate for 100,000 commits, 50 branches:**
- Each commit references ~1 tree + ~5 blobs on average = ~6 objects per commit
- 100,000 commits × 6 = ~600,000 objects to visit in the mark phase
- Total objects in store could be similar — sweep visits all of them once
- Total: roughly 1–2 million object lookups, feasible in seconds

---

### Q6.2 — Race condition between GC and concurrent commit

**The race condition:**

1. GC starts and traverses all refs. At this moment, a new blob object `X` has been written to `.pes/objects/` but the commit referencing it has NOT been written yet.
2. GC does not see `X` referenced by any commit or branch — it marks `X` as unreachable.
3. Concurrently, the commit operation writes the commit object that references `X`, then calls `head_update()`.
4. But GC already decided to delete `X` — it deletes it before or right after step 3.
5. The new commit now references a deleted object. The repository is corrupted.

**How Git avoids this:**

- **Grace period:** Git never deletes objects newer than 14 days, regardless of reachability. This means a new object written during a concurrent commit is safe even if GC misses it.
- **Write ordering:** New objects are always written to disk before any ref is updated. So by the time a branch points to a commit, all its objects exist. GC only needs to worry about objects written after the traversal started.
- **Lock files:** Git uses `.git/gc.pid` and ref lock files so concurrent operations serialize. A commit operation and GC cannot run simultaneously on the same ref.

---

## Summary

All four phases implemented and tested successfully:

| Phase | Status |
|-------|--------|
| Phase 1: Object Storage | PASS |
| Phase 2: Tree Objects | PASS |
| Phase 3: Index / Staging | PASS |
| Phase 4: Commits + History | PASS |
| Integration Test | PASS |
