# Tutorial on Git objects

This is a tutorial on the most significant git objects: blob, tree, and commit. (There's also a
fourth object, tag, but I'm not going to discuss it.)

## Setup, 1 of 3

This tutorial create a directory at /tmp/src/wiki - if you already have something there that you
want to preserve, quit immediately and move that stuff out of the way.

## Setup, 2 of 3

Entirely optional, but if you want to play along, you will need to get some things into your path.
I keep `$HOME/bin` in my path.

```shell
curl -sO https://raw.githubusercontent.com/jgn/wrap/refs/heads/main/wrap
curl -sO https://raw.githubusercontent.com/jgn/tutor/refs/heads/main/tutor
curl -sO https://raw.githubusercontent.com/jgn/watch/refs/heads/main/watch
chmod +x wrap tutor watch
```

## Setup, 3 of 3

And from Homebrew (par is optional):

```shell
brew install glow tree par
```

Also install [iTerm2](https://iterm2.com/).

## Tutorial controls

While running the tutorial, you can also press **q** to quit and **r** to redisplay (in case you
change the terminal width).

Some parts of this script actually "do something." In those cases, you will first see a list of
commands. If you want to run the commands, press return. You can also omit the execution of the
commands by pressing **s** (for "skip").

The next bit will show that it's going to do the `date` and `pwd` commands. Then after you press
return again it will actually do it and show the result.

```shell { .run }
date
pwd
```

## Why is Git so hard?

In my opinion, the reason Git is so hard is that the commands (`git add`, `git commit`,
`git checkout`, etc.), have names whose semantics seem to be mapped onto the history of a variety of
versioning tools, especially Subversion. The actual git commands, however, have behaviors with very
precise meanings regarding specific git entities. For instance, `git add` means: Compress the files
and add them to a git-managed directory, and add references to those files to the binary "index"
file. Even when the official Git docs use the language of the implementation (for instance, we are
told by `git help add` that `git-add` means "Add file contents to the index") it isn't very clear
what "index" means in this context. A lot would be gained by banning words such as "staging area"
and "cache" and just sticking with "index."

## intro

If you'd like to play along:

In iTerm2, create two vertical panes by pressing command-d twice.

In the second pane, enter

```shell
watch /tmp/src/wiki "tree -a -C -F -t ."
```

## plumbing

Conventionally the "friendly" git commands like `git status` are called "porcelain."

Many of the commands I'll be showing here are called "plumbing."

For this tutorial, here's a breakdown:

### porcelain

* git init
* git status
* git add
* git commit
* git show

### plumbing

* git hash-object
* git cat-file
* git ls-files
* git write-tree
* git update-index
* git update-ref

## Clear out any old files

Let's clear out any old files that are lying around.

```shell { .run }
pwd
WIKI=/tmp/src/wiki
rm -vfR $WIKI
mkdir -vp $WIKI
cd $WIKI
pwd
```

## 3rd pane: cd into the wiki

In the 3rd pane, change your directory to the newly-created wiki with

```
cd /tmp/src/wiki
```

## Create a file in the wiki

First we'll create a file in the wiki.

```shell { .run }
echo "### Welcome to the wiki" >home.md
cat home.md
```

And initialize git.

```shell { .run }
pwd
git init
```

## observe the newly-created .git/ dir

At this point we've created the `home.md` file and the `.git/` directory.

Notice that there is nothing in the `.git/` directory that indicates that anything is being tracked.
We can verify this with the porcelain command:

```shell
git status
```

What we're going to do next is add the `home.md` file and see what happens. To do this, we'll use
the porcelain command `git add`.

Then we'll clear out the repo and do it again with "plumbing."

```shell { .run }
git add home.md
```

## added blob

Voila.

After we've done `git add`, the file is said to be "staged." Hold on to that word. "Staged" means
that it's a change to be committed.

You should now see in the `objects/` directory a new directory `81/`, and within that, a file called

```
d4aaeff611587c378d25a90e1d2f178b484727
```

The concatenation of those two is:

```
81d4aaeff611587c378d25a90e1d2f178b484727
```

and is the git-style hash of the contents of `home.md`

(I say "git-style" because it's displayed as the SHA1 of: "blob " concatenated with the file size,
a null byte, and the actual data. In the longer document I provide a link to some Ruby code that
will generate this hash.)

There's a git command to get this git-style hash of any file. To try it, in the third pane
type

```shell
git hash-object home.md
```

Also, a file called index is added, which we'll discuss in a moment.

## contents of a blob

The contents of that file in the `81/` dir is a compressed version of `home.md`. To
see that, in the 3rd pane, type

```shell
git show 81d4aa
```

(Notice how instead of the full hash we can use an abbreviation.)

There is no magic here. You can uncompress the file with some Ruby: `ruby -rzlib -e 'print Zlib::Inflate.new.inflate(STDIN.read)' < .git/objects/81/d4aaeff611587c378d25a90e1d2f178b484727`

That compressed file is a git blob.

Notice that in the `.git/` directory, there's no indicator that this object is a blob (as opposed to
the other types of objects, which we'll get to in a moment.)

We can determine the type of any object with`git cat-file -t`. Example: Into the 3rd pane, type

```shell
git cat-file -t 81d4aa
```

## git is a content-addressable database

I want to make a big point here.

git is a content-addressable database. We're used to finding files by name. (Right?)

But notice that so far, there is no indication in the `objects/` directory that the content from
`home.md` is actually named `home.md`.

If we want to know whether `home.md` is in the database, we can hash it, and then look in the
`objects/` directory and see if there's something matching that hash.

Now we'll add a new copy of `home.md` called `home2.md`, and also `git add` that. **No extra object
will be created.**

```shell { .run }
cp home.md home2.md
git add home2.md
```

See? Even though we've added a second file, since it has the exact same contents, there is only one
blob in the `objects/` directory.

## the index

Before I said that there is no indication in `objects/` that a particular object has a file name.

But there is a place where this is tracked: It's called the index, which is stored in
a binary file called `.git/index`

There are a couple of ways to understand what's in the index: You can use the porcelain command

```shell
git status
```

or the plumbing command

```shell
git ls-files
```

## status vs ls-files

`git status` looks at your files, the index, and the object database.

`git ls-files` looks at the index.

Try those out. The index is really powerful because it is used to compare the state of what's in
your file system, and what's in your object database. That's what `git status` is all about: It can
tell you what is not being tracked (never added) and what is staged (added).

In older versions of `git`, `git ls-files` was written in C, while `git status` was written in
shell. To get a feel of how `git status` relies on `git ls-files` I have put the v1.0.0 version
of `git status` in a [gist](https://gist.github.com/jgn/597a3058ed15b4523436b32eadf8da93).


```shell { .run }
git status
```

```shell { .run }
git ls-files
```

## about ls-files

The output for `ls-files` is incredibly limited, right? That's because the
typical use case is to list files of a certain type.

```shell
ls-files -c  # show cached (tracked)
ls-files -d  # show unstaged deletions
ls-files -u  # show unmerged
```

There is also an option -t that shows status tag (though the tags are weird:
for instance, the tag for a tracked file is: H).

In short, `git status` is for humans, `git ls-files` is a bit more for
scripting and automation.

Incidentally, if you delete the index, you can still get some information out of `git status`.
But `git ls-files` will show nothing. Of course, don't delete the index in real life, because
you'll lose your history.

## commit

At this point, we'll commit our changes and see 2 more objects in the objects database.

```shell { .run }
git commit -m "Adding a couple of files"
```

## play: `git show` and `git cat-file`

Now take a look at the objects with both `git show` (porcelain) and `git cat-file` (plumbing).

```shell
git show 0a74e3
git cat-file -t 0a74e3
git cat-file -p 0a74e3
```

Notice especially how the tree object and the commit object use the hashes to link everything up.

You will notice that the tree object (which has the file names `home.md` and `home2.md`) has the
same hash (0a74e3) on both your system and mine. This is not true of the commit object, because the
commit object has date and committer information in it, so the hashes are different. When I was
writing this, the hash was `1cfa96`.

## reset our work so far

Let's remove the .git directory and the second wiki file and start over.

```shell { .run }
rm -fR $WIKI/.git
cd $WIKI
rm home2.md
git init
```

## using plumbing to add a file

Now we'll do `git-add` the hard way.

## adding a blob with plumbing

It turns out that `git hash-object` has an option `-w` that will create the right subdirectory in
`objects/` and also write out the compressed file.

Note that in what follows we will also capture the hash into a variable.

```shell { .run }
BLOB_HASH=`git hash-object -w home.md`
echo $BLOB_HASH
```

(If we really wanted to, we could skip `-w`, create the directory, and add the compressed file
manually.)

The next thing we will do is get the hash into the index.

```shell { .run }
git update-index --add --cacheinfo 100644 $BLOB_HASH home.md
```

## adding a tree object with plumbing

As noted, so far there is no indication in the object database that the blob for home.md has a name.
Let's do that.

```shell { .run }
TREE_HASH=`git write-tree`
echo $TREE_HASH
```

## adding a commit with plumbing

And now we'll make a commit with plumbing.

```shell { .run }
COMMIT_HASH=`echo "adding a file" | git commit-tree $TREE_HASH`
echo $COMMIT_HASH
git update-ref HEAD $COMMIT_HASH
```
