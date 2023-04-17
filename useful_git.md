#### delete a file
##### a. keep a file but remove from git index
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

##### b. remove a file
```shell
git rm <file>
# or
rm <file> && git add <file>
```
#### what is a index
In Git, the "index" (also called the "staging area" or "cache") is a data structure that holds information about the changes that have been made to the files in your repository since the last commit.
check by
```shell
git ls-files --stage
```