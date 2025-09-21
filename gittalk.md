# Peaceful conflict resolution in version control systems

When collaborating on a codebase where several people are working in parallel on different development branches,
it often happens that different developers try to edit the same piece of code, which leads to problems when both branches have to be merged into the main codebase.

If git is used to collaborate on the codebase, the first developer to merge the branch has an easy day, and the other developer is met with a **merge conflict**,
which is the technical term used when git was unable to automatically integrate the two sets of changes together.

In this article, we present a way of thinking about codebase changes that lets us handle merge conflicts in a novel way that is much less of a hair-pulling frustrating experience.
By using this new way of thinking, we then develop a new mechanical way to work with branches that makes it easy to split off "refactoring work" from "feature work", such that refactors can be merged early, thus reducing the incidence of merge conflicts.

## The classic approach to merge conflicts

When a merge conflict happens, the developer needs to resolve the merge conflict, and unfortunately, the default approach used by git is to show a line-by-line matching of the two versions of the file, sometimes with the "base" file thrown into the mix, and asking the developer to pick which lines from each version should go into the "merged" file.
If the developer already knows what the final merged file looks like, then this task is easy, but often it is the case that the developer is not equipped to start picking lines from the two files right away;
more context is usually required to construct the final version of the file where both sets of changes have been applied.

Let's expand a bit on why git "gives up" and presents a merge conflict to the developer. Suppose you have two sets of changes, A and B, that need to be applied together to a codebase. First, A is applied to the codebase. Then an attempt is made to apply B to the codebase. If B can be applied on top of A, then all is good; if B cannot be applied, it is called a **conflict**.

The way we normally formulate "sets of changes" when interfacing with an automated system like git is using *patches*.
Patches contain line-oriented edit instructions of the form "change line 177 from `resource->version = version;` to `resource->version_ident = version;`".
The patch also contains a few context lines before and after the change, to help locate the correct place to make the line edit in case the line in question is not exactly on line 177 anymore.
In the previous example, A and B are treated by git as patches, and when git tries to apply patch B to the codebase where A has already been applied,
if a line cannot be found (or doesn't match exactly what it says in the patch it's supposed to say), git gives up and presents the "merge conflict" riddle to the developer.

## What's in a patch?

While patches are useful to communicate an exact codebase change in a way that a system can apply with as little uncertainty as possible, patches are much more literal than the way developers would normally communicate specific codebase changes to each other.

If a developer reads a patch, and is tasked with communicating to another developer what the patch does in a way that the other developer could apply the same change, the explanation will typically go something like "go to function `doFoo()` and add a new parameter `bool force`, and skip the permission check if `force` is true; go through the codebase and add `false` as an argument wherever `doFoo()` is called".

Such an explanation can then be interpreted by the other developer and applied more or less directly, and if the other developer in question is familiar with the codebase they are working on, then this process is quick and straightforward.

One key insight from this is that even though the patch formulation of A may cause the patch formulation of B to not apply cleanly when git works on the literal codebase, it might be that B can be applied cleanly to the codebase if a developer paraphrases the patch and applies this paraphrased patch to the codebase by hand.

What we have really done is taken one step towards turning the patch into a more abstract description of a codebase change.
It might be the case that even this paraphrasing of the patch is not enough to be applied cleanly, e.g. if the function to be edited no longer exists in the codebase.
In that case, we might need to make an even more abstract description of the change to be made to the codebase, such as "add a way to run the `foo` functionality without the permission check that we normally do".
One developer might take this description and add a "force" parameter, but maybe after the first patch A has been applied, this no longer makes sense, and instead we need to refactor some existing function, extracting a new function that does the operation without the check.

### Thinking about conflicting changes

Two modifications to a codebase can conflict semantically, or just syntactically.

A semantic conflict is when two developers are trying to take the codebase in different directions, e.g. Alice wants to change the app's "save" button to be green, and Bob wants to change it to be red.
In this case, only one of the two changes can "survive", and the other change's commit must be dropped (on a feature branch) or reverted (if already merged).
It is not necessarily obvious how to resolve the conflict, as it might require involving a product owner or other arbitrator to decide on which change should "win".
When such semantic conflicts happen, it is usually a sign of some missed coordination, since two developers invested effort on something that only one developer should have been working on.

A syntactic conflict is when two developers are changing the codebase in different ways but at the same place in the code, e.g. Alice wants to rename a variable, and Bob wants to extract a couple of lines of code into a new function.
In this case, the two modifications could easily be done by one developer in any order, but the fact that the two changes happened in parallel means that the computer cannot merge the changes together automatically, and instead the developer needs to be involved in merging the changes.

## Resolving conflicts peacefully

Faced with two sets of changes, A and B, that conflict syntactically,
we can now formulate how to resolve the conflict peacefully, without having to deal with git's default approach of asking the developer to pick the correct sets of lines from the two versions.
Instead of asking git to apply the patch for B on top of the codebase with A, the developer has to print out the patch for B, paraphrase it in their head, and apply the paraphrased patch to the codebase by hand.
With only minimal paraphrasing, this is a mechanical task that uses the developers reading and writing skills without requiring a lot of mental work; if minimal paraphrasing of the patch is not enough, then the developer needs to think about the patch in more abstract terms, which in turn adds more mental strain when actually applying the patch.

While it may sound tedious to apply the patch for B by hand in this way, it is useful to think about the skillset that it leverages, compared to the default merge conflict resolution approach of picking which set of lines from two versions of a file to go into the final merged file.
The point is that developers are training their reading and writing skills as part of doing programming work, whereas solving merge conflicts is only a tiny fraction of programming and thus not a skill that developers are specifically training in their day-to-day work, so why present arcane novel merge challenges, when the developer can just do some light reading and writing instead?

In school, we are taught how to show our work in deriving solutions to mathematical questions, and while this can seem tedious, it can in fact be helpful to go through a number of steps in working towards the final answer, and this applies not only to high school mathematics, but to all sorts of tasks. In this view, merge conflict resolution by picking lines from two versions is showing the final answer without showing any work, whereas merging conflicts by re-applying B by hand can be thought of as deriving the final answer by showing the intermediary steps.

## When re-applying the patch is just too tedious

Suppose another developer has already merged the change A that conflicts with your change B. However, the conflicting change A is actually just a minor refactoring in some shared code where you have made a large refactoring effort.

In this case, it would have been much less work to first merge B, and then redo A on top of B, since the A change is much less work to apply by hand.

Luckily, this is possible, since we know that the patch for B can be applied to the codebase where A is not applied:
Simply revert A, then apply the patch for B, then re-apply A by hand.
If we use A^-1 to refer to the revert of A, then the branch now contains commits that apply three sets of changes in sequence:
First A^-1, then B, and finally A.
Note that only the third commit, which applies the `A` change, has to be applied by hand, as the other two commits result from the `git revert` and `git rebase` commands.
By squashing the three commits together, we obtain a single commit which implements the `B` change, and which applies cleanly as a patch on top of the codebase with `A` already merged.

We still had to reapply a patch by hand, but we chose to reapply A by hand instead of reapplying B by hand.


## Why git?

git is a complicated tool; it is difficult to learn and difficult to master.
Nevertheless, it is very widespread in the software development world as the main version control system across open-source and proprietary ecosystems.
The git command-line interface has a lot of subcommands whose names can be difficult to make sense of,
and there are several cases of one subcommand name being able to do lots of seemingly unrelated operations.

There are also GUIs, either standalone or built into editors and IDEs, that seek to replace all day-to-day use of the standard command-line interface.

The goal of this article is to give the reader a new system of thinking about patches and codebase changes, and the formulations lead to techniques that align nicely with certain basic functionalities in git that are not as smooth or first-class in other version control systems.

This article is NOT intended to be a git tutorial, and there is no guarantee that you can take the insights from the article and apply it directly in your day-to-day life, but the hope is that the insights can help you a bit the next time you are faced with merge conflicts and codebase branch reorganizations.
There are lots of useful git tutorials online, for example you can read up on the following:

* https://git-scm.com/book/en/v2
* https://github.com/robparrott/version-control-workshop/blob/master/docs/git.md#git-the-index
* [Mark Dominus' git habits](https://blog.plover.com/prog/git-habits.html): separate the work of *composing* the commits from the work of *ordering* them.

To apply the techniques from this article consistently, it is required to have a lot of experience with git and knowledge of git, where git usage typically falls on one of the following levels of the "git ladder":


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

* Lots of `git rebase -i`, only doing one thing at a time
* `git prune` and `git gc`


## The weight of a change

Now we have outlined two good approaches to dealing with the conflict between a merged commit A and a local conflicting commit B:
Either drop B from the branch, rebase, and then redo B by hand; or revert A, apply B, and redo A by hand.
Which approach is best? Well, it depends: Would you rather redo A or B?

In general, you should prefer to redo the commit that is the quickest for you to redo.
Certain classes of changes are quite easy and fast to redo:
Changes that are done by purely mechanical approaches, such as codebase-wide search-and-replace, or IDE code refactoring actions such as "Rename variable", are easy and quick to redo; just repeat the mechanical action.
Changes that only modify a small handful of lines are also easy and quick to redo; just look at the diff for the change to redo, and do it.
Changes that mostly just delete code are easy to redo; just delete the code.

Changes that don't fall into those categories are more tedious to redo, in that they typically involve many lines, perhaps spanning several files, introducing new functionality. Redoing them by hand typically involves copying code out of the old diff and manually adapting the copied code in the places where it no longer applies cleanly to the codebase.

## Splitting and combining changes

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

## When revert-the-revert gets messy: Introducing the Hammer

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

## Using the Hammer to swap two conflicting commits

Sometimes while preparing a code change, it becomes evident that some refactoring is appropriate to do, usually in a way that requires modifying both the existing codebase and the newly changed code in the current branch.

This can result in the unwanted swapped sequence of commits B A, where B is the feature addition and A is the refactoring.

If it is not possible to merge B immediately, then it is a good idea to merge A early, to avoid conflicts when rebasing the branch later.

The traditional approach is to start a new branch where you implement the refactoring A without the feature B, merge it, and then rebase your feature branch; however, this inevitably causes conflicts.

Instead, use the Hammer to swap A and B without dealing with conflicts. There are two cases, depending on which change is lightest (in terms of effort required to apply manually):

* A: Use the Hammer to insert A A^-1 before B A, then squash A^-1 B A to obtain just B, resulting in A B.
* B^-1: Use the Hammer to insert B^-1 B after B A, then squash B A B^-1 to obtain just A, resulting in A B.

Note that this is really just a retelling of the "splitting a commit" story from earlier, since if you squash B A together, then you obtain a commit AB that you can untangle (using revert-the-revert) to obtain A B.
