# Workshop Conventions

## Repository Purpose

Antora courseware "Image Security with RHACS: From Inventory to Enforcement" on one OpenShift cluster: Step 1 Know Your Images (versions, builds, and fixable CVEs), Step 2 Alert, Then Block (image compliance policies), Step 3 Policies as Code (Argo CD), Step 5 Keep Enforcement On (RHACM guards the ACS configuration). Step 4 (signatures) is out of scope.

## Structure

- `documentation/` contains all AsciiDoc courseware content (Antora component)
- `documentation/modules/ROOT/pages/` contains the course pages in `.adoc` format
- YAML manifests live in the root step directories
- `00-setup/` operators, Central, SecuredCluster, MultiClusterHub
- `01-inventory-cve/` demo namespace, application images with `x.y.z` tags (image streams and an in-cluster build), workloads, inform-only CVE policy
- `02-image-compliance/` alert sink, notifier, `policies/` (inform), `enforce/` (enforce), `tests/` (test workloads)
- `03-policies-as-code/` Argo CD RBAC, Application, `policies/` synced from Git
- `05-acs-acm/` RHACM policy namespace, placement, ACS baseline policy
- Steps are progressive: each builds on the previous one
- The `SecurityPolicy` files in `02-image-compliance/policies/`, `02-image-compliance/enforce/`, and `03-policies-as-code/policies/` differ only by `enforcementActions` and the header comment; keep the three variants in sync
- `VALIDATION.md` lists open validation items; remove it once every TODO marker is resolved

## Validation markers

- `TODO-CAPTURE` in a console-output block: replace with real output from a live run
- `// TODO-TIMING`: measure wall-clock time; add an `icon:clock[]` line only above 10 seconds
- `// TODO-SCREENSHOT`: capture with a browser tool against the live console, save as `documentation/modules/ROOT/images/<NN>-<name>.png`, add caption + `image::` macro
- `// TODO-VALIDATE`: confirm behavior on the live cluster and update the text
- Never fill a marker with invented output, timings, or images

## AsciiDoc Formatting

- Every numbered step must have a *What* and *Why* pair in bold
- Every step must have a collapsible verification block
- Use `[.console-input]` before source blocks for commands
- Use `[.console-output]` before source blocks showing expected output
- Reference YAML manifests by file path in `oc apply -f` commands
- Do not use em dashes. Use regular dashes (`-`) instead
- Use `'''` for horizontal rules between steps
- Cross-reference other pages with `xref:page.adoc[Label]`
- External links use `link:URL[Label]`
- Use `NOTE:`, `TIP:`, `IMPORTANT:`, `WARNING:` admonitions
- All doc links must point to `docs.redhat.com` or `docs.openshift.com`
- Use attributes from `_attributes.adoc` for version numbers
- Write all prose with the humanizer rules: no not-X-but-Y contrasts, no one-line closers, no forced triads, no decorative bold

## CLI Conventions

- Every `oc` command must include an explicit `--context` flag (`hub`)
- Never use `oc login` inside step pages
- YAML files with `${VARIABLE}` placeholders use `envsubst`
- All workshop variables are exported once in `00-setup.adoc` / `myenv.sh`
- `roxctl` and `curl` against Central use `--insecure-skip-tls-verify` / `-k` (self-signed lab certificate)

## Content Rules

- Never include customer names, user names, or email addresses
- All content must be in English
- Prefer `registry.access.redhat.com` or `registry.redhat.io` images
- Workshop application images use `x.y.z` version tags in the internal registry (tests that need `latest` on purpose are the exception)
- Use Red Hat / OpenShift-native components when they cover the use case
- Mention Red Hat products because they fit the solution, not to promote them

## YAML Conventions

- Every YAML file starts with a comment block: filename and purpose
- Use `app.kubernetes.io/part-of: acs-image-workshop` label consistently
- Include resource requests and limits on Deployments
- Include readiness and liveness probes where applicable
