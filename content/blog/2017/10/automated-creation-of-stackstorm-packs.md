+++
author = "Nick Maludy"
author_url = "https://github.com/nmaludy"
categories = ["Nick Maludy", "StackStorm", "pack", "automation"]
date = "2017-10-15"
description = "Automated creation of StackStorm Packs"
linktitle = ""
title = "Automated creation of StackStorm Packs"
type = "post"

+++


In StackStorm a pack is logical organization that contains related actions,
aliases, policies, rules and sensors. To get a good overview of what a pack is
please see the excellent [StackStorm Pack documentation](https://docs.stackstorm.com/packs.html).

Most users of StackStorm wish to either integrate with a new system that's not
available in the [Exchange](https://exchange.stackstorm.org/) or use StackStorm
to execute custom automation scripts. The way to accomplish this is to create a
new Pack.


# Pack Format

A Pack is simply a folder structure and set of files:

``` bash
actions/                # contains all actions and workflows
aliases/                # all action aliases
policies/               # action execution policies
rules/                  # rules for executing actions based on triggers
sensors/                # sensors that monitor external systems and emit triggers
config.schema.yaml      # schema for a pack's config
packname.yaml.example   # example config following the schema in config.schema.yaml
pack.yaml               # pack definition and metadata
requirements-tests.txt  # list of python modules needed for running a pack's tests
requirements.txt        # list of python modules needed for runtime execute of a pack
```

For more information on this structure please see the StackStorm documentation
[Create and Contribute a pack](https://docs.stackstorm.com/reference/packs.html).

One option for creating a pack is to manually create all of these folders and
files in the proper formats or automatically create them.


# Automatic Pack Creation

In true StackStorm and DevOps fashion we at Encore thought it would be best to
automate the creation of Packs. To accomplish this we utilize a tool called
[cookiecutter](https://github.com/audreyr/cookiecutter) that creates folder
structure and file content based on a template. We've created a template in the
standard Pack structure and file content. Cookicutter works by prompting
a user for a set of variables defined by the template. It then uses these variables
to fill in [Jinja2](http://jinja.pocoo.org/docs/latest/) expressions for folder
names and file content.

Encore's cookiecutter template for StackStorm can be found on [GitHub at EncoreTechnologies/cookiecutter-stackstorm](https://github.com/EncoreTechnologies/cookiecutter-stackstorm).

# Example

In this example we're going to showcase how to automatically create a StackStorm
Pack using Encore's `coockiecutter-stackstorm` template.

``` bash
# Create a repo on GitHub (or other remote repository)
# Clone the repository
$ git clone git@github.com:YourOrganization/stackstorm-cookiecutterexample.git

# Install the cookiecutter package (could do this in a virtualenv if you like)
$ pip install cookiecutter

# Generate the pack content automatically
$ cookiecutter -f https://github.com/EncoreTechnologies/cookiecutter-stackstorm.git
pack_name [xxx]: cookiecutterexample
pack_description [Pack description.]: An example pack directory structure created using cookiecutter.
version [0.1.0]:
keywords_csv []: cookiecutter,example
author_name [First Last]: Nick Maludy
author_email [first.last@domain.tld]: code@encore.tech

# Check out our newly created Pack (will be in a folder called stackstorm-<pack_name>
$ ls -l stackstorm-cookiecutterexample
actions/
aliases/
policies/
rules/
sensors/
CHANGES.md
circle.yml
config.schema.yaml
CONTRIBUTORS.md
cookiecutterexample.yaml.example
LICENSE
Makefile
pack.yaml
README.md
requirements-tests.txt
requirements.txt

# Commit and push your changes
$ cd stackstorm-cookiecutterexample
$ git add .
$ git commit -m "Initial pack creation using Encore's cookiecutter-stackstorm repo"
$ git push origin master
```

The name of the folder is based on what the user entered for `[pack_name]` in this
case it is `stackstorm-cookciecutterexample/`. Several of the files have
content filled in based on the variables entered by the user:

``` bash
############################
$ cat pack.yaml
---
ref: cookiecutterexample
name: cookiecutterexample
description: test
keywords:
    - cookiecutter
    - example
version: 0.1.0
author: nick maludy
email: code@encore.tech

############################
$ cat CONTRIBUTORS.md 
## Pack Contributors
* nick maludy code@encore.tech

############################
$ cat README.md 
# cookiecutterexample Integration Pack

## Configuration
TODO: Describe configuration


# Sensors

## Example Sensor
TODO: Describe sensor


# Actions

## example
TODO: Describe action
```


# Encore Extras

As you may have noticed, there are some extra files in this automatically created
Pack. We at Encore thought it best to include additional files above and beyond
the standard structure to promote better GitHub repos and automated unit testing
within Packs.

First the additional files for GitHub are:

``` bash
CHANGES.md              # a changelog for the pack
CONTRIBUTORS.md         # list of contributors to the pack
LICENSE                 # license information of the pack (default: Apache 2.0)
README.md               # readme for the pack detailing usage and configuration information
```

These files are just skeletons, but provide a good starting point for pack authors
to fill out important information.

Secondly are files related to automated unit testing:

``` bash
circle.yml              # how to execute unit tests in CircleCI
Makefile                # helper tasks for executing tests
```

The `Makefile` is a light wrapper script that clones the [Encore StackStorm CI repository](https://github.com/EncoreTechnologies/ci-stackstorm)
that provides tasks for testing a pack. These tests are based off of the testing
standard required by the [StackStorm Exachange CI repository](https://github.com/StackStorm-Exchange/ci).
If an author wishes to submit their pack to the Exchange, these tests must pass.
Encore's CI `Makefile` executes all tests required by the Exchange except for 
testing registration of actions, sensors, rules, etc (this test requires MongoDB
be installed).

To execute the tests using the `Makefile` simple run:

``` bash
$ make
```

Individual tests can be executed one at a time:

``` bash
$ make flake8
$ make pylint
$ make packs-tests
```

There are other tests available and can be listed by running:

``` bash
$ make list
```

When executing tests the `Makefile` creates a `ci/` folder within your Pack's 
repository. As with any good makefile it should be able to clean up any cruft
it creates. To clean up the CI data simply run:

``` bash
$ make clean
```


# Conclusion

In this post we've detailed how to automatically create a Pack's structure using
cookiecutter and the Encore cookiecutter-stackstorm repository. We've also disussed
some of the extra features that this repository offers above and beyond a default
creation using `mkdir` and `touch`.


-Nick Maludy
