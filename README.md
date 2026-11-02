# GSoC 2026: B-tree Leaf Page Merging in PostgreSQL

## Introduction

I am Salma Elsayed, and during Google Summer of Code 2026 I worked on merging B-tree leaf pages in PostgreSQL.

After a heavy INSERT workload, B-tree pages get split. After a heavy DELETE workload, many pages are left almost empty. VACUUM only removes pages that are completely empty, so these pages stay and the index becomes bloated. Today the only fix is to rebuild the index (REINDEX).

In this project I implemented a leaf page merge algorithm. Merging is easy if you are the only database user. The hard part is doing it while other users are reading and writing the index, without blocking them and without their scans missing or repeating rows.

This project was known to be too big for GSoC from the start. Many hackers advised against taking it on, because merging B-tree pages has been an open problem in PostgreSQL for decades. Now it is on its way to being solved.

The project is not finished yet. Our minimum goal was a proof of concept, and we went far past that: the patch is not committed, but it is a working patch ([v4](https://www.postgresql.org/message-id/CANBEAPFHkuSHYDdp7%2BSGvPKvDarSHCATzoiU1wn0zwXQo6LGuw%40mail.gmail.com)) under review on pgsql-hackers, not just a proof of concept.

- **Project description**: [B-tree Index Bloat Reduction](<https://wiki.postgresql.org/wiki/GSoC_2026#B-tree_Index_Bloat_Reduction_(Page_Merge)>)
- **Proposal**: [GSoC proposal](https://docs.google.com/document/d/1PIMG0N__4BIB0uDWOfcK-AruatNL8SCTlfTlBTCaoCo/edit?usp=sharing)
- **Introduction email**: [[GSoC 2026] - B-tree Index Bloat Reduction - Introduction](https://www.postgresql.org/message-id/CANBEAPFW64QUGmMYoPekV6jGrn5Yx8kTNfeEtcTG9CCBUUBoLQ@mail.gmail.com)
- **Mailing list thread**: [[GSoC 2026] - B-tree Index Bloat Reduction - Approach & Questions](https://www.postgresql.org/message-id/flat/CANBEAPFq3YAOydjUS3xwcUG9L6e3WE5Z4nGPk_Q3RsjSFWTJNA%40mail.gmail.com)
- **Mentors**: [Kirk Wolak](https://www.linkedin.com/in/kirk-wolak-7706737/), [Andreas Karlsson](https://hackorum.dev/person/andreas.karlsson@percona.com), [Pavlo Golub](https://github.com/pashagolub)
- **Email**: salmasayed182003@gmail.com
- **LinkedIn**: [Salma Elsayed](https://www.linkedin.com/in/salma-elsayed-322063224/)

## Design overview

- L and R are two neighbor leaf pages with the same parent page, and both are almost empty, so all their tuples fit in one page.
- L's tuples are moved into R.
- L's key space is taken over by its right sibling, like in normal page deletion, so scans and inserts that arrive late can still move right.
- R becomes M (merged). L becomes MA (merged away), a tombstone that stays for a while so scans that were about to read L or R can still find their way.
- MA keeps a boundary key (L's last key), so a scan knows which tuples in R came from L.
- M saves the MA block number.
- A scan that reaches an MA or M page knows a merge happened and fixes its position, so it doesn't miss or repeat rows.
- Later, VACUUM clears the flags and deletes the MA page once no old scan can need it.
- Merges are WAL-logged, so they work after a crash and on replicas.
- Normal scans, parallel scans, and scans that change direction in the middle (like a cursor) all handle the M/MA design.
- For now, a merge is started manually with the `bt_merge()` function in pageinspect.

## Work done

1. [Proposal (Feb to Apr)](https://docs.google.com/document/d/1PIMG0N__4BIB0uDWOfcK-AruatNL8SCTlfTlBTCaoCo/edit?usp=sharing): For three months before the official GSoC period started, I worked under my mentors' guidance on leaf page [inspection tools](https://github.com/salmaaliia/postgres/commits/tooling/) (`bt_find_merge_candidates`, `bt_merge_detail`) to get an overview of how the merge would work. I also had many discussions with them about the design that I wrote in my proposal.

2. [Design discussion (May to Jul)](https://www.postgresql.org/message-id/CANBEAPFq3YAOydjUS3xwcUG9L6e3WE5Z4nGPk_Q3RsjSFWTJNA%40mail.gmail.com): At first we worked on how scans recover after they read an M or MA page. The initial design left the tuples in MA, so scans in recovery could read it like a normal page. Matthias van de Meent and Peter Geoghegan showed on pgsql-hackers that this would corrupt the index: VACUUM would not remove the TIDs from the copies left in MA, so the same heap TID could exist in two places, and scans could return rows whose TIDs were already reused. Robert Haas also pointed out that after an M page splits, the pages cannot tell which merge they belong to. This is why every M page saves the MA block number.

3. [v1 (Aug 4)](https://www.postgresql.org/message-id/CANBEAPHj3iZ97i4W9EL5TicWXRyNQ1FzK_pgxaT%3DTJbms0AHxg%40mail.gmail.com): After a lot of discussion and experiments with the code, I had the first working patch. It was a proof of concept. Recovery changed from keeping the tuples in the MA page to saving them in the scan's own memory, which made recovery simple and fixed the corruption problem.
   - MA became a tombstone page. It only keeps a `safemergexid`, which VACUUM uses later to know when the M/MA pages can be cleaned up.
   - Forward scan recovery was simple. When a scan reads an M page, it checks if it saw the MA page. If it did, no recovery is needed. If not, the scan read L before the merge, and reading M normally would return L's tuples twice. The scan still has L's tuples saved, so it reads R and removes the duplicates.
   - Backward scan recovery needed more work, but it was not too complicated. When a scan reads an MA page, it checks if it read an M page first. If it did, the MA page is skipped. If not, R was the last page read before the merge, and the scan missed L's tuples, which are now in R. The scan still has R's old tuples saved, so it goes back to the M page, reads it again, and removes R's old tuples. This way it returns L's tuples and does not miss any rows.
   - M pages can split and be deleted, but they keep the M flag and the MA block number as long as they are in the group. When an M page splits, the new page inherits them.
   - We call an MA page and its M pages a merge group. Forward and backward recovery work the same no matter how many M pages are in the group (more than one M page only happens because of splits).

4. [v2 (Aug 16)](https://www.postgresql.org/message-id/CANBEAPHRK_MqaRtSFnShS07M8fDftHmDsRrwOYqyL7%3DMFtmGvA%40mail.gmail.com): Michail Sirotkin found a bug in v1: a row inserted at the same time as a merge could not be found by later scans. The problem was in the case where a scan starts on an M page as its first page, and it was fixed in v2.

5. [v3 (Sep 15)](https://www.postgresql.org/message-id/CANBEAPFy8ig42RBqu0Y8gvi9P5n7ApyoX6dgypZ%3DAwUCPEPgTg%40mail.gmail.com): In this version I added VACUUM cleanup for the MA page and for the M flag on the M pages. Cleanup depends on the `safemergexid` saved in MA, so we are sure no scan that started before the merge is still running, and no scan still needs recovery. VACUUM must keep the merge group connected the whole time, so it clears the M flags starting from the rightmost M page going left. The MA page is cleaned last: VACUUM marks it HALF_DEAD, and then it is deleted like a normal page. I also added WAL logging for the merge and the cleanup operations. The nbtree resource manager had only 1 free WAL record type (each resource manager can have 16) and we needed 3, so we added a new nbtree2 resource manager. This is still open for discussion.

6. [v4 (Oct 28)](https://www.postgresql.org/message-id/CANBEAPFHkuSHYDdp7%2BSGvPKvDarSHCATzoiU1wn0zwXQo6LGuw%40mail.gmail.com): During this version, Nikolay Samokhvalov tested the patch with his AI test harness and found many bugs, which helped us a lot ([issues](https://github.com/NikolayS/postgres/issues/101)). In the first 3 versions, recovery only worked for normal scans. Parallel scans and scans that change direction in the middle were not handled. Saving L's tuples in the scan's memory does not work for parallel scans, because workers do not share this data: if worker A read L and saved its tuples, and then worker B reads R (M), B cannot recover. This limitation is what led us to the current design.
   - Instead of saving all of L's tuples in the scan, the merge saves the boundary tuple between L and R on the MA page, next to `safemergexid`. It is a pivot tuple similar to a high key: it has L's last key and its highest heap TID (in case it is a posting list tuple).
   - Because the boundary is stored on the page, any scan or worker can read it.
   - The scan reads the boundary tuple from the MA page and uses it to filter the tuples while reading R.
   - The scan loads all of M's tuples, including L's, but uses the boundary to decide where to start returning them. If the scan changes direction, L's tuples are already loaded, so this also solved the problem of scans that change direction.
   - Forward scan recovery: when the scan reaches an M page and needs recovery, it reads the boundary tuple from the MA page and then reads M.
   - Backward scan recovery did not change much from the earlier versions. It only uses the boundary tuple to filter, and the rest is the same.
   - Merge takes a cleanup lock on L without waiting. Index-only scans keep a pin on the page while they use its tuples, so if a scan holds one, the merge skips this pair and does not wait.
   - Merge stamps L and R with the VACUUM cycle ID, like a page split does, so a VACUUM running at the same time does not miss the tuples moved into R.
   - Mark/restore: the merge recovery state is now saved with the scan position, so restoring a mark also restores it.
   - Fixed a crash in `_bt_killitems()` that Yao Feng reported, when the page a scan just read became an MA page.
   - Added tests: amcheck checks for merged pages, and TAP tests for WAL replay and for concurrent scans during merges.

## Current status

- The patch is not committed yet. It is under review on pgsql-hackers, and the latest version is [v4](https://www.postgresql.org/message-id/CANBEAPFHkuSHYDdp7%2BSGvPKvDarSHCATzoiU1wn0zwXQo6LGuw%40mail.gmail.com).
- Code: [v4 commits](https://github.com/salmaaliia/postgres/commits/28e3e13ef08e4478be569a364d6db4f4eaa1da8b)
- 3 patches: nbtree, pageinspect, amcheck (about 5600 lines in 30 files).
- It applies on current master (as of October 2026) and passes the tests (core, pageinspect, amcheck, pg_waldump).

## How to try it

Apply the [v4 patches](https://www.postgresql.org/message-id/CANBEAPFHkuSHYDdp7%2BSGvPKvDarSHCATzoiU1wn0zwXQo6LGuw%40mail.gmail.com) on master, build, and install `contrib/pageinspect`, `contrib/pgstattuple` and `contrib/amcheck`. Then:

```sql
CREATE EXTENSION pageinspect;
CREATE EXTENSION pgstattuple;
CREATE EXTENSION amcheck;

CREATE TABLE t (id int);
INSERT INTO t SELECT generate_series(1, 1000000);
CREATE INDEX t_idx ON t (id);

-- delete 95% of the rows, spread over the whole index
DELETE FROM t WHERE id % 20 <> 0;
VACUUM t;

SELECT leaf_pages, deleted_pages, avg_leaf_density FROM pgstatindex('t_idx');
 leaf_pages | deleted_pages | avg_leaf_density
------------+---------------+------------------
       2733 |             0 |             4.73

SELECT bt_merge('t_idx', 10.0, 90.0, 100000);
 bt_merge
----------
     1357

-- VACUUM removes the MA pages. It only scans the index when there is
-- something to remove, so delete one more row first.
DELETE FROM t WHERE id = 20;
VACUUM (INDEX_CLEANUP ON) t;

SELECT leaf_pages, deleted_pages, avg_leaf_density FROM pgstatindex('t_idx');
 leaf_pages | deleted_pages | avg_leaf_density
------------+---------------+------------------
       1376 |          1357 |             9.16

SELECT bt_index_check('t_idx', true);   -- no errors
```

The merge halved the number of leaf pages. The 1357 deleted pages are reused by later page splits, so the index file keeps its size (21 MB), the same as with normal B-tree page deletion.

A page that was just merged (M) is not merged again until VACUUM clears its flag. Running `bt_merge()` and VACUUM a second time gave 683 more merges: 693 leaf pages at 17.95% full.

## Future work

- **Parallel backward scans:** there may be a deadlock in parallel backward recovery when a merge group returns a lot of rows. I am working on a fix.
- **amcheck for merge groups ([#110](https://github.com/NikolayS/postgres/issues/110)):** amcheck checks merged pages one by one, but it does not check a whole merge group or the boundary key yet.
- **Benchmarks:** measure how much bloat merging removes, and how much the merge checks cost normal scans.
- **Automatic merging:** for now a merge only runs when `bt_merge()` is called. In the end it should run on its own, for example during VACUUM.
- **Inner pages:** only leaf pages are merged. Merging inner pages is out of scope for now.

Because of this work, I found a sponsor who support me in continuing after GSoC. I plan to keep working on this patch until it gets committed.

## Challenges and lessons learned

- **Concurrent scans:** merging two pages is easy. The hard part was that scans in PostgreSQL move between pages without holding locks, so a scan can be in the middle of L or R when the merge happens. Every case (forward, backward, parallel, direction change, mark/restore) needed its own plan, and most of my time went into this.
- **Changing the design:** my first design was wrong, and the reviewers on pgsql-hackers explained why. I learned to ask early and share my design before writing a lot of code.
- **Finding race bugs:** many bugs only happen when a merge runs at the exact moment a scan moves to the next page. I learned to use injection points to stop a scan at that moment and reproduce the bug every time.
- **Working with the community:** this was my first time sending patches to pgsql-hackers. I learned how to write patch emails, answer review comments, and credit the people who found bugs.

## Acknowledgments

I want to thank my mentors **Kirk Wolak**, **Andreas Karlsson** and **Pavlo Golub**. I also want to thank my mentors outside GSoC, **Andrey Borodin**, **Nikolay Samokhvalov** and **Andrei Lepikhov**.

I also want to thank all the hackers and reviewers Matthias, Peter, Robert, Oleg, Michail, Yao Feng who helped with their reviews and advice.
