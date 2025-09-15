# Git talk

git is a complicated tool; it is difficult to learn and difficult to master.
Nevertheless, it is very widespread in the software development world as the main version control system across open-source and proprietary ecosystems.
A git repository tracks the evolution of a codebase (set of text files) over time, and git makes it possible to do efficient distributed collaboration on a codebase at both small and large scales.

When two people try to change the same piece of code at the same time, e.g. one developer changing the app's "save" button to be green, and the other changing it to be red, this leads to a conflict when the developers attempt to synchronize their repositories with each other.
Such a conflict requires a decision to be made as to which change should be kept in the "main" version of the codebase.
git makes it possible to resolve such a conflict, but the default ways in which this conflict resolution is presented to the user is unintuitive.

The git command-line interface has a lot of subcommands whose names can be difficult to make sense of,
and there are several cases of one subcommand name being able to do lots of seemingly unrelated operations.

There are also GUIs, either standalone or built into editors and IDEs, that seek to replace all day-to-day use of the standard command-line interface.

This article will develop a mathematical theory of version control and derive some interesting git techniques for dealing with tricky situations.

This article is NOT intended to be a git tutorial, and you should NOT expect to learn anything useful from it.


## Aside: Some links to git tutorials

* https://git-scm.com/book/en/v2
* https://github.com/robparrott/version-control-workshop/blob/master/docs/git.md#git-the-index


## The git ladder

### Beginner

* `git clone URL`
* `git add .`
* `git commit -m "fixes and foxes"`
* `git pull`
* `git push`
* Removing the checkout and starting from scratch with a fresh clone

### Intermediate

* `git add -p`
* `git rebase origin/master`
* `git commit --amend` to change commit message or add more changes

### Experienced

* `gitk` or similar to browse history
* `git checkout -p` to remove untracked changes selectively
* `git reset --hard`
* `git reflog`
* `git reset --soft` to squash before rebasing
* `git commit -v` (or `git config commit.verbose true`) to read the diff while writing the commit message
* `git rebase -i`

### Advanced

* [Mark Dominus' git habits](https://blog.plover.com/prog/git-habits.html): separate the work of *composing* the commits from the work of *ordering* them.
* Lots of `git rebase -i`, only doing one thing at a time
* `git prune` and `git gc`


## Thinking about conflicting changes

Two modifications to a codebase can conflict semantically, or just syntactically.

A semantic conflict is when two developers are trying to take the codebase in different directions, e.g. Alice wants to change the app's "save" button to be green, and Bob wants to change it to be red.
In this case, only one of the two changes can "survive", and the other change's commit must be dropped (on a feature branch) or reverted (if already merged).
It is not necessarily obvious how to resolve the conflict, as it might require involving a product owner or other arbitrator to decide on which change should "win".
When such semantic conflicts happen, it is usually a sign of some missed coordination, since two developers invested effort on something that only one developer should have been working on.

A syntactic conflict is when two developers are changing the codebase in different ways but at the same place in the code, e.g. Alice wants to rename a variable, and Bob wants to extract a couple of lines of code into a new function.
In this case, the two modifications could easily be done by one developer in any order, but the fact that the two changes happened in parallel means that there is now a conflict to resolve.

There are several ways of resolving such a syntactic conflict.

First, the direct approach offered by the git command-line interface is to go into the codebase, where git has helpfully mixed together two versions of the file along with some "conflict markers", and then remove the markers and the unwanted code.
This is tedious and error-prone, and requires some mental gymnastics quite different from normal software development.
The goal of this article is to promote approaches to conflict resolution that lets the developer avoid having to do merges by hand, whether it's through conflict markers, or an interactive three-way view, or some other approach.

Another approach to resolving the syntactic conflict is to drop the commit that caused the conflict and redoing the commit by hand.
This means some effort has to be duplicated, as the dropped commit needs to be applied again.

The third approach is in some ways what this article is all about.
Let's call the already-merged commit A, and the local conflicting commit B.
With the third approach, you revert A, then apply B, and then redo A by hand.
Like the previous approach, this requires duplicate effort, but the difference between the two approaches is whether it is A or B that is redone by hand.

### The weight of a change

Now we have outlined two good approaches to dealing with the conflict between a merged commit A and a local conflicting commit B:
Either drop B from the branch, rebase, and then redo B by hand; or revert A, apply B, and redo A by hand.
Which approach is best? Well, it depends: Would you rather redo A or B?

In general, you should prefer to redo the commit that is the quickest for you to redo.
Certain classes of changes are quite easy and fast to redo:
Changes that are done by purely mechanical approaches, such as codebase-wide search-and-replace, or IDE code refactoring actions such as "Rename variable", are easy and quick to redo; just repeat the mechanical action.
Changes that only modify a small handful of lines are also easy and quick to redo; just look at the diff for the change to redo, and do it.
Changes that mostly just delete code are easy to redo; just delete the code.

Changes that don't fall into those categories are more tedious to redo, in that they typically involve many lines, perhaps spanning several files, introducing new functionality. Redoing them by hand typically involves copying code out of the old diff and manually adapting the copied code in the places where it no longer applies cleanly to the codebase.

### Splitting and combining changes

git makes it quite easy to combine two adjacent commits A and B into a single AB commit:
simply use the `f` (fixup) command in `git rebase -i`, or alternatively use `git reset --soft @^ && git commit --amend` if not doing an interactive rebase.

It's also possible to split a commit into several commits, e.g. using `git reset @^` followed by `git add -p`.

There's another interesting way to split a commit AB into two commits A and B, namely by reverting B by hand, reverting the revert, and then combining the revert into the original commit. If we use notation B^-1 for the revert of B, it means that on top of AB you create commit B^-1, then revert it to create B, leaving you with three commits: AB, B^-1, and B. Then you combine AB with B^-1 to form the commit A.

By alternating between the `git add -p` approach and the revert-the-revert approach, you can quickly split a large commit into tiny units of change.

One common use-case for the revert-the-revert approach is to split a refactoring step from a feature-addition step.
Suppose you have just worked on adding a new button to your app, and in the process you have refactored/tweaked some common shared functionality in the codebase.
You are not yet ready to merge the change that adds the new button, but the refactoring of the common functionality would be good to merge early, to avoid conflicts when rebasing later.
In that case, the revert-the-revert approach can be used to split the commit into commit A that does the refactoring and commit B that adds the button.
To be precise, suppose you have already committed AB, that is, a combined commit of both the refactor and the addition of the button.
Simply delete the button (and all related new code) and commit that as commit B^-1, then revert the most recent commit to form commit B.
Squash AB and B^-1 to form commit A, which is the refactoring that you can then merge.

### When revert-the-revert gets messy: Introducing the Hammer

It's not easy to do a perfect revert-the-revert in one try.
What can happen is that you try to delete all the code for the new button, commit the revert, revert the revert, and squash, but then you look at the diff of the two resulting commits A and B and see that there's still a bit of "button addition" work in commit A even though it belongs in commit B.

What you have done is not create the two pristine commits A and B, but instead created something like "A B1" and "B2", where B = B1 B2, i.e., a part of B has ended up in the first commit.
In this case it is tempting to use `git reflog` to go back in time to when you had commit "AB" (or indeed, just squash "A B1" together with B2), but that is actually not necessary.

Instead, what you must do is an interactive rebase where you insert a `b` "break" command in between the "A B1" commit and the "B2" commit.
Then the interactive rebase lets you edit the codebase on top of "A B1" before continuing with applying B2. At this point, manually revert B1 and commit to create B1^-1, and then immediately revert the revert of B1, creating the commit B1. Then continue the interactive rebase, resulting in four commits: "A B1", B1^-1, B1, and B2.
Next, use another interactive rebase to squash the first two and the last two commits, resulting in two commits "A B1 B1^-1" = A and "B1 B2" = B.

Note that the step of committing B1 and immediately reverting B1 is a no-op from the codebase's point of view, and this fact is crucial to ensure that the interactive rebase doesn't run into merge conflicts when continuing to apply B2. This is such a useful trick, and it is called The Hammer.

**The Hammer**: Insert two commits A and A^-1 at an arbitrary point in a commit sequence, usually through the `b` "break" command in an interactive rebase. Then, squash A and A^-1 into different commits in the commit sequence, using the `f` and `f -C` commands in an interactive rebase.

Note that it is pointless to write a detailed commit message for commit A, since A and A^-1 are anyway going to be squashed into other commits just moments after being committed. For this reason it can be useful to have a shell alias ready to commit the change and immediately revert it: `git commit -am HAMMER && git revert --no-edit @`.

The Hammer can be used to iteratively clean up a revert-the-revert that has gone messy, but it should be thought of as a general tool for moving changes from one commit into another.

### Using the Hammer to swap two conflicting commits

Sometimes while preparing a code change, it becomes evident that some refactoring is appropriate to do, usually in a way that requires modifying both the existing codebase and the newly changed code in the current branch.

This can result in the unwanted swapped sequence of commits B A, where B is the feature addition and A is the refactoring.

If it is not possible to merge B immediately, then it is a good idea to merge A early, to avoid conflicts when rebasing the branch later.

The traditional approach is to start a new branch where you implement the refactoring A without the feature B, merge it, and then rebase your feature branch; however, this inevitably causes conflicts.

Instead, use the Hammer to swap A and B without dealing with conflicts. There are two cases, depending on which change is lightest (in terms of effort required to apply manually):

* A: Use the Hammer to insert A A^-1 before B A, then squash A^-1 B A to obtain just B, resulting in A B.
* B^-1: Use the Hammer to insert B^-1 B after B A, then squash B A B^-1 to obtain just A, resulting in A B.

Note that this is really just a retelling of the "splitting a commit" story from earlier, since if you squash B A together, then you obtain a commit AB that you can untangle (using revert-the-revert) to obtain A B.
