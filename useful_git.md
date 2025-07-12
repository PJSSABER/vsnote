# delete a file
## a. keep a file but remove from git index
```shell
git rm --cache <file>
```
so, git will untrack file you specified, but the file will still exist in your workspace.(Useful for local settings, i.e., .gitignore)


it is possible to tell Git to ignore changes to the .gitignore file itself by using the following command:
```shell
git update-index --assume-unchanged .gitignore
```
this also like a rm in git, because git will not track the file anymore since your last commit. 
When --[no-]assume-unchanged flag is specified, the object names recorded for the paths are not updated. Instead, this option sets/unsets the "assume unchanged" bit for the paths. When the "assume unchanged" bit is on, the user promises not to change the file and allows Git to assume that the working tree file matches what is recorded in the index. If you want to change the working tree file, you need to unset the bit to tell Git. This is sometimes helpful when working with a big project on a filesystem that has very slow lstat(2)system call (e.g. cifs).

## b. remove a file
```shell
git rm <file>
# or
rm <file> && git add <file>
```
### git rm v.s. rm 

git rm:
- Removes the File from the Working Directory: The file is deleted from your filesystem. 
- Removes the File from the Git Index: The file is staged for removal, meaning it will be deleted in the next commit. 
- Optionally Keeps the File in the Working Directory: You can use the --cached option with git rm to remove the file only from the Git index while keeping it in the working directory. This is useful for removing a file from version control but keeping it on your local filesystem.

rm:
- Removes the File Only from the Working Directory: The file is deleted from your filesystem but not from the Git index.
- Leaves the File Staged for Deletion: If you only use rm to delete a file, Git will recognize that the file has been deleted when you run git status, but it won’t stage the deletion for the next commit.
- Requires an Additional Step to Remove from Git: After using rm, you'll need to run git add to stage the file’s deletion for the next commit.

# what is a index
In Git, the "index" (also called the "staging area" or "cache") is a data structure that holds information about the changes that have been made to the files in your repository since the last commit.
check by
```shell
git ls-files --stage
```

## Working Directory/Workspace: This is where you modify files in your project. It's essentially the files and directories on your local filesystem.

## Index/Staging Area: When you use git add, you are moving changes from the working directory into the index. The index is like a middle ground between your working directory and the local repository. It holds the changes that will be included in the next commit.

## Local Repository: This is where your committed changes live. When you commit, the changes in the index are saved into the local repository, creating a new commit in the branch.

## git reset

git reset [<mode>] [<commit>]
           This form resets the current branch head to <commit> and possibly updates the index (resetting it to the tree of <commit>) and the working tree depending on <mode>. If <mode> is omitted, defaults to --mixed. The <mode> must be one of the following:

--soft
    Does not touch the index file or the working tree at all (but resets the head to <commit>, just like all modes do). This leaves all your changed files "Changes to be committed", as git status would put it.

--mixed
    Resets the index but not the working tree (i.e., the changed files are preserved but not marked for commit) and reports what has not been updated. This is the default action.

    If -N is specified, removed paths are marked as intent-to-add (see git-add(1)).

--hard
    Resets the index and working tree. Any changes to tracked files in the working tree since <commit> are discarded.

see ![git_reset](/images/git_reset.png)


# merge in one commit

git log --oneline main..yiyu_dev

git rebase -i main

Change all but the first pick to squash (or s)
Save and exit, then write your commit message.

git push --force-with-lease origin yiyu_dev


# 
see what commits you have that aren't in main:
git log --oneline main..HEAD


reset to main while keep changes

merge with main

git reset --soft main

git commit -m "feat: your comprehensive commit message here"

git push --force-with-lease origin yiyu_dev