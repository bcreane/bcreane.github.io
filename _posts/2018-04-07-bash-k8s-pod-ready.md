---
layout: post
title:  "Blocking until a kubernetes pod is ready - bash edition"
date:   2018-04-07 02:41:11 -0700
categories: kubernetes
---

# TL;DR
Bash is rightly critized as a train wreck when it comes to building
readable, maintainable scripts. See [Stop writing shell Scripts!](http://databio.org/posts/shell_scripts.html)
for a sample.

If you're careful, it's possible to write a bash script that is
sufficiently well-written to (potentially) justify the extra maintenance costs.

I'll solve a useful (to me) problem in both bash as well as golang,
comparing up-front the costs.

First edition: bash script that blocks until a kubernetes pod is "ready,"
or a timeout fires.

# Some bash best practices

We'll use a few features of "modern" bash (if that's not too much of an oyxmoron):

* Subshells: rather than using backticks, isolating a command in a subshell
  is good security practice:

  ```
    status=$(ls)  # grab the output from a subshell
  ```

* Arithmetic:

  ```
    $((count--))  # looks like C if you squint hard enough
  ```

* String regexes: bash supports some python-esque operators which are concise
  and convenient, though you must use the clumsy `[[ ]]` test operator:

  ```
      if [[ $(echo "hello, fool") =~ "fool" ]]; then  # search for "fool" substring
        echo You are indeed a fool.
      fi
  ```

* Use `local` to reduce the scope of variables, e.g.:

  ```
  function fun() {
    local rabbit="marty"  # $rabbit is not visible outside the function "fun()"
    echo "${rabbit}"
  }

  fun             # invoke function "fun()", prints "marty"
  echo "$rabbit"  # in the global scope, "$rabbit" is uninitialized, prints ""
  ```

# Bash is glue

Obviously bash is just the glue that binds a myriad of tools together - similar
to the range of packages available to a golang program.

In this case, we'll use the following standalone programs:

* `kubectl`: reports the pod's state, optionally in json format
* `jq`: parses and filters `kubectl`'s json output

# Bash script to block until a pod is ready (or timeout)

```
#!/bin/bash
# Brendan Creane
# Block until a pod specified by a selector is "ready"
# or a timeout occurs.

#
# podStatus() - takes a selector as argument, e.g. "k8s-app=kube-dns"
# and returns the pod "ready" status as a bool string ("true"). Note that if
# the pod is in the "pending" state, there is no containerStatus yet, so
# podStatus() returns an empty string. Success means seeing the "true"
# substring, but failure can be "false" or an empty string.
#
function podStatus() {
  local label="$1"
  local status=$(kubectl get pods --selector="${label}" -o json --all-namespaces | jq -r '.items[] | .status.containerStatuses[]? | [.name, .image, .ready|tostring] |join(":")')
  echo "${status}"
}

#
# blockUntilPodIsReady() - takes a pod selector and a timeout in seconds
# as arguements. If the pod never stabilizes, bail. Otherwise return as
# soon as the pod is "ready."
#
function blockUntilPodIsReady() {
  local label="$1"
  local secs="$2"
  local friendlyPodName="$3"

  echo -n "Waiting for \"${friendlyPodName}\" to be ready: "
  until [[ $(podStatus "${label}") =~ "true" ]]; do
    if [ "$secs" -eq 0 ]; then
      echo "\"${friendlyPodName}\" never stabilized."
      exit 1
    fi

    : $((secs--))
    echo -n .
    sleep 1
  done
}

blockUntilPodIsReady "k8s-app=kube-dns" 120 "kube-dns"  # Block until kube-dns is running & ready

```

# Cost analysis

Depending on your familiarity with `jq`, `kubectl` and `bash`, writing this script could take
between half an hour up to several hours. It doesn't look much harder to read than modern typed
languages such as golang.

Some of the problems with using bash include:

* Namespaces: bash code usually lives in a single file, and there's obviously no object-oriend
  encapsulation or scope. Using `local` helps a bit since bash variables are global by default.

* Unit and functional tests: writing tests seems possible in theory, but I admit I've never
  enjoyed writing bash tests ... which brings us to the final point.

* Writing bash is not fun. I've never experienced that feeling of easy and flow with bash that
  comes from writing a beautiful C++ or golang program 

Next week, the golang equivalent using [client-go](https://github.com/kubernetes/client-go).

