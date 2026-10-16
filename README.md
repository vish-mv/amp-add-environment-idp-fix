# add-environment.sh — with the IDP credential fix

A patched copy of WSO2 Agent Manager's `deployments/scripts/add-environment.sh`
(from tag `amp/v1.0.0`) that lets the gateway bootstrap job authenticate with
**real** IDP credentials instead of the chart's placeholder.

## The problem

Creating an environment from the console fails at the gateway step:

```
Error: failed pre-install: 1 error occurred:
	* job api-platform-<org>-<env>-bootstrap failed: BackoffLimitExceeded
```

Helm doesn't show why. The bootstrap pod does:

```
[INFO]  Agent Manager API is healthy.
[INFO]  Obtaining JWT from Thunder IDP...
[ERROR] POST .../oauth2/token failed with HTTP 401:
        {"error":"invalid_client","error_description":"Invalid client credentials"}
```

**Cause.** In `wso2-amp-api-platform-gateway-extension`:

- `values.yaml` sets `agentManager.idp.clientSecret: "amp-api-client-secret"` — a
  dev placeholder — and leaves `agentManager.idp.existingSecret` empty.
- `_helpers.tpl` (`wso2-amp-gateway-extension.idpClientSecretEnv`) therefore takes
  its `else` branch and ships that literal string to the job as a plain value.
- `add-environment.sh` never passes the override, and its `HELM_ARGS` array has no
  passthrough — so it cannot be supplied on the command line either.

On a local/dev install the placeholder *is* the real secret, so it works. On any
install where the platform secrets were generated — i.e. every real deployment —
Thunder rejects it with 401 and the pre-install hook exhausts its retries.

This is **not** specific to production environments. The bootstrap job never
references `isProduction`; a non-production environment fails identically.

## The fix

One addition after `HELM_ARGS`, honouring a new `GATEWAY_IDP_EXISTING_SECRET`
variable. When set, the chart switches from embedding the placeholder to a
`secretKeyRef` against your real credentials. When unset, behaviour is identical
to upstream.

## Usage

**1. Create (or locate) a Secret with the real `amp-api-client` credentials.**
It **must live in the gateway's namespace** — a `secretKeyRef` cannot cross
namespaces. That is `GATEWAY_NAMESPACE` if you set it, otherwise the script's
default of `<org>-<env>`.

```bash
kubectl get secret -A | grep -iE "idp|amp-api-client"
```

```bash
kubectl create secret generic gateway-idp-credentials -n <gateway-namespace> \
  --from-literal=client-id=amp-api-client \
  --from-literal=client-secret='<real amp-api-client secret>'
```

**2. Run this script instead of the console's URL**, with the same variables the
console gave you, plus `GATEWAY_IDP_EXISTING_SECRET`:

```bash
curl -fsSL https://raw.githubusercontent.com/<you>/<repo>/main/add-environment.sh \
  | ENV_NAME=prod-test \
    DISPLAY_NAME="prod-test" \
    AGENT_MANAGER_TOKEN=<token from the console> \
    AGENT_MANAGER_URL=https://<your-amp-api-host> \
    CHART_VERSION=1.0.0 \
    IS_PRODUCTION=true \
    GATEWAY_IDP_EXISTING_SECRET=gateway-idp-credentials \
    GATEWAY_NAMESPACE=<namespace holding that secret> \
    bash
```

Key names default to `client-id` / `client-secret`; override with
`GATEWAY_IDP_CLIENT_ID_KEY` / `GATEWAY_IDP_CLIENT_SECRET_KEY` if yours differ.

## Two things to check before you run it

**`gateway.vhost` is frozen at first successful registration.** The bootstrap job
records it once; on every later run it sees the gateway already exists, logs
`already exists`, and exits without reconciling. A wrong value cannot be corrected
by re-running. Check the host and TLS values the console gave you against your
real domain first — on a TLS deployment we saw the console emit
`TLS_ENABLED=false` and a `*.localhost` Thunder domain.

**A failed attempt leaves things behind.** The Environment record and the per-env
Thunder instance are created before the gateway step, so re-run with the **same**
`ENV_NAME` rather than a new one — the script is idempotent.

## Verifying

If it fails again, Helm's message is only the symptom. The real error is here:

```bash
kubectl get pods -n <gateway-namespace> | grep bootstrap
kubectl logs -n <gateway-namespace> <bootstrap-pod>
```

## Provenance

Upstream source: `wso2/agent-manager`, tag `amp/v1.0.0`,
`deployments/scripts/add-environment.sh` (739 lines, sha256 `77dbd4e3a5a24a0a11a9…`).
The only change is the `GATEWAY_IDP_EXISTING_SECRET` block, clearly marked
`PATCH (not upstream)`. Diff against upstream to confirm.
