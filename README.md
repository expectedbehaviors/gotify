# Gotify Helm Chart

Baseline for [Gotify](https://gotify.net) (self-hosted push notifications). Uses bjw-s app-template. Override TZ, ingress host, and TLS in your values. Use OIDC at ingress (e.g. nginx + oauth2-proxy) to protect the app.

## Install

```bash
helm dependency update
helm install gotify . -f my-values.yaml -n gotify --create-namespace
```
