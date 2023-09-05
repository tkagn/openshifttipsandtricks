# Chrony Configuration for RHOCP4

## Dump config to Base64

```bash
chrony_base64=$(cat << EOF | base64 -w0
server clock.corp.redhat.com iburst
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
logdir /var/log/chrony
EOF
)

```

## Generate manifests for Control and Compute Nodes

```bash
for i in master worker; do
cat << EOF > ./${i}-chrony-configuration.yml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: ${i}
  name: ${i}-chrony-configuration
spec:
  config:
    ignition:
      config: {}
      security:
        tls: {}
      timeouts: {}
      version: 2.2.0
    networkd: {}
    passwd: {}
    storage:
      files:
      - contents:
          source: data:text/plain;charset=utf-8;base64,${chrony_base64}
          verification: {}
        filesystem: root
        mode: 420
        path: /etc/chrony.conf
  osImageURL: ""
EOF
done
```

## Apply Manifests to RHOCP4 Clusters

```bash
oc apply -f ./master-chrony-configuration.yml
oc apply -f ./worker-chrony-configuration.yml
```

## References

- Configuring chrony time service - https://docs.openshift.com/container-platform/4.12/post_installation_configuration/machine-configuration-tasks.html#installation-special-config-chrony_post-install-machine-configuration-tasks

