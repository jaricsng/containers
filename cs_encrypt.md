---

copyright:
  years: 2014, 2018
lastupdated: "2018-10-19"

---

{:new_window: target="_blank"}
{:shortdesc: .shortdesc}
{:screen: .screen}
{:pre: .pre}
{:table: .aria-labeledby="caption"}
{:codeblock: .codeblock}
{:tip: .tip}
{:download: .download}


# Protecting sensitive information in your cluster
{: #encryption}

By default, your {{site.data.keyword.containerlong}} cluster uses encrypted disks to store information such as configurations in `etcd` or the container file system that runs on the worker node secondary disks. When you deploy your app, do not store confidential information, such as credentials or keys, in the YAML configuration file, configmaps, or scripts. Instead, use [Kubernetes secrets ![External link icon](../icons/launch-glyph.svg "External link icon")](https://kubernetes.io/docs/concepts/configuration/secret/). You can also encrypt data in Kubernetes secrets to prevent unauthorized users from accessing sensitive cluster information.
{: shortdesc}

For more information on securing your cluster, see [Security for {{site.data.keyword.containerlong_notm}}](cs_secure.html#security).



## Understanding when to use secrets
{: #secrets}

Kubernetes secrets are a secure way to store confidential information, such as user names, passwords, or keys. If you need confidential information encrypted, [enable {{site.data.keyword.keymanagementserviceshort}}](#keyprotect) to encrypt the secrets. For more information on what you can store in secrets, see the [Kubernetes documentation ![External link icon](../icons/launch-glyph.svg "External link icon")](https://kubernetes.io/docs/concepts/configuration/secret/).
{:shortdesc}

Review the following tasks that require secrets.

### Adding a service to a cluster
{: #secrets_service}

When you bind a service to a cluster, you don't have to create a secret to store your service credentials. A secret is automatically created for you. For more information, see [Adding Cloud Foundry services to clusters](cs_integrations.html#adding_cluster).

### Encrypting traffic to your apps with TLS secrets
{: #secrets_tls}

The ALB load balances HTTP network traffic to the apps in your cluster. To also load balance incoming HTTPS connections, you can configure the ALB to decrypt the network traffic and forward the decrypted request to the apps that are exposed in your cluster. For more information, see the [Ingress configuration documentation](cs_ingress.html#public_inside_3).

Additionally, if you have apps that require the HTTPS protocol and need traffic to stay encrypted, you can use one-way or mutual authentication secrets with the `ssl-services` annotation. For more information, see the [Ingress annotations documentation](cs_annotations.html#ssl-services).

### Accessing your registry with credentials stored in a Kubernetes `imagePullSecret`
{: #imagepullsecret}

When you create a cluster, secrets for your {{site.data.keyword.registrylong}} credentials are automatically created for you in the `default` Kubernetes namespace. However, you must [create your own imagePullSecret for your cluster](cs_images.html#other) if you want to deploy a container in the following situations.
* From an image in your {{site.data.keyword.registryshort_notm}} registry to a Kubernetes namespace other than `default`.
* From an image in your {{site.data.keyword.registryshort_notm}} registry that is stored in a different {{site.data.keyword.Bluemix_notm}} region or {{site.data.keyword.Bluemix_notm}} account.
* From an image that is stored in an {{site.data.keyword.Bluemix_notm}} Dedicated account.
* From an image that is stored in an external, private registry.

<br />


## Encrypting Kubernetes secrets by using {{site.data.keyword.keymanagementserviceshort}}
{: #keyprotect}

You can encrypt your Kubernetes secrets by using [{{site.data.keyword.keymanagementservicefull}} ![External link icon](../icons/launch-glyph.svg "External link icon")](/docs/services/key-protect/index.html#getting-started-with-key-protect) as a Kubernetes [key management service (KMS) provider ![External link icon](../icons/launch-glyph.svg "External link icon")](https://kubernetes.io/docs/tasks/administer-cluster/kms-provider/) in your cluster. KMS provider is an alpha feature in Kubernetes for versions 1.10 and 1.11.
{: shortdesc}

By default, Kubernetes secrets are stored on an encrypted disk in the `etcd` component of the IBM-managed Kubernetes master. Your worker nodes also have secondary disks that are encrypted by IBM-managed LUKS keys that are stored as secrets in the cluster. When you enable {{site.data.keyword.keymanagementserviceshort}} in your cluster, your own root key is used to encrypt Kubernetes secrets, including the LUKS secrets. You get more control over your sensitive data by encrypting the secrets with your root key. Using your own encryption adds an layer of security to your Kubernetes secrets and gives you more granular control of who can access sensitive cluster information. If you ever need to irreversibly remove access to your secrets, you can delete the root key.

**Important**: If you delete the root key in your {{site.data.keyword.keymanagementserviceshort}} instance, you cannot access or remove the data from the secrets in your cluster afterward.

Before you begin:
* [Log in to your account. Target the appropriate region and, if applicable, resource group. Set the context for your cluster](cs_cli_install.html#cs_cli_configure).
* Check that your cluster runs Kubernetes version 1.10.8_1524, 1.11.3_1521, or later by running `ibmcloud ks cluster-get --cluster <cluster_name_or_ID>` and checking the **Version** field.
* Verify that you have [**Administrator** permissions](cs_users.html#access_policies) to complete these steps.
* Make sure that the API key that is set for the region that your cluster is in is authorized to use Key Protect. To check the API key owner whose credentials are stored for the region, run `ibmcloud ks api-key-info --cluster <cluster_name_or_ID>`.

To enable {{site.data.keyword.keymanagementserviceshort}}, or to update the instance or root key that encrypts secrets in the cluster:

1.  [Create a {{site.data.keyword.keymanagementserviceshort}} instance](/docs/services/key-protect/provision.html#provision).

2.  Get the service instance ID.

    ```
    ibmcloud resource service-instance <kp_instance_name> | grep GUID
    ```
    {: pre}

3.  [Create a root key](/docs/services/key-protect/create-root-keys.html#create-root-keys). By default, the root key is created without an expiration date.

    Need to set an expiration date to comply with internal security policies? [Create the root key by using the API](/docs/services/key-protect/create-root-keys.html#api) and include the `expirationDate` parameter. **Important**: Before your root key expires, you must repeat these steps to update your cluster to use a new root key. Otherwise, you cannot unencrypt your secrets.
    {: tip}

4.  Note the [root key **ID**](/docs/services/key-protect/view-keys.html#gui).

5.  Get the [{{site.data.keyword.keymanagementserviceshort}} endpoint](/docs/services/key-protect/regions.html#endpoints) of your instance.

6.  Get the name of the cluster for which you want to enable {{site.data.keyword.keymanagementserviceshort}}.

    ```
    ibmcloud ks clusters
    ```
    {: pre}

7.  Enable {{site.data.keyword.keymanagementserviceshort}} in your cluster. Fill in the flags with the information that you previously retrieved.

    ```
    ibmcloud ks key-protect-enable --cluster <cluster_name_or_ID> --key-protect-url <kp_endpoint> --key-protect-instance <kp_instance_ID> --crk <kp_root_key_ID>
    ```
    {: pre}

After {{site.data.keyword.keymanagementserviceshort}} is enabled in the cluster, existing secrets and new secrets that are created in the cluster are automatically encrypted by using your {{site.data.keyword.keymanagementserviceshort}} root key. You can rotate your key at any time by repeating these steps with a new root key ID.
