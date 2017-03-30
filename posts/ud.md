[ud](https://github.com/codingconcepts/ud) is a little CLI I wrote which talks to the Urban Dictionary API and returns the top result by "thumbs up" responses.

I wrote it as part of the Go training material I'm compiling for my colleagues, as it demonstrates the following:

* Writing an HTTP client
* Project layout
* Vendoring

###### Installation

``` bash
$ go get -u github.com/codingconcepts/ud
```

###### Usage

As with the website, some of the definitions and examples don't leave much to the imagination, so user discretion is advised!  Here's an example you might *not* want to try in the office:

``` bash
ud pink falafel hat
```

I leave the execution of this statement as an exercise to the user.