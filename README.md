# Securing webhooks with EventListeners

Triggers now support end to end secure connection.
On OpenShift we support secure connection to eventlistener by adding label to namespace
```text
oc label namespace <ns-name> operator.tekton.dev/enable-annotation=enabled
```

Adding label to namespace will make sure that created eventlistener resources to run as `https`

### Implementation Detail:
When OpenShift Pipleine installed through Operator a `tekton-operator-proxy-webhook-` pod will be running always

Below are the functionality of proxy webhook pod
* Webhook watches for label on namespace and whenever it finds it tries to set up below Annoation
```text
service.beta.openshift.io/serving-cert-secret-name=<secret_name>
```
on the eventlistener object which interns create secrets and required certificates.

Ref: [Add a service certificate](https://docs.openshift.com/container-platform/4.7/security/certificates/service-serving-certificate.html#add-service-certificate_service-serving-certificate)

* Mounts the created secret into the eventlistener pod so that request will be secured.

Ref: [setting secret as env TLS_CERT and TLS_KEY](https://github.com/tektoncd/operator/blob/main/pkg/reconciler/openshift/annotation/annotation.go#L338-L358)

### Providing secure connection with OpenShift Routes

##### Create a route with the re-encrypt TLS termination:
```text
oc create route reencrypt --service=<svc-name> --cert=tls.crt --key=tls.key --ca-cert=ca.crt --hostname=<hostname>
```

Alternatively, you can create a re-encrypt TLS termination YAML file to create a secured route.

##### Example Re-encrypt TLS Termination YAML of the Secured Route
```yaml
apiVersion: v1
kind: Route
metadata:
  name: route-passthrough-secured  #1
spec:
  host: <hostname>
  to:
    kind: Service
    name: frontend #1
  tls:
    termination: reencrypt #2         
    key: [as in edge termination]
    certificate: [as in edge termination]
    caCertificate: [as in edge termination]
    destinationCACertificate: |- #3   
      -----BEGIN CERTIFICATE-----
      [...]
      -----END CERTIFICATE-----
```

###### #1 -> The name of the object, which is limited to 63 characters.
###### #2 -> The termination field is set to reencrypt. This is the only required tls field.
###### #3 -> Required for re-encryption. *destinationCACertificate* specifies a CA certificate to validate the endpoint certificate, securing the connection from the router to the destination pods. If the service is using a service signing certificate, or the administrator has specified a default CA certificate for the router and the service has a certificate signed by that CA, this field can be omitted.

See oc create route reencrypt --help for more options.

### Sample Example

Refering [pipeline-tutorial](https://github.com/openshift/pipelines-tutorial) as a sample example

* Create the `TriggerBinding` resource directly from the pipelines-tutorial Git repository:
```text
oc create -f https://raw.githubusercontent.com/openshift/pipelines-tutorial/master/03_triggers/01_binding.yaml
```
* create the `TriggerTemplate` resource directly from the pipelines-tutorial Git repository:
```text
oc create -f https://raw.githubusercontent.com/openshift/pipelines-tutorial/master/03_triggers/02_template.yaml
```
* Create the Trigger resource directly from the pipelines-tutorial Git repository:
```text
oc create -f https://raw.githubusercontent.com/openshift/pipelines-tutorial/master/03_triggers/03_trigger.yaml
```
* To create an EventListener resource using a secure HTTPS connection
    * Add a label to enable the secure HTTPS connection to the Eventlistener resource:
    ```text
    oc label namespace <ns-name> operator.tekton.dev/enable-annotation=enabled
    ```
    * Create the EvenListener resource directly from the pipelines-tutorial Git repository:
    ```text
    oc create -f https://raw.githubusercontent.com/openshift/pipelines-tutorial/master/03_triggers/04_event_listener.yaml
    ```
    * Create a route with the re-encrypt TLS termination:
    ```text
    oc create route reencrypt --service=<svc-name> --cert=tls.crt --key=tls.key --ca-cert=ca.crt --hostname=<hostname>
    ```
