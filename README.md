# Openshift Tips and Tricks

## Get events sorted by timestamp

``` oc get events -n openshift-storage   --sort-by=.metadata.creationTimestamp ```

## Get only names of pods

``` oc get pods -n openshift-storage -o name | grep noobaa ```

## Delete pods based on name

``` oc delete $(oc get pods -n openshift-storage -o name |grep noob) -n openshift-storage ```

## Delete pods based on status
``` oc delete pods  --field-selector status.phase=Pending -o name ```

## Setup HTPasswd Autentication

Install httpd-tools and create HTPassword file

```bash
dnf install httpd-tools
htpasswd -c -B -b /tmp/htpasswd 'auser' 'theuserpassword'
```

Generate HTPasswd Secret

```bash
oc create secret generic htpasswd --from-file=htpasswd=/tmp/htpasswd -n openshift-config
```

Update `oauth` with htpasswd identity provider

Dump current oauth
```bash
oc get oauth cluster -o yaml > oauth.yml
```
Add htpasswd identity provider

vi
```yaml
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec: 
  identityProviders:
    - name: htpasswd
      type: HTPasswd
      htpasswd:
        fileData:
          name: htpasswd
```
or 

```bash
# Test/review patch
oc patch oauth cluster -p '{"spec":{"identityProviders":[{"htpasswd":{"fileData":{"name":"htpasswd"}},"name":"htpasswd","type":"HTPasswd"}]}}' --type=merge --dry-run=server -o yaml

# Apply patch
oc patch oauth cluster -p '{"spec":{"identityProviders":[{"htpasswd":{"fileData":{"name":"htpasswd"}},"name":"htpasswd","type":"HTPasswd"}]}}' --type=merge

```
## Delete user from Openshift

```bash
oc delete user <username>
oc delete identity <IDP Name>:<username>

ex:
oc delete user dunbar
oc delete identity htpasswd:dunbar   
```

WARNING: If the identity is not deleted the user will not be able to login even though the user has been deleted

## Update htpasswd auth

```bash
# Extract htpasswd file from secret
oc get secret htpasswd-secret -ojsonpath={.data.htpasswd} -n openshift-config | base64 --decode > users.htpasswd

# Update htpasswd file with new user, edit current user password, or remove user
htpasswd -Bb users.htpasswd <username> '<password>'

# Import updated htpasswd file into htpasswd secret
oc create secret generic htpasswd-secret --from-file=htpasswd=users.htpasswd --dry-run=client -o yaml -n openshift-config | oc replace -f -
```

Authorization pods will be restarted in the openshift-authentication namespace.
Verify by `oc get pods -n openshift-authentication`

## Add OpenID authentication for Azure

Create secret in `openshift-config` namespace that contains clientsecret for Azure App Registration

```bash
oc create secret generic openid-client-secret --from-literal=clientSecret=<secret> -n openshift-config

# or copy secret to a file named clientSecret and use the file to poulated the clientSecret key

oc create secret generic openid-client-secret --from-file=clientSecret -n openshift-config
```

NOTE: The client secret can be/should be obtained and entered in a secure fashion. This command will leave the secret
in plain-text in the cli history.

Update oauth object

```bash
oc get oauths.config.openshift.io cluster -o yaml > oauth.yaml
```

Edit oauth.yml with new `identityProvider`

```yaml
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - name: AzureAD
    type: OpenID
    mappingMethod: claim
    openID:
      clientID: <clientID>
      clientSecret:
        name: openid-client-secret
      issuer: https://login.microsoftonline.com/<tenantID>
      claims:
        email:
          - email
        name:
          - name
        preferredUsername:
          - upn
```

Apply updated manifest to cluster

```bash
# test & verify 
oc apply -f oauth.yaml -o yaml --dry-run=server

# apply
 oc apply -f oauth.yaml
```

Resources:

- [Configuring a OpenID Connect identity provider](https://docs.openshift.com/container-platform/4.7/authentication/identity_providers/configuring-oidc-identity-provider.html)
- [Configure Azure Active Directory authentication for an Azure Red Hat OpenShift 4 cluster (Portal)](https://docs.microsoft.com/en-us/azure/openshift/configure-azure-ad-ui)

## Add User to cluster-admin role

```bash
# Create gorup
oc adm groups new ocp-admins

# Add user to group
oc adm groups add-users ocp-admins tkagn

#Add cluster-admin role to group
oc adm policy add-cluster-role-to-group cluster-admin ocp-admins
```


## Configuring control plane nodes as schedulable

```bash
oc edit schedulers.config.openshift.io cluster 
```
Configure the `mastersSchedulable` field

or 

```bash
# Test/review patch
oc patch schedulers/cluster -p '{"spec": {"mastersSchedulable": true}}' --type=merge --dry-run=server -o yaml

# Apply patch
oc patch schedulers/cluster -p '{"spec": {"mastersSchedulable": true}}' --type=merge
```

### Add Custom Notification Banners

```yaml
apiVersion: console.openshift.io/v1
kind: ConsoleNotification
metadata:
  name: bannertop
spec:
  backgroundColor: '#5b9e13'
  color: '#fff'
  location: BannerTop
  text: UNCLASSIFIED - Unauthorized access not permitted
```

### Add MOTD banner for CLI login

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: motd
  namespace: openshift
data:
  message: Welcome to the Red Hat OpenShift
```


### Removing the privilege to create projects
```bash
 oc patch clusterrolebinding.rbac self-provisioners -p '{"subjects": null}'
```

### Let a specific user create a project
```bash
oc adm policy add-cluster-role-to-user self-provisioner <username> --rolebinding-name='self-provisioners'
```

### Create project for User
```bash
oc new-project the-project-name
oc adm policy add-role-to-user admin the-user -n the-project-name --rolebinding-name='admin'
oc patch namespaces my-project-name -p '{"metadata":{"annotations":{"openshift.io/requester": "the-user"}}}'
```

## Configure Time Service

[./chrony configuration.md](./chrony-configuration.md)

