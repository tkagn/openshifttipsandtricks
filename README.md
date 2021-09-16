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


```
