- Feature Name: rtic-git-rebuild
- Start Date: 2021-03-22
- RFC PR: [rtic-rs/rfcs#0000](https://github.com/rtic-rs/rfcs/pull/0000)
- RTIC Issue: [rtic-rs/cortex-m-rtic#0000](https://github.com/rtic-rs/cortex-m-rtic/issues/0000)

# Summary
[summary]: #summary

Size of git repository is too large due to historic CI-generated build artifacts.

In order to maintain a good user experience and not needlessly consume bandwidth/CI-time
a git history cleanup is warranted.

By using `git filter-repo` to delete the offending CI-build artifacts the size of the repo
can be reduced by a factor 600 and the number of trees by a factor 63.

# Motivation
[motivation]: #motivation

The user experience downloading the repo is far from ideal with a ~180 MiB download.

Due to limitations of the current libgit2 implementation, see [Issue 3058][libgit2],
shallow clones are not supported in cargo [Issue 1171][cargo-shallow].

The size on disk is not the primary concern, the number of trees are the primary problem
according to [`git-sizer`][git-sizer].

[libgit2]: https://github.com/libgit2/libgit2/issues/3058
[cargo-shallow]: https://github.com/rust-lang/cargo/issues/1171
[git-sizer]: https://github.com/github/git-sizer

# Detailed design
[design]: #detailed-design

Cloning `cortex-m-rtic` with standard options over SSH:

``` shell
❯ git clone git@github.com:rtic-rs/cortex-m-rtic.git
Cloning into 'cortex-m-rtic'...
remote: Enumerating objects: 140, done.
remote: Counting objects: 100% (140/140), done.
remote: Compressing objects: 100% (73/73), done.
remote: Total 1389138 (delta 91), reused 102 (delta 67), pack-reused 1388998
Receiving objects: 100% (1389138/1389138), 179.40 MiB | 12.28 MiB/s, done.
Resolving deltas: 100% (1381616/1381616), done.

/tmp took 38s
```

On reasonable fast network the issue is not so prominent, but sometimes we
 do not have the luxury of fast connectivity:

```
❯ git clone git@github.com:rtic-rs/cortex-m-rtic.git
Cloning into 'cortex-m-rtic'...
remote: Enumerating objects: 141, done.
remote: Counting objects: 100% (141/141), done.
remote: Compressing objects: 100% (74/74), done.
remote: Total 1389117 (delta 91), reused 102 (delta 67), pack-reused 1388976
Receiving objects: 100% (1389117/1389117), 179.40 MiB | 762.00 KiB/s, done.
Resolving deltas: 100% (1381605/1381605), done.

/tmp took 4m29s 
```

Looking at the size of the repo with the help of `git sizer`:

```
cortex-m-rtic on  master [$?] is 📦 v0.6.0-alpha.1 via R v1.50.0 took 6s 
❯ git sizer -v 
Processing blobs: 1837488                        
Processing trees: 8814                        
Processing commits: 1184                        
Matching commits to trees: 1184                        
Processing annotated tags: 24                        
Processing references: 234                        
| Name                         | Value     | Level of concern               |
| ---------------------------- | --------- | ------------------------------ |
| Overall repository size      |           |                                |
| * Commits                    |           |                                |
|   * Count                    |  1.18 k   |                                |
|   * Total size               |   439 KiB |                                |
| * Trees                      |           |                                |
|   * Count                    |  8.81 k   |                                |
|   * Total size               |  77.7 MiB |                                |
|   * Total tree entries       |  1.92 M   |                                |
| * Blobs                      |           |                                |
|   * Count                    |  1.84 M   | *                              |
|   * Total size               |  9.19 GiB |                                |
| * Annotated tags             |           |                                |
|   * Count                    |    24     |                                |
| * References                 |           |                                |
|   * Count                    |   234     |                                |
|                              |           |                                |
| Biggest objects              |           |                                |
| * Commits                    |           |                                |
|   * Maximum size         [1] |  5.37 KiB |                                |
|   * Maximum parents      [2] |     4     |                                |
| * Trees                      |           |                                |
|   * Maximum entries      [3] |  3.30 k   | ***                            |
| * Blobs                      |           |                                |
|   * Maximum size         [4] |  2.06 MiB |                                |
|                              |           |                                |
| History structure            |           |                                |
| * Maximum history depth      |   685     |                                |
| * Maximum tag depth      [5] |     1     |                                |
|                              |           |                                |
| Biggest checkouts            |           |                                |
| * Number of directories  [6] |  1.19 k   |                                |
| * Maximum path depth     [6] |    15     | *                              |
| * Maximum path length    [6] |   127 B   | *                              |
| * Number of files        [7] |  60.9 k   | *                              |
| * Total size of files    [7] |   399 MiB |                                |
| * Number of symlinks         |     0     |                                |
| * Number of submodules       |     0     |                                |

[1]  e925a3e38f493b835356fb94dc118236ed217708 (refs/remotes/origin/root)
[2]  9a974585d0c3e20e860e5d1ad4cf9df501d229d9
[3]  fa5a6b378d3660eb8a7b0198ddb857f67e3b7ae6 (refs/remotes/upstream/gh-pages:0.4/api/typenum)
[4]  ae0c70624634faa6581abb505c4436c3b5f82402 (5a331d3e75a697a127670085555ccb1a1722c4f2:0.5/api/src/typenum/home/travis/build/rtfm-rs/cortex-m-rtfm/target/debug/build/typenum-045f2683e9dd584d/out/consts.rs.html)
[5]  03adc6ed6d3ad64d0656f25a19ad31fb3860bb56 (refs/tags/v0.1.0)
[6]  7b537a86e93b90f4b42a419a6ceb88dd24ae6530 (refs/remotes/upstream/gh-pages^{tree})
[7]  8bb490aabf403c00db339d51aded85727e9338a5 (6976dc28b54bb98f03eaa3ac1ea5abb4d4bfff7a^{tree})
```

`git filter-repo` has an analyze mode, allowing us to find the offending paths,
another hint is found in the footnotes provided by `git-sizer`:

```
= == All paths by reverse accumulated size ===
Format: unpacked size, packed size, date deleted, path name
    27733613     815159 <present>  0.5/api/search-index.js
    16498260     708489 <present>  dev/book/en/searchindex.json
    40097673     644207 <present>  0.4/api/typenum/consts/index.html
    18362175     533997 <present>  0.4/api/search-index.js
    12012177     477375 <present>  0.5/book/en/searchindex.json
    19857155     418542 <present>  dev/api/search-index.js
     8218009     291244 <present>  0.5/api/src/syn/expr.rs.html
     8218009     291244 <present>  0.4/api/src/syn/expr.rs.html
     4683688     287942 <present>  stable/book/en/searchindex.json
    13828791     279431 <present>  stable/api/search-index.js
     2750259     208897 <present>  dev/book/ru/searchindex.json
     4335695     198587 <present>  stable/api/src/syn/expr.rs.html
     4335695     198587 <present>  dev/api/src/syn/expr.rs.html
      186824     186480 <present>  stable/api/FiraSans-Medium.woff
      186824     186480 <present>  dev/api/FiraSans-Medium.woff
      186824     186480 <present>  0.5/api/FiraSans-Medium.woff
      186824     186480 <present>  0.4/api/FiraSans-Medium.woff
     2629486     186096 <present>  0.5/api/src/syn/parse.rs.html
     2629486     186096 <present>  0.4/api/src/syn/parse.rs.html
     4401622     184920 <present>  0.5/api/src/hashbrown/map.rs.html
     2032590     183640 <present>  0.5/book/ru/searchindex.json
      183268     182795 <present>  stable/api/FiraSans-Regular.woff
      183268     182795 <present>  dev/api/FiraSans-Regular.woff
      183268     182795 <present>  0.5/api/FiraSans-Regular.woff
      183268     182795 <present>  0.4/api/FiraSans-Regular.woff
     8350224     180479 <present>  0.5/api/src/syn/item.rs.html
     8350224     180479 <present>  0.4/api/src/syn/item.rs.html
     1962315     169270 <present>  0.5/api/src/hashbrown/raw/mod.rs.html
< cut >
```

Running `git filter-repo` removing the following paths,
 all containing old and deleted CI artifacts:

* 0.5
* 0.4
* dev
* stable

```
❯ git filter-repo --path 0.5 --path 0.4 --path dev --path stable --invert-paths          
Parsed 1161 commits
New history written in 8.13 seconds; now repacking/cleaning...
Repacking your repo and cleaning out old unneeded objects
Enumerating objects: 7687, done.
Counting objects: 100% (7687/7687), done.
Delta compression using up to 6 threads
Compressing objects: 100% (2679/2679), done.
Writing objects: 100% (7687/7687), done.
Selecting bitmap commits: 1035, done.
Building bitmaps: 100% (115/115), done.
Total 7687 (delta 4815), reused 7686 (delta 4815), pack-reused 0
Completely finished after 8.61 seconds.
```

The following size is achieved:

```
~/rust/cortex-m-rtic-bare-mirror-backup-wip2 took 8s 
❯ git sizer -v                                                                 
Processing blobs: 3517                        
Processing trees: 3056                        
Processing commits: 1088                        
Matching commits to trees: 1088                        
Processing annotated tags: 26                        
Processing references: 1100                        
| Name                         | Value     | Level of concern               |
| ---------------------------- | --------- | ------------------------------ |
| Overall repository size      |           |                                |
| * Commits                    |           |                                |
|   * Count                    |  1.09 k   |                                |
|   * Total size               |   341 KiB |                                |
| * Trees                      |           |                                |
|   * Count                    |  3.06 k   |                                |
|   * Total size               |  1.20 MiB |                                |
|   * Total tree entries       |  33.4 k   |                                |
| * Blobs                      |           |                                |
|   * Count                    |  3.52 k   |                                |
|   * Total size               |  15.5 MiB |                                |
| * Annotated tags             |           |                                |
|   * Count                    |    26     |                                |
| * References                 |           |                                |
|   * Count                    |  1.10 k   |                                |
|                              |           |                                |
| Biggest objects              |           |                                |
| * Commits                    |           |                                |
|   * Maximum size         [1] |  5.37 KiB |                                |
|   * Maximum parents      [2] |     4     |                                |
| * Trees                      |           |                                |
|   * Maximum entries      [3] |    52     |                                |
| * Blobs                      |           |                                |
|   * Maximum size         [4] |  68.2 KiB |                                |
|                              |           |                                |
| History structure            |           |                                |
| * Maximum history depth      |   709     |                                |
| * Maximum tag depth      [5] |     1     |                                |
|                              |           |                                |
| Biggest checkouts            |           |                                |
| * Number of directories  [6] |    31     |                                |
| * Maximum path depth     [7] |     5     |                                |
| * Maximum path length    [8] |    54 B   |                                |
| * Number of files        [9] |   199     |                                |
| * Total size of files    [9] |   431 KiB |                                |
| * Number of symlinks         |     0     |                                |
| * Number of submodules       |     0     |                                |

[1]  e925a3e38f493b835356fb94dc118236ed217708 (refs/heads/root)
[2]  63ae34109355535db83cdbde11d82eebeb2ea9fd (refs/replace/9a974585d0c3e20e860e5d1ad4cf9df501d229d9)
[3]  4c68f4ad6c6073674fe438f4b3a3dd05ed9d2313 (refs/heads/goodby_static_mut:examples)
[4]  dac4fb1d05997a63ec691e333e77729b08e13d6a (refs/heads/gh184:macros/src/codegen.rs)
[5]  03adc6ed6d3ad64d0656f25a19ad31fb3860bb56 (refs/tags/v0.1.0)
[6]  24b56a21f7ce11f62029bd0bef8ca4d3082140c9 (refs/heads/v0.5.x^{tree})
[7]  f9d698f900b6b04b2cc91d7fcb8a5e0bfde4d522 (refs/pull/456/merge^{tree})
[8]  45ae48ed16c09b067f0c5403103f3a944eda1cd0 (refs/heads/goodby_static_mut^{tree})
[9]  56477b57a762bdb9519c97119d4c627c5f93df7a (refs/heads/immutable_resource_proxies^{tree})
```

The change of "Blobs - Total size" is significant, going from

```
| * Blobs                      |           |                                |
|   * Count                    |  1.84 M   | *                              |
|   * Total size               |  9.19 GiB |                                |
```

to

```
| * Blobs                      |           |                                |
|   * Count                    |  3.52 k   |                                |
|   * Total size               |  15.5 MiB |                                |
```

A similar reduction can be observed for trees, maximum entries changes from 3.30k to only 52.

The problem is that `git filter-repo` needs to rewrite history in order to achieve this.

# How We Teach This
[how-we-teach-this]: #how-we-teach-this

In order to prevent the reintroduction of old history into the newly cleaned repo the recommended solution
is to force all users to freshly clone the repo, see point 4 in [`git filter-repo`][git-filter-repo].

Looking at the `cortex-m-rtic` repo [GitHub insights network graph][gh-network] the majority of users have old forks.

Adding a large and bold note in README instructing users to cleanly clone it.

[gh-network]: https://github.com/rtic-rs/cortex-m-rtic/network

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

Rewriting history is generally risky.

As noted in the Discussion section of the [man-pages][git-filter-repo] of `git filter-repo`,
rewriting history in a git project is tricky business.

This immediately becomes the “Hard case” as indicated in [git-rebase documentation][git-rebase].

[git-filter-repo]: https://www.mankier.com/1/git-filter-repo#Discussion
[git-rebase]: https://www.mankier.com/1/git-rebase#Recovering_from_Upstream_Rebase-The_hard_case

# Alternatives
[alternatives]: #alternatives

What other designs have been considered? What is the impact of not doing this?

Not doing this a lot of junk will just pollute git history and make git slower than necessary.

# Unresolved questions
[unresolved]: #unresolved-questions

What parts of the design are still TBD?

How to best communicate the need to completely re-clone the repo.
