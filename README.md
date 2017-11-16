# An Extensible Command-line Client for Gitea and Gogs

`gitea` is an extensible tool for interacting with a gogs or gitea instance via its API.  It's implemented as a [loco](https://github.com/bashup/loco) extension, which means that it's extensible and configurable via a `~/.gitearc` and `/etc/gitea-cli/config`.

You can install it by copying [the binary](bin/gitea) onto your `PATH`, or by using [basher](https://github.com/basherpm/basher) to `basher install bashup/gitea-cli`.  In either case, you must have [jq](https://github.com/stedolan/jq/)  and `curl` installed as well.  You will then gain access to the following commands:

* `gitea new` *repo [create-opts...]*  -- create *repo* with the specified options
* `gitea deploy-key` *repo keytitle key* -- add the named key as a deployment key for *repo*
* `gitea exists` *repo* -- return success if *repo* exists, otherwise fail
* `gitea repo` *repo* `v` *versiontag* `vendor` *[description create-opts...]* -- import the current directory to *repo* under a `vendor` branch tagged as `vendor-versiontag`, using the given *description* and *create-opts* if *repo* didn't previously exist.

You can also add your own subcommands by defining functions in `~/.gitearc` and `/etc/gitea-cli/config` of the form `gitea.X`, where `X` is the name of the desired subcommand.  That is, if you define a function `gitea.foo()`, then `gitea foo` *args...* will invoke your function with the given arguments.

## Configuration

In order for the predefined subcommands to work, you must have the following environment variables set, either by being in your environment to begin with, or by defining them in  `~/.gitearc` or `/etc/gitea-cli/config`:

* `GITEA_USER` -- the user on whose behalf API operations will be performed
* `GITEA_API_TOKEN` -- the user's API token
* `GITEA_URL` -- the https URL of the gitea installation
* `GITEA_DEPLOY_KEY`, `GITEA_DEPLOY_KEY_TITLE`  -- if set, new repositories will have this key automatically added to their deploy keys
* `GITEA_CREATE_DEFAULTS` -- an array of options used as defaults for repository creation.  For example setting `GITEA_CREATE_DEFAULTS=("private=" "true")` will make new repositories private by default, unless you do `gitea new some/repo private= false` to override it.

## Option Syntax

* *repo* -- a repository name, specified as an `org/repo` to designate a particular organization or user; if the `org` is omitted, the `GITEA_USER` is assumed

* *create-opts* -- a series of key/value word pairs, where values are interpreted according to their keys' syntax:

  * keys ending in `=` mean, "the value is raw JSON or a valid jq expression, appropriately quoted or escaped"
  * keys ending in `:` mean, "the value is YAML" (you must have a yaml2json tool like this one on your `PATH`)
  * keys ending in neither `=` nor `:` mean, "the value is a string"

  The specific keys to use are defined by the [gitea API](https://github.com/gogits/go-gogs-client/wiki); for `gitea new` the specific options may be found [here](https://github.com/gogits/go-gogs-client/wiki/Repositories#create)

