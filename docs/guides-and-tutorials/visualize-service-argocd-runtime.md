---
sidebar_position: 5
sidebar_label: Visualize services' k8s runtime using ArgoCD
---

import Tabs from "@theme/Tabs"
import TabItem from "@theme/TabItem"
import PortTooltip from "/src/components/tooltip/tooltip.jsx"

# Visualize your services' Kubernetes runtime using ArgoCD

This guide takes 10 minutes to complete, and aims to demonstrate the value of Port's integration with ArgoCD.

:::tip Prerequisites

- This guide assumes you have a Port account and that you have finished the [onboarding process](/quickstart). We will use the `Service` blueprint that was created during the onboarding process.
- You will need an accessible k8s cluster. If you don't have one, here is how to quickly set-up a [minikube cluster](https://minikube.sigs.k8s.io/docs/start/).
- [Helm](https://helm.sh/docs/intro/install/) - required to install Port's Kubernetes exporter.

:::

<br/>

### The goal of this guide

In this guide we will model and visualize our services' Kubernetes resources (via ArgoCD).

After completing it, you will get a sense of how it can benefit different personas in your organization:

- Developers will be able to easily view the health and status of their services' K8s runtime.
- Platform engineers will be able to create custom views and visualizations for different stakeholders in the organization.
- Platform engineers will be able to set, maintain and track standards for K8s resources.
- R&D managers will be able to track any data about services' K8s resources, using tailor-made views and dashboards.

<br/>

### Install Port's ArgoCD integration

1. Go to your [data sources page](https://app.getport.io/dev-portal/data-sources), click on `+ Data source`, find the `Kubernetes Stack` category and select `ArgoCD`.

2. Copy the installation command, it should look like the snippet below, with your Port `client ID` and `secret` filled in.  
   Replace the following placeholders with your own values:  
   - `integration.secrets.token` - Your [ArgoCD API token](https://argo-cd.readthedocs.io/en/stable/developer-guide/api-docs/#authorization).
   - `integration.config.serverUrl` - The server url of your ArgoCD instance.

```bash showLineNumbers
# The following script will install an Ocean integration in your K8s cluster using helm
helm repo add --force-update port-labs https://port-labs.github.io/helm-charts
helm upgrade --install argocd port-labs/port-ocean \
	--set port.clientId="<CLIENT_ID>"  \
	--set port.clientSecret="<CLIENT_SECRET>"  \
	--set port.baseUrl="https://api.getport.io"  \
	--set initializePortResources=true  \
	--set integration.identifier="argocd"  \
	--set integration.type="argocd"  \
	--set integration.eventListener.type="POLLING"  \
	--set integration.secrets.token="Enter value here"  \
	--set integration.config.serverUrl="Enter value here" 
```

#### What does the integration do?

After installation, the integration will:

1. Create <PortTooltip id="blueprint">blueprints</PortTooltip> in your [Builder](https://app.getport.io/dev-portal/data-model) (as defined [here](https://github.com/port-labs/ocean/blob/main/integrations/argocd/.port/resources/blueprints.json)) that represent ArgoCD resources:

<img src='/img/guides/argoBlueprintsCreated.png' width='100%' border='1px' />

<br/><br/>

<!-- :::info Note

`Workload` is an abstraction of Kubernetes objects which create and manage pods (e.g. `Deployment`, `StatefulSet`, `DaemonSet`).

::: -->

2. Create <PortTooltip id="entity">entities</PortTooltip> in your [Software catalog](https://app.getport.io/services). You will see a new page for each <PortTooltip id="blueprint">blueprint</PortTooltip> containing your resources, filled with data from your Kubernetes cluster (according to the default mapping that is defined [here](https://github.com/port-labs/ocean/blob/main/integrations/argocd/.port/resources/port-app-config.yaml)):

<img src='/img/guides/argoEntitiesCreated.png' width='100%' border='1px' />

<br/><br/>

3. Create a [dashboard](https://app.getport.io/argocdDashboard) in your software catalog that provides you with some visual views of the data ingested from your K8s cluster.

4. Listen to changes in your ArgoCD resources and update your <PortTooltip id="entity">entities</PortTooltip> accordingly.

<br/>

### Define the connection between services and workloads

Now that we have our <PortTooltip id="blueprint">blueprints</PortTooltip> set up, we want to model the logical connection between our ArgoCD resources and the `Service` blueprint that already exists in our builder. This will grant us some helpful context in our Software catalog, allowing us to see relevant ArgoCD application/s in a `Service`'s context, or an application's property directly in its corresponding `Service`.

In this guide we will create one relation named `Prod_runtime` which will represent the production environment of a service. In a real-world setting, we could have another relation for our staging environment, for example.

1. Go to your [Builder](https://app.getport.io/dev-portal/data-model), expand the `Service` blueprint, and click on `New relation`.

2. Fill out the form like this, then click `Create`:

<img src='/img/guides/argoCreateRelation.png' width='50%' border='1px' />

<br/><br/>

When looking at a `Service`, some of its `Running service` properties may be especially important to us, and we would like to see them directly in the `Service's` context. This can be achieved using [mirror properties](https://docs.getport.io/build-your-software-catalog/customize-integrations/configure-data-model/setup-blueprint/properties/mirror-property/), so let's create one:

3. Let's mirror the application's health. Under the relation we just created, click on `New mirror property`:

<img src='/img/guides/argoCreateMirrorProp.png' width='50%' border='1px' />

<br/><br/>

4. Fill the form out like this, then click `Create`:

<img src='/img/guides/argoCreateMirrorPropHealth.png' width='50%' border='1px' />

<br/><br/>

### Map your workloads to their services

You may have noticed that the `Prod_runtime` property and the mirror property we created are empty for all of our `services`. This is because we haven't specified which `Running service` belongs to which `service`. This can be done manually, or via mapping by using a convention of your choice.

In this guide we will use the following convention:  
An Argo application with a label in the form of `portService: <service-identifier>` will automatically be assigned to a `service` with that identifier.

For example, an ArgoCD application with the label `portService: awesomeService` will be assigned to a `service` with the identifier `awesomeService`.

To achieve this, we need to update the ArgoCD integration's mapping configuration:

1. Go to your [data sources page](https://app.getport.io/dev-portal/data-sources), find the ArgoCD exporter card, click on it and you will see a YAML editor showing the current configuration.  
Add the following block to the mapping configuration and click `Resync`:

```yaml showLineNumbers
- kind: application
  selector:
    query: 'true'
  port:
    entity:
      mappings:
        identifier: .metadata.labels.portService
        blueprint: '"service"'
        properties: {}
        relations:
          prod_runtime: .metadata.uid
```

<br/>

2. Go to the [services page](https://app.getport.io/services) of your software catalog. Click on the `Service` for which you created the deployment, and you should see the `Prod_runtime` property filled, along with the `Health` property that we mirrored:

<img src='/img/guides/argoEntityAfterIngestion.png' width='100%' border='1px' />

<br/>

### Visualize data from your Kubernetes environment

We now have a lot of data about our Argo applications, and a dashboard that visualizes it in ways that will benefit the routines of our developers and managers. Since we connected our ArgoCD application(`Running service`) blueprint to our `Service` blueprint, we can now access some of the application's data directly in the context of the service.  
Let's see an example of how we can add useful visualizations to our dashboard:

#### Show all "degraded" production services belonging to a specific team

1. Go to your [ArgoCD dashboard](https://app.getport.io/argocdDashboard), click on the `+ Add` button in the top right corner, then select `Table`.

2. Fill the form out like this, then click `Save`:

    <img src='/img/guides/argoTableDegradedServicesForm.png' width='50%' border='1px' />

3. In your new table, click on the `Filter` icon, then on `+ Add new filter`.  
   Add two filters by filling out the fields like this:

    <img src='/img/guides/argoTableFilterDegraded.png' width='80%' border='1px' />

4. Your table should now display all services belonging to your specified team, whose `Health` is `Degraded`:

    <img src='/img/guides/argoTableDegradedServices.png' width='90%' border='1px' />

### Possible daily routine integrations

- Send a slack message in the R&D channel to let everyone know that a new deployment was created.
- Notify Devops engineers when a service's availability drops.
- Send a weekly/monthly report to R&D managers displaying the health of services' production runtime.

### Conclusion

Kubernetes is a complex environment that requires high-quality observability. Port's ArgoCD integration allows you to easily model and visualize your ArgoCD & Kubernetes resources, and integrate them into your daily routine.  
Customize your views to display the data that matters to you, grouped or filtered by teams, namespaces, or any other criteria.  