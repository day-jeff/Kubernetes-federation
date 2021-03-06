# Kubernetes-federation
A quick overview on how to federate multiple Kubernetes clusters on Azure

# About the group who wrote this guide

This text was created as a way to capture the learnings of a hacking among four Microsoft employees playing around with kubernetes, specifically with the concepts behind the federation of clusters on the Microsoft Azure cloud platform.

The group got together for some days in Seattle and was composed by:

**Jeff Day** - *Program Manager Architect*, from USA

**Diego Protta Casati** - *Senior Software Engineer*, from Canada

**Michelle Cone** -  *Software Development Engineer*, from USA

**Alessandro Jannuzzi** - *Software Development Engineering Manager*, from Brazil


## A quick look on federating Kubernetes clusters

In a nutshell, you might be interested in federating Kubernetes clusters if you plan to run multi-cloud applications or even multi-datacenter, geo-distributed workloads within a single cloud provider.

Imagine if you have a Deployment specifying two replicas of your pods. When you deploy this yaml file to the context of the federation, one pod would run on the first Kubernetes cluster and its replica would end up running on a second cluster.

The location of the clusters is abstracted from you and in the end, everything your development team sees is a reliable API endpoint where you dispatch the manifest files for your application.

The federation is implemented by a set of components on the so-called **control plane**, installed in one of the Kubernetes cluster of your choice.

Once you have the **control plane** in place, you basically join the member clusters into the federation as participants. If you only have two clusters, the **control plane** can run on *cluster A* and you would want to have *cluster A* joining the federation as a member also, along with *cluster B*.

Then, on your client side where *kubectl* runs, you will need the **.kube/config** files from both clusters. They can be merged into a single file or you can set the environment variable KUBECONFIG in your shell to have the path for both config files.

This way you'll be able to switch contexts between *cluster A*, *cluster B* and *Federation*.

The federation context is where you'll deploy the yaml files of the application that will benefit from the underlying distributed architecture.

Once you do this, you will the a Deployment under the federation contexts and the individual pods under the contexts of clusters A and B.

For this application to be discovered, you will need a DNS service external to the clusters that will be able to host records generated by the federation. In this external DNS you'll have CNAMEs and A records pointing to external load balancers defined in your yaml manifests, for example.

In this example, we will describe our experience on setting up two Kubernetes clusters on the Microsoft Azure cloud platform in different regions and configuring a federation among them. We will use Azure DNS as the external DNS for the federation and describe the hacks that made it work, as Azure is still not natively supported by the federation service as a DNS provider (it will be there soon, we hope!)

## Creating two Kubernetes clusters

The Kubernetes clusters we will use in this example will be installed through the [**Azure Container Services Engine (ACS Engine)**](https://github.com/Azure/acs-engine).

ACS Engine is a tool to generate a fully customized Azure ARM template, describing all the componentes of the Kubernetes clusters to run on Azure.

It is not the same as ACS (Azure Container Services), although ACS uses ACS Engine behind the scenes.

In this case we will get the ACS Engine tool and run it directly on the client side, which already have az (Azure client) installed and configured to the right cloud subscription.

In this example, we're using bash on Windows10, under the fantastic [Linux subsystem for Windows](https://docs.microsoft.com/en-us/windows/wsl/install-win10).

### Generating the ARM template and provisioning it on Azure

[Azure Resource Manager (ARM) templates](https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-group-authoring-templates) are files that describe the components to be deployed to Azure. ACS Engine uses a json as an input parameter file, generates the corresponding ARM template and gets it deployed on Azure.

You should download the latest version of the ACS Engine for your platform from [here](https://github.com/Azure/acs-engine/releases/tag/v0.16.0).

After login with "az login" and setting your default subscription (if you have doubts, check [this](https://docs.microsoft.com/en-us/cli/azure/account?view=azure-cli-latest#az-account-set)), let's prepare a json file to be used with ACS Engine.

The parameters of this json file will dictate how your cluster will be provisioned and configured on Azure. We created this one and saved it as k8.json.

```json
{
"apiVersion": "vlabs",
  "properties": {
    "orchestratorProfile": {
      "orchestratorType": "Kubernetes"
    },
    "masterProfile": {
      "count": 1,
      "dnsPrefix": "westuscluster1",
      "vmSize": "Standard_D2_v2"
    },
    "agentPoolProfiles": [
      {
        "name": "agentpool1",
        "count": 2,
        "vmSize": "Standard_D2_v2",
        "availabilityProfile": "AvailabilitySet"
      }
    ],
    "linuxProfile": {
      "adminUsername": "azureuser",
      "ssh": {
        "publicKeys": [
          {
            "keyData": ""
          }
        ]
      }
    },
    "servicePrincipalProfile": {
      "clientId": "",
      "secret": ""
    }
  }
}
```
Based on this file, *westuscuster1* will be our first Kubernetes cluster, with one master and two agents.

There are other parameters we will not set in this file, but will use directly in the command line of the ACS Engine tool.

We will leave the ```servicePrincipalProfile``` entry empty so the ACS Engine will create it automatically when running.

This *k8.json* file should be informed in the command line for the ACS Engine.

Here is how we've run it.

```
./acs-engine deploy --api-model templates/k8.json --subscription-id <my-subscription-id> --location westtus --private-key-path /home/jannuzzi/.ssh/id_rsa
```

This was the output after finishing successfully.
```
WARN[0024] --resource-group was not specified. Using the DNS prefix from the apimodel as the resource group name: westuscluster1
WARN[0033] apimodel: ServicePrincipalProfile was missing or empty, creating application...
WARN[0043] created application with applicationID (...) and servicePrincipalObjectID (...).
WARN[0044] apimodel: ServicePrincipalProfile was empty, assigning role to application...
INFO[0077] Starting ARM Deployment (westuscluster1-1172685674). This will take some time...
INFO[0475] Finished ARM Deployment (westuscluster1-1172685674). Succeeded
```

While this was running, we copied the *k8.json* file to *k8cluster2.json*, edited the location of the cluster to be in a different datacenter and run it again.

After both commands finished, we had our two kubernetes clusters provisioned on Azure.

## Making sure you can log into the Master node

One of your clusters will be the host cluster for the federation. 
In this case, we selected the *wesuscluster1* to be the host cluster.

We will use *westuscluster1* master node to be the place where kubectl will be used to manage the federation. It can be any other place where kubectl have credentials and works.

Just to make sure we can acess *westuscluster1* master, we connected through ssh using the azureuser private key stored in the *_output* directory after acs-engine finshes its execution.

``` ssh -i ../../_output/westuscluster1/azureuser_rsa azureuser@<Public-IP-Master-Node>```

You can get the public IP of the master node under the Azure Portal.

## Preparing the external DNS configuartion

First thing to do is to create a DNS Zone on Azure.
You can do this through the Portal, it's very intuitive.
We named the zone in this example "syncweek.com".
Once you create the zone, you'll have only the SOA and NS records and that's ok for now.

After creating the zone, you'll need a Service Principle with access to the DNS service.

That's how we created ours.

```az ad sp create-for-rbac -n "http://MyApp" --role contributor --scopes /subscriptions/[your subscription id]/resourceGroups/[your resource group]```

In this phase we followed [this awesome hack](https://github.com/xtophs/k8s-setup-federation-cluster) created by our good friend and Kubernetes ninja, [**Christoph Schittko**](https://github.com/xtophs).

To allow kubernetes create records dynamically into the Azure DNS, you'll need to inform this configuration file when creating the federation. So, you should prepare it accordingly.

```
[Global]
subscription-id = 
tenant-id = [sp tenant]
client-id = [sp appId]
secret = [sp password]
resourceGroup = fed-dns-rg
```
## Preparing the things for the federation

Before creating the federation you should run the command below in the master node of your host cluster, to allow access to the federation services.

```$ kubectl create clusterrolebinding permissive-binding   --clusterrole=cluster-admin   --user=client --user=kubeconfig   --group=system:serviceaccounts```

The command that creates the federation is **kubefed**. You should download it into the master.

```
curl -LO https://storage.googleapis.com/kubernetes-release/release/${RELEASE-VERSION}/kubernetes-client-linux-amd64.tar.gz
```
You should replace RELEASE-VERSION with the version of kubernetes you are running.

In our case, it was 1.8.11. 

**Tricky part: attention to the formation, in this case we had to use "v1.8.11" in the URL.** 

```
root@k8s-master-40902532-0:~# kubectl get nodes
NAME                        STATUS    ROLES     AGE       VERSION
k8s-agentpool1-40902532-0   Ready     agent     1h        v1.8.11
k8s-agentpool1-40902532-1   Ready     agent     1h        v1.8.11
k8s-master-40902532-0       Ready     master    1h        v1.8.11
```

After expanding the files, we copied kubefed to the /usr/local/bin folder.

```
root@k8s-master-40902532-0:~/kubefed# tar xzvf kubernetes-client-linux-amd64.tar.gz
kubernetes/
kubernetes/client/
kubernetes/client/bin/
kubernetes/client/bin/kubefed
kubernetes/client/bin/kubectl
```

The DNS file we prepared before should be in the master node now when creating the federation.

There is one more step before creating the federation.

You should bring to this master server the credentials to acess the other cluster. You can do this by copying the ".kube/config" file from the second cluster to here. 

As we created the clusters with acs-engine, this file is also located in the "_output" directory from where you executed acs-engine.

In our case, we did it the following way:

```
scp -i ../westuscluster1/azureuser_rsa kubeconfig/kubeconfig.eastus.json azureuser@<IP-MASTER-NODE-FIRST-CLUSTER>:/home/azureuser/eastus.config
kubeconfig.eastus.json                                                                      100% 9815     9.6KB/s   00:00
```

Now you need to setup the KUBECONFIG environment variable to be able to switch between contexts (clusters).

```
export KUBECONFIG="/home/azureuser/eastus.config:/home/azureuser/.kube/config"
```

To test it, run the following command. You should be able to see both clusters now.

```
root@k8s-master-40902532-0:~# kubectl config get-contexts
CURRENT   NAME             CLUSTER          AUTHINFO               NAMESPACE
*         eastuscluster1   eastuscluster1   eastuscluster1-admin
          westuscluster1   westuscluster1   westuscluster1-admin
```

## Creating the federation

Now that we have everything in place we can create the federation using **kubefed** command in the following way:

```
kubefed init myfederation --host-cluster-context=westuscluster1  --dns-provider="azure-azuredns" --dns-zone-name="syncweek.com" --dns-provider-config=dns.conf --image=xtoph/hyperkube-amd64:xtoph.azuredns.2
```

You should replace the host cluster name and dns zone with your own information.

As Azure DNS is still in the process of being supported by kubernetes, you should inform the --image parameter above to the image containing the hack created by mr. Christoph to allow the federation services to dynamically update the Azure DNS zone. It is a hacked version of the hyberkube, with patched services.

This was the output of kubefed:

```
Creating a namespace federation-system for federation system components... done
Creating federation control plane service.............................................. done
Creating federation control plane objects (credentials, persistent volume claim)... done
Creating federation component deployments... done
Updating kubeconfig... done
Waiting for federation control plane to come up.............................................................. done
Federation API server is running at: <IP-Address>
```

Now, the command below should include the federation contexts in the output:

```
root@k8s-master-40902532-0:~/kubefed# kubectl config get-contexts
CURRENT   NAME             CLUSTER          AUTHINFO               NAMESPACE
*         eastuscluster1   eastuscluster1   eastuscluster1-admin
          myfederation     myfederation     myfederation
          westuscluster1   westuscluster1   westuscluster1-admin
```

Switch contexts to the federation and create the default namespace.

```
kubectl config use-context myfederation
```

```
 kubectl create ns default
 ```

 Now, lets include both clusters in the federation.

 ```
 kubefed join westuscluster1 --host-cluster-context=westuscluster1 --cluster-context=westuscluster1

 kubefed join eastuscluster1 --host-cluster-context=westuscluster1 --cluster-context=eastuscluster1
 ```

 The following command can confirm if the joins were executed properly:

 ```
 root@k8s-master-40902532-0:~/kubefed# kubectl get clusters
NAME             STATUS    AGE
eastuscluster1   Ready     46s
westuscluster1   Ready     1m
```

## Deploying services into the federation

We used [these beautiful yaml files](https://github.com/CSELATAM/istio-scenarios/blob/master/src/scenarios.yaml) created by another great friend, [**Allan Targino**](https://github.com/allantargino) for another project to test the deployment under the federation.

After downloading the files to my master node, we've run the following command:

```
root@k8s-master-40902532-0:~/yamls# kubectl create -f scenarios.yaml
deployment "calculator" created
service "calculator" created
deployment "frontend" created
service "frontend" created
```

The federation conext doesn't have anything else deployed besides the Deployment of this app. So, it is expected that kubectl get <reource> would return information only if we ask for "deployments".

```
root@k8s-master-40902532-0:~/yamls# kubectl get deployments
NAME         DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
calculator   2         2         0            2           1m
frontend     2         2         0            2           1m
```

The two pods here shoudl be spread across the clusters.

Let's check that with the following command:

```
root@k8s-master-40902532-0:~/yamls# kubectl get rs --context=westuscluster1
NAME                    DESIRED   CURRENT   READY     AGE
calculator-75746d7576   1         1         1         6m
frontend-54f85759c8     1         1         1         6m
```

```
root@k8s-master-40902532-0:~/yamls# kubectl get rs --context=eastuscluster1
NAME                    DESIRED   CURRENT   READY     AGE
calculator-75746d7576   1         1         1         7m
frontend-54f85759c8     1         1         1         7m
```

That's exactly what we wanted: one pod per cluster, composing the desired state of having two replicas.

In order to access this test application from the outside, I created the following file for a load balancer service.

The load balancer will be provisioned by Azure and will have a public IP address so we can reach this nice little app.

Here is the manifest for the load balancer. Save it to a file and send it to kubernetes with kubectl.

```json
 {
      "kind": "Service",
      "apiVersion": "v1",
      "metadata": {
        "name": "calculatorlb"
      },
      "spec": {
        "ports": [{
          "port": 5001,
          "targetPort": 5001
        }],
        "selector": {
          "app": "frontend"
        },
        "type": "LoadBalancer"
      }
    }
```

```
root@k8s-master-40902532-0:~/yamls# kubectl create -f extlb.json
service "calculatorlb" created
```

The provisioning of the external load balancer with a public IP address make take some seconds.

You can follow the progress with the following command:

```
root@k8s-master-40902532-0:~/yamls# kubectl describe service calculatorlb
Name:                     calculatorlb
Namespace:                default
Labels:                   <none>
Annotations:              federation.kubernetes.io/service-ingresses={"items":[{"cluster":"westuscluster1","items":[{"ip":"<IP-ADDR>"}]}]}
Selector:                 app=frontend
Type:                     LoadBalancer
IP:
LoadBalancer Ingress:     <IP-ADDR>
Port:                     <unset>  5001/TCP
TargetPort:               5001/TCP
Endpoints:                <none>
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

Once you see a public IP address in place, open your browser and access the following URL:

http://PUBLIC-IP-ADDR:5001/

You should see the message "Welcome to Calculator service!".

Now you can play with this app just like this:

http://PUBLIC-IP-ADDR:5001/add/10/67

Check it out! Nice little test app! 

By this time, the DNS zone should be populated with A and CNAME records.

If your DNS is reacheble in the Internet, feel free to call your service by name!

## Conclusion, no... Questions to follow up!

Kubernetes federation is an interesting concept but you should be clear on what you're trying to solve by using it.

We think it is important to ask questions like:

* What I am trying to achieve with federation? (avoid vendor lock-in? HA? Manageability?)
* Could I get to the same results using different techniques? If so, how  one solution compares to the other in terms of operations overhead?
* Could I use only my CI/CD pipeline and configure the yaml manifests so I can deploy replicas in different clusters?
* What if I use external DNS load balancers such as [Azure Traffic Manager](https://azure.microsoft.com/en-us/services/traffic-manager/) to load balance bewteen multiple clusters?
* How would I benefit from having a single control place to manage the jointed clusters under the federation versus the effort of dealing with one more abstraction?

These are all valid questions to better define the scope of implementing federation with kubernetes.

But this hack was more an exploration of the possibilities. In this aspect, we are much more clear now on how to do it using ACS Engine on Microsoft Azure. 

We hope it was useful for you! 

