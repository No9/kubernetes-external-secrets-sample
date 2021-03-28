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

  image:
    repository: ghcr.io/external-secrets/kubernetes-external-secrets
    tag: master
    ```

1. To install the chart with the release named `kubernetes-external-secrets-release`:
    ```
    $ helm install kubernetes-external-secrets-release . --skip-crds
    ```
    > **Tip:** A namespace can be specified by the `Helm` option '`--namespace kube-external-secrets`', however know this will not [autocreate a namespace](https://helm.sh/docs/faq/#automatically-creating-namespaces) like in Helm V2. To do that, also add the `--create-namespace` flag.

    > **Note**: `--skip-crds` is required in order to ensure the custom resource manager is used and will work for backwards compatibility. In future 4.x releases, this will not be required. See below for how to [disable the custom resource manager](#installing-the-crd) via the chart.

1. Now the service is installed we need to create a secret in Secret Manager that we will make available in k8s. 
    ```
    $ export IBM_CLOUD_SECRETS_MANAGER_API_URL=https://yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy.<Location>.secrets-manager.appdomain.cloud
    $ ibmcloud secrets-manager secret-create --secret-type username_password --metadata '{"collection_type": "application/vnd.ibm.secrets-manager.secret+json", "collection_total": 1}'  --resources '[{"name": "example-username-password-test-secret","description": "Extended description for my secret.","username": "user123","password": "cloudy-rainy-coffee-book"}]'
    
    ..id..
    ..zzzzzzzz-zzzz-zzzz-zzzz-zzzzzzzzzzzz.. 
    ```
1. Now we will create a CRD that will pull the secrets from the secret manager and add it to the namespace in the cluster. 
   Get the id from the output of the created secret and add update the `key` field in `ibmcloud-secrets-manager-example.yml`

    ```
    apiVersion: kubernetes-client.io/v1
    kind: ExternalSecret
    metadata:
      name: ibmcloud-secrets-manager-example
    spec:
      backendType: ibmcloudSecretsManager
      data:
        # The guid id of the secret
        - key: zzzzzzzz-zzzz-zzzz-zzzz-zzzzzzzzzzzz
          name: username
          property: username
          secretType: username_password
        - key: zzzzzzzz-zzzz-zzzz-zzzz-zzzzzzzzzzzz
          name: password
          property: password
          secretType: username_password
    ```

1. Apply the secret CRD to the cluster
    ```
    $ kubectl apply -f ibmcloud-secrets-manager-example.yml
    ```

1. View the created secrets in the cluster
    ```
    $ kubectl describe secrets ibmcloud-secrets-manager-example  
      Name:         ibmcloud-secrets-manager-example
      Namespace:    default
      Labels:       <none>
      Annotations:  <none>

      Type:  Opaque

      Data
      ====
      password:  9 bytes
      username:  5 bytes
    ```