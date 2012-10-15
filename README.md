Introduction
============

This document explains how we used git submodule for an internal project. It's
not clear that submodules is the best solution for the problem (for example, see
[Why your company shouldn’t use Git submodules](http://codingkilledthecat.wordpress.com/2012/04/28/why-your-company-shouldnt-use-git-submodules/)),
but if you do decide to use submodules, then you may find this document useful.

Our use case is that we have two different web applications written in the same
language and framework, and we want them to share the same model code. We need
two different applications, because they will be deployed to different servers,
but they have identical models because they'll interact with the same SQL DB.

Setup
=====

I'm assuming you already have your source code split into three repos: "appA",
"appB" and "model". where "model" will become the submodule under the "appA" and
"appB" repos.

To tell git to use one repo as a submodule for another repo, use the
`git submodule add [path to submodule] [subdirectory name]` command:

    $ pwd
    /home/alice/dev/appA
    $ ls
    app.php
    $ git submodule add git://url.to.your.repo/model.git model
    Cloning into 'model'...
    done.
    $ ls
    app.php model
    $ ls model/
    user.class.php
    $ git status
    # On branch master
    # Changes to be committed:
    #   (use "git reset HEAD <file>..." to unstange)
    #
    #       new file:   .gitmodules
    #       new file:   model
    #
    $ cat .gitmodules
    [submodule "model"]
            path = model
            url = git://url.to.your.repo/model.git
    $  git commit -m "Added model submodule to appA."
    [master 6b3c178] Added model submodule to appA.
     2 files changed, 4 insertions(+)
     create mode 100644 .gitmodules
     create mode 160000 model
    $ git push

Repeat the above for appB (and appC, appD, etc. if you have them).

The .gitmodules file is the file that git uses to store all information about
the submodules that the current project contains.

Every other developer on your team should do a `git pull` to get the latest
.gitmodules file, then `git submodule init` to initialize the submodule, and
then `git submodule update` to actually retrieve the contents of the submodule.

    $ pwd
    /home/bob/dev/appA
    $ ls
    app.php
    $ git pull
    remote: Counting objects: 4, done.
    remote: Compressing objects: 100% (3/3), done.
    remote: Total 3 (delta 0), reused 0 (delta 0)
    Unpacking objects: 100% (3/3), done.
    From git://url.to.your.repo/appA.git
       42f980d..39228b7  master     -> origin/master
    Updating 42f980d..39228b7
    Fast-forward
     .gitmodules | 3 +++
     model       | 1 +
     2 files changed, 4 insertions(+)
     create mode 100644 .gitmodules
     create mode 160000 model
    $ ls
    app.php model
    $ ls model/
    $ git submodule init
    Submodule 'model' (git://url.to.your.repo/model.git) registered for path 'model'
    $ ls model/
    $ git submodule update
    Cloning into 'model'...
    done.
    Submodule path 'model': checked out '3c38e1abf6d2812b21bd1afe5e5aa71edd5f7d97'
    $ ls model/
    user.class.php

As you can see, just doing a `git pull` or a `git submodule init` is not
sufficient to actually retrieve the contents of the submodule.

Making changes to the submodule
===============================

The correct mental model to have is that the parent project "appA" points to a
very specific revision of the submodule "model". Thus, if you make changes to
the model repo, these changes will not be visible in appA unless you update the
pointer accordingly.

The recommended workflow is to never edit the contents of appA/model/ directly,
but rather to update the model repo, and then to update the pointers in your
appA, appB, etc. repos.

First, make the desired changes in your model repo:

    $ pwd
    /home/alice/dev/model
    $ ls
    user.class.php
    $ touch blogpost.class.php
    $ git add .
    $ git commit -m "Implemented model for blog posts."
    [master 2bdea61] Implemented model for blog posts.
     0 files changed
     create mode 100644 blogpost.class.php
    $ git push

Then, in each application, pull in the changes into the submodule, and then
adjust your application's code accordingly.

    $ pwd
    /home/alice/dev/appA
    $ cd model
    $ git pull
    remote: Counting objects: 3, done.
    remote: Compressing objects: 100% (2/2), done.
    remote: Total 2 (delta 0), reused 0 (delta 0)
    Unpacking objects: 100% (2/2), done.
    From git://url.to.your.repo/model.git
       3c38e1a..2bdea61  master     -> origin/master
    Updating 3c38e1a..2bdea61
    Fast-forward
     0 files changed
     create mode 100644 blogpost.class.php
    $ cd ..
    $ touch controllers.blog.php
    $ git add .
    $ git status
    # On branch master
    # Changes to be committed:
    #   (use "git reset HEAD <file>..." to unstage)
    #
    #       new file:   controllers.blog.php
    #       modified:   model
    #
    $ git commit -m "Implemented support for blog posts."
    [master 8ba0fe0] Implemented support for blog posts.
     1 file changed, 1 insertion(+), 1 deletion(-)
     create mode 100644 controllers.blog.php
    $ git push

The key step here is that you must go into the submodule directory
(`cd model`) and perform the `git pull` there, rather than in the root of
the parent repo. Then you must return to the parent repo (i.e. be outside of
the submodule) when you perform the commit and push.

Repeat as necessary for appB, appC, etc.

Receiving changes to the submodule
==================================

The next time your other developers do a git pull, they'll see the following:

    $ pwd
    /home/bob/dev/appA
    $ git pull
    remote: Counting objects: 3, done.
    remote: Compressing objects: 100% (2/2), done.
    remote: Total 2 (delta 1), reused 0 (delta 0)
    Unpacking objects: 100% (2/2), done.
    From git://url.to.your.repo/appA.git
       39228b7..8ba0fe0  master     -> origin/master
    Fetching submodule model
    remote: Counting objects: 3, done.
    remote: Compressing objects: 100% (2/2), done.
    remote: Total 2 (delta 0), reused 0 (delta 0)
    Unpacking objects: 100% (2/2), done.
    From git://url.to.your.repo/model.git
       3c38e1a..2bdea61  master     -> origin/master
    Updating 39228b7..8ba0fe0
    Fast-forward
     model | 2 +-
     1 file changed, 1 insertion(+), 1 deletion(-)
     create mode 100644 controllers.blog.php
    $ ls model
    user.class.php
    $ git status
    # On branch master
    # Changes not staged for commit:
    #   (use "git add <file>..." to update what will be committed)
    #   (use "git checkout -- <file>..." to discard changes in working directory)
    #
    #       modified:   model (new commits)
    #
    no changes added to commit (use "git add" and/or "git commit -a")

As you can see, they do not automatically get the changes to the submodule (the
blogpost.class.php file is missing), but they *do* know that some sort of change
to the module occured via `git status`.

To actually retrieve these changes, run the `git submodule update` command.

    $ git submodule update
    Submodule path 'model': checked out '2bdea6144badf840525620f4f3a10831140debef'
    $ ls model/
    blogpost.class.php user.class.php
    $ git status
    # On branch master
    nothing to commit (working directory clean)

License
=======
This git submodule tutorial by NebuPookins is licensed under the Creative
Commons Attribution-ShareAlike 3.0 Unported License. To view a copy of this
license, visit http://creativecommons.org/licenses/by-sa/3.0/.