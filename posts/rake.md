The `rake` build automation tool is a Ruby implementation of the `make` tool, which has been around since _ever_.

I tried it out for the first time as part of a Go application I'm currently pushing flogging into Production and found it a joy to work with.  So much so, that I've taken to creating a `Rakefile` for projects that aren't destined for deployment, as a wrapper for repetitive tasks like `go build -o thing.exe; .\thing.exe`

###### Getting started
First of all, you'll need to have an installation of Ruby.  Head over to [Ruby Installer](https://rubyinstaller.org/) to download the latest version.

Then, you'll need to install `rake`:

``` bash
$ gem install rake
```

Add it to your path so you can call `rake` from the command line.

As with `make`, `rake` allows you to write named tasks with commands to execute.  I won't delve into the nitty-gritty details of tasks and namespaces but here's an example `Rakefile` I use to make the process of building and running my Go code quicker.  This particular example is taken from my [Guilded Rose](https://github.com/codingconcepts/katas/guilded-rose) kata:

``` ruby
require 'rake'

OUTPUT = 'rose.exe'

task :run do
    sh('go', 'build', '-o', OUTPUT)
    sh(".\\#{OUTPUT}")
end
```

...and here's a breakdown of what everything's doing:

``` ruby
require 'rake'
```

Bring in the `rake` module, same as you would bring in any other Ruby module.

``` ruby
OUTPUT = 'rose.exe'
```

As the resulting executable is never going to need referencing by name (outside of this file), I provide a hard-coded name I can build to and run from.

``` ruby
task :run do
end
```

A `rake` task definition called "run".  This will be accessible on the command line when invoking the `rake` command.

``` ruby
sh("go build -o #{OUTPUT}")
# OR
sh('go', 'build', '-o', OUTPUT)
```

Executes the "go build -o rose.exe" command on the command line and fails the `rake run` task if it isn't successful.  Either of the above commnds work as expected, the second one just specifies arguments explicitly.

``` ruby
sh(".\\#{OUTPUT}")
```

Runs the resulting executable file creating during the "go build" step.

###### Result

Kick off the `rake run` task on the command line and you'll see the following output in the console:

``` bash
go build -o rose.exe
.\rose.exe
{+5 Dexterity Vest 9 19}
{Aged Brie 1 1}
{Elixir of the Mongoose 4 6}
{Sulfuras, Hand of Ragnaros 0 80}
{Backstage passes to a TAFKAL80ETC concert 14 21}
{Conjured Mana Cake 2 4}
```

...assuming you've CD'd into the directory your `Rakefile` is in and it's in the same directory as your code.