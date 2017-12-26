# Migrate a SVN repository to git, then host it on github

Author: Robert Nikutta

Version: 2017-12-26

Version original: 2016-06-13

## Primer
We wanted to migrate an SVN repository called "dusty" that has years of development
history, to git, and then host it on github.  The SVN repo is hosted
at ```svn+ssh://some.server.edu/path/to/svn/dusty```, but is also accessible
through https at ```https://some.server.edu/path/to/svn/dusty```

## Requirements
The migration should fulfill all these requirements:

- preserve all history
- map authors to their github accounts (for those who have a github account)
- turn SVN 'tags' branches into git tags
- turn SVN 'branches' into git branches
- remove 'trunk' once all is said and done (since 'master' is the git equivalent)

## Steps to get it done



### Make a directory to store the git-tified SVN dump

Make a temporary directory and change to it
```
DIR=~/tmp/foo
mkdir $DIR
cd $DIR
```

### Compile a list of contributors
The SVN repo has committer/user names, and github identifies
contributors by they registered email addresses. We need to map the
two. A file like this will do just that:
```
cat ~/authors.txt
jane.doe = Jane Doe <fortranninja@gmail.com>
jimmy = James G. MacGee <omg@nsa.gov>
zombie = Zeezee Top <abc@astro.foo.edu>
```
Every line is one committer, and the format is
```
svnuser = Real Name <github-email>
```

### Pull from the SVN repo
```
git svn clone --prefix=svn/ -s -A ~/authors.txt svn+ssh://validuser@some.server.edu/path/to/svn/dusty dusty
```

Be ready to type (or better paste) your SSH password many times!
(basically every time the clone descends into another branch). IIRC I
had to provide the password a total of 11 times.

### Turn SVN branches into git branches, and SVN tag branches to git tags

Have this shell script ready in a file:
```
cat ~/make_tags_branches.sh
#!/bin/bash

set -u
set -e

cd $PWD;

### Make tags
echo "\n///////// TAGS"
git for-each-ref --format='%(refname)' refs/remotes/svn/tags/* | while read r; do
    tag=${r#refs/remotes/svn/tags/}
    sha1=$(git rev-parse "$r")

    commiterName="$(git show -s --pretty='format:%an' "$r")"
    commiterEmail="$(git show -s --pretty='format:%ae' "$r")"
    commitDate="$(git show -s --pretty='format:%ad' "$r")"

    # make the tags
    git show -s --pretty='format:%s%n%n%b' "$r" | \
    echo "Creating tag $tag"
    env GIT_COMMITTER_EMAIL="$commiterEmail" GIT_COMMITTER_DATE="$commitDate" GIT_COMMITTER_NAME="$commiterName" \
        git tag -a -m "" "$tag" "$sha1"

    # Remove the svn/tags/* ref
    echo "Deleting remote branch ref $r"
    git update-ref -d "$r"
done

### create local branches out of svn branches
echo "\n///////// BRANCHES"
git for-each-ref --format='%(refname)' refs/remotes/svn/ | while read branch_ref; do
    branch=${branch_ref#refs/remotes/svn/}
    # Only use select branches
    if [[ "$branch" =~ "trunk" ]]; then
        echo "Deleting remote branch_ref: $branch_ref"
        git update-ref -d "$branch_ref"
    else
        echo "Creating git local branch: $branch"
        git branch --no-track "$branch" "$branch_ref"
        echo "Deleting remote branch_ref: $branch_ref"
        git update-ref -d "$branch_ref"
    fi
    echo
done
```
make sure it is executable, then run it
```
chmod 700 ~/make_tags_branches.sh   # make script executable
cd dusty   # change to the newly pulled repo
~/make_tags_branches.sh   # run it in the repo
```

### Create a new git repo on github
Log into your account on github, click on "Repositories", create a new
one. I you initialize it (i.e. add some files, and possibly a LICENSE
file), then don't skip the step [Pull changes from remote](#pull). In
any case, copy the https URL to the new repo on github which you'll
see when you click on the green button ```Clone or download```,
e.g. here
```
https://github.com/rnikutta/dusty.git
```

### Add new git repo as remote origin branch
Add the new github repo as remote tracking branch to the local repo
```
git remote add origin https://github.com/rnikutta/dusty.git
```

<a id="pull"></a>

### Pull changes from remote
If you've already initialized the github repo with some files, then
pull then changes from remote to local first
```
git pull origin master
```
Confirm the merge message when asked about it (i.e. save the
temporarily opened file, and exit the editor).

### Push all branches and tags to remote repo
Now we're ready to push the master branch to the remote...
```
git push -u origin master
```

And all other branches and tags, too
```
git push origin --all   # push all branches
git push origin --tags  # push all tags
```

## Tag the current HEAD on master branch

We wanted to tag the current version on master as "Dusty 4.2". This
name is purely historical, that's what we called it internally while
we developed on SVN. From here on we will adhere to the [Semantic
versioning](http://semver.org/).

Let's figure out the sha1sum of the last commit on master, via either
```
git log
git reflog
```
The sha1sum could be, for instance: d6ab3e9

Now make a tag
```
git tag -a -m "Tagging HEAD on master as Dusty 4.2" "4.2" d6ab3e9
```
Replace d6ab3e9 with the actual sha1sum.

Check if the tag was created
```
git tag -l
```

Finally, push the new tag to the remote on github
```
git push origin --tags
```

## Add a-posteriori an older 1.0 version (as a tag)

We had a very old but still often-used version of DUSTY that we always
called "DUSTY 1.0", but which never enjoyed VC-based development. We
wanted to have it on github as a tagged version; it would not be used
for development, but for easy downloads and for reference only. Git
tags are the right mechanism.

Create a new feature branch
```
git checkout -b dusty1.0
```

Delete everything from tracking. Also delete empty directories, if
any; git won't track empty dirs, and thus won't remove them. We're
deleting only on the newly created branch, so no worries.

```
git rm -rf .   # git-remove tracked files
rm -rf *       # remove not tracked files and empty directories
```

Copy to here all files to be added to this new branch from some ```sourcedir/```,
add everything to tracking, and commit to the new branch

```
cp -pra sourcedir/* .   # copy recursively from source directory to here
git add *   # stage everything for commit
git ci -a -m "Made new branch dusty1, which contains Dusty v1.0 files."  # commit to branch
```

Note the sha1 sum of the commit reported with the last command, e.g.
```
[dusty1.0 b2961f9]  # the 7-character string is the (partial) sha1 sum we need
```

Or look it up in the last commit via one of these commands
```
git log
git reflog
```

Have the sha1 sum ready. Now make a tag
```
git tag -a -m "Creating tag for Dusty 1.0" "1.0" sha1sum
```

Check out master again
```
git checkout master
```

Delete the feature branch again (we don't need it anymore, since we have the tag that we wanted)
```
git branch -D dusty1.0
```

Push to remote
```
git push origin --tags
```

Check what you have got
```
git branch -a
git tag -l
```

## What didn't work in general
- Accessing the SVN repo through https, because the hosting server has
new(er) SSL certificates. The clients (which all use ```git svn```
under the hood) would not accept the certificate, neither
(t)emporarily, nor (p)ermanently.

## What tools didn't work
Despite all the googling in the world, most tools and scripts
available online either work only partially, or are outright broken.

Specifically:

- The ruby script [svn2git](https://github.com/nirvdrum/svn2git) has
  serious troubles accessing an SVN repo through svn+ssh:// Some users
  online report that each time it asks for the SSH password, one has
  to press ENTER first, then type the password, then press ENTER
  again. This did not work most of the time. In 20 tries I got it to
  start pulling from SVN maybe two times, but only up to the next
  password request. Turning on super-verbose mode (the ```-vvv```
  switch) in my subversion SSH config file

```
$grep "^ssh" ~/.subversion/config
ssh = $SVN_SSH ssh -vvv -o ControlMaster=no
```

didn't help (but you get to see all the nitty-gritty of SSH in action).

- I also tried Atlassian's java-based migration scripts, and while
  they worked in the past, it had trouble at some stages this
  time. Atlassian's documentation and discussion of the process are
  **highly recommended** though, and some of the commands used above
  were adapted from their scripts.

  Check out:

  [https://www.atlassian.com/git/tutorials/migrating-convert/](https://www.atlassian.com/git/tutorials/migrating-convert/)
  
  [http://blogs.atlassian.com/2012/01/moving-confluence-from-subversion-to-git/](http://blogs.atlassian.com/2012/01/moving-confluence-from-subversion-to-git/)

## Some other tools (not tried)

- KDE has an own tool called
  [Svn2Git](https://techbase.kde.org/Projects/MoveToGit/UsingSvn2Git),
  but despite the same name this is not the Ruby-based
  [svn2git](https://github.com/nirvdrum/svn2git).
  