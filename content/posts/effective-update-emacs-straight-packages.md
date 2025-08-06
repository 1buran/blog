---
title: "Effective Update Emacs Straight Packages"
date: 2023-11-05T19:59:25+03:00
draft: false
toc: true
images:
tags:
  - Emacs
  - Golang
---

# Intro to problem

The Straight is a great Emacs package manager, it pulls all emacs packages from their git repos,
places it locally to `~/.emacs.d/straight/repos` and builds (if needed) in `~/.emacs.d/straight/build` directory. If you face to some package issue or you want to make some kind of experiment with the package, you may just switch to another git branch/tag/commit. This is pretty nice!

But over time of use and as your experience grows, the number of packages grows, and it takes longer and longer to update them because emacs and straight do **sequentially** one by one package.

The packages also have dependencies, that also need to be updated. On my laptop ~100 straight packages installed, there is nothing extra ordinary, i just work with: ReactJS, CSS, HTML, Doker, Python, Golang, Nginx etc. Also I have emacs specific: magit, zenburn theme, org mode, rg, company, hl-todo etc.

Again, the update all this stuff may take ~2-5 minutes, depending on internet connection.
But no matter how your internet connection is good, it anyway will take minutes,
because straight utility will do that in **sequential** manner!

Sometimes it can be annoying.


# Solution

Make update in **parallel/concurrent** manner.

The steps of update process may be these ones:
1. Go into every git repo (walk trought `../straight/repos` directories)
1. Create `Updated.At` git tag (it will help you for rollback the update if it crashed something)
1. Pull the changes from origin
1. [optional] Print git log messages (new commits)
1. Restart Emacs if at least one package has been updated, Straight builds new version of package during start up process

Once we code these steps, we able to apply this update procedure **concurrency** to all localy installed packages. Let's do it!

## Break down main.go

Let's break down our main.go into small pieces - steps of update process.

### Define constants

Later, if we wanted to make this public and then we have to move this into configurable options(config file or cli parameters).

```go
const (
	EmacsStraightReposPath = "/home/buran/.emacs.d/straight/repos"
	TagName                = "Updated.At"
)
```

### Get the list of all installed packages

Use [filepath.Glob](https://pkg.go.dev/path/filepath#Glob):

```go
func ListEmacsStraightRepos(p string) (repos []string, err error) {
	repos, err = filepath.Glob(EmacsStraightReposPath + "/*")
	return
}

```

### Create or modify a git tag

First we create a tag, but in the next time of running our program,
we have to change the reference of the tag to the current HEAD - move tag forward.
So we have to cover both cases.

Use [go-git](https://github.com/go-git/go-git):

```go
// Create a new tag with name Updated.At or change its reference to ref
func CreateOrModifyGitTag(r *git.Repository, t string, ref *plumbing.Reference) (*plumbing.Reference, error) {
	tag, err := r.Tag(t)
	switch err {
	case nil: // CASE 1:  tag exists, set new reference of tag
		tag = plumbing.NewHashReference(tag.Name(), ref.Hash())
		if err = r.Storer.SetReference(tag); err != nil {
			return nil, err
		}
	case git.ErrTagNotFound: // CASE 2: tag does not exist, create a tag
		if tag, err = r.CreateTag(TagName, ref.Hash(), nil); err != nil {
			return nil, err
		}
	}
	return tag, nil
}
```



### Pull the changes

You may think that the good approach is to detect changes during this action byt the check of
returned err:

```go
// Pull git changes and return true if the local workdir has updated
func PullGitChanges(r *git.Repository) (bool, error) {
	w, err := r.Worktree()
	if err != nil {
		return false, err
	}
	err = w.Pull(&git.PullOptions{})
	switch err {
	case git.NoErrAlreadyUpToDate:
		return false, nil
	default:
		return true, nil
	}
}
```

But actually it useless without deep analyze of what exactly has been updated!
For example the appereance of new branch on remote is also - update!
The lib should not fetch such branch updates, its docs says that it will pull changes only for current branch, but on practic - it fetches all updates.

So for detection of updates files we have to check git logs since our last update - find new commits made since commit referenced by tag `Updated.At`.


### Print git log and count new commits

Here is we inspect our git log: we try to find new commits made since our last update.
The senseable way is use `r.Log(&git.LogOptions{From: tag.Hash()})`,
but it looks like the lib has a bug and this doesnt' work (there are some not resolved old GitHub issues). So the workaround is to use `r.Log(&git.LogOptions{Since: &t})` - use the time of commit on which referrence the tag `Updated.At` (in fact - since our last update).

To make funcy color output use [gookit/color](https://github.com/gookit/color):

```go
// Print git log to buffer, inspect commits since given time,
// count the number of commits and save to n
func PrintGitLog(r *git.Repository, ref *plumbing.Reference, buf *strings.Builder, n *int) error {
	// KLUDGE use LogOptions.From doesn't work, use alternative method LogOptions.Since instead
	// cIter, err := r.Log(&git.LogOptions{From: tag.Hash(), Order: git.LogOrderDFSPost})

	c, err := r.CommitObject(ref.Hash())
	if err != nil {
		return err
	}

	// KLUDGE hide the Updated.At tagged commit, show only after it
	t := c.Committer.When.Add(time.Second)
	cIter, err := r.Log(&git.LogOptions{Since: &t})

	defer cIter.Close()

	// process every single commit
	f := func(n *int) func(c *object.Commit) error {
		return func(c *object.Commit) error {
			*n++
			buf.WriteString(
				color.Sprintf("\t<red>%s</> <gray>%s %s <%s></>\n\t\t<green>%s</>\n",
					c.Committer.When.Format("2006-01-02"),
					c.Hash.String()[:6],
					c.Author.Name, c.Author.Email, strings.ReplaceAll(c.Message, "\n", "\n\t\t"),
				))
			return nil
		}
	}(n)
	err = cIter.ForEach(f)
	return err
}
```


### Restart emacs

Here is break down of this procedure:
1. we create helper function `runCommand` which is recieve the cmd and args in a golang way and run it
2. and the restart function itself: it uses `runCommand` for kill and start emacs again

Use [exec.Command](https://pkg.go.dev/os/exec@go1.21.3#Command):

```go
func runCommand(s ...string) (err error) {
	color.Set(color.Red)
	cmd := exec.Command(s[0], s[1:]...)
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr
	err = cmd.Run()
	color.Reset()
	return
}

func restartEmacs() {
	commands := []string{"emacsclient -e (kill-emacs)", "emacs -nw --daemon"}
	for _, v := range commands {
		err := runCommand(strings.Split(v, " ")...)
		if err != nil {
			log.Fatal(err)
		}
	}
}
```

## Let's put all together

Create global variables for buffering output of `PrintGitLog` function
and flag `restartEmacsIsNeeded` which we will use further.

```go
var buf strings.Builder
var restartEmacsIsNeeded bool

func UpdateEmacsStraightRepo(p string, wg *sync.WaitGroup) {
	var (
		r         *git.Repository
		tag, head *plumbing.Reference
		rr        *git.Remote
		err       error
	)

	defer wg.Done()

	if r, err = git.PlainOpen(p); err != nil {
		log.Fatal(err)
	}
	if head, err = r.Head(); err != nil {
		log.Fatal(err)
	}
	if rr, err = r.Remote("origin"); err != nil {
		log.Fatal(err)
	}

	if tag, err = CreateOrModifyGitTag(r, TagName, head); err != nil {
		log.Fatal(err)
	}

	if _, err = PullGitChanges(r); err != nil {
		log.Fatal(err)
	}

	var n int
	err = PrintGitLog(r, tag, &buf, &n)

	if n > 0 {
		restartEmacsIsNeeded = true
		color.Comment.Printf("Fetched from %s", rr.Config().URLs[0])
		color.C256(214).Printf(" %d new commits\n", n)
		color.C256(247).Printf("local path: %s\n", p)
		fmt.Print(buf.String())
		buf.Reset()
	}
}
```

## The main function

Use goroutines for run `UpdateEmacsStraightRepo` cuncurrently and
`sync.WaitGroup` for synchronization:

```go
func main() {
	// walk trought emacs straight repos directories
	repos, err := ListEmacsStraightRepos(EmacsStraightReposPath)
	if err != nil {
		log.Fatal(err)
	}

	wg := &sync.WaitGroup{}
	for _, v := range repos {
		wg.Add(1)
		go UpdateEmacsStraightRepo(v, wg)
	}

	wg.Wait()
	if restartEmacsIsNeeded {
		restartEmacs()
	}
}
```

{{< admonition warning >}}
Actually this is not good approach the run as many processes for update the git repos as you have repos, but it's not be araising the  Disk I/O Wait catostrafically for hundreds repos, on SSD.
{{< /admonition >}}

## Run the concurrent update

Ok, previously we had created all parts of our program, it's time to run it!

Run `go install` for create `updemacs` binary file, assuming that you already have `$(go env GOPATH)/bin` in `$PATH` env, if update is available you will see something like this:

{{< figure src="images/2023-11-05-235956.png"  caption="Example of run updemacs" >}}

## The bug

Previous we saw that the idea works, but actually an implementation has a bug,
if you ran many times the utility you might notice a mess in the output like this:

{{< figure src="images/2023-11-29-185853.png"  caption="Log messages are mixed up" >}}

git repository log messages are mixed up: messages from one repo are displayed in the log of another and vice verse.

This is because we used shared buffer, and more than one goroutine may write to this buffer.

> Do not communicate by sharing memory; instead, share memory by communicating.

Do you remember that? This is it.

So the fix is to use go channels or just return back a full string of git log messages for repo.
Let's do the second, here are the changes:

```diff
modified   golang/updemacs/main.go
@@ -62,13 +62,15 @@ func PullGitChanges(r *git.Repository) (bool, error) {

 // Print git log to buffer, inspect commits since given time,
 // count the number of commits and save to n
-func PrintGitLog(r *git.Repository, ref *plumbing.Reference, buf *strings.Builder, n *int) error {
+func GetGitLog(r *git.Repository, ref *plumbing.Reference, n *int) (string, error) {
 	// KLUDGE use LogOptions.From doesn't work, use alternative method LogOptions.Since instead
 	// cIter, err := r.Log(&git.LogOptions{From: tag.Hash(), Order: git.LogOrderDFSPost})

+	var buf strings.Builder
+
 	c, err := r.CommitObject(ref.Hash())
 	if err != nil {
-		return err
+		return "", err
 	}

 	// KLUDGE hide the Updated.At tagged commit, show only after it
@@ -91,10 +93,9 @@ func PrintGitLog(r *git.Repository, ref *plumbing.Reference, buf *strings.Builde
 		}
 	}(n)
 	err = cIter.ForEach(f)
-	return err
+	return buf.String(), err
 }

-var buf strings.Builder
 var restartEmacsIsNeeded bool

 func UpdateEmacsStraightRepo(p string, wg *sync.WaitGroup) {
@@ -125,16 +126,18 @@ func UpdateEmacsStraightRepo(p string, wg *sync.WaitGroup) {
 		log.Fatal(err)
 	}

-	var n int
-	err = PrintGitLog(r, tag, &buf, &n)
+	var (
+		n int
+		l string
+	)
+	l, err = GetGitLog(r, tag, &n)

 	if n > 0 {
 		restartEmacsIsNeeded = true
 		color.Comment.Printf("Fetched from %s", rr.Config().URLs[0])
 		color.C256(214).Printf(" %d new commits\n", n)
 		color.C256(247).Printf("local path: %s\n", p)
-		fmt.Print(buf.String())
-		buf.Reset()
+		fmt.Print(l)
 	}
 }
```


# Conclusion
Now we have known why we should follow the one most of very important principle/concept of Golang:
> Do not communicate by sharing memory; instead, share memory by communicating.

Also we've learned how to update installed Emacs Straight packages in more effective way,
without waiting for this feature to be available in Emacs/Straight.

Feel the power of concurrency: the time of execution of check for update of ~100 repositories, on my laptop (i7-8550U CPU @ 1.80GHz, RAM 16 GB DDR4 2400, SSD NVMe 256 GB), takes ~2 seconds!

Thank you for reading, see you next time!
