# An Extensible Command-line Client for Gitea and Gogs

### Contents

<!-- toc -->

- [Installation and Use](#installation-and-use)
  * [Prefix Options](#prefix-options)
  * [Vendor Imports](#vendor-imports)
- [Configuration and Extension](#configuration-and-extension)
  * [Configuration Files](#configuration-files)
  * [Required Configuration](#required-configuration)
  * [Optional Configuration](#optional-configuration)
  * [Creating Custom Commands](#creating-custom-commands)
  * [Customizing `curl` Behavior](#customizing-curl-behavior)

<!-- tocstop -->

## Installation and Use

`gitea` is an extensible tool for interacting with a gogs or gitea instance via its API.  It's implemented as a [loco](https://github.com/bashup/loco) extension, which means that it's extensible and configurable via multiple [configuration files](#configuration-files), including the ability to define custom subcommands of your own.

You can install the tool by copying [the binary](bin/gitea) onto your `PATH`, or by using [basher](https://github.com/basherpm/basher) to `basher install bashup/gitea-cli`.  In either case, you must have [jq](https://github.com/stedolan/jq/)  and `curl` installed, as well as the [required configuration](#required-configuration) settings to access your gitea or gogs instance.  You can then use the following commands:

* `gitea new` *repo [create-opts...]*  -- create *repo* with the specified options
* `gitea delete` *repo* -- delete the named repo (use with caution!)
* `gitea deploy-key` *repo keytitle publickey* -- add the named key as a deployment key for *repo*
* `gitea exists` *repo* -- return success if *repo* exists, otherwise fail
* `gitea vendor` *[create-opts...]* -- import current directory to a vendor branch, optionally creating a repository; for full details see [Vendor Imports](#vendor-imports) below.

The built-in commands use the following argument syntax:

* *repo* -- a repository name, specified as an `org/repo` to designate a particular organization or user; if the `org` is omitted, the `GITEA_USER` is assumed

* *create-opts* -- a series of key/value word pairs, where values are interpreted according to their keys' syntax.  If a key ends in `=`, the value must be raw JSON or a valid jq expression, appropriately quoted or escaped.  Otherwise, the value is assumed to be an unescaped string and is encoded as such.

  (The specific keys to use are defined by the [gitea API](https://github.com/gogits/go-gogs-client/wiki); for `gitea new` the specific options may be found [here](https://github.com/gogits/go-gogs-client/wiki/Repositories#create).  Remember to include `=` at the end of keys whose value isn't a string.)

* *keytitle* -- the name of the deployment key

* *publickey* -- the SSH public key to be used

You can also add your own subcommands by defining functions in one of the standard [configuration files](#configuration-files).  A functions named `gitea.X` implements a subcommand named `X`.  So, if for example you run  `gitea foo arg1 arg2...`, the CLI tool will call a function named `gitea.foo` with the supplied arguments.  All of the functions defined in the [tool source](gitea.md) are available to you to use, as well as those provided by [loco](https://github.com/bashup/loco).

### Prefix Options

The following options can be used *before* various commands to alter their behavior.

Creation prefixes (use before `new` or `vendor`):

* `--description`, `--desc`, or `-d` -- set the description for the created repository, e.g. `gitea -d "something" new foo/bar`.
* `--public` or `-p` -- make the created repository public, e.g. `gitea --public new some/thing`
* `--private` or `-P` -- make the created repository private, e.g. `gitea --private new some/thing`

Vendor option prefixes (use before `vendor` only):

* `--repo` or `-r` -- override the automatically-generated repository settings, e.g. `gitea -r some/thing vendor` sets `PROJECT_ORG=some` and`PROJECT_REPO=thing`.
* `--tag` or `-t` -- set `PROJECT_TAG` to the supplied version; e.g. `gitea -t 1.2 vendor` will tag the import as `vendor-1.2`.

### Vendor Imports

The `gitea vendor` command attempts to intelligently import the contents of the current directory as a vendor distribution into a possibly-new repository.  It's intended for managing forks of projects that only supply their sources via archive files or a foreign revision control system.  To use it, you need to unpack the vendor-supplied sources and run the command from the resulting directory.  The directory must *not* contain a `.git` subdirectory.

By default, the repository name and organization will be the name of the current directory and `$GITEA_USER` respectively, but you can override these by setting `PROJECT_NAME` and `PROJECT_ORG` on the command line, or via the `--repo` prefix option.

If the repository doesn't exist, it's created with the supplied *create-opts*, if any.  `GITEA_CREATE` and the default deploy key (if any) are applied.

On the other hand, if the repository *does* exist, then  a `PROJECT_TAG` *must* be supplied (e.g. via `gitea --tag someversion vendor`).  It will be used to tag the newly imported version (as `vendor-someversion`).

In either case, the code is imported on the `vendor` branch of the repository, which will be created if it does not exist.  (If the vendor branch doesn't exist, the snapshot is commited first to `master`, then branched.) The commit will be tagged as `vendor-$PROJECT_TAG` if `$PROJECT_TAG` is set  (and it *must* be set if the repository already exists).  Code is commited as `"$GITEA_VENDOR_NAME <$GITEA_VENDOR_EMAIL>"`, defaulting to `Vendor <vendor@example.com>`.

After the code is imported, branched, tagged, and pushed, the current branch is switched to `master` so that you can begin work on merging.  If you've defined a `gitea.vendor-merge` function in any of the config files, it will be called after the switch to `master`.

As a convenient way of specifying `PROJECT_NAME`, `PROJECT_ORG`, and `PROJECT_TAG`, you can use the `--repo` and `--tag` prefix options.  `--repo` sets `PROJECT_NAME` and `PROJECT_ORG` by splitting its argument, while `--tag` sets `PROJECT_TAG` from its argument.  So, for example, this command:

```shell
$ gitea --repo foo/bar --tag 1.2 vendor
```

would run a vendor import with `PROJECT_ORG=foo`, `PROJECT_NAME=bar` and `PROJECT_TAG=1.2`.   If you are creating a new repository, you can also include creation prefix options like `--desc`, `--private`, `--public`, etc.

## Configuration and Extension

### Configuration Files

Configuration is loaded by sourcing bash script files from the following locations, in this order, if they exist:

* `/etc/gitea-cli/gitearc`
* `$HOME/.config/gitearc`
* The *nearest* `.gitearc` found in or above the current directory (i.e., only one is loaded, even if more than one exists)

These files can set variables or define functions to change gitea's behavior or add new commands.  If more than one file sets the same variable or defines the same function, the later ones override the earlier ones.

### Required Configuration

In order for the predefined subcommands to access your gitea instance, you must have the following environment variables set, either by being in your environment to begin with, or by defining them in one of the configuration files listed above:

* `GITEA_USER` -- the user on whose behalf API operations will be performed
* `GITEA_API_TOKEN` -- the user's API token
* `GITEA_URL` -- the https URL of the gitea installation

### Optional Configuration

These variables can also be set via config file(s) or runtime environment:

* `GITEA_GIT_URL` -- the URL prefix used by the `vendor` command for `git clone` operations; defaults to `GITEA_URL` if not specified.  Can be a hostname and `:` for simple ssh URLs, or a full URL base like `git+ssh://user@host`.
* `GITEA_DEPLOY_KEY`, `GITEA_DEPLOY_KEY_TITLE`  -- if set, new repositories will have this key automatically added to their deploy keys
* `GITEA_CREATE` -- an array of options used as defaults for repository creation.  For example setting `GITEA_CREATE=("private=" "true")` will make new repositories private by default, unless you do `gitea new some/repo private= false` (or the equivalent, `gitea --public new some/repo`) to override it.

### Creating Custom Commands

Configuration files can define functions to create new commands.  A function named `gitea.X` adds a new `gitea X` command.  For example, if you wanted to create a command to list a repository's issues in JSON form, you might put something like this in an appropriate config file, to make `gitea issues foo/bar` dump the issues for repository `foo/bar` in JSON:

```shell
gitea.issues() {
	split_repo "$1"
	auth curl --silent "${GITEA_URL%/}/api/v1/repos/$REPLY/issues"
}
```

Several functions are available for you to use in creating your commands:

* `split_repo` *repo* -- set `$REPLY` to the full repo name, `${REPLY[1]}` to the organization, and `${REPLY[2]}` to the repo name within the organization.  If *repo* doesn't contain a `/`, the organization is the `$GITEA_USER`.

* `auth` *curl-using-command args...* -- runs *curl-using-command args...* with an added authorization token header after *args*.

* `json`  *curl-using-command args...* -- runs *curl-using-command args...* with added arguments after *args* to configure curl to read data from stdin and submit it as an `application/json` request body.

* `jmap` *[key value]...* -- convert the given key-value argument pairs into a JSON object and write it to stdout.  If a key name ends in `=`, the value must be raw JSON or a valid jq expression, appropriately quoted or escaped.  Otherwise, the value is assumed to be an unescaped string and is encoded as such.  (Typically, you'll pipe the output of this to a `json auth api` command to post or put the data and check the return status.)

* `api` *success failure api-path curl-opts...* -- runs `curl` with *curl-opts* on a url equal to *api-path* prefixed with `${GITEA_URL%/}/api/v1/`.  The returned http status is checked against *success* and *failure*, returning true (0) if the status matches *success*, or false (1) if it matches *failure*.

  Both *success* and *failure* can be either a single status code, or multiple codes separated by `|`.  (Note: you must quote arguments that contain `|`.)  If the returned http status doesn't match either, an error message is output to stderr and the return code is a number greater or equal to 64.  (See [sysexits(3)](https://www.freebsd.org/cgi/man.cgi?query=sysexits&sektion=3#DESCRIPTION) and the [gitea-cli source](gitea.md).)

You can also invoke other gitea-cli commands normally.

### Customizing `curl` Behavior

If you need to customize any `curl` options for *all* gitea-cli commands, you can define a `curl` function in your configuration file that invokes `command curl` with extra options.  For example:

```shell
# Add a connect timeout
curl() { command curl --connect-timeout 2 "$@"; }

# Allow self-signed certs
curl() { command curl --insecure "$@"; }

# Do both
curl() { command curl --insecure --connect-timeout 2 "$@"; }
```

If your configuration files define more than one `curl()` function, the last one in the nearest file takes precedence.