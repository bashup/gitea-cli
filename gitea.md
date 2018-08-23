#!/usr/bin/env bash
: '
<!-- ex: set syntax=markdown : '; eval "$(mdsh -E "$BASH_SOURCE")"; # -->

```shell mdsh
@module gitea.md
@require pjeby/license @comment LICENSE
@require bashup/loco mdsh-source "$BASHER_PACKAGES_PATH/bashup/loco/loco.md"
@main loco_main
```

### Contents

<!-- toc -->

- [An Extensible CLI for gitea and gogs](#an-extensible-cli-for-gitea-and-gogs)
  * [Prefix Options](#prefix-options)
    + [--with *key* *val*](#--with-key-val)
    + [--description / --desc / -d *description*](#--description----desc---d-description)
    + [--public / -p](#--public---p)
    + [--private / -P](#--private---p)
    + [--repo / -r *repo*](#--repo---r-repo)
    + [--tag / -t *version*](#--tag---t-version)
  * [Commands](#commands)
    + [gitea exists *repo*](#gitea-exists-repo)
    + [gitea delete *repo*](#gitea-delete-repo)
    + [gitea deploy-key *repo keytitle key*](#gitea-deploy-key-repo-keytitle-key)
    + [gitea new *repo [opts...]*](#gitea-new-repo-opts)
    + [gitea vendor [create-opts...]](#gitea-vendor-create-opts)
  * [Utilities](#utilities)
    + [Repository Names](#repository-names)
      - [split_repo](#split_repo)
    + [json/jq](#jsonjq)
      - [jmap](#jmap)
    + [API Wrappers](#api-wrappers)
  * [loco configuration](#loco-configuration)

<!-- tocstop -->

# An Extensible CLI for gitea and gogs

This file is the source code and main tests for the [generated gitea client](bin/gitea).  Tests look like this:

~~~shell
# Source the functions in this file and initialize as if the loco command were running:
    $ source $TESTDIR/$TESTFILE; set +e
    $ gitea.no-op() { :;}

# Ignore/null out all configuration for testing
    $ loco_user_config() { :;}
    $ loco_site_config() { :;}
    $ loco_findproject() { LOCO_PROJECT=/dev/null; }
    $ loco_main no-op

# dummy `curl` for testing
    $ curl_status=200
    $ GITEA_URL=https://example.com/gitea/
    $ GITEA_USER=some_user
    $ GITEA_API_TOKEN=EXAMPLE_TOKEN
    $ curl() {
    >     { printf -v REPLY ' %q' "curl" "$@"; echo "${REPLY# }"; cat; } >&2;
    >     echo $curl_status;
    > }
~~~

## Prefix Options

The prefix options work by setting variables and invoking the remainder of the command line, so for testing we'll make a subcommand that dumps out those variables:

~~~shell
    $ gitea.dump() {
    >     declare -p PROJECT_ORG PROJECT_NAME PROJECT_TAG GITEA_CREATE 2>/dev/null || true
    > }
    $ gitea dump
    declare -- PROJECT_NAME="gitea.md"
    declare -a GITEA_CREATE=()

# Try some combos and abbreviations:

    $ gitea -t 1.2 -r bada/bing --desc foobly --with x y -p -P dump
    declare -- PROJECT_ORG="bada"
    declare -- PROJECT_NAME="bing"
    declare -- PROJECT_TAG="1.2"
    declare -a GITEA_CREATE=([0]="description" [1]="foobly" [2]="x" [3]="y" [4]="private=" [5]="false" [6]="private=" [7]="true")
~~~

### --with *key* *val*

`--with` alters `GITEA_CREATE` to add the given key-value pair:

```shell
GITEA_CREATE=()
gitea.--with() {
    local GITEA_CREATE=(${GITEA_CREATE[@]+"${GITEA_CREATE[@]}"} "${@:1:2}")
    gitea "${@:3}"
}
```

~~~shell
    $ gitea --with foo bar dump
    declare -- PROJECT_NAME="gitea.md"
    declare -a GITEA_CREATE=([0]="foo" [1]="bar")
~~~

### --description / --desc / -d *description*

These options are short for `--with description` *description*:

```shell
gitea.--description() { gitea --with description "$1" "${@:2}"; }
gitea.--desc()        { gitea --description "$@"; }
gitea.-d()            { gitea --description "$@"; }
```

~~~shell
    $ gitea --description something dump
    declare -- PROJECT_NAME="gitea.md"
    declare -a GITEA_CREATE=([0]="description" [1]="something")
~~~

### --public / -p

```shell
gitea.--public() { gitea --with private= false "$@"; }
gitea.-p()       { gitea --public "$@"; }
```

~~~shell
    $ gitea --public dump
    declare -- PROJECT_NAME="gitea.md"
    declare -a GITEA_CREATE=([0]="private=" [1]="false")
~~~

### --private / -P

```shell
gitea.--private() { gitea --with private= true "$@"; }
gitea.-P()        { gitea --private "$@"; }
```

~~~shell
    $ gitea --private dump
    declare -- PROJECT_NAME="gitea.md"
    declare -a GITEA_CREATE=([0]="private=" [1]="true")
~~~

### --repo / -r *repo*

Set `PROJECT_ORG` and `PROJECT_NAME` from *repo*:

```shell
gitea.--repo() {
    split_repo "$1"; local PROJECT_ORG="${REPLY[1]}" PROJECT_NAME="${REPLY[2]}"; gitea "${@:2}"
}
gitea.-r() { gitea --repo "$@"; }
```

~~~shell
    $ gitea --repo foo/bar dump
    declare -- PROJECT_ORG="foo"
    declare -- PROJECT_NAME="bar"
    declare -a GITEA_CREATE=()

    $ gitea --repo baz dump
    declare -- PROJECT_ORG="some_user"
    declare -- PROJECT_NAME="baz"
    declare -a GITEA_CREATE=()
~~~

### --tag / -t *version*

Set `PROJECT_TAG` from *version*:

```shell
gitea.--tag()  { local PROJECT_TAG="$1"; gitea "${@:2}" ; }
gitea.-t()     { gitea --tag "$@"; }
```

~~~shell
    $ gitea --tag a.b dump
    declare -- PROJECT_NAME="gitea.md"
    declare -- PROJECT_TAG="a.b"
    declare -a GITEA_CREATE=()
~~~

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

### gitea delete *repo*

Delete the repository:

```shell
gitea.delete() { split_repo "$1" && auth api 204 "/repos/$REPLY" -X DELETE; }
```

~~~shell
    $ gitea delete foo/bar </dev/null
    curl --silent --write-out %\{http_code\} --output /dev/null -X DELETE -H Authorization:\ token\ EXAMPLE_TOKEN https://example.com/gitea/api/v1/repos/foo/bar
    [1]
    $ curl_status=204 gitea delete foo/bar </dev/null; echo [$?]
    curl --silent --write-out %\{http_code\} --output /dev/null -X DELETE -H Authorization:\ token\ EXAMPLE_TOKEN https://example.com/gitea/api/v1/repos/foo/bar
    [0]
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

Create the repository; *opts* are key-value pairs to pass to the API, such as `description` and `private=`.  Defaults from `${GITEA_CREATE[@]}` are used first, if set.  A deploy key is automatically set if `$GITEA_DEPLOY_KEY` is non-empty; its title will be `$GITEA_DEPLOY_KEY_TITLE` or `"default"` if empty or missing:

```shell
gitea.new() {
    split_repo "$1"; local org="${REPLY[1]}" repo="${REPLY[2]}"
    if [[ $org == "$GITEA_USER" ]]; then org=user; else org="org/$org"; fi
    jmap name "$repo" "${GITEA_CREATE[@]}" "${@:2}" |
        json api "200|201" "$org/repos?token=$GITEA_API_TOKEN"
    [[ ! "${GITEA_DEPLOY_KEY-}" ]] || gitea deploy-key "$1" "${GITEA_DEPLOY_KEY_TITLE:-default}" "$GITEA_DEPLOY_KEY"
}
```

~~~shell
# Defaults apply before command line; default API url is /org/ORGNAME/repos
    $ GITEA_CREATE=(private= true)
    $ export GIT_AUTHOR_NAME="PJ Eby" EMAIL="null@example.com"
    $ gitea new biz/baz description whatever
    curl --silent --write-out %\{http_code\} --output /dev/null -X POST -H Content-Type:\ application/json -d @- https://example.com/gitea/api/v1/org/biz/repos\?token=EXAMPLE_TOKEN
    {
      "name": "baz",
      "private": true,
      "description": "whatever"
    }

# When the repo is the current user, the API url is /user/repos
    $ gitea --public -d something new some_user/spam
    curl --silent --write-out %\{http_code\} --output /dev/null -X POST -H Content-Type:\ application/json -d @- https://example.com/gitea/api/v1/user/repos\?token=EXAMPLE_TOKEN
    {
      "name": "spam",
      "private": false,
      "description": "something"
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

### gitea vendor [create-opts...]

```shell
gitea.vendor-merge() { :; }

branch-exists() { git rev-parse --verify "$1" &>/dev/null; }

gitea.vendor() {
    [[ ! -d .git ]] || loco_error ".git repo must not exist here";
    [[ -n "${PROJECT_ORG-}"  ]] || PROJECT_ORG=$GITEA_USER
    [[ -n "${PROJECT_NAME-}" ]] || loco_error "PROJECT_NAME not set"

    local GITEA_GIT_URL=${GITEA_GIT_URL-$GITEA_URL}
    [[ $GITEA_GIT_URL == *: ]] || GITEA_GIT_URL="${GITEA_GIT_URL%/}/";
    local GIT_REPO="$GITEA_GIT_URL$PROJECT_ORG/$PROJECT_NAME.git"

    if gitea exists "$PROJECT_ORG/$PROJECT_NAME"; then
        [[ -n "${PROJECT_TAG-}"  ]] || loco_error "PROJECT_TAG not set"
        local MESSAGE="Vendor update to $PROJECT_TAG"
        git clone --bare -b vendor "$GIT_REPO" .git ||
        git clone --bare "$GIT_REPO" .git   # handle missing-branch case
    else
        local MESSAGE="Initial import"
        gitea new "$PROJECT_ORG/$PROJECT_NAME" "$@"
        git clone --bare "$GIT_REPO" .git
    fi

    git config --local --bool core.bare false
    git config --local user.name "${GITEA_VENDOR_NAME:-Vendor}"
    git config --local user.email "${GITEA_VENDOR_EMAIL:-vendor@example.com}"

    git add .; git commit -m "$MESSAGE"             # commit to master or vendor
    branch-exists vendor || git checkout -b vendor  # split off vendor branch if needed
    git push --all

    [[ -z "${PROJECT_TAG-}" ]] || { git tag "vendor-$PROJECT_TAG"; git push --tags; }

    git checkout master
    gitea vendor-merge
}
```

~~~shell
# Mock a gitea repo w/a file URL
    $ GITEA_GIT_URL=$PWD
    $ mkdir -p some_user/foo
    $ git --git-dir=some_user/foo.git init --bare  # fake gitea new
    Initialized empty Git repository in /*/gitea.md/some_user/foo.git/ (glob)

# New Repository
    $ mkdir foo; cd foo; echo "v1" >f
    $ curl_status=404 gitea -p -r foo -t 1.1 vendor </dev/null 2>&1|grep -v '^ Author:'
    curl --silent --write-out %\{http_code\} --output /dev/null -H Authorization:\ token\ EXAMPLE_TOKEN https://example.com/gitea/api/v1/repos/some_user/foo
    curl --silent --write-out %\{http_code\} --output /dev/null -X POST -H Content-Type:\ application/json -d @- https://example.com/gitea/api/v1/user/repos\?token=EXAMPLE_TOKEN
    {
      "name": "foo",
      "private": false
    }
    Cloning into bare repository '.git'...
    warning: You appear to have cloned an empty repository.
    done.
    [master (root-commit) *] Initial import (glob)
     1 file changed, 1 insertion(+)
     create mode 100644 f
    Switched to a new branch 'vendor'
    To /*/some_user/foo.git (glob)
     * [new branch]      master -> master
     * [new branch]      vendor -> vendor
    To /*/some_user/foo.git (glob)
     * [new tag]         vendor-1.1 -> vendor-1.1
    Switched to branch 'master'

# Existing Repository
    $ rm -rf .git
    $ echo "v2" >>f
    $ gitea -r foo -t 1.2 vendor </dev/null 2>&1|grep -v '^ Author:'
    curl --silent --write-out %\{http_code\} --output /dev/null -H Authorization:\ token\ EXAMPLE_TOKEN https://example.com/gitea/api/v1/repos/some_user/foo
    Cloning into bare repository '.git'...
    done.
    [vendor *] Vendor update to 1.2 (glob)
     1 file changed, 1 insertion(+)
    To /*/some_user/foo.git (glob)
     *..* vendor -> vendor (glob)
    To /*/some_user/foo.git (glob)
     * [new tag]         vendor-1.2 -> vendor-1.2
    Switched to branch 'master'

    $ cd ..
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
            ((arg++))
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
    REPLY=$(curl --silent --write-out '%{http_code}' --output /dev/null "${@:3}" "${GITEA_URL%/}/api/v1/${2#/}")
    eval 'case $REPLY in '"$1) true ;; 000) echo "Failure: Invalid server response. Check GITEA_URL"; false ;; 401) echo "Failure: Unauthorized. Check GITEA_USER and GITEA_API_TOKEN"; false ;; *) echo "Failure: Server returned HTTP code $REPLY" ; false ;; esac"
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

We override loco's configuration process in a few ways: first, our command name/function prefix is always `gitea`, and we use `/etc/gitea-cli/config` file as the site-wide configuration file.  We also change loco's "find the project root" functionality so it looks for a `.gitearc` but doesn't change to that directory.  We also default a `PROJECT_NAME` setting to the basename of the current working directory (which is helpful for `gitea vendor`.)

```shell
loco_preconfig() {
    LOCO_SCRIPT=$BASH_SOURCE
    LOCO_SITE_CONFIG=/etc/gitea-cli/gitearc
    LOCO_USER_CONFIG=$HOME/.config/gitearc
    LOCO_NAME=gitea
    LOCO_FILE=(.gitearc)
    PROJECT_NAME="${PROJECT_NAME-$(basename "$PWD")}"
}
loco_findproject() {
    findup "$LOCO_PWD" "${LOCO_FILE[@]}" && LOCO_PROJECT=$REPLY || LOCO_PROJECT=/dev/null
}
loco_findroot() { LOCO_ROOT=$LOCO_PWD; }
```
