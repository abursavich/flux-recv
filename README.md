[![CircleCI](https://circleci.com/gh/fluxcd/flux-recv.svg?style=svg)](https://circleci.com/gh/fluxcd/flux-recv)

# Flux webhook receiver

The container image built herein acts as a webhook endpoint, that will
notify Flux of git push and image push events.

Flux has a `notify` method in its API, but this is unsuitable to
exposing to the internet, because

 - it handles a payload defined by the Flux API, rather than handling
   the events posted by GitHub etc.
 - to use it as a webhook endpoint, you would have to expose the Flux
   API listener to the internet; and that is terribly, terribly
   unsafe.

## Supported webhook sources

`flux-recv` understands

 - `GitHub` push events (and ping events)
 - `DockerHub` image push events
 - `GitLab` push events

## How to use it

In short:

 - construct a configuration, including shared secrets
 - run `fluxcd/flux-recv` as a sidecar in the flux deployment
 - expose the flux-recv listener to the internet
 - install webhooks at the source provider (e.g., GitHub)
 - check if it works

These are explained in more depth in the following sections.

### Constructing a configuration

Each webhook you want to receive must have its own shared secret,
which is supplied as a file mounted into the container.

In addition, there is a config YAML that tells `flux-recv` which
secrets correspond to which kinds of webhook, so it knows what to
expect.

Here is how to make a Kubernetes Secret with a shared secret and a
configuration in it:

 - create the secret -- [GitHub
   recommends](https://developer.github.com/webhooks/securing/) 20
   bytes of randomness (NB I am using `print` rather than `puts`
   because newlines will mess things up):

```
$ ruby -rsecurerandom -e 'print SecureRandom.hex(20)' > ./github.key
```

 - create a configuration that refers to it:

```sh
$ cat >> fluxrecv.yaml <<EOF
fluxRecvVersion: 1
endpoints:
- keyPath: github.key
  source: GitHub
EOF
```

The value of `source` is one of the sources supported (listed above,
and in [`sources.go`](./sources.go)).

 - create a kustomization.yaml that will construct the Secret for you:

```sh
$ cat >> kustomization.yaml <<EOF
secretGenerator:
- name: fluxrecv-config
  files:
  - github.key
  - fluxrecv.yaml
generatorOptions:
  disableNameSuffixHash: true
EOF
```

 - use kubectl to apply the kustomization:

```sh
kubectl apply -k .
```

You now have a Kubernetes secret named `fluxrecv-config`.

### Running flux-recv as a sidecar

The ideal is to run `flux-recv` as a sidecar to `fluxd`, so that the
flux API is only exposed on localhost. The additional bits you need in
the flux deployment are:

 - a volume definition that refers to the fluxrecv-config constructed
   above
 - a container spec to run the flux-recv container itself


The first bit goes under `.spec.template.volumes`:

```yaml
      # volumes:

      - name: fluxrecv-config
        secret:
          secretName: fluxrecv-config
          defaultMode: 0400
```

The second bit goes under `.spec.template.containers`:

```yaml
      # containers:

      - name: recv
        image: fluxcd/flux-recv:0.2.0
        imagePullPolicy: IfNotPresent
        args:
        - --config=/etc/fluxrecv/fluxrecv.yaml
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: fluxrecv-config
          mountPath: /etc/fluxrecv
```

You do not need to alter the container spec for the `flux` container,
though you may want to supply the argument `--listen=localhost:3030`
to limit API access to localhost, if you don't already.

> If you do restrict API access to localhost, make sure you also
> 
>  - supply `--listen-metrics=:3031`
>  - annotate the pod with `prometheus.io/port: "3031"`, so
>    Prometheus knows which port to scrape).
>  - remove any probes that rely on reaching the API

### Expose flux-recv to the internet

To expose the flux-recv listener to the internet so that webhooks can
reach it, you will need to:

 - make a Kubernetes Service for `flux-recv`; and,
 - either
   - tell an Ingress to route requests to the service; OR
   - use ngrok to tunnel requests through to flux-recv

#### Making a service for flux-recv

To be able to route requests to `flux-recv` from anywhere else, it
needs a Kubernetes Service. Here's a suitable definition:

```sh
$ cat > flux-recv-service.yaml <<EOF
---
apiVersion: v1
kind: Service
metadata:
  name: flux-recv
spec:
  type: ClusterIP
  ports:
    - name: recv
      port: 8080
      targetPort: 8080
  selector:
    name: flux
EOF
$ kubectl apply -f ./flux-recv-service.yaml
```

The selector assumes that your flux deployment (which now includes the
flux-recv sidecar) has a label `name: flux`.

#### Using an Ingress

If running in the cloud, you will need to route though an Ingress or
an analogue, and that will vary depending how you're already using
that. However, these things will be in common:

 - the URLs at flux-recv are of the form `/hook/<digest>`, where
   `<digest>` is the SHA265 digest of a hook's shared secret,
   hex-encoded. flux-recv will print out these endpoints when it
   starts, or you can calculate them with e.g., `sha256 -b
   ./github.key`;

 - the backend will be the `flux-recv` service created previously,
   with the port `8080`.

#### Using ngrok

If running locally (e.g., while developing flux-recv itself), it will
be more convenient to use `ngrok` to tunnel requests through to your
cluster.

In general, ngrok will create a new hostname each time it runs, and
you will want to run it in its own deployment so it doesn't get
restarted too much.

The kustomization in [`./example/ngrok`](./example/ngrok/) contains an
_almost_ complete configuration. You need to obtain an auth token by
signing up for an ngrok.com account, and paste it into the field
indicated in `example/ngrok.yml`, before running

```sh
$ kubectl apply -k ./example
```

> Alternatively, you can _not_ sign up to ngrok.com, and remove the
> volume mount and `-config` arg from the deployment. In this case,
> you will need to restart ngrok every few hours to get a fresh
> tunnel, and reinstall any webhooks with the new hostname for the
> tunnel.

The _host_ part of your webhook URLs will be something like
`https://abcd1234.ngrok.io`, and it will change each time ngrok starts
(which is why we went to some trouble to run it in its own
deployment).

The example ngrok configuration includes a Service with a NodePort, so
you can access ngrok's web UI. You may wish to remove the NodePort, or
the service entirely, and use `kubectl port-forward` instead.

### Install webhook at the source

Each webhook source has its own user interface or API for installing
webhooks. In general though, you will usually need

 - a URL for the webhook
 - the shared secret

You can get the webhook URL for an endpoint by combining the host --
which will depend on how you exposed `flux-recv` to the internet in
the previous step -- and the path, which is

    '/hook/' + sha256sum(secret)

You can see the path in the log output of `flux-recv`, or calculate
the digest yourself with, e.g.,

```
$ sha256sum -b ./github.key
```

The webhook URL in total will look something like

    https://abcd1234.ngrok.io/hook/7962c728be656d9580d0ce9bda78320c946d8321a4ba7f31ea15c7f2d471bd26

The path may differ if you are routing through an ingress or load
balancer.

GitHub (and others) require the shared secret, which you can take
directly from the file created in the first step (be careful not to
introduce extra characters into the file, if you load it in an
editor).

### Check if it works

The easiest way to do this is to watch fluxd's logs, and push a new
commit (or image) to the repo for which you installed the hook.

    kubectl logs deploy/flux -f


You should see a log message indicating that fluxd has either

 * refreshed the git repo

```
ts=2019-11-20T16:29:26.871873132Z caller=loop.go:127 component=sync-loop event=refreshed url=ssh://git@github.com/squaremo/flux-example
```

 * ignored the notification because it's the wrong branch

```
ts=2019-11-19T15:03:06.477412849Z caller=daemon.go:540 component=daemon msg="notified about unrelated change" url=git@github.com:squaremo/flux-example.git branch=refs/tags/flux-write-check
```

 * scheduled an image repo refresh

```
ts=2019-11-20T16:34:30.134889169Z caller=warming.go:95 component=warmer priority=squaremo/helloworld
```

Some sources (e.g., GitHub, GitLab) let you test or re-rerun hooks
from their webhook configuration, which can be useful for
debugging. If you are using ngrok, it's also possible to re-run a hook
from its dashboard.

It is safe to re-run hooks, because `fluxd` treats notifications as a
trigger to refresh state, rather than as authoritative themselves. For
example, when informed of an image push, fluxd does not add the image
mentioned to its database -- it polls the image registry in question
to determine whether there is a new image.
