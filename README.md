# How to deploy Hyperledger on IBM Container Services

This lab will show you how to deploy your Hyperledger-based business network on IBM Container Services on Bluemix.

IBM Container Services is a hosted container service on IBM's Bluemix Cloud platform, using open source Docker technology. It uses Kubernetes clusters to orchestrate, schedule and manage those containers.

Deploying Hyperledger in this way is fine for development and proof-of-concept work, but it is not suitable for production Blockchain workloads.  For that you can use the **IBM Blockchain Platform**, which is a highly available, secure and resilient managed Blockchain service on IBM Cloud. See [here](https://www.ibm.com/blockchain/platform/) for more details.


## Install the pre-requisites
As you'll be interacting with Kubernetes, IBM Bluemix and Hyperledger Composer in this lab, you need to install three  command line tools at the levels shown (or higher):
- kubectl: v1.7 (the Kubernetes command line tool)
- bx: v0.5 (the Bluemix command line tool).
- Hyperledger Composer CLI: always use the latest version.

Install `kubectl` from https://kubernetes.io/docs/tasks/tools/install-kubectl/ - use the section titled _Install kubectl binary via curl_. You will need to select your OS and follow some basic instructions to download a file, mark it as executable, and move it into position.

Install `bx` from
https://console.bluemix.net/docs/cli/reference/bluemix_cli/index.html#install_bluemix_cli - use the command shown in the _Online installation_ section.

Validate the installations with `kubectl version` and `bx -v`.

Now add the container service plugin - this will let you interact with the IBM Container Service. Add the repo first (this tells the following command where to find the plugin to be installed) - you may get a message saying it already exists, if so that's fine.

```bash
bx plugin repo-add bluemix https://plugins.ng.bluemix.net
bx plugin install container-service -r bluemix
```

Install the Hyperledger Composer CLI tool with
```bash
npm install -g composer-cli
```
> **NB:** don't try to install this with _sudo_, it will cause errors.

## Clone the repository
Clone this git repository to your local machine
```bash
git clone https://github.com/idpattison/hyperledger-ibmcs.git
cd hyperledger-ibmcs
```

## Set up a container cluster
Point the Bluemix CLI at the API endpoint for your Bluemix setup, then login
```bash
bx api api.ng.bluemix.net
bx login
```
This will ask for your userid and the account password.

> **NB:** the API endpoint used is for IBM's US South region. If you need to use another region you will need to replace _ng_ in the API with the code for that region, e.g. _eu-gb_ for the UK.

You will be asked to select an organisation (usually your email address) and a space (call it something like 'blockchain'). If you don't get asked, you can specify them with
```bash
bx target -o <org-name> -s <space-name>
```

If you'd like to you can create a new space with the command `bx iam space-create <space-name>`

Now create the cluster on the IBM Container Service
```bash
bx cs cluster-create --name blockchain
```

This could take up to 30 minutes. You can check progress with
```bash
bx cs clusters
```
Once the _State_ shows _normal_, it's done.  You should see something like this:
```bash
Listing clusters...
OK
Name         ID                                 State    Created                    Workers
blockchain   0783c15e421749a59e2f5b7efdd351d1   normal   2017-05-09T16:13:11+0000   1
```

Once that's done, you can check the status of the worker node:
```bash
bx cs workers blockchain
```
This will show the public and private IP addresses.  Note down the public IP address, as you will use this later to access the Blockchain network.

## Configure kubectl to use the cluster
Issue the following command
```bash
bx cs cluster-config blockchain
```

The output will contain an `EXPORT` command which will point your local `kubectl` to the cluster.  Copy and paste that command into the command line and run it. It will be something like this:
```bash
export KUBECONFIG=/home/*****/.bluemix/plugins/container-service/clusters/blockchain/kube-config-prod-dal10-blockchain.yml
```

## Install the Blockchain network
The _kube-configs_ directory defines a Blockchain implementation which consists of:
- Hyperledger Fabric (2 peers, 2 databases, an orderer and a CA)
- Hyperledger Composer Playground
- Hyperledger Composer Rest Server
- utility containers for creating the keys and certificates
- utility containers for creating and joining a channel
- utility containers for installing and instantiating the test chaincode
- persistent storage to allow containers to share cryptographic material

To deploy the Blockchain:
```bash
cd scripts
./create_all.sh --with-couchdb
```

Once that's complete you can use the Kubernetes Dashboard to explore the services and pods which have been created.  Run
```bash
kubectl proxy
```

Now browse to http://localhost:8001/ui and you will see the dashboard.

You can get all of this information from the command line (try `kubectl get pods -a`), but it's convenient to have it all just a few clicks away.

## Create a local connection profile
We're going to deploy the business network, but to do that we need a connection profile to tell our local Hyperledger Composer CLI where to deploy it.  Local connection profiles are stored in _~/.composer-connection-profiles/_ by default

Create a new connection profile directory for IBM Container Services and copy the example profile.
```bash
mkdir ~/.composer-connection-profiles/ibmcs
cp profile/connection.json ~/.composer-connection-profiles/ibmcs
```

Edit it to use the public IP address of your container cluster - the one from `bx cs workers blockchain`.

## Copy the credentials from the running Hyperledger instance
When the Hyperledger instance was deployed to IBM Container Service, a set of cryptographic credentials was created.  We need to copy some of those off the peer so we can use it locally.

Start by getting the container name of the _Org1_ peer - it will be something like _blockchain-org1peer1-1820571918-bdqrv_.
```bash
kubectl get pods
```

Now extract two files from that peer - these are the certificate and key for the admin user.  _You need to use the container name you found in the previous step_.
```bash
kubectl cp blockchain-org1peer1-xxxxxxxxxx-xxxxx:/shared/crypto-config/peerOrganizations/org1.example.com/users/Admin\@org1.example.com/msp/signcerts/Admin@org1.example.com-cert.pem cert.pem
kubectl cp blockchain-org1peer1-xxxxxxxxxx-xxxxx:/shared/crypto-config/peerOrganizations/org1.example.com/users/Admin\@org1.example.com/msp/keystore/key.pem key.pem
```

Import that identity into the local credential store.  Clear out the store first to remove any old keys.
```bash
rm -rf ~/.composer-credentials/*
composer identity import -p ibmcs -u PeerAdmin -c cert.pem -k key.pem
```

## Create and deploy the business network
An example business network is provided, or you can use your own.  Make sure that _composer-cli_ is at the latest version, or you will get compatibility errors.

```bash
composer archive create -a digital-property.bna -t dir -n business-network/
composer network deploy -a digital-property.bna -p ibmcs -i PeerAdmin -s anything
```

> **NB:** if you want to update an existing business network you need to use `composer network update`:
```bash
composer archive create -a digital-property.bna -t dir -n business-network/
composer network update -a digital-property.bna -p ibmcs -i PeerAdmin -s anything
```


## Deploy the REST server
When the Composer REST server starts, it reads the model information from the business network which is deployed in the Blockchain.  Therefore you can't start it until _after_ you have deployed the business network.

That's the reason we didn't deploy the REST server along with all the other services; we're going to do that now.

Run the following command, adding in your business network name (that's the one defined in the _package.json_ file, not necessarily the file name).
```bash
create/create_composer-rest-server.sh <business-network-name>
```

Examine the running pods (either with the Kubernetes Dashboard or `kubectl get pods`) to see that the REST server has started.  View the logs from the Dashboard (or use `kubectl logs <container-name>`) to ensure that it is serving on port 3000 - this will be exposed externally as port 31090.

Now access the Composer REST server explorer; browse to http://your-ip-address:31090/explorer

**Congratulations!** You've successfully deployed a Hyperledger Fabric Blockchain and a business network to IBM Container Services, and exposed it as an API.


## Cleaning up
You can remove the containers you deployed via Kubernetes with a single script.
```bash
cd scripts
./delete_all.sh
```
