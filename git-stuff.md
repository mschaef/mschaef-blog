title: Git Stuff
date: 2019-09-18
tags: tech git

Amazingly enough, [git](https://git-scm.com/) is now
[14 years old](https://en.wikipedia.org/wiki/Git). What started
out as Linus Torvald's
['three day'](https://www.linuxjournal.com/content/git-origin-story) replacement for
[BitKeeper](https://lwn.net/Articles/130746/) is now dominant enough in its
domain that even the [Windows Kernel](https://devblogs.microsoft.com/bharry/the-largest-git-repo-on-the-planet/) is hosted on git. (If you really are amazed by the age
of git, that last bit might be even more amazing.) In any event, I also use git
and have done so for close to ten years. Along with a compiler and an editor, I'd
consider it one of the three *essential* development tools. That experience has
left me with a set of preconceived notions about how git should be used
and some tips and tricks on how to use it better. I've been meaning to get it
all into a single place for a while, and this is the attempt.

This isn't really the place to *start* learning git (that would be a
[tutorial](https://git-scm.com/docs/gittutorial)). This is for people
that have used git for a while, understand the basic mechanics, and
want to look for ways to elevate their game and streamline their
workflow.

# The Underlying Data Model

git is built on a distinct data structure, and the implications of
this structure permeate the user experience.

Understanding the underlying data model is important, and not that
complicated from a computer science perspective. 

* Every revision of a source tree managed by git can be considered a
  complete snapshot of every source file. This is called a commit.
* Every commit has a name (or address), which is a hash of the entire
  contents of the commit. These names are not user friendly (They look
  like `d674bf514fc5e8301740534efa42a28ca4466afd`), but they're
  essentially guaranteed to be unique.
* If two commits have different contents, they also have different
  hashes. A hash is enough to completely identify a state of a source
  tree.
* Because hashes are a pain to work with, git also has refs. Refs are
  user friendly symbolic names (`master`, `fix-bug-branch`) that can
  each point to a commit by hash.
* Commits can't be mutated, because any change to their contents would
  change their name/hash. Refs are where git allows mutations to
  occur.
* If you think of a ref as a variable that contains a hash and points
  to a commit, you're not far off.
* Commits can themselves refer to other commits - Each commit can
  contain references to zero or more predecessors. These backlinks
  what allow git to construct a history of commits (and therefore a
  history of a source code tree).
* The 'first commit' has zero predecessors, a merge commit has two or
  more.

The result of all this is that the core data structure is a
[directed acyclic graph](https://en.wikipedia.org/wiki/Directed_acyclic_graph), 
[covered nicely in this post by Tommi Virtanen](https://eagain.net/articles/git-for-computer-scientists/).


