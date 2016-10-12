kubernetes-che
==============

Example deploying **[Eclipse Che](https://github.com/eclipse/che/)** on a [Kubernetes](http://kubernetes.io/) cluster.
So what you get is a Che server running on your cluster behind [SPDY Proxy](https://github.com/igrigorik/node-spdyproxy) to handle authencation and encryption.

Note: Currently alpha state; barely tested.


Usage
-----

### Deployment

 1. You need a running [Kubernetes](http://kubernetes.io/) cluster (for example [Google Container Engine](https://cloud.google.com/container-engine/)).
 2. Edit `kubernetes.yml` and see the `TODO`; update with your values.
 3. Run `kubectl apply -f kubernetes.yml`
 4. Follow client-side set up (see below).

### Client-side set up

For **Chrome** install [SwitchyOmega](https://chrome.google.com/webstore/detail/proxy-switchyomega/padekgcemlokbadohgkifijomclgjgif).
Then add a PAC Profile with the [PAC Script](https://en.wikipedia.org/wiki/Proxy_auto-config) below:

    function FindProxyForURL(url, host) {
      if (dnsDomainIs(host, "che")) {
        # TODO: Replace with your host/IP pointing to SPDY Proxy.
        return "HTTPS che.example.com:44300";
      }

      return "DIRECT";
    }

Click the lock to set username/password according to values you've set in `kubernetes.yml`.

Supposing you've set up your DNS to point to the Ingress Controller external IP
(or actually to SPDY Proxy) and the SSL certificate of that host is trusted;
you should now be able to connect to `http://che/`.


How it works
------------

The beginning of `kubernetes.yml` is to get a valid TLS certificate. It sets up an Ingress Controller
which, if you don't already have one, can be used for other services running on your cluster.
It's not critical for the system to work, but you need a valid certificate when trying proxy through
SPDY Proxy else your browser will refuse to use it.

SPDY Proxy is used to add **authentication, encryption**, and access che running on your cluster.
The rules are set up so that it's only used when trying to access Che.

The critical part is that `che` resolves to the container running the Docker server used by the Che server,
from withing your cluster (and thus also from outside your cluster via SPDY Proxy).

It runs the Che server and links it to a running Docker daemon (Docker-in-Docker is running also on Kubernetes).

### Isn't Docker-in-Docker discouraged?

The [original blog post](https://jpetazzo.github.io/2015/09/03/do-not-use-docker-in-docker-for-ci/) is partially outdated. The current [`docker:dind`](https://hub.docker.com/_/docker) is slow (because it uses `--storage-driver=vfs`) but should not corrupt data regardless of the underlying file system.

It's required here and desired for the following reasons:

  * CRITICAL: Kubernetes spawns on multiple machines but you need to have Docker and Che on the same host to be able to mount the same directory on both. These volume mounts are required by Che:
       - `lib-copy` used to bootstrap a workspace
       - `storage` used to access some settings like SSH keys
       - `workspaces` used to run `docker run -v ...` from within a Che workspace
  * Cleaner: Deleting the Pod or if it fails health checks it'll not block or pollute the file system of Kubernetes itself.
  * Provide a simple way to access the Docker host from within Kubernetes cluster (passed via `--remote:che` to Che).

What could be improved
----------------------

 * Requires clients to setup and use that SPDY Proxy (at least until [#1560](https://github.com/eclipse/che/issues/1560) is fixed)
 * Currently only **listing some ports**: All ports in range 32768-65535 should point to the Pod running `docker:dind`.
   One way would be to delete the `che` Service, and instead of using a Deployment, directly create a Pod
   named `che` (but I don't like that idea). Another idea is just to wait for issue [#1560](https://github.com/eclipse/che/issues/1560).
 * Long lived HTTP requests like the terminal, or possibly other parts, seem to disconnect after a while if there was no I/O. Not tested which part is responsible for this issue.
