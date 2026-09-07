# Git Commands Reference — Everything Used So Far

Every git command used across all SOC lab sessions, not just today. Grouped by
purpose, with a general explanation of what each one does.

## One time setup

**git init**
Turns the current folder into a git repository, creating a hidden .git folder
that will track all future history. Only needs to be run once, ever, per
repository.

**git config --global user.name "Your Name"**
Sets the name that will be attached to every commit you make on this computer,
for every repository. This is metadata only, it does not affect GitHub login.

**git config --global user.email "your@email.com"**
Sets the email attached to every commit, same as above, applies to every
repository on this computer.

**git config --global --list**
Shows all of your current global git settings, useful for confirming that
user.name and user.email actually saved correctly.

**ssh-keygen -t ed25519 -C "a label"**
Not a git command itself, but part of setting up SSH based authentication with
GitHub. Generates a new public/private key pair. The private key stays on your
computer, the public key gets added to GitHub so it can recognize your
machine.

**ssh -T git@github.com**
Tests whether your SSH key is correctly set up and recognized by GitHub. A
successful response confirms authentication is working, without actually
doing anything to a repository.

## Connecting a local repo to GitHub

**git remote add origin url**
Tells your local repository where its remote counterpart lives on GitHub.
"origin" is just the conventional name for this connection, you could name it
something else, but almost everyone uses "origin."

**git remote -v**
Shows which remote repository your local repo is connected to, and confirms
the URL. Useful for double checking you are pointed at the correct repository
before pushing anything.

**git clone url destination**
Downloads an entire existing repository from GitHub onto your computer for
the first time, including all of its history. This is the command used to
get a fresh copy of a repo onto a brand new machine.

## Checking status and history

**git status**
Shows what branch you are currently on, and lists any files that are staged,
modified, or untracked. This is the command to run any time you want to know
the current state of your working folder before doing anything else.

**git branch**
Lists all of your local branches. The branch you are currently standing on is
marked with an asterisk.

**git branch -a**
Same as git branch, but also lists remote branches, meaning the branches that
exist on GitHub, so you can see everything in one list.

**git log --oneline**
Shows the commit history, one line per commit, with a short ID and the commit
message. Adding a number like -5 limits it to showing only the most recent 5
commits. You can point it at a specific branch name, like git log master
--oneline, to see that branch's history specifically. You can also point it
at a remote branch, like git log origin/master --oneline, to see what GitHub's
copy looks like without switching anything locally.

**git ls-files**
Lists every file git currently tracks in the repository, meaning files that
have been committed at least once. A file that's only sitting in the folder
but has never been added and committed will NOT show up here — this is
different from git status, which shows untracked files too. Useful for
confirming a file was never actually committed before relying on .gitignore
alone to keep it out of the repo. Can be combined with a filter, for example
git ls-files | findstr /I "keyword" on Windows, to search the tracked list
for a specific name without scrolling through everything.

**git show --stat HEAD**
Shows details about the most recent commit specifically, including exactly
which files it changed and how many lines were added or removed in each,
without showing the full line by line content.

## Syncing with the remote

**git fetch origin**
Reaches out to GitHub and downloads information about what commits exist
there, without changing any of your own files or branches. This just updates
your local knowledge of what's actually out on GitHub.

**git pull origin master**
Downloads GitHub's current version of a branch, master in this case, and
applies those changes to your local copy of that branch. This is really a
fetch plus a merge done in one step. You have to be standing on the branch you
want updated for this to apply where you expect.

**git push -u origin master**
Uploads your local commits to GitHub for the very first time on a brand new
repository, and sets up tracking between your local master branch and
GitHub's master branch, so future pushes remember where to go automatically.
Only needed once, the very first time a repository is pushed.

**git push origin branchname**
Uploads your local commits on a specific branch up to GitHub, making them
visible there for the first time if the branch is new, or adding new commits
to an existing branch on GitHub.

**git push**
Once tracking is already set up between a local branch and its remote
counterpart, this shorter version uploads new commits without needing to
specify origin or the branch name every time.

## Branching

**git branch branchname**
Creates a new branch with the given name, but does not switch you onto it.
You stay on whatever branch you were already on.

**git checkout branchname**
Switches your working folder over to an existing branch, so your files on
disk now reflect that branch's version of everything.

**git checkout -b branchname**
Creates a new branch and switches onto it immediately, in one step. This is
just git branch and git checkout combined together.

## Saving work

**git add .**
Stages every changed, new, or deleted file in the current folder, meaning it
marks them as ready to be included in the next commit. This does not create
the commit itself, it just prepares what will go into it.

**git add filename**
Stages one specific file by name, instead of everything in the folder. Useful
when only one file actually needs to go into the next commit — for example,
staging just .gitignore after adding a new exclusion, without also staging
unrelated changes sitting elsewhere in the working folder at the same time.

**git commit -m "message"**
Takes everything currently staged and permanently saves it as a new commit on
your current branch, with the given message describing what changed and why.

## Moving commits between branches

**git cherry-pick commitid**
Takes one specific commit from anywhere in your history and reapplies it on
top of whatever branch you're currently standing on, as a brand new commit.
Useful when a commit ended up on the wrong branch and you want to bring just
that one piece of work over to a different branch, without bringing along
everything else from the original branch.

## Cleaning up branches

**git branch -d branchname**
Deletes a local branch, but only if git can confirm that branch's work is
already fully merged into your current branch. This is the safe version, git
will refuse and warn you if it thinks you might lose unmerged work.

**git branch -D branchname**
Same as above, but skips the safety check and deletes the branch regardless of
whether git can confirm it was merged. This should only be used when you are
already confident the branch's work is safe elsewhere, since it will delete
unmerged work without warning if you are wrong.

## A note on GitHub's website versus these commands

Actions like creating a pull request, reviewing a diff, merging a pull
request, and deleting a branch on GitHub's website are not git commands run
from the terminal at all, they are actions performed through GitHub's own
interface. GitHub does the equivalent git operations on its own servers behind
the scenes when you click those buttons. That's why, after merging or
deleting something on GitHub, your local copy does not automatically know
about it until you run a command like git fetch or git pull to catch up.
