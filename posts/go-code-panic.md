This has caught me out before, so this is as much for me as it is for you!

###### Scenario 1 - editor and gocode out-of-sync

If you write your Go code using a text editor which takes its Go auto-completion from [gocode](https://github.com/nsf/gocode) (it will), it may get out-of-sync with your editor every now and then, if your top-level Go package [goplus for Atom](https://atom.io/packages/go-plus) or [vscode-go for VSCode](https://github.com/Microsoft/vscode-go) etc. doesn't update it for you.

Perhaps the most obvious symptom of this issue is the auto-complete dropdown.  If you're only seeing `PANIC` as an auto-complete option, run through the following steps to rectify:

``` bash
gocode close
go get -u github.com/nsf/gocode
```

You may need to close your editor for the changes to take effect but this seems to do the trick for me.

###### Scenario 2 - gb workspace layout

If you've configured gocode to work in [gb](https://getgb.io/) mode using the following command and things aren't working:

``` bash
$ gocode set package-lookup-mode gb
```

You have two options:

**1** - roll forward and get it working in gb mode:

``` bash
$ gb install all
```

**2** - roll back to the default mode:

``` base
$ gocode set package-lookup-mode go
```