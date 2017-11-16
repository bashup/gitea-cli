#!/usr/bin/env bash
: '
<!-- ex: set syntax=markdown : '; eval "$(mdsh -E "$BASH_SOURCE")"; # -->

# An Extensible CLI for gitea and gogs

This file is the source code and main tests for the [generated gitea client](bin/gitea).  Tests look like this:

~~~shell
# Source the functions in this file and initialize as if the loco command were running:
    $ source $TESTDIR/$TESTFILE; set +e
    $ gitea.no-op() { :;}
    $ loco_config() {
    >     LOCO_SITE_CONFIG=/dev/null
    >     LOCO_USER_CONFIG=/dev/null
    >     _loco_config
    > }
    $ loco_main no-op

# dummy `curl` for testing
    $ curl_status=200
    $ GITEA_URL=https://example.com/gitea
    $ GITEA_API_TOKEN=EXAMPLE_TOKEN
    $ curl() {
    >     { printf -v REPLY ' %q' "curl" "$@"; echo "${REPLY# }"; cat; } >&2;
    >     echo $curl_status;
    > }
~~~

And source code like this:

```shell
#!/usr/bin/env bash
```

## Commands

### gitea exists *repo*

Return success if the repository exists:

```shell
gitea.exists() { split_repo "$1" && auth api 200 "repos/$REPLY" ; }
```

~~~shell
    $ gitea exists foo/bar </dev/null; echo [$?]
    curl --silent --write-out %\{http_code\} --output /dev/null -H Authorization:\ token\ EXAMPLE_TOKEN https://example.com/gitea/api/v1/repos/foo/bar
    [0]
    $ curl_status=404 gitea exists foo/bar </dev/null; echo [$?]
    curl --silent --write-out %\{http_code\} --output /dev/null -H Authorization:\ token\ EXAMPLE_TOKEN https://example.com/gitea/api/v1/repos/foo/bar
    [1]
~~~



## Utilities

### Repository Names

#### split_repo

`split_repo` splits `$1` into an organization in `${REPLY[1]}` and a repo in `${REPLY[2]}`.  The organization defaults to `$GITEA_USER` if there's no slash in `$1`.   `$REPLY` (aka `${REPLY[0]}`) contains the fully qualified name, with the defaulted organization included:

```shell
split_repo() {
    [[ "$1" == */* ]] || set -- "$GITEA_USER/$1";
    REPLY=("$1" "${1%/*}" "${1##*/}")
}
```

~~~shell
    $ split_repo foo/bar; printf '%s\n' "${REPLY[@]}"
    foo/bar
    foo
    bar
    $ GITEA_USER=baz; split_repo spam; printf '%s\n' "${REPLY[@]}"
    baz/spam
    baz
    spam
~~~

### json/jq

### API Wrappers

The main gogs/gitea API actions are done by stacking various "adapters" (`json` and `auth`) over an application of `curl`.  These adapters modify the command line arguments passed to curl by adding headers, changing method types, etc.  They invoke their arguments, followed by additional arguments:

```shell
json() { "$@" -X POST -H "Content-Type: application/json" -d @-; }
auth() { "$@" -H "Authorization: token $GITEA_API_TOKEN"; }
```

~~~shell
# `json` makes it a curl POST with content-type application/json:
    $ json curl </dev/null
    curl -X POST -H Content-Type:\ application/json -d @-
    200
# `auth` adds the current API token:
    $ GITEA_API_TOKEN=foo auth curl </dev/null
    curl -H Authorization:\ token\ foo
    200
~~~

The actual API invocation is handled by the  `api` function, which takes a list of `|`-separated "success" statuses followed by an API path and any extra curl options.  If the curl status matches one of the given statuses, success is returned, otherwise failure:

```shell
api() {
    REPLY=$(curl --silent --write-out %{http_code} --output /dev/null "${@:3}" "$GITEA_URL/api/v1/${2#/}")
    eval 'case $REPLY in '"$1) true ;; *) false ;; esac"
}
```

~~~shell
    $ api 200 /x extra args </dev/null; echo [$?]
    curl --silent --write-out %\{http_code\} --output /dev/null extra args https://example.com/gitea/api/v1/x
    [0]
    $ curl_status=401 api 200 /y </dev/null
    curl --silent --write-out %\{http_code\} --output /dev/null https://example.com/gitea/api/v1/y
    [1]
    $ curl_status=401 api '200|401' /z </dev/null; echo [$?]
    curl --silent --write-out %\{http_code\} --output /dev/null https://example.com/gitea/api/v1/z
    [0]
~~~



## loco configuration

We override loco's configuration process in a few ways: first, our command name/function prefix is always `gitea`, and we use `/etc/gitea-cli/config` file as the site-wide configuration file.  We also disable loco's "find the project root" functionality because it's not relevant for most use cases.  Instead, we default a `PROJECT_NAME` setting to the basename of the current working directory (which is helpful for `gitea vendor`.)

```shell
loco_preconfig() {
    LOCO_SCRIPT=$BASH_SOURCE
    LOCO_SITE_CONFIG=/etc/gitea-cli/config
    LOCO_COMMAND=gitea
    PROJECT_NAME="${PROJECT_NAME-$(basename "$PWD")}"
}
loco_loadproject() { :; }
loco_findproject() { LOCO_PROJECT=''; }
loco_findroot() { LOCO_ROOT=$PWD; }
```

Having configured everything we need, we can simply include loco's source code to do the rest:

```shell mdsh
cat "$(command -v loco)"
```