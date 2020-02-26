I've got the blues: working with `go get` on Azure
==================================================

`go get` works smoothly with github, but sadly it's not as fun when using Azure. Microsoft have put up some [documentation](https://docs.microsoft.com/en-us/azure/devops/repos/git/go-get?view=azure-devops) but there are a few errors in there, and the long Azure paths make it easy to misconfigure things.

So, if you can't use github for whatever reason (because your infrastructure is all built on Azure), let me give you some guidance.

## Hosting a module on Azure Devops
If you're using a Go package that's hosted on Azure, the first thing to do is to make sure that the module of the hosted package is set up correctly. I'm assuming that you've got control of the source - if you were importing someone else's code, it would probably be a public package on github and you wouldn't be reading this blog in the first place.

Hosting a module on Azure is largely the same as hosting on GitHub, but the module path is a bit more complex:
```
go mod init dev.azure.com/<organisation>/<project>/_git/<repo>.git
```
Don't include the <>, those are just there to show the parts of the path that you need to edit.
- `organisation`: Your organisation (or username if it's a personal project)
- `project`: Your Azure project name
- `repo`: The name of the actual repository

Once you've written the content of your module, you can push it to azure using git as normal. 

### Sample Module
For reference, here's a very simple module I built to experiment with `go get`.
```
testproject
├─  go.mod
├─  main.go
├─  README.md
│
└───magicfoo
    └─  magicfoo.go
```

go.mod:
```
module dev.azure.com/dummyorganisation/dummyproject/_git/testgoget.git

go 1.13
```

main.go (note the .git in the import path):
```
// Executable file
package main

import (
	"fmt"

	"dev.azure.com/dummyorganisation/dummyproject/_git/testgoget.git/magicfoo"
)

func main() {
	fmt.Println("Welcome to the go get practice, the magic number is:", magicfoo.MagicAdd(1, 2))
}
```

magicfoo.go
```
// I created this file to practice importing a package from within a module
package magicfoo

const Magic uint16 = 0xdead
const Foo uint16 = 0xbeef

func MagicAdd(foo, bar int) (int){
    return foo + bar + 1
}
```

## Installing from a public Azure repo with `go get`
Installing from a public repo is reasonably straightforward. If you want the top-level package:
```
go get dev.azure.com/<organisation>/<project>/_git/<repo>.git # install the top level package
```
If you want to install a package from a subfolder, just append the subfolder path to the end:
```
go get dev.azure.com<organisation>/<project>/_git/<repo>.git/<path>/<to>/<package> # install package from subfolder
```
Note the `.git` after the repo name.

For example, to import the package `magicfoo` from my repo above:
```
go get dev.azure.com/dummyorganisation/dummyproject/_git/testgoget.git/magicfoo
```

If you make a mess and want to clear out your downloaded modules, use `go clean modcache`

*Note:*
The microsoft [documentation](https://docs.microsoft.com/en-us/azure/devops/repos/git/go-get?view=azure-devops) is missing the `<project>` part of the path. This should be fixed soon.

## Installing from a private Azure repo with `go get` 
This is the most gnarly part, and of course it's also the most relevant if you're working with proprietary code. Before you can get anywhere, you need to create a [personal access token](https://docs.microsoft.com/en-us/azure/devops/organizations/accounts/use-personal-access-tokens-to-authenticate) (PAT) with read access to the repo(s) you want to `go get` from.

Once you have a token, stick it in your gitconfig file as follows:
```
[url "https://<user>:<token>@dev.azure.com/<organization>/<project>/_git/"]
    insteadOf = https://dev.azure.com/<organization>/<project>/_git/
```
Any time git tries to access the insteadOf url, it will instead access your replacement. If you had a token with access to multiple projects or organisations, you could shorten these urls to be more general.

Next, append the start of the repo path to your `GOPRIVATE` environment variable.
```
GOPRIVATE='dev.azure.com/<organisation>/<project>'
```
You can get away with just appending `dev.azure.com/*`, but I'd recommend at least adding the organisation name, if not the project. You can check what you set with `go env`. Thanks to @imantumorang for [documenting this step](https://medium.com/mabar/today-i-learned-fix-go-get-private-repository-return-error-reading-sum-golang-org-lookup-93058a058dd8).

You can now use `go get` for repos associated with your auth token just as you would for unauthenticated repos:
```
go get dev.azure.com/<organisation>/<project>/_git/<repo>.git # install the top level package
go get dev.azure.com<organisation>/<project>/_git/<repo>.git/<path>/<to>/<package> # install package from subfolder
```

Windows users, if you get a pop-up asking for your azure credentials, then something is wrong and the git credentials manager is taking over. I've not been able to get `go get` to work at all with the credentials manager, so it's best at that point to abort and see if you've made a mistake. Most likely either your credentials or the repo path you're trying to `get` are wrong. If you like, you can disable the credentials manager altogether and go back to the much nicer command-line interface:
```
git config --global credential.helper=store
git config --system credential.helper=store
```
