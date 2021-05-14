# 💂 Kubernetes External Secrets on IBM Cloud

[Kubernetes External Secrets](https://github.com/external-secrets/kubernetes-external-secrets) allows you to use external secret management systems such as [IBM Secrets Manager](https://cloud.ibm.com/catalog/services/secrets-manager) to securely add secrets in Kubernetes. Read more about the design and motivation for Kubernetes External Secrets on the [GoDaddy Engineering Blog](https://godaddy.github.io/2019/04/16/kubernetes-external-secrets/).

This is a sample chart to demonstrate running it on IBM Cloud

## Prerequisites

* [Kubernetes 1.12+](https://cloud.ibm.com/kubernetes/catalog/create)
* [Secrets Manager](https://cloud.ibm.com/catalog/services/secrets-manager)

## Installing the Chart

1. Create your IAM API credentials.
    ```
    $ ibmcloud iam api-key-create kubernetes-external-secret-key
    ID            ApiKey-0705eaa9-c6f3-4c03-815e-dbc84c59d6db   
    Name          kubernetes-external-secret-key   
    Description      
    Created At    2021-03-14T17:44+0000   
    API Key       xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx   
    Locked        false 
    ```

1. Get the secret manager instance info where `Secrets Manager-01` is the name of your secrets manager instance.
    ```
    $ ibmcloud resource service-instance "Secrets Manager-01"
    Name:                  Secrets Manager-01   
    ID:                    crn:v1:bluemix:public:secrets-manager:us-south:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx::   
    GUID:                  yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy  
    Location:              us-south   
    Service Name:          secrets-manager   
    Service Plan Name:     lite   
    Resource Group Name:   default   
    State:                 active   
    Type:                  service_instance   
    Sub Type:                 
    Created at:            2020-10-19T19:45:05Z   
    Created by:            awhalley@ie.ibm.com   
    Updated at:            2020-10-19T19:49:56Z 
    ```

1. Create the kubernetes secrets using the API Key and GUID and location info from the output in the previous commands

    ```
    kubectl create secret generic ibmcloud-credentials --from-literal=apikey=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx --from-literal=endpoint=https://yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy.<Location>.secrets-manager.appdomain.cloud --from-literal=authtype=iam
    ```

1. To install the chart with the release named `kubernetes-external-secrets-release`:

    ```bash
    $ git clone https://github.com/external-secrets/kubernetes-external-secrets.git
    $ cd kubernetes-external-secrets
    ```

1. Edit the following values in the values.yaml in the kubernetes-external-secrets repo. The file `values-exanple.yaml` in this repo is there for reference.

    ```
    IBM_CLOUD_SECRETS_MANAGER_API_APIKEY:
    secretKeyRef: ibmcloud-credentials
    key: apikey
    IBM_CLOUD_SECRETS_MANAGER_API_ENDPOINT:
    secretKeyRef: ibmcloud-credentials
    key: endpoint
    IBM_CLOUD_SECRETS_MANAGER_API_AUTH_TYPE:
    secretKeyRef: ibmcloud-credentials
    key: authtype    
    ```

1. To install the chart with the release named `kubernetes-external-secrets-release`:
    ```
    $ helm install kubernetes-external-secrets-release .
    ```
    > **Tip:** A namespace can be specified by the `Helm` option '`--namespace kube-external-secrets`', however know this will not [autocreate a namespace](https://helm.sh/docs/faq/#automatically-creating-namespaces) like in Helm V2. To do that, also add the `--create-namespace` flag.

## Configure a username-password CRD

Now the service is installed we need to create a `username_password` secret in Secret Manager that we will make available in k8s.

1.  Ensure the secret manager environment variable is set
    ```
    $ export SECRETS_MANAGER_URL=https://yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy.<Location>.secrets-manager.appdomain.cloud
    ```

1. Now add a secret to secret manager taking note of the id generated
    ```    
    $ ibmcloud secrets-manager secret-create --secret-type username_password --resources '[{"name": "example-username-password-test-secret","description": "Extended description for my secret.","username": "user123","password": "cloudy-rainy-coffee-book"}]'
    
    ..id..
    ..zzzzzzzz-zzzz-zzzz-zzzz-zzzzzzzzzzzz.. 
    ```
1. Now we will create a CRD that will pull the secrets from the secret manager and add it to the namespace in the cluster. 
   Get the id from the output of the created secret and add update the `key` field in `username-password-example.yml`

    ```
    apiVersion: kubernetes-client.io/v1
    kind: ExternalSecret
    metadata:
      name: username-password-example
    spec:
      backendType: ibmcloudSecretsManager
      data:
        # The guid id of the secret
        - key: <guid>
          name: username
          property: username
          secretType: username_password
        - key: <guid>
          name: password
          property: password
          secretType: username_password
    ```

1. Apply the secret CRD to the cluster
    ```
    $ kubectl apply -f username-password-example.yml
    ```

1. View the created secrets in the cluster
    ```
    $ kubectl describe secrets username-password-example  
      Name:         username-password-example
      Namespace:    default
      Labels:       <none>
      Annotations:  <none>

      Type:  Opaque

      Data
      ====
      password:  9 bytes
      username:  5 bytes
    ```
## Configure an arbitrary CRD

Now the service is installed we need to create a `arbitrary` secret in Secret Manager that we will make available in k8s.

1.  Ensure the secret manager environment variable is set
    ```
    $ export SECRETS_MANAGER_URL=https://yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy.<Location>.secrets-manager.appdomain.cloud
    ```

1. Now add a secret to secret manager taking note of the id generated
    ```
    $ ibmcloud secrets-manager secret-create --secret-type arbitrary --resources '[{"name": "example-arbitrary-test-secret","description": "Extended description for my secret.", "payload":"avalue"}]'
    
    ..id..
    ..zzzzzzzz-zzzz-zzzz-zzzz-zzzzzzzzzzzz.. 
    ```
    
1. Now we will create a CRD that will pull the secrets from the secret manager and add it to the namespace in the cluster. 
   Get the id from the output of the created secret and add update the `key` field in `arbitrary-example.yml`

    ```
    apiVersion: kubernetes-client.io/v1
    kind: ExternalSecret
    metadata:
      name: arbritrary-example
    spec:
      backendType: ibmcloudSecretsManager
      data:
        - key: b6f0c056-382c-3ea9-991b-9172a1652c9e 
          name: example-arbitrary-test-secret
          property: payload
          secretType: arbitrary
    ```

1. Apply the secret CRD to the cluster
    ```
    $ kubectl apply -f arbitrary-example.yml
    ```

1. View the created secrets in the cluster
    ```
    $ kubectl describe secrets arbritrary-example 
      Name:         arbritrary-example
      Namespace:    default
      Labels:       <none>
      Annotations:  <none>

      Type:  Opaque

      Data
      ====
      example-arbitrary-test-secret:  6 bytes
    ```