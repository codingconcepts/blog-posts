[semver](http://semver.org/), or "semantic versioning" is a simple but effective way to version an application.

In the past, I've relied on my CI process (Jenkins, BuildMaster, ActiveBatch etc.) to keep track of my application's version.  This method became non-deterministic and hard to manage when there were multiple versions of my application in different branches of code.

###### Enter semver

semver aims to simplify the process of versioning your application by placing *you* in control of your application's version by way of a `.semver` file which lives alongside your code:

``` bash
--- 
:major: 0
:minor: 1
:patch: 0
:special: ""
```

You'll likely be familar with the tried and tested major.minor.patch version system.  In my company, we use the `special` version part to denote the current build number in our Jenkins CI pipeline, ensuring our version is always incremented, even if we forget to checkin an update.

###### Rakefile example

The following is an example `Rakefile` which reads from the `.semver` file and sets a `buildVersion` property using [ldflags](https://golang.org/cmd/link/):

**main.go**

``` go
package main

import "fmt"

var buildVersion string

func main() {
	fmt.Printf("Version: '%s'\n", buildVersion)
}
```

**Rakefile**

``` ruby
require 'rake'
require 'semver'

task :build do
    buildVersion = SemVer.find.to_s

    ldBuildVersion = "-X \"main.buildVersion=#{buildVersion}\""

    Dir.chdir('cmd\\server') do
        sh('go', 'build', '-ldflags', "#{ldBuildVersion}")
    end
end
```

Running the following command will extract the current version from your `.semver` file and build it *into* your executable:

``` bash
$ rake build
```

Running the application with this semver file will yield:

``` bash
Version: '0.1.0'
```