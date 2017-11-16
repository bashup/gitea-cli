# An Extensible Command-line Client for Gitea and Gogs

## Installation and Use

`gitea` is an extensible tool for interacting with a gogs or gitea instance via its API.  It's implemented as a [loco](https://github.com/bashup/loco) extension, which means that it's extensible and configurable via multiple [configuration files](#configuration-files), including the ability to define custom subcommands of your own.

You can install the tool by copying [the binary](bin/gitea) onto your `PATH`, or by using [basher](https://github.com/basherpm/basher) to `basher install bashup/gitea-cli`.  In either case, you must have [jq](https://github.com/stedolan/jq/)  and `curl` installed, as well as the [required configuration](#required-configuration) settings to access your gitea or gogs instance.  You can then use the following commands:

* `gitea new` *repo [create-opts...]*  -- create *repo* with the specified options
* `gitea deploy-key` *repo keytitle publickey* -- add the named key as a deployment key for *repo*
* `gitea exists` *repo* -- return success if *repo* exists, otherwise fail

The built-in commands use the following argument syntax:

* *repo* -- a repository name, specified as an `org/repo` to designate a particular organization or user; if the `org` is omitted, the `GITEA_USER` is assumed

* *create-opts* -- a series of key/value word pairs, where values are interpreted according to their keys' syntax.  If a key ends in `=`, the value must be raw JSON or a valid jq expression, appropriately quoted or escaped.  Otherwise, the value is assumed to be an unescaped string and is encoded as such.

  (The specific keys to use are defined by the [gitea API](https://github.com/gogits/go-gogs-client/wiki); for `gitea new` the specific options may be found [here](https://github.com/gogits/go-gogs-client/wiki/Repositories#create).  Remember to include `=` at the end of keys whose value isn't a string.)

* *keytitle* -- the name of the deployment key

* *publickey* -- the SSH public key to be used

You can also add your own subcommands by defining functions in one of the standard [configuration files](#configuration-files).  A functions named `gitea.X` implements a subcommand named `X`.  So, if for example you run  `gitea foo arg1 arg2...`, the CLI tool will call a function named `gitea.foo` with the supplied arguments.  All of the functions defined in the [tool source](gitea.md) are available to you to use, as well as those provided by [loco](https://github.com/bashup/loco).





## Configuration and Extension

### Configuration Files

Configuration is loaded by sourcing bash script files from the following locations, in this order, if they exist:

* `/etc/gitea-cli/gitea-config`
* `$HOME/.config/gitearc`
* The *nearest* `.gitearc` found in or above the current directory (i.e., only one is loaded, even if more than one exists)

If more than one file sets the same variable or defines the same function, the later ones override the earlier ones.

### Required Configuration

In order for the predefined subcommands to access your gitea instance, you must have the following environment variables set, either by being in your environment to begin with, or by defining them in one of the configuration files listed above:

* `GITEA_USER` -- the user on whose behalf API operations will be performed
* `GITEA_API_TOKEN` -- the user's API token
* `GITEA_URL` -- the https URL of the gitea installation

### Optional Configuration

These variables can also be set via config file(s) or runtime environment:

* `GITEA_DEPLOY_KEY`, `GITEA_DEPLOY_KEY_TITLE`  -- if set, new repositories will have this key automatically added to their deploy keys
* `GITEA_CREATE_DEFAULTS` -- an array of options used as defaults for repository creation.  For example setting `GITEA_CREATE_DEFAULTS=("private=" "true")` will make new repositories private by default, unless you do `gitea new some/repo private= false` to override it.



