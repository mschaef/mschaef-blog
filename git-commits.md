title: Git Commits and Rebasing vs. Merging
date: 2022-03-05
tags: tech git

Over most of the ten years I've been using `git`, I've been a strong
proponent of merging over rebasing. It seemed more honest to avoid
rewriting commits and more likely to produce a complete history.
There are also problems that arise when you rewrite shared history,
and you can avoid those entirely if you just never rewrite history at
all. While all of this is true, the hidden costs of the approach came
to play an increasing role in my thinking, and these days, I
essentially avoid merge entirely. The result has been an easier
workflow, with a more useful history of more coherent commits.

History tracking in a tool like `git` serves a few development
purposes, some tactical and some strategic.  Tactically speaking, it's
nice to be able to have confidence that you can always reset to a
particular state of the codebase, no matter how badly you've screwed
it up. It's easier to make "risky" changes to code when you know that
you're a split second away from your last known good state. Further,
git remotes give you easy access to a form of off site backup and tags
give you the ability to label released. Not only does the history in a
tool like `git` make it easier to get to your last known good state during
development, it also makes it easier to get back to the version you
released last month before your dog destroyed your laptop.

At a strategic level, history tracking can give other longer term
benefits. With a little effort, it's an excellent way to document the
how and way your code evolves over time. Correctly done (and with an
IDE), a good version history gives developers immediate access to the
origin of each line of code, along with an explanation of how and why
it got there. Of course, it takes effort to get there. Your history
can easily devolve into a bunch of `"WIP"` messages in a randomly
associated stream of commits. Like everything else in life worth
doing, it takes effort to ensure that you actually have a commit
history that can live up to its strategic value.

This starts with a commit history that people bother to read, and like
everthing else, it takes effort to produce something worth reading.
For people to bother reading your commit history, they need to believe
that it's worth the time spent to do so. For that to happen, enough
effort needs to have been spent assembling the history that it's
possible to understand what's being said. This is where the notion of
a _complete history_ runs into trouble. Just like historians curate
facts into readable narratives, it is our responsibility as developers
to take some time to curate our projects' change history. At least if
we expect them to be read.  My argument for rebasing over merging
boils down to the fact that rebase/squash makes it easier to do this
curation and produce a history that has these useful properties.

For a commit to be useful in the future as a point of documentation,
it needs to contain a coherent unit of work. `git` thinks in terms of
commits, so it's important that you also think in terms of
commits. Being able to trust that a single commit contains a complete
single change is usetul both from the point of view of interpreting a
history, and also from the point of view of using `git` to manipulate
the history. It's easier to `cherry-pick` one commit with a useful
change than it is three commits, each with a part of that one change.

Another way of putting this is that nobody cares about the history of
how you developed a given feature. Imagine adding a field to a
screen. You make a back end change in one commit, a front end change
in the next, and then submit them both in one branch as a PR. A year
after, does it really matter to you or to anybody else that you
modified the back end first and then the front end? The two commits
are just noise in the history. They document a state that never
existed in anything like a production environment.

These two commits also introduce a certain degree of ongoing
risk. Maybe you're trying to backport the added field into an earlier
maintenance release of your software. What happens if you
`cherry-pick` just one of the two commits into the maintenance
release?  Most likely, that results in a wholly invalid state that you
may or may not detect in testing. Sure, the two commits honestly
documented the history, but there's a cost. You lose documentation of
the fact that both the front and back end changes are necessary parts
of a single whole.

Given this argument for squashing, or _curating_, commits into useful
atomic units, development branches largely reduce down to single
commits. You may have a sequence of commits during development to
personally track your work, but by the time you merge, you've squashed
it down to one atomic commit describing one useful change. This
simplifies your history directly, but it also makes it easier to
rebase your evelopment branch. Rebasing a branch with a single commit
avoids introducing historical states that "never existed". The single
commit also dramatically simplifies the process of merge conflict
resolution.  Rebase a branch with 10 commits, and you may have 10 sets
of merge conflicts to resolve. Do you really care about the first
nine?  Will you really go back to those commits and verify that they
still work post-rebase? If you don't, you're just dumping garbage in
your commit log that might not even compile, much less run. 

I'll close with the thought that this approach also lends itself to
better commit messages. If there are fewer commits, there are fewer
commit messages to write. With fewer commit messages to write, you can
take more time on each to write something useful. It's also easier to
write commit messages when your commits are self-contained atomic
units. Squashing and curating commits is useful by itself in that it
leads to a cleaner history, but it also leads to more opportunities to
produce good and useful commit messages. It points in the direction of
a virtuous cycle where positive changes drive other positive changes.

[git]: https://git-scm.com/
