---
layout: post
title:  "Blocking until a kubernetes pod is ready - golang edition"
date:   2018-05-10 12:18:11 -0700
categories: kubernetes
---

# TL;DR

While building a simple install script for [Tigera's CNX](https://www.tigera.io/cnx/)
product, I received equally emphatic advice to use [bash](https://www.gnu.org/software/bash/)
as well as [golang](https://golang.org/).

I decided to build the same (subset of) functionality twice: once in
[bash](https://bcreane.github.io/kubernetes/2018/04/07/bash-k8s-pod-ready.html)
and once again in [golang](https://github.com/bcreane/k8sutils/blob/master/utils.go).

## Outcome

* The bash function took less than half the time to write and it appears equally robust and not particularly hard to read.
* The golang executable is better for scale: unit tests, strongly typed variables, scoped functionality, and more make
  it easier to maintain and extend safely. 

# Wait till pod is running - golang edition

My package builds upon the
[kubernetes e2e utils framework](https://github.com/kubernetes/kubernetes/blob/master/test/e2e/framework/util.go).
This robust framework monitors the state of a kubernetes cluster using
[client-go](https://github.com/kubernetes/client-go).

```golang

package k8sutils

import (
	"fmt"
	"k8s.io/api/core/v1"
	meta_v1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/util/wait"
	"k8s.io/client-go/kubernetes"
	"k8s.io/kubernetes/pkg/client/conditions"
	"time"
)

// return a condition function that indicates whether the given pod is
// currently running
func isPodRunning(c kubernetes.Interface, podName, namespace string) wait.ConditionFunc {
	return func() (bool, error) {
		fmt.Printf(".") // progress bar!

		pod, err := c.CoreV1().Pods(namespace).Get(podName, meta_v1.GetOptions{IncludeUninitialized: true})
		if err != nil {
			return false, err
		}

		switch pod.Status.Phase {
		case v1.PodRunning:
			return true, nil
		case v1.PodFailed, v1.PodSucceeded:
			return false, conditions.ErrPodCompleted
		}
		return false, nil
	}
}

// Poll up to timeout seconds for pod to enter running state.
// Returns an error if the pod never enters the running state.
func waitForPodRunning(c kubernetes.Interface, namespace, podName string, timeout time.Duration) error {
	return wait.PollImmediate(time.Second, timeout, isPodRunning(c, podName, namespace))
}

// Returns the list of currently scheduled or running pods in `namespace` with the given selector
func ListPods(c kubernetes.Interface, namespace, selector string) (*v1.PodList, error) {
	listOptions := meta_v1.ListOptions{IncludeUninitialized: true, LabelSelector: selector}
	podList, err := c.CoreV1().Pods(namespace).List(listOptions)

	if err != nil {
		return nil, err
	}
	return podList, nil
}

// Wait up to timeout seconds for all pods in 'namespace' with given 'selector' to enter running state.
// Returns an error if no pods are found or not all discovered pods enter running state.
func WaitForPodBySelectorRunning(c kubernetes.Interface, namespace, selector string, timeout int) error {
	podList, err := ListPods(c, namespace, selector)
	if err != nil {
		return err
	}
	if len(podList.Items) == 0 {
		return fmt.Errorf("no pods in %s with selector %s", namespace, selector)
	}

	for _, pod := range podList.Items {
		if err := waitForPodRunning(c, namespace, pod.Name, time.Duration(timeout)*time.Second); err != nil {
			return err
		}
	}
	return nil
}
```

In addition to the `k8sutils` package above, I built a simple [`main`](https://github.com/bcreane/k8sutils/blob/master/watch/watch.go)
executable package:

```golang
package main

import (
	"flag"
	"fmt"
	"github.com/bcreane/k8sutils"
	log "github.com/sirupsen/logrus"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/util/homedir"
	"os"
	"path/filepath"
)

func main() {
	var kubeConfig *string
	if home := homedir.HomeDir(); home != "" {
		kubeConfig = flag.String("kubeconfig", filepath.Join(home, ".kube", "config"), "(optional) absolute path to the kubeconfig file")
	} else {
		kubeConfig = flag.String("kubeconfig", "", "absolute path to the kubeconfig file")
	}
	var namespace = flag.String("namespace", "default", "namespace")
	var selector = flag.String("selector", "", "pod selector")
	var timeout = flag.Int("timeout", 30, "timeout in seconds")
	flag.Parse()

	config, err := clientcmd.BuildConfigFromFlags("", *kubeConfig)
	if err != nil {
		panic(err)
	}
	clientSet, err := kubernetes.NewForConfig(config)
	if err != nil {
		panic(err)
	}

	// Block up to timeout seconds for listed pods in namespace/selector to enter running state
	err = k8sutils.WaitForPodBySelectorRunning(clientSet, *namespace, *selector, *timeout)
	if err != nil {
		log.Errorf("\nThe pod never entered running phase\n")
		os.Exit(1)
	}
	fmt.Printf("\nAll pods in namespace=\"%s\" with selector=\"%s\" are running!\n", *namespace, *selector)
}
```

Invoking the program is simple: 

```bash
    # Wait up to 30 seconds for pods w/ selector and namespace to enter running phase
    ./watch --selector=k8s-app=kube-dns --namespace=kube-system --timeout=30`
```

This works about the same as the bash script I wrote a few weeks ago.

# Take away

* Bash is quicker to write and often more concise than golang.
* golang is more maintainable and robust.

If you need a simple script that's less than a few hundred lines, consider bash.
Otherwise golang's higher upfront cost but much more robust extensibility and
verifiability are a better choice.

The real install script has grown to about 1,000 lines of bash. Don't forget to allow
head room for feature creep - your 50 line bash script may grow into 500 lines before
you know it!
