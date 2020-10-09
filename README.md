# Openshift Tips and Tricks

### Get Events sorted by timestamp ######

``` oc get events -n openshift-storage   --sort-by=.metadata.creationTimestamp ```
