# Readme

---

## Initialization

### Create namespace

```bash
oc new-project ldap
oc project ldap
```

### Allow root containers in namespace


```bash
oc adm policy add-scc-to-user anyuid -z default -n ldap
```

---

## Openldap

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

--
