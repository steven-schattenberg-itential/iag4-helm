# Helm charts for Itential Automation Gateway 4

This repo contains helm charts for running Itential Automation Gateway in Kubernetes. Requires
Helm version `v3.15.0`.

## Itential Automation Gateway (IAG)

The application is installed using a Kubernetes statefulset. It also includes persistent volume
claims, ingress, and other Kubernetes objects suitable to run the application.

### Usage

```bash
helm install iag . -f values.yaml
```

This will install IAG according to how its configured in the values.yaml file.

```bash
helm install iag . -f values.yaml --set image.tag=4.3.4
```

This will install IAG with the "4.3.4" image.

### Requirements & Dependencies

| Repository | Name | Version |
|:-----------|:-----|:--------|
| <https://charts.jetstack.io> | cert-manager | 1.12.3 |
| <https://kubernetes-sigs.github.io/external-dns/> | external-dns | 1.17.0 |

#### Secrets

The chart assumes the following secrets, they are not included in the Chart.

##### imagePullSecrets

This is the secret that will pull the image from the Itential ECR. Name to be determined by the user
 of the chart and that name must be provided in the values file (`imagePullSecrets[0].name`).

##### itential-ca

This secret represents the CA used by cert-manager to derive all the TLS certificates. Name to be
provided by the user of the chart in the values file (`issuer.caSecretName`) if using cert-manager.

##### Hashicorp Vault

If this installation is configured to use Hashicorp Vault then an additional secret should be
provided to secure the connection using TLS. The secret must have the following keys:

| Key | Description |
|:----|:------------|
| ca.crt | The CA file for the Hashivault certificates |
| tls.crt | The TLS certificate for the Hashivault |
| tls.key | The key for the certificate |
| token | The token for the vault server |

##### LDAP Secrets

If this installation is configured to use LDAP authentication then an additional secret should be
provided to include the LDAP password and any TLS certificates. This secret must have the following
keys:

| Key | Description |
|:----|:------------|
| tls.crt | The TLS certificate for the LDAP server. |
| password | The token for the vault server |

#### Certificates

The chart will require a Certificate Authority to be added to the Kubernetes environment. This is
used by the chart when running with `useTLS` flag enabled. The chart will use this CA to generate
the necessary certificates using a Kubernetes `Issuer` which is included. The Issuer will issue the
certificates using the CA. The certificates are then included using a Kubernetes `Secret` which is
mounted by the pods. Creating and adding this CA is outside of the scope of this chart.

Both the `Issuer` and the `Certificate` objects are realized by using the widely used Kubernetes
add-on called `cert-manager`. Cert-manager is responsible for making the TLS certificates required
by using the CA that was installed separately. The installation of cert-manager is outside the scope
of this chart. To check if this is already installed run this command:

```bash
kubectl get crds | grep cert-manager
```

For more information see the [Cert Manager project](https://cert-manager.io/).

If `cert-manager` can not be used then the TLS certificates must be manually added to the Kubernetes
cluster. The Helm templates expect them to be in a secret named `<Chart.name>-tls-secret`. It
expects the following keys:

| Key | Description |
|:----|:------------|
| tls.crt | The TLS certificate that identifies this "server". |
| tls.key | The private key for this certificate. |
| ca.crt | The Certificate Authority used to generate these certificates and keys. |

#### SSH config
If there is any custom ssh client configuration needed, create the ssh_config file and place it
in ```files/ssh_config```. The chart will place it in ```/etc/ssh/ssh_config```.

#### DNS

This is not a requirement but more of an explanation.

Itential used the ExternalDNS project to facilitate the creation of DNS entries. ExternalDNS
synchronizes exposed Kubernetes Services and Ingresses with DNS providers. This is not a
requirement for running the chart. The chart and the application can run without this if needed. We
chose to use this to provide us with external addresses.

For more information see the [ExternalDNS project](https://github.com/kubernetes-sigs/external-dns).

### Volumes

The application does require some volumes to exist. These are mounted and used by the application
as the location for custom scripts and for the location of the onboard database. They should persist
after a shutdown to prevent any data loss.

| Name | Type  | Description  |
|:-----|:------|:-------------|
| config-volume   | Configmap               | The chart uses this volume to mount important configurations files like ansible.cfg |
| iag-data-volume | Persistent Volume Claim | This represents the claim for the persistence for the sqlite data required by IAG.                                       |
| iag-code-volume | Persistent Volume Claim | This represents the claim for the location of all the customer authored scripts, ansible code, and other customizations. |

#### How to construct the iag-code-volume

This volume is intended to store any custom assets that the user is bringing to IAG. Its contents will reflect a customer's unique usage of IAG. There is an expectation in the container of the structure of the files in this volume. It should look like this:

```bash
.
├── ansible/ # custom ansible assets
│   ├── collections
│   ├── inventory
│   ├── modules
│   ├── playbooks
|   ├── ansible_venvs
│   └── roles
├── venv/ # custom python virtual environments
├── git/
│   ├── ssh
│   └── repos
├── nornir/
│   ├── conf
│   ├── inventory
│   └── modules
└── scripts/ # custom script assets
```

### Tests

The chart comes with a suite of unit tests. Those can be executed by running:

```bash
helm unittest . --strict
```

#### Values

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| affinity | object | `{}` | Additional affinity |
| applicationPort | int | `8443` | The application port. |
| applicationSettings.ansibleEnabled | bool | `true` | Enables the Ansible feature set in IAG. |
| applicationSettings.env.automation_gateway_ansible_debug | bool | `false` | Enable Ansible debug logs. |
| applicationSettings.env.automation_gateway_audit | bool | `false` | Enable audit trail in SQLite. |
| applicationSettings.env.automation_gateway_audit_retention_days | int | `30` | The number of days to retain audit information. |
| applicationSettings.env.automation_gateway_authentication_idle_timeout | int | `600` | Idle session timeout. |
| applicationSettings.env.automation_gateway_http_server_threads | int | `8` | Controls the number of threads available to use for handling HTTP requests. Generally, this should be set 2 - 4 times the number CPUs available. |
| applicationSettings.env.automation_gateway_logging_level | string | `"INFO"` | Log level controlling what appears in the log messages. |
| applicationSettings.env.automation_gateway_no_cleanup | bool | `false` | A flag to prevent deletion of the temporary files generated by executing Ansible content. |
| applicationSettings.gitEnabled | bool | `true` | Enables the Git integration feature set in IAG. |
| applicationSettings.grpcEnabled | bool | `true` | Enables the GRPC feature set in IAG. |
| applicationSettings.httpRequestsEnabled | bool | `true` | Enables the HTTP request feature set in IAG. |
| applicationSettings.hvCertVerification | bool | `false` | Enables certificate verification. |
| applicationSettings.hvEnabled | bool | `false` | Enables the Hashicorp Vault integration. |
| applicationSettings.hvHost | string | `"hashivault.example.com"` | The host name of the Hashicorp Vault server. |
| applicationSettings.hvMountPoint | string | `"secret"` | The mount point. |
| applicationSettings.hvSecretName | string | `"hv-secret"` | The name of the Secret object containing Hashicorp TLS information. |
| applicationSettings.hvTLS | bool | `true` | Use TLS when connecting to Hashicorp Vault server. |
| applicationSettings.ldapCertVerificationEnabled | bool | `false` | Enable validation of the TLS certificates. |
| applicationSettings.ldapEnabled | bool | `false` | Enable LDAP authentication. |
| applicationSettings.ldapSecretName | string | `"ldap-secret"` | The name of the secret containing the TLS certificates and LDAP password. |
| applicationSettings.ldapTLSEnabled | bool | `false` | Enable TLS while connecting to LDAP.. |
| applicationSettings.netconfEnabled | bool | `true` | Enables the Netconf feature set in IAG. |
| applicationSettings.netmikoEnabled | bool | `true` | Enables the Netmiko feature set in IAG. |
| applicationSettings.nornirEnabled | bool | `false` | Enables the Nornir feature set in IAG. |
| applicationSettings.pythonVenvEnabled | bool | `true` | Enables the Python Virtual Environment feature set in IAG. |
| applicationSettings.scriptsEnabled | bool | `true` | Enables the custom scripts feature set in IAG. |
| certManager | object | `{"enabled":false}` | Toggle to use certManager as a sub chart. |
| certificate.duration | string | `"2160h"` | Specifies how long the certificate should be valid for (its lifetime). |
| certificate.enabled | bool | `false` | Toggle to use the certificate object or not |
| certificate.hostname | string | `"iag.example.com"` | The static DNS name to pass to the certificate. |
| certificate.issuerRef.kind | string | `"Issuer"` | The issuer type |
| certificate.issuerRef.name | string | `"iag-ca-issuer"` | The name of the issuer with the CA reference. |
| certificate.renewBefore | string | `"48h"` | Specifies how long before the certificate expires that cert-manager should try to renew. |
| external-dns | object | `{"enabled":false}` | Optional dependency to generate a static external DNS name |
| image.pullPolicy | string | `"IfNotPresent"` | The image pull policy |
| image.repository | string | `"497639811223.dkr.ecr.us-east-2.amazonaws.com/automation-gateway"` | The image repository |
| image.tag | string | `nil` | The image tag |
| imagePullSecrets | list | `[]` | The secrets object used to pull the image from the repo |
| ingress.annotations | string | `nil` | The annotations for this ingress object. These are passed into the template as is and will render the contents as you see here. |
| ingress.className | string | `""` | The ingress controller class name |
| ingress.enabled | bool | `true` | The ingress object can be disabled and will not be created with this set to false |
| ingress.hosts | list | `[{"host":"iag.example.com","paths":[{"path":"/","pathType":"Prefix"}]}]` | The list of hosts for this ingress and their associated properties |
| ingress.name | string | `"iag-ingress"` | The name of this Kubernetes ingress object |
| issuer.caSecretName | string | `"some-secret"` | The CA secret to be used by this issuer when creating TLS certificates. |
| issuer.enabled | bool | `false` | Toggle true/false to use issuer. |
| issuer.kind | string | `"Issuer"` | The issuer kind. |
| issuer.name | string | `"iag-ca-issuer"` | The name of the issuer. |
| nodeSelector | object | `{"itential.io/app":"iag"}` | Additional nodeSelectors |
| persistentVolumeClaims.codeClaim | object | `{"storage":"10Gi"}` | This represents the claim for the location of all the customer authored scripts, ansible code, and other customizations. |
| persistentVolumeClaims.codeClaim.storage | string | `"10Gi"` | The requested amount of storage |
| persistentVolumeClaims.dataClaim | object | `{"storage":"10Gi"}` | This represents the claim for the persistence for the sqlite data required by IAG. |
| persistentVolumeClaims.dataClaim.storage | string | `"10Gi"` | storage The requested amount of storage |
| persistentVolumeClaims.enabled | bool | `true` | Toggle the use of persistentVolumeClaims |
| podAnnotations | object | `{}` | Additional pod annotations |
| podLabels | object | `{}` | Additional pod labels |
| podSecurityContext | object | `{"fsGroup":1001,"runAsNonRoot":true,"runAsUser":1001}` | The pods will mount some persistent volumes. These settings allow for that to happen. |
| replicaCount | int | `1` | The number of replicas. IAG is intended to be a standalone application. Its not recommended to run anymore than 1. |
| securityContext | object | `{}` | Additional security context |
| service.name | string | `"iag-service"` | The name of this Kubernetes service object |
| service.port | int | `443` | The port that this service object is listening on |
| service.type | string | `"ClusterIP"` | The service type |
| storageClass.enabled | bool | `true` | Toggle the use of storageClass |
| storageClass.name | string | `"iag-ebs-gp3"` | The name of the storageClass |
| storageClass.parameters | string | `nil` | Key-value pairs passed to the provisioner |
| storageClass.provisioner | string | `""` | Specifies which volume plugin provisions the storage |
| storageClass.reclaimPolicy | string | `"Retain"` | What happens to PersistentVolumes when released. Itential recommends "retain". |
| storageClass.volumeBindingMode | string | `"WaitForFirstConsumer"` | Controls when volume binding occurs |
| tolerations | list | `[{"effect":"NoSchedule","key":"itential.io/role","operator":"Equal","value":"iag"}]` | Additional tolerations |
| useTLS | bool | `true` | Toggle to turn on/off TLS support. Itential recommends running this application with TLS. |
| volumeMounts | list | `[]` | Additional volumeMounts on the output Statefulset definition. |
| volumes | list | `[]` | Additional volumes on the output Statefulset definition. |