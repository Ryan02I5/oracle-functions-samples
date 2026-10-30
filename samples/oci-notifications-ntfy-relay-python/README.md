# Relay OCI Notifications to ntfy with Oracle Functions

This sample shows how to relay OCI Notifications messages to an ntfy topic by
using an OCI Function.

The intended path is:

```text
OCI Alarm -> OCI Notifications Topic -> OCI Function -> ntfy topic
```

As you make your way through this sample, the important point is that OCI
Monitoring Alarm and OCI Notifications continue to own alert state transitions
and deduplication. The function only adapts the OCI Notifications payload into
an ntfy publish request.

## Prerequisites

Before you deploy this sample, make sure you have:

- completed step A, B and C of the [Oracle Functions Quick Start Guide for Cloud Shell](https://www.oracle.com/webfolder/technetwork/tutorials/infographics/oci_functions_cloudshell_quickview/functions_quickview_top/functions_quickview/index.html)
- an OCI Notifications topic
- an OCI Vault
- an ntfy server reachable from OCI Functions

This sample assumes the following Vault secret names by default:

- `ntfy-alert-topic`
- `ntfy-function-relay-token`

## Review and customize your function

Review the following files in the current folder:

- [func.py](./func.py)
- [requirements.txt](./requirements.txt)
- [func.yaml](./func.yaml)
- [Dockerfile](./Dockerfile)

## Configure the function

Set these function configuration keys:

- `NTFY_HOST`
- `OCI_VAULT_ID`
- `NTFY_TOPIC_SECRET_NAME`
- `NTFY_TOKEN_SECRET_NAME`

Example:

```bash
fn config function <app-name> oci-notifications-ntfy-relay-python NTFY_HOST ntfy.example.com
fn config function <app-name> oci-notifications-ntfy-relay-python OCI_VAULT_ID <vault-ocid>
fn config function <app-name> oci-notifications-ntfy-relay-python NTFY_TOPIC_SECRET_NAME ntfy-alert-topic
fn config function <app-name> oci-notifications-ntfy-relay-python NTFY_TOKEN_SECRET_NAME ntfy-function-relay-token
```

## Create or update your Dynamic Group

In order to read OCI Vault secrets, your function must be part of a dynamic
group. For information on how to create a dynamic group, refer to the
[documentation](https://docs.oracle.com/en-us/iaas/Content/Identity/Tasks/managingdynamicgroups.htm#To).

When specifying the matching rule, a function-specific rule can look like:

```text
ALL {resource.type = 'fnfunc', resource.compartment.id = '<compartment-ocid>'}
```

## Add IAM policy

The function resource principal needs permission to read the two Vault secrets:

```text
Allow service FaaS to use virtual-network-family in compartment id <compartment-ocid>
Allow service FaaS to read repos in tenancy
Allow dynamic-group <functions-dynamic-group> to read secret-bundles in compartment id <compartment-ocid> where all {target.secret.name='ntfy-alert-topic'}
Allow dynamic-group <functions-dynamic-group> to read secret-bundles in compartment id <compartment-ocid> where all {target.secret.name='ntfy-function-relay-token'}
```

## Deploy the function

In Cloud Shell, run:

```bash
fn -v deploy --app <app-name>
```

## Create the Notifications subscription

Create an OCI Notifications subscription on your topic with:

- protocol: `ORACLE_FUNCTIONS`
- endpoint: this function

After that, OCI Alarm notifications published into the topic will invoke the
function, which will then publish to the ntfy topic.

## ntfy expectations

The function publishes to:

```text
https://<NTFY_HOST>/<topic>
```

It uses:

- `Authorization: Bearer <token>`
- `Title: ...`
- `Tags: ...`

The token therefore needs write access to the target topic.

## Sample payload behavior

This function is written to work with standard OCI Notifications message
bodies, including alarm notifications routed through OCI Notifications.

It attempts to surface:

- title / alarm display name
- severity
- lifecycle state (for alarms, the top-level `type`: `FIRING_TO_OK` is shown as
  a recovery and `RESET` stays a warning)
- region
- timestamp
- summary/body (for alarms, the plain-text `body` or `alarmSummary`)
- the alarm URL from `alarmMetaData`

Alarm fields follow the
[Alarm Message Format](https://docs.oracle.com/en-us/iaas/Content/Monitoring/alarm-message-format.htm)
reference. The `ONS_OPTIMIZED` message format applies only to Email
subscriptions, so a Functions subscription receives the raw alarm JSON.

If the payload contains non-Latin-1 title text, the message body preserves the
full Unicode value while the HTTP `Title` header is downgraded to a header-safe
variant.

## Scope

This sample is not:

- a custom alert state machine
- a replacement for OCI Alarm deduplication
- a full ntfy deployment stack
- a Terraform, Argo CD, or Kubernetes reference implementation

It is only the minimal relay needed to bridge OCI Notifications into ntfy.
