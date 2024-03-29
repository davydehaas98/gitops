# Create user and database
```
echo $(kubectl get secret --namespace postgresql postgresql-credentials -o jsonpath="{.data.postgres-password}" | base64 -d)

kubectl exec -it postgresql-0 -n postgresql -- psql -U postgres
```

```
CREATE DATABASE <NAME>;
CREATE USER <NAME> WITH ENCRYPTED PASSWORD '<PASSWORD>';
GRANT ALL PRIVILEGES ON DATABASE <NAME> TO <NAME>;
ALTER DATABASE <NAME> OWNER TO <NAME>;
```

# Upgrade
Dump database to .sql file.
```
k port-forward -n postgresql postgresql-0 5432:5432
pg_dumpall -U postgres -h localhost -p 5432 > dump.sql
```
Remove pvc from Longhorn and update the container to the new Postgresql version.
```
psql -h localhost -p 5432 -U postgres < dump.sql
```

# Sealed Secret
```
cat <<EOF > secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgresql-credentials
  namespace: postgresql
type: Opaque
stringData:
  postgres-password: RgwwwG623PwhLQreHxq
  replication-password: 6Qb9MQEXLVKwhFBJyRDj
EOF

cat secret.yaml | kubeseal \
  --format yaml > sealed-secret.yaml

cat sealed-secret.yaml
```
