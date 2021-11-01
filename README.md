# Postfacto platform Deployment.

 [This is the repository](https://github.com/bgarcial/postfacto-platform/) where the postfacto custom helm chart and besides resources files are:


## Some decisions made


1. **Fully managed PostgreSQL database cloud service**

I decided to get out of the cluster the database server. I consider the following cons here: 
- Decoupling the application  from the persistence data. The only thing the cluster is responsible is for the code and
some static assets files content that the applications used for the user interface. So not writes operations over databases
and no left over files when pods are deleted.

- The creation of persistent volumes (and pvc and storage class resources) and attach them to the pods are not necessart, 
so I got one less thing to maintain. 

- In this sense the postgresql headless service that comes with the helm chart to communicate postfacto pods with the
containerized postgres chart, both are not needed.

As a "cons" since a fully SaaS Postresql database is created, we got something additional to pay for. 

2. **Customizing Postfacto helm chart**

I used the [postfacto helm chart](https://github.com/pivotal/postfacto/blob/master/deployment/helm/README.md) 
created to deploy the applications, but along the way I've made the following modifications:

2.1 **A PostgreSQL secret resource is created.** 

Within templates directory, I've created the [`helm/charts/postfacto/templates/pg_secrets.yaml`](https://github.com/bgarcial/postfacto-platform/blob/staging/helm/charts/postfacto/templates/pg_secrets.yaml#L7) file
applying the `pre-install` hook in order to create this secret as long the helm chart templares are rendered, but before
the postfacto application K8 deployment take place in the cluster.

It also has the `before-hook-creation` deletion policy which consists in that this secret will be deleted
right before a new hook (the `pre-install` one in this case) is created.

The behavior here is: 
- The secret is created right before the helm chart is deployed on the cluster
- This secret is not eliminated if we execute `helm uninstall postfacto-release`, it will persist
- But as long a new `helm upgrade --install postfacto-release` command is executed, the secret will be eliminated and created again

2.2. **The `pg_secrets.yaml` file has fake data hardcoded**

Since a secret object is being defined in yaml format, in order to don't hardcode the real postgres secrets values 
(the base64 encode values for username, password, database and hostname) and upload them to the github repository,
I implemented the following approach to protect them and inject them on runtime in the release pipeline

So, the [`set_pg_secrets.py`](https://github.com/bgarcial/postfacto-platform/blob/staging/set_pg_secrets.py) script
enter into action during the pipeline to read the real secrets information and send them as string positional arguments 
[to override them over specifical YAML attributes in the secret resource](https://github.com/bgarcial/postfacto-platform/blob/staging/helm/charts/postfacto/templates/pg_secrets.yaml#L10-L14). 

Then in Azure DevOps a variable groups called **PostgreSQL database secrets** is created containing the real sensitive information
to connect with PostgreSQL from postfacto pods applications


![](https://cldup.com/psMwIgZjhz.png)

Then during the release pipeline the real secrets are being injected, passed as string positional arguments to the 
[`pg_secrets.yaml`](https://github.com/bgarcial/postfacto-platform/blob/staging/set_pg_secrets.py#L48-L52) file.

![](https://cldup.com/4v5kx5Jfx3.png) 

---

**IMPORTANT TO HIGHLIGHT**

Right now there are another approaches to pursue this same operation of protect or even don't create secrets into the
AKS cluster, this is, synchronize the secrets by fetching them directly from the pods to Vault services like Azure KeyVault or
Hashicorp Vault. 
In that sense, tools like [akv2k8s.io](https://akv2k8s.io/) or the [Azure KeyVault Provider for Secrets storage](https://azure.github.io/secrets-store-csi-driver-provider-azure/)
constitute a better approach to follow if we are dealing with recurrent sync operations regarding secrets retrieval. 

---

2.3 **The secrets are being retrieve from the helm chart.**

As long the [`secret/postfacto-ext-postgresql`(https://github.com/bgarcial/postfacto-platform/blob/staging/helm/charts/postfacto/templates/pg_secrets.yaml#L4) 
resource is created, and right after the helm chart is deployed, those secrets are readed [from postfacto pods application helm chart](https://github.com/bgarcial/postfacto-platform/blob/staging/helm/charts/postfacto/templates/deployment.yaml#L67-L91)
---

## Postfacto Platform requirements

[A release pipeline](https://dev.azure.com/bgarcial/postfacto-infra/_release?_a=releases&view=mine&definitionId=4) (public project) 
was created in azure DevOps in order to deploy the postfacto applications and fulfill the following platform requirements:

---

1. **Postfacto must be accessible from the internet through HTTPS.**

Cert manager and nginx ingress controller are being used here in order to contact with Let'sEncrypt as an authority certificate
and get back tls certificates for the following domains:
- [postfacto.bgarcial.me](https://postfacto.bgarcial.me/admin/) for latest postfacto app version - 4.3.11 deployment
- [postfacto-4-1-0.bgarcial.me](https://postfacto-4-1-0.bgarcial.me/admin/) for 4.1.0 postfacto app version deployment
- [postfacto-4-2-0.bgarcial.me](https://postfacto-4-2-0.bgarcial.me/admin/) for 4.2.0 postfacto app version deployment

![](https://cldup.com/8NbD0La83a.png)

---

2. To satisfy the QA team, we need to run two versions, 4.1.0 and 4.2.0, of the Postfacto app on the same cluster simultaneously. 


This release pipeline is not triggered in an automatic way via gitops since I want to dedicate special careful when
deploy the different stages created there.

![](https://cldup.com/Y9jXPq9b0J.png)

You can use the following user to sign in to all of the postfacto deployments: 
- username: b.garcia.l@outlook.com  
- password: gabo2020

Specific [docker public images](https://hub.docker.com/r/postfacto/postfacto) for the below versions are being used here

- Postfacto: 4.3.11:  It corresponds to the staging environment, being this the last version.
  
  You can find it here: https://postfacto.bgarcial.me/. And  [the admin dashboard](https://postfacto.bgarcial.me/admin), 


- Postfacto 4.1.0, https://postfacto-4-1-0.bgarcial.me/admin 

- Postfacto 4.2.0. https://postfacto-4-2-0.bgarcial.me/admin

---

3. Postfacto must route requests based on X-POSTFACTO-VERSION HTTP header value, to the pods that contains the correspondent version of the app.

I wanted to check [the way to define custom headers on nginx ingress controller](https://kubernetes.github.io/ingress-nginx/examples/customization/custom-headers/) since it is considered my kind of gateway resource
but I couldn't find the time to do it. It was the only requirement that was missing from my side

