# Validation handoff

This repository is a scaffold. The commands, manifests, page structure, and Reset sections are written, but nothing has run against a cluster yet. Every console output, timing, and screenshot is a marked placeholder. Delete this file once every marker below is resolved.

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

Never fill a marker with invented output, timing, or image.

## Before the first run

1. Replace `GITHUB_USER` in `site.yml`, `package.json`, `supplemental-ui/partials/footer-nav.hbs`, `README.adoc`, `00-setup.adoc`, and `myenv.sh` with the real GitHub account, push the repository, and set `GIT_REPO_URL` in `myenv.sh` to that repository (Step 3 pushes to it, and Argo CD reads it, so it must be public).
2. Tools on the machine that runs the validation: `oc`, `jq`, `curl`, `git`, `envsubst`, network access to the cluster API, the `*.apps` routes, and `registry.access.redhat.com`.
3. Skills: `antora-workshop` and `humanizer`. A browser automation tool is needed for the screenshots.
4. `myenv.sh` (gitignored) already holds the cluster URL and credentials.

## Run order

1. Setup (Option A), then Steps 1, 2, 3, 5, including every Verify block. Fix and iterate when something breaks.
2. Resets in reverse order: Step 5, 3, 2, 1. Step 0 stays. Then check that nothing from Steps 1-5 is left (see the leftover check below).
3. Run Steps 1, 2, 3, 5 again from a clean Step 0 state and confirm the outputs match the first run.

Leftover check after the resets:

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

## Timing log (fill during the run)

| Step | Sub-step | Measured | Notes (cluster type, cold vs warm) |
|---|---|---|---|
| 0 | 4 operators | | |
| 0 | 5 MultiClusterHub Running | | |
| 0 | 6 Central | | |
| 0 | 8 SecuredCluster healthy | | |
| 0 | 9 scanner returns CVEs | | |
| 1 | total | | |
| 2 | total | | |
| 3 | total | | |
| 5 | total | | |

Target: Steps 1, 2, 3, 5 (participant time, including reading) fit in a 2-hour session. Step 0 is expected to run before the session.

## Open questions to settle on the cluster

Highest risk first:

1. `02-image-compliance/enforce/*.yaml` and `03-policies-as-code/policies/*.yaml`: does the `SecurityPolicy` CRD accept `FAIL_DEPLOYMENT_CREATE_ENFORCEMENT` and `FAIL_DEPLOYMENT_UPDATE_ENFORCEMENT`, and does the admission controller reject with them? (`oc explain securitypolicy.spec.enforcementActions`)
2. Step 2.5: what happens to already running violating deployments when enforcement turns on (kept and reported, or scaled to zero)?
3. Step 2.1: generic notifier JSON schema (`02-image-compliance/02-notifier.json`), and whether Central can reach the `alert-sink` Service.
4. Step 2.2: `Image Age` value format (`"365"` = days).
5. Step 5.3: RHACM `musthave` on a `SecuredCluster` that the ACS operator also manages: no fight between the two controllers?
6. Step 5.5: deleting the `SecuredCluster`, then RHACM recreating it: do the init bundle secrets survive, and does Sensor reconnect under the same name?
7. `oc login` against the cluster: certificate trusted, or does it prompt?
8. ACS + RHACM + GitOps capacity on one cluster.
9. All UI navigation labels (Steps 0, 1, 3, 5) against the installed versions.
10. All `docs.redhat.com` links resolve (pages: setup, 01, 02, 03, 05).
11. `_attributes.adoc`: record the installed OpenShift, RHACS, RHACM, and GitOps versions.

## Design decisions to keep or change

- All operators and the RHACM hub install in Step 0, so the resets never touch them and a rerun starts fast.
- Step 5 targets `local-cluster` only. Importing a second cluster is described, not tested.
- Step 4 (signatures) is skipped; page numbering keeps `05` so a Step 4 page can be added later.
