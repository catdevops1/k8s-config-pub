# When one hostname blocks a certificate renewal

I run a single TLS certificate covering all the hostnames served by my Gateway.
It renewed itself quietly for months, then one day it didn't, and the way it
failed took a while to understand. This is what I learned.

## One bad name stops everything

An ACME order is all or nothing. If the certificate lists nine hostnames, all
nine have to pass validation. Eight passing and one failing gets you nothing.

That's easy to miss because nothing looks broken at first. The old certificate is
still valid, sites keep working, and the Certificate object still reports
`READY: True`. All that means is the certificate you already have hasn't expired
yet. It says nothing about whether the renewal went through.

The thing to check is the expiry date:

```bash
kubectl get certificate <name> -n <ns> -o jsonpath='{.status.notAfter}{"\n"}'
```

If that date is still the old one 30 days out, the renewal is stuck.

## The two ways a name fails

**The DNS record is gone.** You retire or pause an app, delete its DNS record,
and forget that its hostname is still listed on the certificate. cert-manager
tries to reach it and gets nothing back:

```
Waiting for HTTP-01 challenge propagation: failed to perform self check GET request
... dial tcp: lookup app.your-domain.com on 10.96.0.10:53: no such host
```

A hostname lives in three places: the certificate, the tunnel or proxy config,
and DNS. Remove it from one and not the others and you've planted a failure that
won't show up until the next renewal, which could be months away. That delay is
what makes it confusing when it finally breaks.

**The name sits behind a login.** If a Zero Trust or SSO layer intercepts
requests before they reach the cluster, the challenge request gets a login page
instead of the token:

```
Waiting for HTTP-01 challenge propagation: did not get expected response when
querying endpoint, expected "<token>" but got: <!DOCTYPE html>...
```

HTTP-01 can't work here. Let's Encrypt can't log in. Either add a bypass rule for
`/.well-known/acme-challenge/*` or move to DNS-01.

## Why it got worse over time

cert-manager keeps retrying a failing order, and with the HTTP-01 solver pointed
at a Gateway, each retry leaves another solver HTTPRoute behind. They pile up.

Every route change makes the Envoy Gateway control plane re-translate the whole
route set and push it again over xDS. As the pile grew, so did the work, until
the controller couldn't answer its health probes or renew its leader lease in
time and the kubelet started killing it. It restarted in a loop for hours.

Traffic never stopped, which is the interesting part. The Envoy proxies keep
serving the last configuration they were given, so visitors saw nothing wrong.
The cost only shows up when something needs to change: the renewed certificate
sat in its Secret and never reached Envoy, because the controller kept dying
before it could push it.

That's why it's worth checking what's actually on the wire rather than trusting
the Secret:

```bash
echo | openssl s_client -connect <YOUR-METALLB-IP>:443 -servername your-domain.com 2>/dev/null \
  | openssl x509 -noout -enddate
```

## Finding the stuck name

```bash
kubectl get challenges -A -o wide
```

Each pending challenge names the hostname and the reason. That one command would
have saved me a lot of guessing.

The leftover solver routes are visible too:

```bash
kubectl get httproute -n envoy-gateway-system | grep acme
```

They clean themselves up once the order completes. If the challenges are gone but
routes remain:

```bash
kubectl delete httproute -n envoy-gateway-system -l acme.cert-manager.io/http01-solver=true
```

## Getting out of it

The quick fix is to drop the hostnames that can't validate from the Certificate's
`dnsNames`. The names that already passed reuse their existing authorizations, so
issuance usually happens within a minute or two.

The real fix is DNS-01 with a wildcard:

```yaml
solvers:
  - dns01:
      cloudflare:
        apiTokenSecretRef:
          name: cloudflare-api-token
          key: api-token
```

```yaml
dnsNames:
  - your-domain.com
  - "*.your-domain.com"
```

DNS-01 proves you own the domain by writing a TXT record through your DNS
provider's API. It never visits the site, so the tunnel, the proxy, a login layer
and the state of the Gateway stop mattering. And with a wildcard the certificate
no longer lists individual app hostnames, so adding, pausing or retiring an app
can't break a renewal again.

One catch: a wildcard covers a single label. `*.your-domain.com` doesn't cover
the apex, so list both.

## What I'd monitor

- A Certificate stuck with an `Issuing` condition for more than an hour
- Expiry under 21 days, which means renewal has been failing for over a week
- Restart count climbing on the Envoy Gateway control-plane Deployment, since
  nothing else makes that visible
