# Openshift Tips and Tricks

### Get events sorted by timestamp ######

``` oc get events -n openshift-storage   --sort-by=.metadata.creationTimestamp ```

### Get only names of pods ####

``` oc get pods -n openshift-storage -o name | grep noobaa ```

### Delete pods based on name ####

``` oc delete $(oc get pods -n openshift-storage -o name |grep noob) -n openshift-storage ```

### Setup HTPasswd Autentication ###

Install httpd-tools and create HTPassword file

```bash
dnf install httpd-tools
htpasswd -c -B -b /tmp/htpasswd 'auser' 'theuserpassword'
```
Generate HTPasswd Secret
```bash
oc create secret generic httpasswd-secret --from-file=htpasswd=/tmp/htpasswd -n openshift-config
```



### Delete user from Openshift

```bash
oc delete user <username>
oc delete identity <IDP Name>:<username>

ex:
oc delete user dunbar
oc delete identity htpasswd:dunbar   
```
WARNING: If the identity is not deleted the user will not be able to login even though the user has been deleted


### Update htpasswd auth 

```bash
oc get secret htpasswd-secret -ojsonpath={.data.htpasswd} -n openshift-config | base64 --decode > users.htpasswd
htpasswd -Bb users.htpasswd <username> '<password>'
oc create secret generic htpasswd-secret --from-file=htpasswd=users.htpasswd --dry-run=client -o yaml -n openshift-config | oc replace -f -
```
Authorization pods will be restarted in the openshift-authentication namespace.
Verify by ` oc get pods -n openshift-authentication`

### Add User to cluster-admin role

```bash
# Create gorup
oc adm groups new ocp-admins

# Add user to group
oc adm groups add-users ocp-admins tkagn

#Add cluster-admin role to group
oc adm policy add-cluster-role-to-group cluster-admin ocp-admins
```


