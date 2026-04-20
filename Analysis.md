# PES-VCS Report
**Author:** Guntupalli Manaswi
**SRN/Username:** PES1UG24CS548

## Phase 5: Branching and Checkout

**Q5.1: How would you implement `pes checkout <branch>`?**
A branch in Git is just a file under `.pes/refs/heads/` (e.g., `main`). To implement checkout:
1. **Change HEAD:** Update `.pes/HEAD` to contain the new symbolic reference `ref: refs/heads/<branch>`.
2. **Update Working Directory:** Look up the branch's target commit hash, load its corresponding Tree object, and update the working directory to match that snapshot perfectly by creating/updating files to match the blob contents.
3. **Complexity:** The operation is complex because it must preserve untracked files, safely remove files tracking in the old branch but missing in the new branch, and ensure that we do not clobber user's uncommitted work inside tracked files.

**Q5.2: How would you detect a "dirty working directory" conflict?**
Using only the **index** and the **object store**:
1. Iterate over the `Index` array. Compare the recorded `mtime` and `size` in the index against `stat()` derived metadata on the actual file. If they differ locally, recompute the actual file's hash to confirm if it has genuinely been modified.
2. Read the target branch's commit tree recursively to find the `ObjectID` (hash) for that file. 
3. If the local file is modified (dirty) AND its targeted destination tree hash differs from its current indexed base hash, there is a conflict. Refuse the checkout!

**Q5.3: What happens on "Detached HEAD" commits?**
If HEAD contains a hash instead of a branch ref (`ref: ...`), creating commits builds a new history chain where HEAD travels along with new commits. However, since no persistent branch file pointer tracks this chain, checking out another branch will immediately strand the commits, making them "unreachable".
**Recovery:** Users can recover them using `git fsck --lost-found` (or finding dangling commits), manually viewing the CLI terminal scrollback for the lost hashes, or inspecting `git reflog`—then creating a branch pointer directly assigning the lost commit hash.

## Phase 6: Garbage Collection (GC) and Space Reclamation

**Q6.1: Algorithm to find and delete unreachable objects?**
*Algorithm (Mark and Sweep):*
1. **Mark Phase:** Iterate through all branch references in `.pes/refs/heads/*` and the `HEAD` reference file. Follow each commit, marking its hash and its parent commits' hashes. From every marked commit, recursively parse its root tree, sub-trees, and blobs, marking them all as "reachable".
2. **Sweep Phase:** Walk through all subdirectories in `.pes/objects/`. Any file whose hash name isn't marked as "reachable" gets deleted.
*Data Structure:* A **Hash Set** efficiently tracks the `32-byte` ObjectIDs during reachability marking.
*Estimation:* For 100,000 commits & 50 branches, many trees/blobs are heavily duplicated and shared. The number of unique reachable objects might be on the magnitude of millions, but visiting each is fast with an O(1) in-memory Hash Set.

**Q6.2: Why is concurrent GC dangerous?**
*Race Condition:* When a user calls `pes commit`, the command creates `blob` files, `tree` files, and finally a `commit` object. *Before* it successfully updates the branch reference file (`head_update`), a concurrent GC might sweep the repository. Because the new objects aren't pointed to by any branch yet, GC will see them as unreachable and delete them mid-commit!
*Git's Fix:* Real Git handles this by adding a **grace period** (only purging loose objects older than a specific timeframe, default 2 weeks). This mathematically guarantees active transactions won't have newly created objects deleted. Git also uses reference logs (`reflog`) to keep temporary "reachability" anchors.

---
### Artifact Images / Screenshot Instructions
- **1A, 1B**: Display passing tests and object shard output.
- **2A, 2B**: Demonstrate tree tests and `xxd` binary structure outputs.
- **3A, 3B, 4A, 4B, 4C, Final**: Staging operations, commit traversal, and integration pipelines passing reliably.
