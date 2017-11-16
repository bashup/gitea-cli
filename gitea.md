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
    $ GITEA_USER=some_user
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

### gitea deploy-key *repo keytitle key*

Add a deployment key *key* named *keytitle* to *repo*.  Returns success if the key was successfully added.

```shell
gitea.deploy-key() {
    split_repo "$1"
    jmap title "$2" key "$3" | json auth api 201 /repos/$REPLY/keys
}
```

~~~shell
    $ curl_status=201 gitea deploy-key foo/bar baz spam
    curl --silent --write-out %\{http_code\} --output /dev/null -X POST -H Content-Type:\ application/json -d @- -H Authorization:\ token\ EXAMPLE_TOKEN https://example.com/gitea/api/v1/repos/foo/bar/keys
    {
      "title": "baz",
      "key": "spam"
    }
~~~

### gitea new *repo [opts...]*

Create the repository; *opts* are key-value pairs to pass to the API, such as `description` and `private=`.  Defaults from `${GITEA_CREATE_DEFAULTS[@]}` are used first, if set.  A deploy key is automatically set if `$GITEA_DEPLOY_KEY` is non-empty; its title will be `$GITEA_DEPLOY_KEY_TITLE` or `"default"` if empty or missing:

```shell
gitea.new() {
    split_repo "$1"; local org="${REPLY[1]}" repo="${REPLY[2]}"
    if [[ $org == "$GITEA_USER" ]]; then org=user; else org="org/$org"; fi
    jmap name "$repo" "${GITEA_CREATE_DEFAULTS[@]}" "${@:2}" |
        json api "200|201" "$org/repos?token=$GITEA_API_TOKEN"
    [[ ! "${GITEA_DEPLOY_KEY-}" ]] || gitea deploy-key "$1" "${GITEA_DEPLOY_KEY_TITLE:-default}" "$GITEA_DEPLOY_KEY"
}
```

~~~shell
# Defaults apply before command line; default API url is /org/ORGNAME/repos
    $ GITEA_CREATE_DEFAULTS=(private= true)
    $ gitea new biz/baz description whatever
    curl --silent --write-out %\{http_code\} --output /dev/null -X POST -H Content-Type:\ application/json -d @- https://example.com/gitea/api/v1/org/biz/repos\?token=EXAMPLE_TOKEN
    {
      "name": "baz",
      "private": true,
      "description": "whatever"
    }
# When the repo is the current user, the API url is /user/repos
    $ gitea new some_user/spam private= false
    curl --silent --write-out %\{http_code\} --output /dev/null -X POST -H Content-Type:\ application/json -d @- https://example.com/gitea/api/v1/user/repos\?token=EXAMPLE_TOKEN
    {
      "name": "spam",
      "private": false
    }
# Deployment happens if you provide a GITEA_DEPLOY_KEY
    $ GITEA_DEPLOY_KEY=example-key curl_status=201 gitea new foo/bar
    curl --silent --write-out %\{http_code\} --output /dev/null -X POST -H Content-Type:\ application/json -d @- https://example.com/gitea/api/v1/org/foo/repos\?token=EXAMPLE_TOKEN
    {
      "name": "bar",
      "private": true
    }
    curl --silent --write-out %\{http_code\} --output /dev/null -X POST -H Content-Type:\ application/json -d @- -H Authorization:\ token\ EXAMPLE_TOKEN https://example.com/gitea/api/v1/repos/foo/bar/keys
    {
      "title": "default",
      "key": "example-key"
    }
# and it can have a GITEA_DEPLOY_KEY_TITLE
    $ GITEA_DEPLOY_KEY=example-key GITEA_DEPLOY_KEY_TITLE=sample-title curl_status=201 gitea new foo/bar
    curl --silent --write-out %\{http_code\} --output /dev/null -X POST -H Content-Type:\ application/json -d @- https://example.com/gitea/api/v1/org/foo/repos\?token=EXAMPLE_TOKEN
    {
      "name": "bar",
      "private": true
    }
    curl --silent --write-out %\{http_code\} --output /dev/null -X POST -H Content-Type:\ application/json -d @- -H Authorization:\ token\ EXAMPLE_TOKEN https://example.com/gitea/api/v1/repos/foo/bar/keys
    {
      "title": "sample-title",
      "key": "example-key"
    }
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

#### jmap

The `jmap` function takes a series of key-value argument pairs and outputs a JSON-encoded mapping using `jq`.  Keys with a trailing `=` treat their value as a JSON-encoded value or jq expression; all others are treated as strings for jq to encode:

```shell
jmap() {
    local filter='{}' opts=(-n) arg=1
    while (($#)); do
        if [[ $1 == *= ]]; then
            filter+=" | .$1 $2"
        else
            filter+=" | .$1=\$__$arg"
            opts+=(--arg "__$arg" "$2")
            let arg++
        fi
        shift 2
    done
    jq "${opts[@]}" "$filter"
}
```

~~~shell
    $ jmap foo bar baz spam thing= true calc= 21 blue= '.calc*2'
    {
      "foo": "bar",
      "baz": "spam",
      "thing": true,
      "calc": 21,
      "blue": 42
    }
~~~

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