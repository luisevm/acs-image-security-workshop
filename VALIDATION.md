# Validation handoff

Setup and Steps 1, 2, 3, and 5 were run twice against a live cluster on 2026-10-08, with the Resets (5, 3, 2, 1) and the leftover check in between. On 2026-10-09 Step 1 was reworked around `x.y.z` application versions, and all Resets and Steps 1, 2, 3, and 5 were run a third time. A fourth run on the same day captured the ACS console screenshots at the matching point of each step. Console outputs, timings, and screenshots on the pages come from those runs. One marker is left (see below). Delete this file once it is resolved.

## Validation cluster

OpenShift 4.20.40: one schedulable control plane node (32 vCPU, 128 GiB) and three workers (16 vCPU, 32 GiB). RHACS 4.11.4, RHACM 2.17.3, OpenShift GitOps 1.22.1. The cluster came from the Demo Platform with RHACS 4.10, RHACM 2.16, and GitOps 1.20 already installed. Running Option A on top of it upgraded all three operators (see the WARNING in sub-step 4 of the setup page).

## Markers

| Marker | Where | What to do |
|---|---|---|
| `TODO-CAPTURE` | inside `[.console-output]` blocks | Replace with the real output of the live run (trim long output, keep it real) |
| `// TODO-TIMING` | above each sub-step's What/Why | Measure wall-clock time. If above 10 s, add `icon:clock[] ~N minutes (breakdown)` above What/Why. Delete the comment either way |
| `// TODO-SCREENSHOT` | where a screenshot belongs | Capture with a browser tool against the live console, save to `documentation/modules/ROOT/images/<file>`, add a `.Caption` line and `image::<file>[Alt text]` |
| `// TODO-VALIDATE` | next to an unconfirmed behavior | Confirm on the cluster, correct the text or manifest, delete the comment |

Count what is left:

```bash
grep -rn -E 'TODO-(CAPTURE|TIMING|SCREENSHOT|VALIDATE)' documentation 00-setup 01-inventory-cve 02-image-compliance 03-policies-as-code 05-acs-acm
```

Left: `00-setup.adoc`, Reset section, "operator CRDs stay on the cluster after this reset". The Step 0 Reset was not run, on purpose.

Never fill a marker with invented output, timing, or image.

## Leftover check after the resets

```bash
oc --context hub get ns acs-image-demo acs-alert-sink acs-policies --ignore-not-found
oc --context hub get securitypolicies -n stackrox -l workshop=acs-image-workshop
oc --context hub get applications.argoproj.io -n openshift-gitops acs-image-policies --ignore-not-found
oc --context hub get role,rolebinding -n stackrox argocd-securitypolicy-manager --ignore-not-found
oc --context hub get policy -A | grep acs-image-enforcement-baseline
curl -sk -H "Authorization: Bearer ${ROX_API_TOKEN}" "https://${ACS_CENTRAL_ROUTE}/v1/policies" | jq -r '.policies[].name' | grep '^Workshop - '
curl -sk -H "Authorization: Bearer ${ROX_API_TOKEN}" "https://${ACS_CENTRAL_ROUTE}/v1/notifiers" | jq -r '.notifiers[].name' | grep workshop-alert-sink
curl -skG -H "Authorization: Bearer ${ROX_API_TOKEN}" "https://${ACS_CENTRAL_ROUTE}/v1/alerts" --data-urlencode "query=Violation State:ATTEMPTED" | jq -r '.alerts[].policy.name' | grep '^Workshop - '
git log --oneline -5   # fork back at 365 days, registry policy file present
```

After the first run's resets, every line came back empty except the `ATTEMPTED` alerts, which the Step 1 Reset now resolves.

## Timing log

Command wall-clock time only. Reading, portal exploration, and typing come on top.

| Step | Sub-step | Measured | Notes |
|---|---|---|---|
| 0 | 4 operators | RHACS 41s, GitOps 51s, RHACM 85s | Fresh install, from the cluster's CSV events |
| 0 | 5 MultiClusterHub | `local-cluster` joined after 5m20s | Fresh install. The 2.16 to 2.17 upgrade took about 9 minutes |
| 0 | 6 Central | `Deployed` 18s, API after ~80s | Fresh install. A spec change restart took 204s |
| 0 | 8 SecuredCluster | ~2 min to the last admission controller pod | Re-apply on this cluster: 96s |
| 0 | 9 scanner | Vulnerability data loaded ~11 min after Central; first scan 40s | |
| 1 | total | ~2 min | Image import and build of `current-app:2.0.0` 33-35s, deploy 5s, scans through the cluster ~13s each, image check ~9s |
| 2 | total | ~2-3 min | Receiver 27s cold / 5s warm, deployment check 12-27s, enforcement active ~5s |
| 3 | total | ~3 min | First sync 26-46s, Git change ~15s, self-heal loop 60s, delete and restore ~20s |
| 5 | total | ~4 min | Admission controller rollout after the drift ~90s, SecuredCluster recreate to `HEALTHY` 62-64s |

Estimated participant time with reading and the portal: Step 1 30-35 min (10 sub-steps since the image-version rework), Step 2 30-35 min, Step 3 25-30 min, Step 5 15-20 min, so 100-120 minutes. That is at the limit of a 2-hour session; Step 1 sub-step 5 (cluster-wide search) is the easiest to drop if time is short. This fits a 2-hour session when Step 0 runs before the session (the scanner alone needs about 11 minutes after Central starts).

## Answers to the open questions

1. The `SecurityPolicy` CRD accepts `FAIL_DEPLOYMENT_CREATE_ENFORCEMENT` and `FAIL_DEPLOYMENT_UPDATE_ENFORCEMENT`, and the admission controller rejects creates, updates, and scale requests with them.
2. Turning on enforcement leaves running deployments alone, `SCALE_TO_ZERO_ENFORCEMENT` included. Not scaled after 90 s, not on the break-glass deployment either.
3. The generic notifier JSON works and Central reaches the receiver (test message and real alerts arrive). ACS leaves `lifecycleStage` out for `DEPLOY`; the receiver now prints `DEPLOY` for it.
4. `Image Age` `"365"` means days.
5. No fight between RHACM and the RHACS operator on the `SecuredCluster`: compliant at once, generation unchanged.
6. The init bundle secrets have no owner and survive the deletion. Sensor reconnects under the same name and cluster ID. RHACM recreates the resource with only the fields in the policy.
7. The API certificate is signed by the cluster's internal CA, so `oc login` fails on a fresh workstation without `--insecure-skip-tls-verify=true` (now in `myenv.sh`). The Central route had a publicly trusted certificate on this cluster.
8. Capacity: fine on the validation cluster (CPU requests 32-61% per node).
9. UI labels checked and updated: ACS 4.11 *Vulnerability Management > Results* (*User workload vulnerabilities*, *CVE fixed in*), *Violations* filters, *Platform Configuration > Clusters*, *Policy Management* (*Origin: Externally managed*), console application launcher *Cluster Argo CD*, RHACM *Fleet management > Governance > Policies*.
10. All `docs.redhat.com` links return 200. They point at RHACS 4.11 now; the old 4.9 secured cluster options page was a 404.
11. `_attributes.adoc` holds OpenShift 4.20, RHACS 4.11, RHACM 2.17, GitOps 1.22.

## Other findings fixed in the pages

- With RHACM installed, the short names `subscription` and `application` resolve to RHACM's and the Kubernetes `Application` CRDs. The pages use `subscriptions.operators.coreos.com` and `applications.argoproj.io`.
- `roxctl` fails when `ROX_ADMIN_PASSWORD` and `ROX_API_TOKEN` are both set. The password variable is now `ACS_ADMIN_PASSWORD`.
- `roxctl` downloads are `roxctl-<os>-<arch>`; `roxctl-darwin` does not exist.
- The built-in policy *Fixable Severity at least Important* fails the build by default, so `roxctl image check` of the legacy image exits 1 already in Step 1.
- `config-controller` only rewrites a policy in Central when the custom resource's spec changes. A portal edit stays until then.
- In three of four runs, an Argo CD prune left a `SecurityPolicy` stuck on its finalizer after the policy was gone from Central (`policy "" is not externally managed`). The Step 3 page has the check and the workaround.
- After a manual change to `admissionControl.enforcement`, RHACM restores it within a second, but the admission controller rollout admits requests for about a minute.
- CVE data changes during the day: between the two runs a fix for an `openssl-libs` CVE was published that the current UBI 9 minimal image does not include yet. Step 2 notes the effect on the "compliant" restart.

- The demo applications run from the internal registry with `x.y.z` tags (`legacy-app:1.0.0`, `current-app:2.0.0`, `alert-sink:1.0.0`). `current-app:2.0.0` is built in the cluster with `microdnf -y update`, because the Red Hat UBI 9 minimal image can lag behind published fixes (an `openssl-libs` fix on the validation date).
- Central cannot pull from the internal registry itself, so `roxctl image scan` and `roxctl image check` of the application images need `--cluster`.
- A pod created in the same second as its namespace can miss the internal-registry pull secret and stay in `ImagePullBackOff`. Step 2 waits for the secret before it deploys the receiver.
- With `imagePullPolicy: IfNotPresent`, a node keeps running the old build after a version tag moves. The application deployments use `Always`.
- After the overnight power-off, the cluster's controllers lagged: an `ImageStream` did not import and a pod missed its pull secret. Both worked within 2 seconds once the cluster had settled.

## Design decisions to keep or change

- All operators and the RHACM hub install in Step 0, so the resets never touch them and a rerun starts fast.
- Step 5 targets `local-cluster` only. Importing a second cluster is described, not tested.
- Step 4 (signatures) is skipped; page numbering keeps `05` so a Step 4 page can be added later.
- Alerts go to a generic webhook receiver in the cluster. An email or Slack notifier would need an SMTP server or an incoming-webhook URL.
