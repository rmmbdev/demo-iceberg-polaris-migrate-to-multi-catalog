# demo-iceberg-polaris-migrate-to-multi-catalog
##### Taking a live, single-catalog Polaris with STS unavailable to a two-catalog, per-catalog-STS setup — without moving any data

The companion project [demo-iceberg-polaris-multi-catalog-minio](../demo-iceberg-polaris-multi-catalog-minio) builds
that two-catalog setup in one pass, on an empty cluster. This one starts from an
environment that is already running, already has tables in it, and cannot be
torn down, and ends at exactly the configuration
`catalogs/lakehouse.json`, `catalogs/lakehouse_fast.json`, `values/polaris.yaml`
and `values/trino.yaml` describe. No table is rewritten and no object is moved.

| | before | after |
|---|---|---|
| catalogs | `lakehouse` | `lakehouse`, `lakehouse_fast` |
| object storage | `minio-a` | `minio-a`, `minio-b` |
| catalog storage config | `"stsUnavailable": true`, no `storageName` | `storageName` + `stsEndpoint`, STS in use |
| what Polaris sends MinIO | long-lived root keys | `AssumeRole` credentials, ~1 hour, confined to one bucket |
| Trino S3 credentials | `minio-a`'s **root** user | `cat-a-user` / `cat-b-user`, each scoped to its bucket |

**Run every command from the directory holding this file, in one terminal.**
Every path below — `secrets/`, `values/`, `catalogs/`, `manifests/`,
`overlays/` — is relative to it. Start with:

```bash
  cd /home/reza/projects/git/demo-iceberg-polaris-migrate-to-multi-catalog
```

Run the steps in order, and the blocks inside each step top to bottom. Each step
is self-contained: it takes nothing from your shell that it did not set itself.
Every step that talks to Polaris opens with the same block, which restarts the
port-forward in the background and issues a fresh `$TOKEN`. Every step that
needs Trino's Polaris credentials reads them back out of the `trino-secrets`
Secret, which is the one place they are kept. A new terminal, an expired token
or a replaced Polaris pod therefore never breaks a step.

Blocks that check something print a line starting `OK:` or `STOP:`. On `STOP:`,
do not go on to the next block. Blocks that change something refuse to run on a
bad value, so a `STOP:` never leaves anything half-written.

The from-scratch build of the companion project is reproduced at the end of
this file for comparison. Do not run both on one cluster.

## 1. Helm repositories

```bash
  helm repo add polaris https://downloads.apache.org/polaris/helm-chart
```

```bash
  helm repo add trino https://trinodb.github.io/charts
```

```bash
  helm repo add minio https://charts.min.io/
```

```bash
  helm repo update
```

## 2. Cluster and images

```bash
  docker pull postgres:17.2-bookworm && \
  docker pull docker.arvancloud.ir/minio/minio:RELEASE.2025-01-20T14-49-07Z && \
  docker tag  docker.arvancloud.ir/minio/minio:RELEASE.2025-01-20T14-49-07Z quay.io/minio/minio:RELEASE.2025-01-20T14-49-07Z && \
  docker pull docker.arvancloud.ir/minio/mc:RELEASE.2025-01-17T23-25-50Z && \
  docker tag  docker.arvancloud.ir/minio/mc:RELEASE.2025-01-17T23-25-50Z quay.io/minio/mc:RELEASE.2025-01-17T23-25-50Z && \
  docker pull apache/polaris:1.5.0 && \
  docker pull apache/polaris-admin-tool:1.5.0 && \
  docker pull trinodb/trino:476
```

The two MinIO images come from the `docker.arvancloud.ir` mirror of Docker Hub,
because an anonymous pull from `quay.io` stops at a `Login prior to pull:`
prompt and, inside this `&&` chain, waits there indefinitely instead of failing.
Each is re-tagged with its `quay.io` name: the MinIO chart runs
`quay.io/minio/minio` and `quay.io/minio/mc`, and with `pullPolicy:
IfNotPresent` it only finds an image loaded into minikube under that exact name.
If `docker.arvancloud.ir` is unavailable, `hub.hamdocker.ir` serves the same two
tags with the same digests.

```bash
  minikube start --memory=8192 --cpus=4 && \
  minikube image load postgres:17.2-bookworm && \
  minikube image load quay.io/minio/minio:RELEASE.2025-01-20T14-49-07Z && \
  minikube image load quay.io/minio/mc:RELEASE.2025-01-17T23-25-50Z && \
  minikube image load apache/polaris:1.5.0 && \
  minikube image load apache/polaris-admin-tool:1.5.0 && \
  minikube image load trinodb/trino:476
```

## 3. Secrets

Three, not five. In the before state there is no per-catalog MinIO user and no
second instance.

```bash
  kubectl create secret generic postgres-secrets   --from-env-file=secrets/postgres-secrets.env && \
  kubectl create secret generic polaris-db-secrets --from-env-file=secrets/polaris-db-secrets.env && \
  kubectl create secret generic minio-a-secrets    --from-env-file=secrets/minio-a-secrets.env
```

## 4. Postgres and one MinIO

```bash
  kubectl apply -f manifests/00-postgres.yaml
```

```bash
  helm upgrade --install minio-a minio/minio --version 5.4.0 \
    --values values/minio-a.yaml \
    --values overlays/minio-no-scoped-user.yaml \
    --timeout 10m
```

```bash
  kubectl wait --for=condition=available deploy/minio-a deploy/postgres-polaris --timeout=300s
```

## 5. Polaris, with STS unavailable

```bash
  kubectl run polaris-bootstrap --rm -it --restart=Never \
  --image=apache/polaris-admin-tool:1.5.0 \
  --image-pull-policy=IfNotPresent \
  --env="polaris.persistence.type=relational-jdbc" \
  --env="quarkus.datasource.username=postgres" \
  --env="quarkus.datasource.password=postgres" \
  --env="quarkus.datasource.jdbc.url=jdbc:postgresql://postgres-polaris:5432/polaris" \
  -- bootstrap -r POLARIS -c POLARIS,root,root
```

```bash
  helm upgrade --install polaris polaris/polaris --version 1.5.0 \
    --values values/polaris.yaml \
    --values overlays/polaris-before.yaml \
    --timeout 10m
```

```bash
  kubectl rollout status deploy/polaris --timeout=600s
```

Connect to Polaris. This stops anything listening on local port 8181, starts
the port-forward in the background — there is no second terminal — and checks
that `root` can log in:

```bash
  lsof -t -i tcp:8181 -s tcp:LISTEN | xargs -r kill ; sleep 1 ; \
  (kubectl port-forward svc/polaris 8181:8181 > /dev/null 2>&1 &) ; \
  for i in $(seq 60); do curl -s -o /dev/null http://localhost:8181/api/catalog/v1/config && break; sleep 1; done ; \
  TOKEN=$(curl -s -X POST http://localhost:8181/api/catalog/v1/oauth/tokens \
    -d "grant_type=client_credentials&client_id=root&client_secret=root&scope=PRINCIPAL_ROLE:ALL" \
    | jq -r '.access_token // empty') ; export TOKEN ; \
  if [ -n "$TOKEN" ]; then echo 'OK: Polaris is reachable and TOKEN is set'; else echo 'STOP: no TOKEN -- Polaris is not answering on localhost:8181'; fi
```

## 6. The catalog, the principal, and Trino

Connect to Polaris:

```bash
  lsof -t -i tcp:8181 -s tcp:LISTEN | xargs -r kill ; sleep 1 ; \
  (kubectl port-forward svc/polaris 8181:8181 > /dev/null 2>&1 &) ; \
  for i in $(seq 60); do curl -s -o /dev/null http://localhost:8181/api/catalog/v1/config && break; sleep 1; done ; \
  TOKEN=$(curl -s -X POST http://localhost:8181/api/catalog/v1/oauth/tokens \
    -d "grant_type=client_credentials&client_id=root&client_secret=root&scope=PRINCIPAL_ROLE:ALL" \
    | jq -r '.access_token // empty') ; export TOKEN ; \
  if [ -n "$TOKEN" ]; then echo 'OK: Polaris is reachable and TOKEN is set'; else echo 'STOP: no TOKEN -- Polaris is not answering on localhost:8181'; fi
```

```bash
  curl -X POST http://localhost:8181/api/management/v1/catalogs \
    -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
    -d @catalogs/lakehouse-before.json
```

```bash
  curl -X PUT http://localhost:8181/api/management/v1/catalogs/lakehouse/catalog-roles/catalog_admin/grants \
    -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
    -d '{"type": "catalog", "privilege": "CATALOG_MANAGE_CONTENT"}'
```

Create Trino's principal and store its credentials in `trino-secrets` in the
same block. Polaris returns a principal's secret exactly once, in the response
to this call, and keeps only a hash — the Secret is the only copy there will
ever be. The block writes the Secret only if Polaris actually returned a
`clientId` and `clientSecret`; any error (a `401`, or a `409` because
`trino-svc` already exists) writes nothing and prints `STOP:`.

`ACCESS_KEY_MINIO` is `minio-a`'s **root** user here; `cat-a-user` does not exist yet.

```bash
  POLARIS_AUTH=$(curl -s -X POST http://localhost:8181/api/management/v1/principals \
    -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
    -d '{"principal": {"name": "trino-svc", "type": "SERVICE"}}' \
    | jq -r 'if .credentials.clientId and .credentials.clientSecret then .credentials.clientId + ":" + .credentials.clientSecret else empty end') ; \
  export POLARIS_AUTH ; \
  if [ -n "$POLARIS_AUTH" ]; then \
    kubectl create secret generic trino-secrets \
      --from-literal=POLARIS_AUTH="$POLARIS_AUTH" \
      --from-literal=ACCESS_KEY_MINIO='lakehouse-key' \
      --from-literal=SECRET_KEY_MINIO='lakehouse-secret-0123456789' \
    && echo 'OK: trino-svc created and its pair stored in trino-secrets' ; \
  else echo 'STOP: Polaris returned no credentials for trino-svc -- nothing was written'; fi
```

```bash
  curl -X POST http://localhost:8181/api/management/v1/principal-roles \
    -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
    -d '{"principalRole": {"name": "data_engineer"}}'
```

```bash
  curl -X PUT http://localhost:8181/api/management/v1/principal-roles/data_engineer/catalog-roles/lakehouse \
    -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
    -d '{"catalogRole": {"name": "catalog_admin"}}'
```

```bash
  curl -X PUT http://localhost:8181/api/management/v1/principals/trino-svc/principal-roles \
    -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
    -d '{"principalRole": {"name": "data_engineer"}}'
```

The pair in the Secret must log in to Polaris:

```bash
  POLARIS_AUTH=$(kubectl get secret trino-secrets -o jsonpath='{.data.POLARIS_AUTH}' | base64 -d) ; export POLARIS_AUTH ; \
  TRINO_TOKEN=$(curl -s -X POST http://localhost:8181/api/catalog/v1/oauth/tokens \
    -d "grant_type=client_credentials&client_id=${POLARIS_AUTH%%:*}&client_secret=${POLARIS_AUTH##*:}&scope=PRINCIPAL_ROLE:ALL" \
    | jq -r '.access_token // empty') ; export TRINO_TOKEN ; \
  if [ -n "$TRINO_TOKEN" ]; then echo 'OK: the trino-svc pair in trino-secrets authenticates'; else echo 'STOP: the POLARIS_AUTH pair in trino-secrets does not authenticate'; fi
```

```bash
  helm upgrade --install trino trino/trino --version 1.42.2 \
    --values values/trino-before.yaml \
    --timeout 10m
```

```bash
  kubectl rollout status deploy/trino-coordinator --timeout=600s && \
  kubectl rollout status deploy/trino-worker --timeout=600s
```

## 7. Confirm the starting point

Connect to Polaris:

```bash
  lsof -t -i tcp:8181 -s tcp:LISTEN | xargs -r kill ; sleep 1 ; \
  (kubectl port-forward svc/polaris 8181:8181 > /dev/null 2>&1 &) ; \
  for i in $(seq 60); do curl -s -o /dev/null http://localhost:8181/api/catalog/v1/config && break; sleep 1; done ; \
  TOKEN=$(curl -s -X POST http://localhost:8181/api/catalog/v1/oauth/tokens \
    -d "grant_type=client_credentials&client_id=root&client_secret=root&scope=PRINCIPAL_ROLE:ALL" \
    | jq -r '.access_token // empty') ; export TOKEN ; \
  if [ -n "$TOKEN" ]; then echo 'OK: Polaris is reachable and TOKEN is set'; else echo 'STOP: no TOKEN -- Polaris is not answering on localhost:8181'; fi
```

No `storageName`, no `stsEndpoint`:

```bash
  curl -s http://localhost:8181/api/management/v1/catalogs/lakehouse \
    -H "Authorization: Bearer $TOKEN" | jq '.storageConfigInfo'
```

No identity on `minio-a` besides root, and no `bucket-a-rw`:

```bash
  kubectl exec deploy/minio-a -- sh -c "
      mc alias set a http://minio-a:9000 lakehouse-key lakehouse-secret-0123456789 > /dev/null
      echo '--- users ---'   ; mc admin user list a
      echo '--- policies ---'; mc admin policy ls a"
```

Now put some data in. Leave the CLI with `quit;`.

```bash
  kubectl exec -it deploy/trino-coordinator -- trino --server localhost:8080 --user admin
```

```sql
SHOW CATALOGS;

CREATE SCHEMA lakehouse.bronze;
CREATE TABLE lakehouse.bronze.customers (customer_id BIGINT, name VARCHAR, city VARCHAR);
INSERT INTO lakehouse.bronze.customers VALUES (1,'Ada','Tehran'),(2,'Grace','Shiraz'),(3,'Alan','Mashhad');
SELECT * FROM lakehouse.bronze.customers;
```

Ask Polaris for that table as an Iceberg client would. This is the sharpest
check after the migration, so it is worth seeing what it answers now. First
log in as Trino, with the pair from `trino-secrets`:

```bash
  POLARIS_AUTH=$(kubectl get secret trino-secrets -o jsonpath='{.data.POLARIS_AUTH}' | base64 -d) ; export POLARIS_AUTH ; \
  TRINO_TOKEN=$(curl -s -X POST http://localhost:8181/api/catalog/v1/oauth/tokens \
    -d "grant_type=client_credentials&client_id=${POLARIS_AUTH%%:*}&client_secret=${POLARIS_AUTH##*:}&scope=PRINCIPAL_ROLE:ALL" \
    | jq -r '.access_token // empty') ; export TRINO_TOKEN ; \
  if [ -n "$TRINO_TOKEN" ]; then echo 'OK: the trino-svc pair in trino-secrets authenticates'; else echo 'STOP: the POLARIS_AUTH pair in trino-secrets does not authenticate'; fi
```

```bash
  curl -s http://localhost:8181/api/catalog/v1/lakehouse/namespaces/bronze/tables/customers \
    -H "Authorization: Bearer $TRINO_TOKEN" \
    -H "X-Iceberg-Access-Delegation: vended-credentials" \
    | jq -rn '(try input catch null) as $r
       | if $r.metadata != null then $r.config["s3.session-token"] // "table returned, but no session token"
         elif ($r.error.message // "" | test("no credentials are available")) then "Polaris refuses to vend credentials -- it is not calling AssumeRole"
         else "request failed: \($r.error.message // "empty response -- check $TRINO_TOKEN and the port-forward")" end'
```

The answer you want here is `Polaris refuses to vend credentials`. With
`"stsUnavailable": true` Polaris does not return the table without credentials:
it rejects the request outright with `400 IllegalArgumentException: Credential
vending was requested for table bronze.customers, but no credentials are
available`. That rejection is the proof: it is Polaris itself saying it has
nothing to vend. The same request *without* the `X-Iceberg-Access-Delegation`
header answers `200`, which is why Trino, which does not send it, is unaffected.

Anything starting with `request failed:` is some other error — a `401`, a `404`,
a missing grant — and has told you nothing. Do not read it as a pass.

## 8. Add the second MinIO

```bash
  kubectl create secret generic minio-b-secrets --from-env-file=secrets/minio-b-secrets.env
```

```bash
  helm upgrade --install minio-b minio/minio --version 5.4.0 \
    --values values/minio-b.yaml \
    --values overlays/minio-no-scoped-user.yaml \
    --timeout 10m
```

```bash
  kubectl wait --for=condition=available deploy/minio-b --timeout=300s
```

Write, read, delete — from inside the cluster, where `minio-b` resolves:

```bash
  kubectl exec deploy/minio-b -- sh -c "
      set -e
      mc alias set b http://minio-b:9000 minio-b-admin minio-b-admin-secret-0123456789 > /dev/null
      echo '--- buckets ---' ; mc ls b
      echo '--- write ---'   ; echo 'minio-b reachable' | mc pipe b/neshan-lakehouse-fast/probe.txt
      echo '--- read ---'    ; mc cat b/neshan-lakehouse-fast/probe.txt
      echo '--- clean up ---'; mc rm b/neshan-lakehouse-fast/probe.txt"
```

The last line answers `Created delete marker`, not a plain removal. That does
not mean the probe survives. Versioning on both buckets is *suspended* (`mc
version info` says so), so the delete replaces the object's `null` version with
a `null`-version delete marker: the 18 bytes are gone, and what remains is a
0-byte marker that `mc ls --versions` shows as `null v1 DEL probe.txt`.

## 9. Scoped users on both MinIO instances

Dropping the overlay is what creates them. The chart does it from a
`post-upgrade` job, so buckets and data are untouched — but the user and the
policy also enter the chart's ConfigMap, which changes the Deployment's
`checksum/config` annotation and so replaces the MinIO pod. Each of the two
upgrades below restarts its instance; at `replicas: 1` that is an
object-storage outage of about two seconds, not merely a job run.

```bash
  kubectl create secret generic catalog-users-secrets --from-env-file=secrets/catalog-users-secrets.env
```

```bash
  helm upgrade --install minio-a minio/minio --version 5.4.0 --values values/minio-a.yaml --timeout 10m
```

```bash
  helm upgrade --install minio-b minio/minio --version 5.4.0 --values values/minio-b.yaml --timeout 10m
```

```bash
  kubectl exec deploy/minio-a -- sh -c "
      mc alias set a http://minio-a:9000 lakehouse-key   lakehouse-secret-0123456789   > /dev/null
      mc alias set b http://minio-b:9000 minio-b-admin   minio-b-admin-secret-0123456789 > /dev/null
      echo '--- minio-a ---'; mc admin user list a; mc admin policy info a bucket-a-rw
      echo '--- minio-b ---'; mc admin user list b; mc admin policy info b bucket-b-rw"
```

`AssumeRole` credentials inherit the user's policy, so proving it on the user
proves it for every session Polaris mints. The last write must fail.

Read the failure for what it is. `neshan-lakehouse-fast` does not exist on
`minio-a`, so the check proves the policy confines `cat-a-user` to its own
bucket on its own instance: the answer is `Insufficient permissions`, not a
missing-bucket error, because MinIO evaluates the policy first. It says nothing
about `minio-b`, where `cat-a-user` does not exist at all. Isolation between the
two instances comes from their being separate identity stores, not from any
policy. The probe's `mc rm` leaves a `null`-version delete marker in
`neshan-lakehouse`, as versioning is suspended; the object itself is gone.

```bash
  kubectl exec deploy/minio-a -- sh -c "
      mc alias set scoped http://minio-a:9000 cat-a-user cat-a-secret-0123456789 > /dev/null
      echo '--- buckets this user can see ---'
      mc ls scoped
      echo '--- write to its own bucket ---'
      echo ok | mc pipe scoped/neshan-lakehouse/probe.txt && mc rm scoped/neshan-lakehouse/probe.txt
      echo '--- write to the other catalog bucket (must fail) ---'
      echo ok | mc pipe scoped/neshan-lakehouse-fast/probe.txt || echo 'denied, as it should be'"
```

Move Trino's own S3 keys off root. `kubectl patch` changes only the two keys it
names; `POLARIS_AUTH` is not in the command, so it stays exactly as it is:

```bash
  kubectl patch secret trino-secrets --type merge \
    -p '{"stringData": {"ACCESS_KEY_MINIO": "cat-a-user", "SECRET_KEY_MINIO": "cat-a-secret-0123456789"}}'
```

```bash
  kubectl rollout restart deploy/trino-coordinator deploy/trino-worker && \
  kubectl rollout status deploy/trino-coordinator --timeout=600s && \
  kubectl rollout status deploy/trino-worker --timeout=600s
```

```bash
  kubectl exec -i deploy/trino-coordinator -- trino --server localhost:8080 --user admin \
    --execute "SELECT count(*) FROM lakehouse.bronze.customers"
```

## 10. Per-storage credentials on Polaris

`overlays/polaris-storagea-only.yaml` turns `RESOLVE_CREDENTIALS_BY_STORAGE_NAME`
on and adds the `storagea` pair. It keeps the `AWS_*` root keys, which the
catalog still needs until it is patched.

```bash
  helm upgrade --install polaris polaris/polaris --version 1.5.0 \
    --values values/polaris.yaml \
    --values overlays/polaris-storagea-only.yaml \
    --timeout 10m
```

```bash
  kubectl rollout status deploy/polaris --timeout=600s
```

The pod was replaced, which killed the port-forward. Connect to Polaris again:

```bash
  lsof -t -i tcp:8181 -s tcp:LISTEN | xargs -r kill ; sleep 1 ; \
  (kubectl port-forward svc/polaris 8181:8181 > /dev/null 2>&1 &) ; \
  for i in $(seq 60); do curl -s -o /dev/null http://localhost:8181/api/catalog/v1/config && break; sleep 1; done ; \
  TOKEN=$(curl -s -X POST http://localhost:8181/api/catalog/v1/oauth/tokens \
    -d "grant_type=client_credentials&client_id=root&client_secret=root&scope=PRINCIPAL_ROLE:ALL" \
    | jq -r '.access_token // empty') ; export TOKEN ; \
  if [ -n "$TOKEN" ]; then echo 'OK: Polaris is reachable and TOKEN is set'; else echo 'STOP: no TOKEN -- Polaris is not answering on localhost:8181'; fi
```

```bash
  kubectl get configmap polaris -o go-template='{{index .data "application.properties"}}' \
    | grep RESOLVE_CREDENTIALS_BY_STORAGE_NAME
```

Nothing observable has changed — the catalog still says `"stsUnavailable": true`:

```bash
  kubectl exec -i deploy/trino-coordinator -- trino --server localhost:8080 --user admin \
    --execute "SELECT count(*) FROM lakehouse.bronze.customers"
```

## 11. Patch the catalog

This is the migration. Connect to Polaris first:

```bash
  lsof -t -i tcp:8181 -s tcp:LISTEN | xargs -r kill ; sleep 1 ; \
  (kubectl port-forward svc/polaris 8181:8181 > /dev/null 2>&1 &) ; \
  for i in $(seq 60); do curl -s -o /dev/null http://localhost:8181/api/catalog/v1/config && break; sleep 1; done ; \
  TOKEN=$(curl -s -X POST http://localhost:8181/api/catalog/v1/oauth/tokens \
    -d "grant_type=client_credentials&client_id=root&client_secret=root&scope=PRINCIPAL_ROLE:ALL" \
    | jq -r '.access_token // empty') ; export TOKEN ; \
  if [ -n "$TOKEN" ]; then echo 'OK: Polaris is reachable and TOKEN is set'; else echo 'STOP: no TOKEN -- Polaris is not answering on localhost:8181'; fi
```

Back up the metastore. It costs one `pg_dump` and no downtime, and it is the
only thing that covers a mistake the patch cannot undo by itself. `pg_dump` is
transactionally consistent, so it runs with everything live. The dump contains
every principal's credentials; `backup/` is in `.gitignore` — keep it that way,
and treat the file as a secret. Each backup gets its own timestamped directory,
created with a plain `mkdir` that refuses an existing name, so a backup never
overwrites an earlier one. The block checks that the dump holds Polaris's tables
(a dump against the wrong database succeeds and is empty) and that both JSON
records came back from the API:

```bash
  BACKUP=backup/$(date +%Y%m%dT%H%M%S) ; export BACKUP ; \
  mkdir -p backup && mkdir "$BACKUP" && \
  kubectl exec deploy/postgres-polaris -- \
    pg_dump -U postgres -d polaris --format=plain --clean --if-exists \
    > "$BACKUP/polaris-metastore.sql" && \
  [ "$(grep -c 'COPY polaris_schema' "$BACKUP/polaris-metastore.sql")" -gt 0 ] && \
  curl -sf http://localhost:8181/api/management/v1/catalogs \
    -H "Authorization: Bearer $TOKEN" > "$BACKUP/catalogs.json" && \
  curl -sf http://localhost:8181/api/management/v1/principal-roles \
    -H "Authorization: Bearer $TOKEN" > "$BACKUP/principal-roles.json" && \
  echo "OK: backup complete in $BACKUP" || echo "STOP: the backup in $BACKUP is incomplete -- do not patch the catalog"
```

```bash
  jq -r '.catalogs[] | "\(.name) entityVersion=\(.entityVersion) storageName=\(.storageConfigInfo.storageName)"' "$BACKUP/catalogs.json"
```

Patch. `currentEntityVersion` is an optimistic lock, so the block reads it and
sends the `PUT` in one go. It sends nothing unless the backup above exists in
this shell and the version was read successfully:

```bash
  VERSION=$(curl -sf http://localhost:8181/api/management/v1/catalogs/lakehouse \
    -H "Authorization: Bearer $TOKEN" | jq -r '.entityVersion // empty') ; export VERSION ; \
  if [ -s "$BACKUP/polaris-metastore.sql" ] && [ -s "$BACKUP/catalogs.json" ] && [ -n "$VERSION" ]; then \
    curl -s -w '\nHTTP %{http_code}\n' -X PUT http://localhost:8181/api/management/v1/catalogs/lakehouse \
      -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
      -d "$(jq -n --argjson version "$VERSION" \
            --slurpfile storage catalogs/lakehouse-storage-after.json \
            '{currentEntityVersion: $version, storageConfigInfo: $storage[0]}')" ; \
  else echo 'STOP: no backup in this shell, or the entityVersion of lakehouse could not be read -- nothing was sent. Run the two blocks above again.'; fi
```

It must end with `HTTP 200`. The net change is three fields, and `properties` is
left out of the request so `default-base-location` is untouched:

```diff
+ "storageName": "storagea",
+ "stsEndpoint": "http://minio-a:9000",
- "stsUnavailable": true
+ "stsUnavailable": false
```

```bash
  curl -s http://localhost:8181/api/management/v1/catalogs/lakehouse \
    -H "Authorization: Bearer $TOKEN" | jq '{entityVersion, properties, storageConfigInfo}'
```

`"matches catalogs/lakehouse.json"` means the live catalog satisfies every
field of `catalogs/lakehouse.json` — the definition the from-scratch build
creates it with. A list is the fields that differ. `request failed: ...` means
Polaris never returned the catalog, so nothing was compared:

```bash
  curl -s http://localhost:8181/api/management/v1/catalogs/lakehouse \
    -H "Authorization: Bearer $TOKEN" \
    | jq -n --slurpfile target catalogs/lakehouse.json \
      '(try input catch null) as $r
       | if $r.storageConfigInfo == null then "request failed: \($r.error.message // "empty response -- check $TOKEN and the port-forward")"
         else $r.storageConfigInfo as $live
           | $target[0].catalog.storageConfigInfo
           | to_entries | map(select(.value != $live[.key]))
           | if length == 0 then "matches catalogs/lakehouse.json" else . end end'
```

Keep the failure branch. Without it an error still produces output: a `404`
has no `storageConfigInfo`, so every field of the target looks different, and
the result is a list of all ten keys that reads as a patch gone badly wrong
rather than a request that never landed.

## 12. Verify the migration

Connect to Polaris, and log in as Trino with the pair from `trino-secrets`:

```bash
  lsof -t -i tcp:8181 -s tcp:LISTEN | xargs -r kill ; sleep 1 ; \
  (kubectl port-forward svc/polaris 8181:8181 > /dev/null 2>&1 &) ; \
  for i in $(seq 60); do curl -s -o /dev/null http://localhost:8181/api/catalog/v1/config && break; sleep 1; done ; \
  TOKEN=$(curl -s -X POST http://localhost:8181/api/catalog/v1/oauth/tokens \
    -d "grant_type=client_credentials&client_id=root&client_secret=root&scope=PRINCIPAL_ROLE:ALL" \
    | jq -r '.access_token // empty') ; export TOKEN ; \
  if [ -n "$TOKEN" ]; then echo 'OK: Polaris is reachable and TOKEN is set'; else echo 'STOP: no TOKEN -- Polaris is not answering on localhost:8181'; fi
```

```bash
  POLARIS_AUTH=$(kubectl get secret trino-secrets -o jsonpath='{.data.POLARIS_AUTH}' | base64 -d) ; export POLARIS_AUTH ; \
  TRINO_TOKEN=$(curl -s -X POST http://localhost:8181/api/catalog/v1/oauth/tokens \
    -d "grant_type=client_credentials&client_id=${POLARIS_AUTH%%:*}&client_secret=${POLARIS_AUTH##*:}&scope=PRINCIPAL_ROLE:ALL" \
    | jq -r '.access_token // empty') ; export TRINO_TOKEN ; \
  if [ -n "$TRINO_TOKEN" ]; then echo 'OK: the trino-svc pair in trino-secrets authenticates'; else echo 'STOP: the POLARIS_AUTH pair in trino-secrets does not authenticate'; fi
```

Before the patch, Polaris refused this request with `400`. Now it succeeds and
carries a session token, and the token's `parent` claim names the MinIO user
Polaris assumed:

```bash
  curl -s http://localhost:8181/api/catalog/v1/lakehouse/namespaces/bronze/tables/customers \
    -H "Authorization: Bearer $TRINO_TOKEN" \
    -H "X-Iceberg-Access-Delegation: vended-credentials" \
    | jq -r '.config["s3.session-token"] | split(".")[1] | gsub("-";"+") | gsub("_";"/") | @base64d'
```

```json
  {"accessKey":"...","exp":...,"parent":"cat-a-user","sessionPolicy":"eyJWZXJ..."}
```

The old rows are still there, and new writes work:

```bash
  kubectl exec -it deploy/trino-coordinator -- trino --server localhost:8080 --user admin
```

```sql
SELECT * FROM lakehouse.bronze.customers ORDER BY customer_id;
INSERT INTO lakehouse.bronze.customers VALUES (4,'Katherine','Tabriz'),(5,'Edsger','Isfahan');
SELECT * FROM lakehouse.bronze.customers ORDER BY customer_id;

SELECT snapshot_id, committed_at, operation
FROM lakehouse.bronze."customers$snapshots" ORDER BY committed_at;
```

The metadata landed where it always did — one bucket, one prefix, the early
files written with root keys and the later ones under STS:

```bash
  kubectl exec deploy/minio-a -- sh -c "
      mc alias set a http://minio-a:9000 lakehouse-key lakehouse-secret-0123456789 > /dev/null
      mc ls --recursive a/neshan-lakehouse/bronze/" | grep metadata.json
```

```
  09:13  customers-ca4f229f.../metadata/00000-....metadata.json   <- before the migration
  09:13  customers-ca4f229f.../metadata/00001-....metadata.json   <- before
  09:13  customers-ca4f229f.../metadata/00002-....metadata.json   <- before
  09:18  customers-ca4f229f.../metadata/00003-....metadata.json   <- after it
  09:18  customers-ca4f229f.../metadata/00004-....metadata.json   <- after it
```

Five files, not two: the `CREATE TABLE` and `INSERT` made before the patch wrote
`00000` through `00002`, and the `INSERT` above wrote `00003` and `00004`. The
count depends on how many commits the table has taken; what the check is for is
the single shared prefix and the timestamps straddling the patch. `mc` runs
inside the MinIO pod, whose image has no `grep`, which is why the filter is on
the host side of the pipe.

## 13. Add the second catalog

Dropping the overlay is what makes `storageb` known, and removes the `AWS_*`
root keys the before state needed.

```bash
  helm upgrade --install polaris polaris/polaris --version 1.5.0 \
    --values values/polaris.yaml --timeout 10m
```

```bash
  kubectl rollout status deploy/polaris --timeout=600s
```

The pod was replaced, which killed the port-forward. Connect to Polaris again:

```bash
  lsof -t -i tcp:8181 -s tcp:LISTEN | xargs -r kill ; sleep 1 ; \
  (kubectl port-forward svc/polaris 8181:8181 > /dev/null 2>&1 &) ; \
  for i in $(seq 60); do curl -s -o /dev/null http://localhost:8181/api/catalog/v1/config && break; sleep 1; done ; \
  TOKEN=$(curl -s -X POST http://localhost:8181/api/catalog/v1/oauth/tokens \
    -d "grant_type=client_credentials&client_id=root&client_secret=root&scope=PRINCIPAL_ROLE:ALL" \
    | jq -r '.access_token // empty') ; export TOKEN ; \
  if [ -n "$TOKEN" ]; then echo 'OK: Polaris is reachable and TOKEN is set'; else echo 'STOP: no TOKEN -- Polaris is not answering on localhost:8181'; fi
```

`AWS_ACCESS_KEY_ID` is gone and both per-storage pairs are present. The chart's
`polaris.storage.aws.access-key` / `.secret-key` pair remains: `values/polaris.yaml`
points `storage.secret` at `minio-a-secrets`, and it is only the fallback for a
catalog that names no `storageName`, which neither catalog does from here on:

```bash
  kubectl get deploy polaris -o jsonpath='{range .spec.template.spec.containers[0].env[*]}{.name}{"\n"}{end}' \
    | grep -E 'AWS_ACCESS|AWS_SECRET|polaris.storage.aws'
```

```bash
  curl -X POST http://localhost:8181/api/management/v1/catalogs \
    -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
    -d @catalogs/lakehouse_fast.json
```

```bash
  curl -X PUT http://localhost:8181/api/management/v1/catalogs/lakehouse_fast/catalog-roles/catalog_admin/grants \
    -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
    -d '{"type": "catalog", "privilege": "CATALOG_MANAGE_CONTENT"}'
```

```bash
  curl -X PUT http://localhost:8181/api/management/v1/principal-roles/data_engineer/catalog-roles/lakehouse_fast \
    -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
    -d '{"catalogRole": {"name": "catalog_admin"}}'
```

Give Trino the second catalog's S3 keys. As before, `kubectl patch` changes only
the keys it names and leaves `POLARIS_AUTH` untouched:

```bash
  kubectl patch secret trino-secrets --type merge \
    -p '{"stringData": {"ACCESS_KEY_MINIO": "cat-a-user", "SECRET_KEY_MINIO": "cat-a-secret-0123456789", "ACCESS_KEY_MINIO_SD": "cat-b-user", "SECRET_KEY_MINIO_SD": "cat-b-secret-0123456789"}}'
```

```bash
  helm upgrade --install trino trino/trino --version 1.42.2 --values values/trino.yaml --timeout 10m
```

The `helm upgrade` already replaces both Trino pods, because adding
`lakehouse_fast` changes the pods' `checksum/catalog-config` annotation. The
restart below therefore rolls them a second time, and on a first pass it is
redundant. Keep it anyway: the chart has no checksum for `trino-secrets`, so on a
re-run of this step, where the catalogs are unchanged and only the Secret
differs, the restart is the only thing that loads the new credentials.

```bash
  kubectl rollout restart deploy/trino-coordinator deploy/trino-worker && \
  kubectl rollout status deploy/trino-coordinator --timeout=600s && \
  kubectl rollout status deploy/trino-worker --timeout=600s
```

## 14. Verify both catalogs

```bash
  kubectl exec -it deploy/trino-coordinator -- trino --server localhost:8080 --user admin
```

```sql
SHOW CATALOGS;

CREATE SCHEMA lakehouse_fast.bronze;
CREATE TABLE lakehouse_fast.bronze.orders (order_id BIGINT, customer_id BIGINT, amount DECIMAL(10,2));
INSERT INTO lakehouse_fast.bronze.orders VALUES (100,1,250.00),(101,1,75.50),(102,2,400.25),(103,3,15.00),(104,4,90.00);
SELECT * FROM lakehouse_fast.bronze.orders;

SELECT c.name, c.city, count(o.order_id) AS orders, sum(o.amount) AS total
FROM lakehouse.bronze.customers c
JOIN lakehouse_fast.bronze.orders o ON o.customer_id = c.customer_id
GROUP BY c.name, c.city
ORDER BY total DESC;
```

The join reads both MinIO instances at once — the same end state the
from-scratch build produces, reached without rewriting a table.

Connect to Polaris, and log in as Trino with the pair from `trino-secrets`:

```bash
  lsof -t -i tcp:8181 -s tcp:LISTEN | xargs -r kill ; sleep 1 ; \
  (kubectl port-forward svc/polaris 8181:8181 > /dev/null 2>&1 &) ; \
  for i in $(seq 60); do curl -s -o /dev/null http://localhost:8181/api/catalog/v1/config && break; sleep 1; done ; \
  TOKEN=$(curl -s -X POST http://localhost:8181/api/catalog/v1/oauth/tokens \
    -d "grant_type=client_credentials&client_id=root&client_secret=root&scope=PRINCIPAL_ROLE:ALL" \
    | jq -r '.access_token // empty') ; export TOKEN ; \
  if [ -n "$TOKEN" ]; then echo 'OK: Polaris is reachable and TOKEN is set'; else echo 'STOP: no TOKEN -- Polaris is not answering on localhost:8181'; fi
```

```bash
  POLARIS_AUTH=$(kubectl get secret trino-secrets -o jsonpath='{.data.POLARIS_AUTH}' | base64 -d) ; export POLARIS_AUTH ; \
  TRINO_TOKEN=$(curl -s -X POST http://localhost:8181/api/catalog/v1/oauth/tokens \
    -d "grant_type=client_credentials&client_id=${POLARIS_AUTH%%:*}&client_secret=${POLARIS_AUTH##*:}&scope=PRINCIPAL_ROLE:ALL" \
    | jq -r '.access_token // empty') ; export TRINO_TOKEN ; \
  if [ -n "$TRINO_TOKEN" ]; then echo 'OK: the trino-svc pair in trino-secrets authenticates'; else echo 'STOP: the POLARIS_AUTH pair in trino-secrets does not authenticate'; fi
```

Each catalog's credentials are minted for its own MinIO user:

```bash
  for t in lakehouse/namespaces/bronze/tables/customers lakehouse_fast/namespaces/bronze/tables/orders; do
    printf '%s -> ' "${t%%/*}"
    curl -s "http://localhost:8181/api/catalog/v1/$t" \
      -H "Authorization: Bearer $TRINO_TOKEN" \
      -H "X-Iceberg-Access-Delegation: vended-credentials" \
      | jq -r '.config["s3.session-token"] | split(".")[1] | gsub("-";"+") | gsub("_";"/") | @base64d | fromjson | .parent'
  done
```

Both catalogs match their definitions in `catalogs/`:

```bash
  for c in lakehouse lakehouse_fast; do
    printf '%s: ' "$c"
    curl -s "http://localhost:8181/api/management/v1/catalogs/$c" \
      -H "Authorization: Bearer $TOKEN" \
      | jq -cn --slurpfile target "catalogs/$c.json" \
        '(try input catch null) as $r
         | if $r.storageConfigInfo == null then "request failed: \($r.error.message // "empty response -- check $TOKEN and the port-forward")"
           else $r.storageConfigInfo as $live
             | $target[0].catalog.storageConfigInfo
             | to_entries | map(select(.value != $live[.key]))
             | if length == 0 then "matches" else . end end'
  done
```

## Why the steps are in this order

1. **Storage before catalog.** `minio-b` is stood up and tested as plain S3 in
   step 8. A failure there is a storage problem; found after the catalog existed,
   it would look like a catalog problem.
2. **Scoped identity before STS.** MinIO needs no STS feature turned on. What it
   needs is a non-root identity with a bucket policy, because the temporary
   credentials inherit that policy — assumed as root they reach everything.
3. **Server credentials before the catalog names them.** A catalog whose
   `storageName` the server has no credentials for fails with
   `Storage name 'storagea' is not configured on the server` the first time
   credentials are needed for it. Hence step 10 before step 11, and `storageb`
   only in step 13, with the catalog that names it.
4. **Patch, verify, then extend.** Step 11 moves the live catalog; step 12 checks
   it on its own, before a second catalog joins the picture.

The migration is finished at step 12. Steps 13 and 14 only add the second
catalog.

## Before you run this in production

The steps above were executed end to end and every claim in this file was
checked against a running cluster. The mechanism is sound: step 11 changes three
fields on one row, and no object is moved or rewritten. What follows is what the
demo does **not** tell you, and what will hurt if you carry this to a live
lakehouse unchanged.

**`DROP TABLE` deletes your data, asynchronously.** `values/polaris.yaml` sets
`DROP_WITH_PURGE_ENABLED: true`, and Trino sends `DELETE ...?purgeRequested=true`
by default on a REST catalog. Polaris then runs `TableCleanupTaskHandler` and
`BatchFileCleanupTaskHandler` to remove data files and manifests. The deletion
lands *after* the `DROP TABLE` returns: a bucket listing taken immediately
afterwards showed every object still present, and they were gone seconds later.
A quick post-drop check will tell you everything is fine when it is not. Set
`DROP_WITH_PURGE_ENABLED: false` for the duration of any recovery work.

**The roll-forward escape hatch does not work as configured.** [What a restore
costs you](#what-a-restore-costs-you) says a reverted commit "can be recovered by
rolling the Iceberg table forward to the newer metadata file by hand". Three
things block that here:

1. `register_table` is disabled — neither Trino values file sets
   `iceberg.register-table-procedure.enabled=true`, so the call answers
   `register_table procedure is disabled`.
2. You cannot register under a new name at the same location. Polaris refuses
   with `conflicts with existing table or namespace at location`, so the
   reverted table has to be dropped first.
3. Dropping it purges the objects the newer metadata points at — see above.

It *is* recoverable: enable the procedure, snapshot the table prefix, drop,
verify the objects survived, then register the table naming the newer
`.metadata.json` explicitly. That sequence was tested and returned every row.
Snapshot first — without it, the recovery destroys what it was recovering.

**Nothing here consumes the vended credentials.** Both catalogs set
`iceberg.rest-catalog.vended-credentials-enabled=false`, so Trino authenticates
to MinIO with the *static* `cat-a-user` / `cat-b-user` keys. The `AssumeRole`
credentials Polaris mints are read by nothing but the checks in steps 12 and 14.
The demo proves the plumbing, not the security benefit. The vended path does
work — with delegation on and the static keys removed, `SELECT`, `INSERT` and
`CREATE TABLE` all succeed against session credentials — but it is a separate
step that this file never takes, so treat it as unverified for your deployment.

**The upgrades are not zero-downtime.** Measured with an in-cluster prober at
4 req/s against `svc/polaris`, at `replicaCount: 1`:

| step | what goes away | for how long |
|---|---|---|
| each `helm upgrade` of Polaris (steps 5, 10, 13) | the catalog | ~2–4 s |
| each `helm upgrade` of MinIO (step 9) | that MinIO instance | ~2 s each: 1.6–2.2 s (`minio-a`), 1.6–1.8 s (`minio-b`) over three runs, sometimes as two short gaps — see below |
| the catalog `PUT` (step 11) | nothing | — |
| the metastore restore | the catalog | ~13 s |

The Deployment surges a new pod before terminating the old, so it looks safe,
but there is still an endpoint-propagation gap at cutover. The ~13 s restore
figure is from a near-empty metastore and tells you nothing about a real one —
measure it against a production-sized copy.

The MinIO row is the weakest figure in this table. It comes from an in-cluster
prober that polled `/minio/health/live` on `svc/minio-a` and `svc/minio-b` about
every 0.28 s while both step 9 upgrades ran, on three separate end-to-end runs.
Each instance missed one to three probes, and the window runs from the last
successful probe before the cutover to the first one after it. The cutover is
not always one clean gap. On one run `minio-a` failed, answered, then failed
again 1.8 s later, so the Service flapped between the old and new pods. Count a
flap from its first failure, about 2.2 s in that case. The spread between runs
(1.6 s, 2.2 s, 2.2 s for `minio-a`) is a reminder that three samples bound
nothing; plan for more. Three things it does not tell you. The
buckets were near-empty. A liveness answer is not a served `GET` or `PUT`, so
the gap for real S3 traffic may be longer. And the effect on a multipart upload
that is in flight across the cutover was not tested. Measure it with real
traffic before you schedule that step: unlike the Polaris rows, it takes
**object storage** down, which stops readers and writers that never touch the
catalog.

**The restore's data loss is real and silent.** Restoring a dump taken before
step 11 onto the finished cluster reverted `customers` from five rows to three
and removed `lakehouse_fast` entirely, with no error from any component. The
`$POLARIS_AUTH` pair kept working, so clients did not need re-issuing. Take the
dump as late as possible, and either freeze writers for the window or plan the
replay.

Smaller items: this Postgres has no PVC, so the metastore dies with the pod; the
deployment is single-realm and single-replica; and concurrent writers during the
patch were never exercised.

## Notes

**The `PUT` in step 11.** `storageConfigInfo` replaces, it does not merge, which
is why `catalogs/lakehouse-storage-after.json` is a complete document rather
than a delta. `roleArn` must not change: Polaris parses the AWS account id out
of it and refuses with `Cannot modify AWS account ID in storage config`. MinIO
ignores the ARN entirely, but Polaris still validates it.

**The comparison in steps 11 and 14 is one-directional.** It walks the keys of
the repo's catalog definition and reports any the live catalog does not match,
so a `"matches"` answer means the target is satisfied — not that the documents
are key-for-key equal. The live catalog carries `"stsUnavailable": false`, which
the file omits and which is Polaris's default anyway.

**Why `overlays/polaris-before.yaml` sets `AWS_ACCESS_KEY_ID`.** With
`"stsUnavailable": true` Polaris never calls `AssumeRole`, so it never builds
credentials from the storage configuration at all — the S3 client it uses for
metadata falls back to the AWS default provider chain, and those variables are
what the chain finds. Leave them out and the first `CREATE TABLE` fails with
`SdkClientException: Unable to load credentials from any of the providers in the
chain`. Helm replaces lists wholesale, so `polaris-storagea-only.yaml` has to
restate them to keep them.

**One root pair survives the migration.** `values/polaris.yaml` sets
`storage.secret.name: minio-a-secrets`, which no overlay touches, so the chart
renders `polaris.storage.aws.access-key` / `.secret-key` from minio-a's root
user into all three Polaris deployments here. It is the fallback identity for a
catalog that names no `storageName`; both catalogs name one by step 13, so
nothing reaches for it. Retiring it means clearing `storage.secret`, which
changes the end state the from-scratch build defines.

**The MinIO checks run inside the MinIO pod**, which already has `mc` on board.
That image ships no `grep`, `sed` or `awk`, so filter on the host side of the
pipe, as step 12 does.

## When it goes wrong

| symptom | cause |
|---|---|
| `Storage name 'storagea' is not configured on the server` | the catalog names a `storageName` before step 10. The `PUT` answers `200`; the failure surfaces later as `400` on `loadTable` |
| `Failed to update Catalog; currentEntityVersion ...` | the `entityVersion` in the `PUT` is stale — read it again and retry |
| `Cannot modify AWS account ID in storage config` | `roleArn` differs from the stored one |
| `400` with an **empty body**, and no `ExceptionMapper` line in the log | `storageType` was left out of the replacement document. Without it Jackson cannot resolve which `storageConfigInfo` subtype to build, so the request is rejected while it is still being deserialised — before any Polaris validation runs. There is no mapped exception to grep for; the access log shows `"PUT /api/management/v1/catalogs/lakehouse HTTP/1.1" 400 -` |
| `Unsupported storage type: ...` | `storageType` was changed. `values/polaris.yaml` narrows `SUPPORTED_CATALOG_STORAGE_TYPES` to `S3`, so any other value is rejected |
| `STOP: no TOKEN`, or a call answers `401` | the token expired (they last an hour) or the port-forward died. Re-run the *Connect to Polaris* block at the top of the step you are in; it restarts the port-forward and issues a new `$TOKEN` |
| Trino says `Not authorized:` with an empty message, Polaris logs `401` on `GET /api/catalog/v1/config`, or a block prints `STOP: the POLARIS_AUTH pair in trino-secrets does not authenticate` | `POLARIS_AUTH` in `trino-secrets` is empty or not a working pair. The steps never rewrite it after the principal is created, so this comes from a Secret written by an older version of this file. Rebuild the principal with the blocks in [If you lose the POLARIS_AUTH pair](#if-you-lose-the-polaris_auth-pair) |
| `curl: (7) Failed to connect` or `curl: (52) Empty reply` on port 8181 | a `helm upgrade` of Polaris replaced the pod and killed the port-forward — `(52)` if the old forward has not exited yet. Re-run the *Connect to Polaris* block of the step you are in |
| `bind: address already in use` from `kubectl port-forward`, and the *Connect to Polaris* block prints `STOP: no TOKEN` | a process the block cannot stop — one owned by another user — is holding 8181. The block stops whatever *you* have listening on 8181 before it starts the forward. `sudo lsof -i :8181` names the process; stop it and re-run the block |
| `AccessDenied` on metadata writes after the patch | the MinIO user's policy does not cover the bucket, or `storageName` points at the wrong pair |
| Trino still authenticating as the old user | `trino-secrets` changed but the pods did not restart |
| `pods "polaris-bootstrap" already exists` | `kubectl delete pod polaris-bootstrap`, then run it again |
| `secrets "..." already exists` | re-running an earlier step — `kubectl delete secret <name>` first |
| `ImagePullBackOff` | an image was not loaded into minikube |
| `error reading secrets/postgres-secrets.env: no such file or directory`, or any `no such file or directory` on `values/`, `catalogs/`, `manifests/` or `overlays/` | the shell is not in the directory holding this README. `cd` there and re-run the step; nothing was created, so there is nothing to clean up |
| `stsEndpoint` unreachable | it must resolve **from Polaris**, i.e. the in-cluster service name |
| `Failed to load table` right after a rollback | the rollback was done after step 13 — see [Rolling back](#rolling-back) |
| `lakehouse` works but `lakehouse_fast` answers `Failed to load table`, and Polaris logs `Storage name 'storageb' is not configured on the server` | Polaris was redeployed with `polaris-storagea-only.yaml` after step 13. Redeploy with `polaris-recovery.yaml` (to stay rolled back) or `values/polaris.yaml` alone (to return to the end state) |
| `View does not exist: bronze.customers` in the Polaris log | not an error: Trino checks whether a name is a view before treating it as a table |

Polaris reports most of these as mapped exceptions — the exception is the
deserialisation failure above, which never reaches the mapper:

```bash
  kubectl logs deploy/polaris --tail=200 | grep -a ExceptionMapper
```

`-a` is not optional: the startup banner contains NUL bytes, so once `--tail`
reaches back that far `grep` decides the stream is binary and prints nothing at
all — which reads exactly like a clean log.

## Backup and restore

### What to back up

| | why |
|---|---|
| the `polaris` Postgres database | the metastore. Catalogs, namespaces, every table's current metadata pointer, principals, roles and grants all live in `polaris_schema` — this is everything the migration can damage |
| the catalogs and roles as JSON, from the management API | a readable record to diff against afterwards, so you can see exactly what changed |
| the `$POLARIS_AUTH` pair already in `trino-secrets` | Polaris issues a principal's secret only when it creates it. If you ever rebuild the principal rather than restoring it, every client holding that pair has to be updated |

**You do not need to back up the buckets.** The migration writes nothing to
object storage — it only changes which credentials Polaris uses to reach it.
Step 12 shows the same metadata objects, at the same keys, before and after.

### Taking it

`pg_dump` is transactionally consistent, so this needs no downtime and no
quiescing — run it with everything live. Write it to your workstation, not into
the pod: this Postgres has no PVC, so a dump left inside it dies with the pod.

The dump contains `principal_authentication_data` — every principal's
credentials. `backup/` is in this repo's `.gitignore`; keep it that way, and
treat the file as a secret wherever you put it.

Each backup goes into its own timestamped directory, named by `$BACKUP`. The
directory is created with a plain `mkdir`, which refuses a name that already
exists, so a backup can never overwrite an earlier one. That matters most at
the worst moment: if step 11 goes wrong and you take a backup again before
restoring, a fixed file name would replace the last good dump with the broken
state, and nothing would warn you.

Connect to Polaris:

```bash
  lsof -t -i tcp:8181 -s tcp:LISTEN | xargs -r kill ; sleep 1 ; \
  (kubectl port-forward svc/polaris 8181:8181 > /dev/null 2>&1 &) ; \
  for i in $(seq 60); do curl -s -o /dev/null http://localhost:8181/api/catalog/v1/config && break; sleep 1; done ; \
  TOKEN=$(curl -s -X POST http://localhost:8181/api/catalog/v1/oauth/tokens \
    -d "grant_type=client_credentials&client_id=root&client_secret=root&scope=PRINCIPAL_ROLE:ALL" \
    | jq -r '.access_token // empty') ; export TOKEN ; \
  if [ -n "$TOKEN" ]; then echo 'OK: Polaris is reachable and TOKEN is set'; else echo 'STOP: no TOKEN -- Polaris is not answering on localhost:8181'; fi
```

The block checks that the dump holds Polaris's tables — a dump that ran against
the wrong database succeeds and produces a file with no data in it — and that
both JSON records came back from the API:

```bash
  BACKUP=backup/$(date +%Y%m%dT%H%M%S) ; export BACKUP ; \
  mkdir -p backup && mkdir "$BACKUP" && \
  kubectl exec deploy/postgres-polaris -- \
    pg_dump -U postgres -d polaris --format=plain --clean --if-exists \
    > "$BACKUP/polaris-metastore.sql" && \
  [ "$(grep -c 'COPY polaris_schema' "$BACKUP/polaris-metastore.sql")" -gt 0 ] && \
  curl -sf http://localhost:8181/api/management/v1/catalogs \
    -H "Authorization: Bearer $TOKEN" > "$BACKUP/catalogs.json" && \
  curl -sf http://localhost:8181/api/management/v1/principal-roles \
    -H "Authorization: Bearer $TOKEN" > "$BACKUP/principal-roles.json" && \
  echo "OK: backup complete in $BACKUP" || echo "STOP: the backup in $BACKUP is incomplete"
```

```bash
  jq -r '.catalogs[] | "\(.name) entityVersion=\(.entityVersion) storageName=\(.storageConfigInfo.storageName)"' "$BACKUP/catalogs.json"
```

### Which recovery you need

| what went wrong | remedy |
|---|---|
| the catalog was patched with the wrong `storageConfigInfo`, nothing else touched, and you are still before step 13 | the `PUT` in [Rolling back](#rolling-back). Seconds, no restart, no data loss |
| the wrong catalog was patched, several were touched, or a principal, role or grant was damaged | the metastore restore below. The rollback `PUT` cannot help: the storage config is not what is broken |
| you are past step 13 | restore Polaris's `AWS_*` pair first — redeploy with `overlays/polaris-recovery.yaml`, as in [Rolling back](#rolling-back) — then either remedy. **Not** `polaris-storagea-only.yaml`: it has no `storageb` pair, so it breaks `lakehouse_fast` |

### Restoring the metastore

Point `$BACKUP` at the dump you mean to restore, and check it before you stop
anything. In the shell that took the backup it is already set. In any other
shell, `ls backup/` lists every dump by time; choose the last one taken *before*
the change that went wrong, which is not necessarily the newest.

```bash
  [ -s "$BACKUP/polaris-metastore.sql" ] && ls -l "$BACKUP" \
    || echo 'BACKUP does not name a dump -- export BACKUP=backup/<timestamp> first; do not run the next command'
```

Stop Polaris first, so nothing writes while the dump is loading. This is a
catalog outage for its duration; Trino queries already running against cached
metadata may survive, new table loads will not.

```bash
  kubectl scale deploy/polaris --replicas=0 && \
  kubectl rollout status deploy/polaris --timeout=300s
```

```bash
  kubectl exec -i deploy/postgres-polaris -- \
    psql -U postgres -d polaris -v ON_ERROR_STOP=1 -q < "$BACKUP/polaris-metastore.sql"
```

```bash
  kubectl scale deploy/polaris --replicas=1 && \
  kubectl rollout status deploy/polaris --timeout=600s
```

The pod is new, which killed the port-forward. Connect to Polaris again:

```bash
  lsof -t -i tcp:8181 -s tcp:LISTEN | xargs -r kill ; sleep 1 ; \
  (kubectl port-forward svc/polaris 8181:8181 > /dev/null 2>&1 &) ; \
  for i in $(seq 60); do curl -s -o /dev/null http://localhost:8181/api/catalog/v1/config && break; sleep 1; done ; \
  TOKEN=$(curl -s -X POST http://localhost:8181/api/catalog/v1/oauth/tokens \
    -d "grant_type=client_credentials&client_id=root&client_secret=root&scope=PRINCIPAL_ROLE:ALL" \
    | jq -r '.access_token // empty') ; export TOKEN ; \
  if [ -n "$TOKEN" ]; then echo 'OK: Polaris is reachable and TOKEN is set'; else echo 'STOP: no TOKEN -- Polaris is not answering on localhost:8181'; fi
```

Then confirm the restore — the `diff` printing nothing is the check:

```bash
  curl -s http://localhost:8181/api/management/v1/principal-roles \
    -H "Authorization: Bearer $TOKEN" | jq -r '.roles[].name'
```

```bash
  curl -s http://localhost:8181/api/management/v1/catalogs -H "Authorization: Bearer $TOKEN" \
    | jq -S '.catalogs |= sort_by(.name)' \
    | diff <(jq -S '.catalogs |= sort_by(.name)' "$BACKUP/catalogs.json") - && echo "matches the backup"
```

Both sides are sorted by catalog name because Polaris does not list catalogs in
a fixed order. `jq -S` sorts the keys inside each object, not the entries of an
array. Without the `sort_by`, a restore that brought back exactly the right
catalogs can still print a long diff: the same two entries, swapped.

```bash
  kubectl exec -i deploy/trino-coordinator -- trino --server localhost:8080 --user admin \
    --execute "SELECT count(*) FROM lakehouse.bronze.customers"
```

### What a restore costs you

It returns the metastore to the instant of the dump. Any Iceberg commit made
after that instant is still sitting in object storage, but the catalog no longer
points at it, so those tables silently revert to their older snapshot — no
error, just missing rows. Either stop writers for the migration window, or take
the dump as late as possible and accept that anything committed afterwards has
to be replayed.

Restoring does not touch object storage, so nothing is deleted and a reverted
commit can be recovered by rolling the Iceberg table forward to the newer
metadata file by hand.

### If you lose the POLARIS_AUTH pair

Polaris hands a principal its secret once, when it creates it, and keeps only a
hash. The pair in `trino-secrets` is the only copy: it is not in the metastore,
not in the API and not in the logs. Two things that look like recoveries are
not, and both were tried against a running cluster:

* **Creating the principal again.** Against an existing `trino-svc` it answers
  `409 AlreadyExistsException` and returns no credentials.
* **Rotating the credentials.** `POST /principals/{name}/rotate` exists, but the
  service admin may not call it for somebody else. As `root` it answers
  `403 Principal 'root' ... is not authorized for op ROTATE_CREDENTIALS`. Only
  the principal itself may rotate, using the secret you no longer have.

If you have a dump taken on this cluster, restoring the metastore brings the
original pair back and every client keeps working. Otherwise the principal has
to be rebuilt, and that issues a **new `clientId`**, so every client holding the
old pair has to be updated. The blocks below rebuild it and put the new pair
into `trino-secrets`; they work at any point after the principal was first
created.

Connect to Polaris:

```bash
  lsof -t -i tcp:8181 -s tcp:LISTEN | xargs -r kill ; sleep 1 ; \
  (kubectl port-forward svc/polaris 8181:8181 > /dev/null 2>&1 &) ; \
  for i in $(seq 60); do curl -s -o /dev/null http://localhost:8181/api/catalog/v1/config && break; sleep 1; done ; \
  TOKEN=$(curl -s -X POST http://localhost:8181/api/catalog/v1/oauth/tokens \
    -d "grant_type=client_credentials&client_id=root&client_secret=root&scope=PRINCIPAL_ROLE:ALL" \
    | jq -r '.access_token // empty') ; export TOKEN ; \
  if [ -n "$TOKEN" ]; then echo 'OK: Polaris is reachable and TOKEN is set'; else echo 'STOP: no TOKEN -- Polaris is not answering on localhost:8181'; fi
```

```bash
  curl -X DELETE http://localhost:8181/api/management/v1/principals/trino-svc \
    -H "Authorization: Bearer $TOKEN"
```

Create the principal again and store the new pair in the same block.
`kubectl patch` replaces only `POLARIS_AUTH`; the S3 keys already in the Secret
stay as they are. Nothing is written unless Polaris returned credentials:

```bash
  POLARIS_AUTH=$(curl -s -X POST http://localhost:8181/api/management/v1/principals \
    -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
    -d '{"principal": {"name": "trino-svc", "type": "SERVICE"}}' \
    | jq -r 'if .credentials.clientId and .credentials.clientSecret then .credentials.clientId + ":" + .credentials.clientSecret else empty end') ; \
  export POLARIS_AUTH ; \
  if [ -n "$POLARIS_AUTH" ]; then \
    kubectl patch secret trino-secrets --type merge -p "{\"stringData\": {\"POLARIS_AUTH\": \"$POLARIS_AUTH\"}}" \
    && echo 'OK: trino-svc re-created and its new pair stored in trino-secrets' ; \
  else echo 'STOP: Polaris returned no credentials for trino-svc -- nothing was written'; fi
```

Deleting a principal takes its principal-role assignments with it, so the
binding has to go back. The grants that hang off `data_engineer` itself are
untouched, which is why only this one call is needed and the catalog roles are
not reassigned:

```bash
  curl -X PUT http://localhost:8181/api/management/v1/principals/trino-svc/principal-roles \
    -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
    -d '{"principalRole": {"name": "data_engineer"}}'
```

```bash
  curl -s http://localhost:8181/api/management/v1/principals/trino-svc/principal-roles \
    -H "Authorization: Bearer $TOKEN" | jq -c '[.roles[].name]'
```

That must print `["data_engineer"]`. The pair in the Secret must log in:

```bash
  POLARIS_AUTH=$(kubectl get secret trino-secrets -o jsonpath='{.data.POLARIS_AUTH}' | base64 -d) ; export POLARIS_AUTH ; \
  TRINO_TOKEN=$(curl -s -X POST http://localhost:8181/api/catalog/v1/oauth/tokens \
    -d "grant_type=client_credentials&client_id=${POLARIS_AUTH%%:*}&client_secret=${POLARIS_AUTH##*:}&scope=PRINCIPAL_ROLE:ALL" \
    | jq -r '.access_token // empty') ; export TRINO_TOKEN ; \
  if [ -n "$TRINO_TOKEN" ]; then echo 'OK: the trino-svc pair in trino-secrets authenticates'; else echo 'STOP: the POLARIS_AUTH pair in trino-secrets does not authenticate'; fi
```

Trino reads the Secret only at start-up:

```bash
  kubectl rollout restart deploy/trino-coordinator deploy/trino-worker && \
  kubectl rollout status deploy/trino-coordinator --timeout=600s && \
  kubectl rollout status deploy/trino-worker --timeout=600s
```

```bash
  kubectl exec -i deploy/trino-coordinator -- trino --server localhost:8080 --user admin \
    --execute "SELECT count(*) FROM lakehouse.bronze.customers"
```

## Rolling back

The patch is reversible, because it only changes how Polaris authenticates —
**but only while the `AWS_*` pair is still on the server, i.e. before step 13.**
A rolled-back catalog goes back to `"stsUnavailable": true`, which sends Polaris
to the default credentials provider chain; step 13 emptied that chain. Roll back
after step 13 and every query fails with `Failed to load table`, and Polaris logs
`SdkClientException: Unable to load credentials from any of the providers in the
chain`.

**After step 13, redeploy Polaris with `overlays/polaris-recovery.yaml` first**,
then send the `PUT` below. Before step 13, skip straight to the `PUT`: the
`AWS_*` pair is still there.

Do not use `overlays/polaris-storagea-only.yaml` for this, even though it also
carries the `AWS_*` pair. Helm replaces `extraEnv` wholesale, and that overlay
has no `storageb` entry. Redeploying with it after step 13 fixes `lakehouse` and
breaks `lakehouse_fast`: `SELECT` on it fails with `Failed to load table`, and
Polaris logs `Storage name 'storageb' is not configured on the server`.
`polaris-recovery.yaml` is exactly the step 13 environment plus the `AWS_*`
pair. With it, and with `lakehouse` rolled back, both catalogs were verified to
read, write and join.

```bash
  helm upgrade --install polaris polaris/polaris --version 1.5.0 \
    --values values/polaris.yaml \
    --values overlays/polaris-recovery.yaml \
    --timeout 10m
```

```bash
  kubectl rollout status deploy/polaris --timeout=600s
```

Connect to Polaris. Run this whether or not you redeployed — a redeploy
replaces the pod and kills the port-forward:

```bash
  lsof -t -i tcp:8181 -s tcp:LISTEN | xargs -r kill ; sleep 1 ; \
  (kubectl port-forward svc/polaris 8181:8181 > /dev/null 2>&1 &) ; \
  for i in $(seq 60); do curl -s -o /dev/null http://localhost:8181/api/catalog/v1/config && break; sleep 1; done ; \
  TOKEN=$(curl -s -X POST http://localhost:8181/api/catalog/v1/oauth/tokens \
    -d "grant_type=client_credentials&client_id=root&client_secret=root&scope=PRINCIPAL_ROLE:ALL" \
    | jq -r '.access_token // empty') ; export TOKEN ; \
  if [ -n "$TOKEN" ]; then echo 'OK: Polaris is reachable and TOKEN is set'; else echo 'STOP: no TOKEN -- Polaris is not answering on localhost:8181'; fi
```

Read `currentEntityVersion` and send the `PUT` in one block. Nothing is sent if
the version could not be read. It must end with `HTTP 200`:

```bash
  VERSION=$(curl -sf http://localhost:8181/api/management/v1/catalogs/lakehouse \
    -H "Authorization: Bearer $TOKEN" | jq -r '.entityVersion // empty') ; export VERSION ; \
  if [ -n "$VERSION" ]; then \
    curl -s -w '\nHTTP %{http_code}\n' -X PUT http://localhost:8181/api/management/v1/catalogs/lakehouse \
      -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
      -d "$(jq -n --argjson version "$VERSION" \
            --slurpfile before catalogs/lakehouse-before.json \
            '{currentEntityVersion: $version, storageConfigInfo: $before[0].catalog.storageConfigInfo}')" ; \
  else echo 'STOP: the entityVersion of lakehouse could not be read -- nothing was sent'; fi
```

## Teardown

```bash
  lsof -t -i tcp:8181 -s tcp:LISTEN | xargs -r kill ; minikube stop && minikube delete
```

## Files

Everything the steps reference lives here. The files that exist only to describe
the *before* state are marked; the rest are the shared deployment definitions,
identical to the companion project's.

```
  secrets/*.env                          root and per-catalog MinIO users, Postgres, Polaris JDBC
  manifests/00-postgres.yaml             the Polaris metastore
  values/minio-a.yaml                    minio-a: bucket neshan-lakehouse, user cat-a-user
  values/minio-b.yaml                    minio-b: bucket neshan-lakehouse-fast, user cat-b-user
  values/polaris.yaml                    Polaris at the END state: STS resolution on, both storage pairs
  values/trino.yaml                      Trino at the END state: both catalogs
  catalogs/lakehouse.json                the first catalog as the from-scratch build creates it
  catalogs/lakehouse_fast.json           the second catalog, created directly on STS in step 13

  overlays/minio-no-scoped-user.yaml     BEFORE: minio-a/minio-b without their user and bucket policy
  overlays/polaris-before.yaml           BEFORE: Polaris without per-storage credential resolution
  overlays/polaris-storagea-only.yaml    MID:    Polaris knowing storagea but not yet storageb
  overlays/polaris-recovery.yaml         RECOVERY: the end state plus the AWS_* pair, for a rollback after step 13
  values/trino-before.yaml               BEFORE: Trino with one catalog and minio-a's root keys
  catalogs/lakehouse-before.json         BEFORE: the catalog with "stsUnavailable": true
  catalogs/lakehouse-storage-after.json  the replacement storageConfigInfo the patch sends
```

## Appendix: the from-scratch build

**Do not run this on the same cluster as the steps above.** It is the other
route to the same destination — empty cluster to finished two-catalog setup in
one pass — reproduced verbatim from the companion project [demo-iceberg-polaris-multi-catalog-minio](../demo-iceberg-polaris-multi-catalog-minio) so this file stands on its own. Its
steps 1 and 2 are this file's steps 1 and 2.

| from-scratch | here | the difference |
|---|---|---|
| 3. Secrets | 3, 8, 9 | all five Secrets at once; migrating, `minio-b-secrets` and `catalog-users-secrets` arrive only when they exist |
| 4. Postgres and both MinIO | 4, 8, 9 | both instances arrive with their scoped user; migrating, both start without one |
| 5. Polaris | 5, 10, 13 | `values/polaris.yaml` alone; migrating, the same state in three deployments through two overlays |
| 6. Catalogs and principal | 6, 11, 13 | `lakehouse` is created already on STS; migrating, it is created without it and patched in step 11 |
| 7. Trino | 6, 9, 13 | one release with both catalogs and both scoped users; migrating, one catalog on root keys first |
| 8. Verify | 7, 12, 14 | one check at the end; migrating, one after every step |

### A.3 Secrets

```bash
  kubectl create secret generic postgres-secrets      --from-env-file=secrets/postgres-secrets.env && \
  kubectl create secret generic polaris-db-secrets    --from-env-file=secrets/polaris-db-secrets.env && \
  kubectl create secret generic minio-a-secrets       --from-env-file=secrets/minio-a-secrets.env && \
  kubectl create secret generic minio-b-secrets       --from-env-file=secrets/minio-b-secrets.env && \
  kubectl create secret generic catalog-users-secrets --from-env-file=secrets/catalog-users-secrets.env
```

### A.4 Postgres and both MinIO instances

```bash
  kubectl apply -f manifests/00-postgres.yaml
```

```bash
  helm upgrade --install minio-a minio/minio --version 5.4.0 --values values/minio-a.yaml --timeout 10m
```

```bash
  helm upgrade --install minio-b minio/minio --version 5.4.0 --values values/minio-b.yaml --timeout 10m
```

```bash
  kubectl wait --for=condition=available deploy/minio-a deploy/minio-b deploy/postgres-polaris --timeout=300s
```

### A.5 Polaris

```bash
  kubectl run polaris-bootstrap --rm -it --restart=Never \
  --image=apache/polaris-admin-tool:1.5.0 \
  --image-pull-policy=IfNotPresent \
  --env="polaris.persistence.type=relational-jdbc" \
  --env="quarkus.datasource.username=postgres" \
  --env="quarkus.datasource.password=postgres" \
  --env="quarkus.datasource.jdbc.url=jdbc:postgresql://postgres-polaris:5432/polaris" \
  -- bootstrap -r POLARIS -c POLARIS,root,root
```

```bash
  helm upgrade --install polaris polaris/polaris --version 1.5.0 --values values/polaris.yaml --timeout 10m
```

```bash
  kubectl rollout status deploy/polaris --timeout=600s
```

This one blocks. Leave it running and open a second terminal for the rest.

```bash
  kubectl port-forward svc/polaris 8181:8181
```

### A.6 Catalogs and principal

In the second terminal:

```bash
  export TOKEN=$(curl -s -X POST http://localhost:8181/api/catalog/v1/oauth/tokens \
    -d "grant_type=client_credentials&client_id=root&client_secret=root&scope=PRINCIPAL_ROLE:ALL" \
    | jq -r '.access_token')
```

```bash
  curl -X POST http://localhost:8181/api/management/v1/catalogs \
    -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
    -d @catalogs/lakehouse.json
```

```bash
  curl -X POST http://localhost:8181/api/management/v1/catalogs \
    -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
    -d @catalogs/lakehouse_fast.json
```

```bash
  curl -X PUT http://localhost:8181/api/management/v1/catalogs/lakehouse/catalog-roles/catalog_admin/grants \
    -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
    -d '{"type": "catalog", "privilege": "CATALOG_MANAGE_CONTENT"}'
```

```bash
  curl -X PUT http://localhost:8181/api/management/v1/catalogs/lakehouse_fast/catalog-roles/catalog_admin/grants \
    -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
    -d '{"type": "catalog", "privilege": "CATALOG_MANAGE_CONTENT"}'
```

```bash
  export POLARIS_AUTH=$(curl -s -X POST http://localhost:8181/api/management/v1/principals \
    -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
    -d '{"principal": {"name": "trino-svc", "type": "SERVICE"}}' \
    | jq -r '.credentials.clientId + ":" + .credentials.clientSecret')
```

```bash
  curl -X POST http://localhost:8181/api/management/v1/principal-roles \
    -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
    -d '{"principalRole": {"name": "data_engineer"}}'
```

```bash
  curl -X PUT http://localhost:8181/api/management/v1/principal-roles/data_engineer/catalog-roles/lakehouse \
    -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
    -d '{"catalogRole": {"name": "catalog_admin"}}'
```

```bash
  curl -X PUT http://localhost:8181/api/management/v1/principal-roles/data_engineer/catalog-roles/lakehouse_fast \
    -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
    -d '{"catalogRole": {"name": "catalog_admin"}}'
```

```bash
  curl -X PUT http://localhost:8181/api/management/v1/principals/trino-svc/principal-roles \
    -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
    -d '{"principalRole": {"name": "data_engineer"}}'
```

### A.7 Trino

```bash
  kubectl create secret generic trino-secrets \
    --from-literal=POLARIS_AUTH="$POLARIS_AUTH" \
    --from-literal=ACCESS_KEY_MINIO='cat-a-user' \
    --from-literal=SECRET_KEY_MINIO='cat-a-secret-0123456789' \
    --from-literal=ACCESS_KEY_MINIO_SD='cat-b-user' \
    --from-literal=SECRET_KEY_MINIO_SD='cat-b-secret-0123456789'
```

```bash
  helm upgrade --install trino trino/trino --version 1.42.2 --values values/trino.yaml --timeout 10m
```

```bash
  kubectl rollout status deploy/trino-coordinator --timeout=600s
```

### A.8 Verify

```bash
  kubectl exec -it deploy/trino-coordinator -- trino --server localhost:8080 --user admin
```

```sql
SHOW CATALOGS;

CREATE SCHEMA lakehouse.bronze;
CREATE TABLE lakehouse.bronze.customers (customer_id BIGINT, name VARCHAR, city VARCHAR);
INSERT INTO lakehouse.bronze.customers VALUES (1,'Ada','Tehran'),(2,'Grace','Shiraz'),(3,'Alan','Mashhad');
SELECT * FROM lakehouse.bronze.customers;

CREATE SCHEMA lakehouse_fast.bronze;
CREATE TABLE lakehouse_fast.bronze.orders (order_id BIGINT, customer_id BIGINT, amount DECIMAL(10,2));
INSERT INTO lakehouse_fast.bronze.orders VALUES (100,1,250.00),(101,1,75.50),(102,2,400.25),(103,3,15.00);
SELECT * FROM lakehouse_fast.bronze.orders;

SELECT c.name, c.city, count(o.order_id) AS orders, sum(o.amount) AS total
FROM lakehouse.bronze.customers c
JOIN lakehouse_fast.bronze.orders o ON o.customer_id = c.customer_id
GROUP BY c.name, c.city
ORDER BY total DESC;
```

The last query joins across both catalogs, reading from both MinIO instances at
once. That is the demo working.

The `orders` insert above is the companion project's, with four rows. Step 14
inserts a fifth, `(104,4,90.00)`, so that the join also covers a customer added
*after* the migration. The two routes reach the same configuration, not the same
contents, so the join's output differs by one row depending on which you ran.

### Teardown

```bash
  minikube stop && minikube delete
```
