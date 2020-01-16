# ingress-controller-conformance

The goal of this project is to act as an executable specification in the form of a test suite, implementing a standard, actively maintained ingress-controller conformance specification.

The conformance test suite will both ensure consistency across implementations, as well as simplify the work needed for other implementations to conform to the specification. The test suite can also be viewed through human readable descriptions of what it is testing so that implementers can understand the tests without reading source code.

Currently, the `ingress-controller-conformance` supports the `Ingress` resource from [`networking.k8s.io/v1beta1` API](https://kubernetes.io/docs/concepts/services-networking/ingress/), with the goal to serve as a benchmark for the Ingress resource under the `networking.k8s.io/v1` API and the [Ingress/Service V2 evolution](https://kubernetes-sigs.github.io/service-apis/).

## Coverage

The current suite of tests covers the following features of the `Ingress` resource:
- Plain text HTTP/1.1 requests
- Exact and wildcard Host rules
- Prefix path matches rules
- No rules single-service matching

Future tests should align with the Ingress resource specification and support:
- Requests not matching any ingress rules should result in `404 Not Found` responses. Currently, all unmatched requests will fallback to a catch-all route.
- Multiple exposed ingress addresses and ports.
- Ingress resources with backend services across different namespaces.
- HTTPS with SSL termination and SNI at the ingress-controller.
- Support different path match modes for `networking.k8s.io/v1`: "exact" and "prefix" path types.
- Other TCP protocols and UDP.
- Load-balancing between multiple upstream instances. 

## Running

### CLI

At the moment, `ingress-controller-conformance` does not apply modifications to the running resources in the target Kubernetes cluster.

You must manually install and setup you environment targeted by the `kubectl config current-context`:
1. Apply the backing ingress-controller implementation. Samples are available under `examples/`.
1. Apply all, or a subset, of the ingress and service resources found under `deployments`. Files under the deployments folder match implemented check names.

#### Context

```
$ ./ingress-controller-conformance context
Using active Kubernetes context 'docker-desktop'
The target Kubernetes cluster is running verion v1.14.6
  Supports Ingress kind APIVersion: 'extensions/v1beta1'
  Supports Ingress kind APIVersion: 'networking.k8s.io/v1beta1'
```

#### List

List, in a human-readable form, all Ingress verifications
```
$ ./ingress-controller-conformance list
- Ingress with host rule should send traffic to the correct backend service (host-rules)
- [...]
- Ingress with no rules should send traffic to the correct backend service (single-service)
```

#### Verify

Execute a series of assertions on the deployed ingress-controller, using your `kubectl` current-context.
```
$ ./ingress-controller-conformance verify
Running 'all' verifications...
Running 'host-rules' verifications...
Running 'path-rules' verifications...
Running 'path-rules-foo' verifications...
        1) Assertion failed: Expected the request path would be '/foo' but was '/'
  Check failed: path-rules-foo
Running 'path-rules-foo-trailing' verifications...
        1) Assertion failed: Expected the request path would be '/foo/' but was '//'
  Check failed: path-rules-foo-trailing
Running 'path-rules-bar' verifications...
        1) Assertion failed: Expected the request path would be '/bar/' but was '/'
  Check failed: path-rules-bar
Running 'path-rules-bar-subpath' verifications...
        1) Assertion failed: Expected the request path would be '/bar//bershop' but was '//bershop'
  Check failed: path-rules-bar-subpath
Running 'single-service' verifications...
--- Verification completed ---
3 checks passed! 4 failures!
in 1.777148914s
```

#### Help

```
$ ./ingress-controller-conformance help

Kubernetes ingress controller conformance test suite in Go.
  Complete documentation is available at https://github.com/kubernetes-sigs/ingress-controller-conformance

Usage:
  ingress-controller-conformance [command]

Available Commands:
  context     Print the current Kubernetes context, server version, and supported Ingress APIVersions
  help        Help about any command
  list        List all Ingress verifications
  verify      Run Ingress verifications for conformance

Flags:
  -h, --help   help for ingress-controller-conformance

Use "ingress-controller-conformance [command] --help" for more information about a command.
```

### ingress-conformance-echo

The `ingress-conformance-echo` binary is published as docker image of the same name. The purpose of this component is to handle backend-requests made through an Ingress interface and respond using data from the original request. This, in turn, allows to build assertions on the original HTTP request as it is relayed through the ingress-controller.

```
# You may chose to pass in a TEST_ID environment variable to assert which running process actually handles the downstream request.
$ TEST_ID=sample ./echoserver
Starting server, listening on port 3000
Reporting TestId 'sample'

# You can then send test HTTP requests on localhost:3000
$ curl localhost:3000
{"TestId":"sample","Path":"/","Host":"localhost:3000","Method":"GET","Proto":"HTTP/1.1","Headers":{"Accept":["*/*"],"User-Agent":["curl/7.54.0"]}}
```

---

## Building

### Local build

#### Build and push the `ingress-conformance-echo` docker image:
```console
$ docker build -f build/package/Dockerfile . -t agervais/ingress-conformance-echo:latest
$ docker push agervais/ingress-conformance-echo:latest
```

#### Build the `ingress-controller-conformance` CLI and the `echo-server`:
```console
$ go build
$ go build -o echoserver tools/echoserver.go
```

### Release build

```console
$ export VERSION=0.0.1 # Or git tag
$ go build -ldflags "-X github.com/kubernetes-sigs/ingress-controller-conformance/cmd.VERSION=$VERSION"
```