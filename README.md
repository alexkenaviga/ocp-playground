# Readme

- [Openldap](#openldap)
    - [Initialization](#initialization)
    - [Install](#install)
    - [List resources](#list-resources)
    - [Cleanup](#cleanup)
- [PostgreSQL](#postgresql)
- [Keycloak](#keycloak)
    - [Operator](#operator)
    - [Database](#database)
    - [CRD](#crd)

---

## Openldap

### Initialization

Create **namespace**:

```bash
oc new-project ldap
oc project ldap
```

Allow **root-containers** in **namespace**

```bash
oc adm policy add-scc-to-user anyuid -z default -n ldap
```

### Install

```bash
oc apply -R -f ldap
oc apply -R -f phpldapadmin
```

wait for `deployments` availability:

```bash
oc wait -n ldap  deploy/openldap --for=condition=Available --timeout=5m
oc wait -n ldap  deploy/phpldapadmin --for=condition=Available --timeout=5m
```

Once `openldap` is **ready** install some user with:

```bash
LDAP_POD=$(oc get po -n ldap -o name | grep "openldap-" | cut -d "/" -f 2)
oc cp ldap/ldif/init.ldif $LDAP_POD:/tmp
oc rsh $LDAP_POD ldapadd -x -D "cn=admin,dc=keycloak,dc=training" -w "your-admin-password" -f /tmp/init.ldif
```

### List resources

```bash
oc get all,pvc,cm,secret -l app=test-openldap -o name
oc get all,pvc,cm,secret -l app=phpldapadmin -o name
```

### Cleanup

```bash
oc delete all,pvc,cm,secret -l app=test-openldap
oc delete all,pvc,cm,secret -l app=phpldapadmin
```

---

## PostgreSQL

To **install** the database use:

```bash
oc new-project postgresql

POSTGRE_NS=postgresql
DB_NAME=test_database

oc process -f postgresql/postgresql-persistent-template.json \
   -p POSTGRESQL_VERSION=15 \
   -p POSTGRESQL_USER=postgres \
   -p POSTGRESQL_PASSWORD=postgres \
   -p POSTGRESQL_DATABASE=$DB_NAME \
   -p VOLUME_CAPACITY=2Gi \
   -p DATABASE_SERVICE_NAME=psql-postgresql | oc create -n $POSTGRE_NS -f -

oc set image deployment/psql-postgresql -n $POSTGRE_NS postgresql=registry.redhat.io/rhel9/postgresql-15

oc wait -n $POSTGRE_NS deploy/psql-postgresql --for=condition=Available --timeout=5m

POSTGRE_POD=$(oc get po -o name | grep psql | cut -d "/" -f 2)

oc exec -n $POSTGRE_NS $POSTGRE_POD -i -- psql -U postgres -d postgres <<EOF
CREATE DATABASE $DB_NAME;
GRANT ALL PRIVILEGES ON DATABASE $DB_NAME TO postgres;
EOF
```

**NOTE**: to delete resources use:

```bash
oc delete -n $POSTGRE_NS \
    secret/psql-postgresql \
	service/psql-postgresql \
	persistentvolumeclaim/psql-postgresql \
	deployment.apps/psql-postgresql 
```

---

## Keycloak

### Operator

Get `keycloak` operator name with:

```bash
oc get packagemanifests -n openshift-marketplace | grep -i rhbk-operator
```

Then apply the `Subscription` and `OperatorGroup` in a new **namespace**:

```bash
oc new-project keycloak-operator
oc apply -f keycloak/keycloak-operator-subscription.yaml
```

To check resources use:

```bash
oc get csv,subscription,operatorgroup,installplan -n keycloak-operator
```

Once the `installplan` is ready, approve it with:

```bash
INSTALL_PLAN=$(oc get installplan -n keycloak-operator | grep "rhbk-" | cut -d " " -f 1)
oc patch installplan $INSTALL_PLAN -n keycloak-operator --type=merge -p '{"spec":{"approved":true}}'
```

### Database

Create database in `postgresql`


```bash
POSTGRE_NS=postgresql;
DB_NAME=keycloak;

POSTGRE_POD=$(oc get po -o name -n $POSTGRE_NS | grep psql | cut -d "/" -f 2);

oc exec -n $POSTGRE_NS $POSTGRE_POD -i -- psql -U postgres -d postgres <<EOF
CREATE DATABASE $DB_NAME;
GRANT ALL PRIVILEGES ON DATABASE $DB_NAME TO postgres;
EOF
```

Drop database:

```bash
POSTGRE_NS=postgresql;
DB_NAME=keycloak;

POSTGRE_POD=$(oc get po -o name -n $POSTGRE_NS | grep psql | cut -d "/" -f 2);

oc exec -n $POSTGRE_NS $POSTGRE_POD -i -- psql -U postgres -d postgres <<EOF 
DROP DATABASE $DB_NAME;
EOF
```

### CRD

Apply the resources:

```bash
HOSTNAME=keycloack.apps.cluster-9bvn9.9bvn9.sandbox2300.opentlc.com # replace with yors
oc apply -f keycloak/keycloak-crd.yaml
oc patch keycloak keycloak -n keycloak-operator --type=merge -p "{\"spec\":{\"hostname\":{\"hostname\":\"$HOSTNAME\"}}}"

KC_POD=$(oc get po -n keycloak-operator -l app=keycloak -o name)
oc wait -n keycloak-operator $KC_POD --for=condition=Ready --timeout=5m
```

Once connected you can use the `username/password` defined in **secret** `keycloak-initial-admin` to login to `admin-console`:

```bash
oc get secret keycloak-initial-admin -n keycloak-operator -o jsonpath='{.data}' \
    | jq -r 'to_entries[] | .key + ": " + (.value | @base64d)'
```

Delete resources:

```bash
oc delete secret,keycloak,all -n keycloak-operator -l app=test-keycloak
oc delete secret keycloak-initial-admin -n keycloak-operator
```

**NOTE**: if you delete the CRD, also the database has to be removed (see [Drop database](#database))

---


