# 04: Git Internals

## Notes

Command `git init`creates a new Git repository.

Git stores the state and the history of the repository in a hidden directory `.git/`.

```
$ ls -a
./  ../  .git/  LICENSE  notes.md  README.md
```

**Never make any changes to the `.git` directory.**

Contents of the `.git/` directory:

- file `config`
- file `description`
- file `HEAD`
- directory `objects`
- ...

Git has its own internal file system.
All files of this system are called objects and are stored in the `.git/objects/` directory.

There are four Git object types:

- `Blob` -- stores data
- `Tree` -- stores structure
  - `trees` contain `blobs` and/or other `trees`
- `Commit` -- stores state
- `Annotated Tag` -- stores commit alias

Git low-level commands:

- `git hash-object` -- creates a new `blob` object and returns its hash
- `git cat-file` -- reads from a Git object
- `git mktree` -- creates a new `tree` object

Creating a temporary `blob` object with contents `"Hello, Git\n"`:

```
$ echo "Hello, Git" | git hash-object --stdin
b7aec520dec0a7516c18eb4c68b64ae1eb9b5a5e
$ ls .git/objects/
info/  pack/
$ ls -a
./  ../  .git/  LICENSE  notes.md  README.md
```

Creating a permanent `blob` object with contents `"Hello, Git\n"`:

```
$ echo "Hello, Git" | git hash-object --stdin -w
b7aec520dec0a7516c18eb4c68b64ae1eb9b5a5e
$ ls .git/objects/
b7/  info/  pack/
$ ls .git/objects/b7
aec520dec0a7516c18eb4c68b64ae1eb9b5a5e
$ ls -a
./  ../  .git/  LICENSE  notes.md  README.md
```

Notice that hash `b7aec...a5e` becomes `b7/aec...a5e`, i.e. Git stores object `b7aec...a5e` in the `b7` directory in the `aec...a5e` file.

Remarks:

- there are at most `256` object directories: from `00/` to `ff/`.

Hash function:

- a function that takes a variable length string and outputs a fixed length digest

```
$ echo "Hello" | shasum
1d229271928d3f9e2bb0375bd6ce5db6c6d348d9 *-
$ echo "Hello, Git" | shasum
c9d5d04925b93d2fb99c73ab2b5869bde7405ca4 *-
```

- deterministic, i.e. same output for same input

```
$ echo "Hello, Git" | shasum
c9d5d04925b93d2fb99c73ab2b5869bde7405ca4 *-
$ echo "Hello, Git" | shasum
c9d5d04925b93d2fb99c73ab2b5869bde7405ca4 *-
```

- unpredictable, i.e. small change in input data will output a completely different hash

```
$ echo "Hello, Git" | shasum
c9d5d04925b93d2fb99c73ab2b5869bde7405ca4 *-
$ echo "Hello, Git!" | shasum
e40153b3e43a5ed7fa00ce6bd7a576763b88dab2 *-
```

- one-way, i.e. based on output it is not possible to tell the input without brute-force guessing
  `c9d5d04925b93d2fb99c73ab2b5869bde7405ca4 = SHA1(??)`

Hash function examples:

- function `MD5` --> 128 bit digest **(depreciated)**
- family `SHA`:
  - function `SHA1` --> 160 bit (40 hex) digest
  - function `SHA256` --> 256 bit digest
  - function `SHA512` --> 512 bit digest

Git uses `SHA1` hash function, therefore Git can store `2^160` or approx. `10^48` objects.

Command `git cat-file` options:

```
$ echo "Hello, Git" | git hash-object --stdin
b7aec520dec0a7516c18eb4c68b64ae1eb9b5a5e
```

- `git cat-file -p <hash>` -- object contents

```
$ git cat-file -p b7aec520dec0a7516c18eb4c68b64ae1eb9b5a5e
Hello, Git
```

- `git cat-file -s <hash>` -- object size

```
$ git cat-file -s b7aec520dec0a7516c18eb4c68b64ae1eb9b5a5e
11
```

- `git cat-file -t <hash>` -- object type

```
$ git cat-file -t b7aec520dec0a7516c18eb4c68b64ae1eb9b5a5e
blob
```

Creating a `blob` object based on a file:

```
$ echo "Second file in Git repository" > new-file.txt
$ ls
LICENSE  new-file.txt  notes.md  README.md
$ git hash-object new-file.txt
4400aae52a27341314f423095846b1f215a7cf08
$ ls .git/objects/
b7/  info/  pack/
$ git hash-object new-file.txt -w
4400aae52a27341314f423095846b1f215a7cf08
$ ls .git/objects/
44/  b7/  info/  pack/
```

Reading form a `blob` object:

```
$ git cat-file -p 4400aae52a27341314f423095846b1f215a7cf08
Second file in Git repository
$ git cat-file -s 4400aae52a27341314f423095846b1f215a7cf08
30
$ git cat-file -t 4400aae52a27341314f423095846b1f215a7cf08
blob
```

Remarks:

- `blob` objects do not store file names.
- `blob` object persist even when the original files are destroyed:

```
$ ls
LICENSE  new-file.txt  notes.md  README.md
$ ls .git/objects/
44/  b7/  info/  pack/
$ rm new-file.txt
$ ls
LICENSE  notes.md  README.md
$ ls .git/objects/
44/  b7/  info/  pack/
```

- `blob` objects store:
  - their type
  - their size
  - their content
- `blob` object format:
  ```
  "blob <length>\0<content>"
  ```
  where `\0` is the `NULL` character separating meta from content
- all the above are used to compute `blob` object hash:

```
$ echo "Hello, Git" | git hash-object --stdin
b7aec520dec0a7516c18eb4c68b64ae1eb9b5a5e
$ echo -e "blob 11\0Hello, Git" | shasum
b7aec520dec0a7516c18eb4c68b64ae1eb9b5a5e *-
```

`Tree` object has the following format:

```
<obj-permission> <obj-type> <obj-hash>\t<obj-name>
<obj-permission> <obj-type> <obj-hash>\t<obj-name>
...
<obj-permission> <obj-type> <obj-hash>\t<obj-name>
```

where `<obj-type>` can be either `blob` or `tree`,
and `<obj-permission>` can be:

- `040000` -- directory
- `100644` -- non-executable file
- ...

Git stores permissions because it has a separate file system and it needs to be able to create files and directories in the file system of the OS.

Creating a `tree` object based on a file:

```
$ cat > temp-tree.txt
100644 blob b7aec520dec0a7516c18eb4c68b64ae1eb9b5a5e    file1.txt
100644 blob 4400aae52a27341314f423095846b1f215a7cf08    file2.txt

$ cat temp-tree.txt
100644 blob b7aec520dec0a7516c18eb4c68b64ae1eb9b5a5e    file1.txt
100644 blob 4400aae52a27341314f423095846b1f215a7cf08    file2.txt

$ cat temp-tree.txt | git mktree
3b95df0ac6365c72e9b0ff6c449645c87e6e1159
```

Reading from a `tree` object:

```
$ git cat-file -p 3b95df0ac6365c72e9b0ff6c449645c87e6e1159
100644 blob b7aec520dec0a7516c18eb4c68b64ae1eb9b5a5e    file1.txt
100644 blob 4400aae52a27341314f423095846b1f215a7cf08    file2.txt

$ git cat-file -s 3b95df0ac6365c72e9b0ff6c449645c87e6e1159
74
$ git cat-file -t 3b95df0ac6365c72e9b0ff6c449645c87e6e1159
tree
```

Remarks:

- `<obj-names>` `file1.txt` and `file2.txt` differ from the file names.
  Usually they are the same, but they don\'t have to be.

So far we have worked directly with the Git repository, but Git has actually three areas:

- Git repository
- Staging area (index)
- Working directory

Staging area (index) prepares new and/or modified files to be stored in the Git repository and prepares files taken from the Git repository to be put in the working directory.

Command `git ls-files` options:

- `git ls-files` lists all files stored in the working directory

```
$ git ls-files
LICENSE
README.md
```

- `git ls-files -s` lists all files stored in the staging area (index)

```
$ git ls-files -s
100644 d2b1b34ac63635f34e73cf660ca0a4ac3ff9e6dd 0       LICENSE
100644 1c0a8a81ae6fad495403034e4781d716c1fe4741 0       README.md
```

Command `git read-tree <hash>` reads from the Git repository the `tree` object with the `<hash>` hash and puts it in the staging area (index).

```
// $ cat temp-tree.txt | git mktree
//3b95df0ac6365c72e9b0ff6c449645c87e6e1159
$ git read-tree 3b95df0ac6365c72e9b0ff6c449645c87e6e1159
$ git ls-files -s
100644 b7aec520dec0a7516c18eb4c68b64ae1eb9b5a5e 0       file1.txt
100644 4400aae52a27341314f423095846b1f215a7cf08 0       file2.txt
$ git ls-files
file1.txt
file2.txt
$ ls
LICENSE  notes.md  README.md
```

Remarks:

- `git ls-files` format:
  `<obj-permissions> <obj-hash> <obj-flag>\t<obj-name>`
  where `<obj-flag>` is `0` when object is unmodified.

Command `git checkout-index -a` reads all files from the staging area (index) and puts them in the working directory:

```
$ git checkout-index -a
$ git ls-files -s
100644 b7aec520dec0a7516c18eb4c68b64ae1eb9b5a5e 0       file1.txt
100644 4400aae52a27341314f423095846b1f215a7cf08 0       file2.txt
$ git ls-files
file1.txt
file2.txt
$ ls
file1.txt  file2.txt  LICENSE  notes.md  README.md
```
