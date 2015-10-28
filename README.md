#Using MODAClouds CloudML and Puppet for Multi-Cloud Deployments

This report describes how [BOC](https://www.boc-group.com)  makes use of MODAClouds [CloudML](http://cloudml.org/)  together with [Puppet](https://puppetlabs.com/puppet/puppet-open-source)  to deploy their applicatins in a multi-cloud environment. As such it may be useful for anybody considering (multi-)cloud deployments with the [MODAclouds toolchain](http://multiclouddevops.com/technologies.html#MODACloudsToolbox)  extended with the state-of-the-art configuration management system Puppet.

**Some background on BOC** – being in the market for modelling software tools and consultancy services for 20 years, BOC has gained an excellent expertise in supporting enterprises in the management domains for business processes, enterprise architecture and IT services, risk control and governance. With the BOC management office a well-integrated suite of software tools supporting management of the assets mentioned above. With the cloud service ADONIS:cloud BOC has now also entered the market of software provided as fully managed service to clients all around the world.

**Some background on CloudML** – as part of the MODAClouds toolbox it is a the component that takes design time models form the Creator 4Clouds IDE and communicates with the target cloud platform APIs in order to have the required infrastructure (IaaS) and platform (IaaS) environments provisioned. On these cloud resources it performs further steps in order to have all application components present and running and all their dependencies fulfilled. Among others one of the possible methods to achieve component installation, configuration and start-up is by exploiting CloudML’s Puppet extension.

**Some background on Puppet** – together with tools like Chef, CFEngine and Ansible, Puppet it is among the market leaders for configuration management tools. The open-source based software stack implements a concept that allows the user to describe the desired state of a server or virtual machine in terms of so called manifests written in Puppet domain specific language (DSL). It takes care of reaching this state by applying module manifests that build on a set of fundamental resource types (such as file, package, service, user, ...) to have software packages installed and configured. There is a large community of contributors maintaining such modules for popular open-source software packages, like for example Apache httpd, MariaDB, HAproxy. Besides that anybody can implement their own modules for software not yet covered by community modules and that is exactly what BOC is doing for their proprietary software products such as ADONIS and ADOit.

###The deployment approach
Being committed to efficient processes not only in their customer projects but also with respect to their internal procedures BOC has put strong focus on automating deployment and monitoring tasks in their SaaS environment. They have built up their platform based on Puppet which lets them deploy application stacks on existing servers with a few clicks. In the course of their participation in the EU funded FP7 project MODAClouds they have extended this approach so that provisioning the required servers in an IaaS provider independent way is now taken care of by the CloudML component which also integrates with the Puppet layer to which it delegates the software component management. [Figure 1](#Figure1) shows the basic interaction between the components involved in the deployment procedure targeting two different IaaS providers in a multi-cloud setup.

<a name="Figure1"></a>![Figure 1: Interaction steps of a deployment procedure](https://raw.githubusercontent.com/boc-modaclouds/cloudml-with-puppet/master/img/interaction.png)

The procedure consists of the following steps enacted by the CloudML component:
1.	The deployment model annotated with Puppet Resources is uploaded to Models@Runtime.
2.	The IaaS providers’ APIs are instructed to create new virtual machines based on a tailored operating system image.
3.	Once the virtual machines are available, CloudML clones the manifests repository ([Mercurial](https://www.mercurial-scm.org/)) from the Puppet Master Server, adds the manifests for the components to be deployed and pushes the changes back to the Puppet Master.
4.	Finally it instructs the newly created virtual machines to run their puppet agents so that they pick up the latest configuration and execute the required commands to reach the desired state.

###Preparations
In order to be able to use this procedure a couple of preparation tasks must be accomplished. 

First the Puppet Master server needs to be prepared. This is a quite simple machine that has the puppet master applications installed and running. Corresponding packages are available as [RPM and DEB packages  form Puppetlabs](https://docs.puppetlabs.com/guides/puppetlabs_package_repositories.html). For every user interacting with this machine a dedicated user account with SSH public key authentication shall be created. This account shall own a mercurial repository of node manifests that is included from the main site manifest. These repositories shall be configured in a way that any push to them also updates the working copy. Please refer to the [Mercurial documentation](https://www.selenic.com/mercurial/hgrc.5.html) for details.

Next the MS Windows based CloudML workstation shall be prepared with the latest distribution of CloudML from the [open-source repository](https://github.com/SINTEF-9012/cloudml/releases). The service required from the suite is the websocket component which provides the Models@Runtime (M@R) functionality. While a single deployment can also be executed with the CloudML shell, the M@R approach is better suited for continuous adaptation of the actually deployed environment. With the HTML/JavaScript based web editor shipped with the suite, a well usable graphical interface to the M@R backend is available.

Finally operating system images need to be provided at the relevant IaaS accounts that are tailored in a way that an administrative user account is present that can be accessed from M@R via SSH or Powershell in order to trigger the execution of the Puppet agent.

##Actual deployment by example
This recipe will present what it takes in order successfully deploy an application instance (a sample Java web application in this case) with the procedure depicted in [Figure 1](#Figure1).

###Step 1 – upload the deployment model to M@R
Starting from a point where the preparation steps for the Puppet Master and the CloudML workstation described above have been completed, the deployment model is the first essential artefact that is needed. This is best obtained from the [Creator 4Clouds IDE](http://forge.modelio.org/projects/creator-4clouds) that supports graphical modelling of a component architecture which can be taken through various transformation steps in order to receive a deployment description that is understood by CloudML. One aspect necessary for the Puppet based method is that all _Internal Components_ must be annotated with so-called _Puppet Resources_ that carry the information needed to assemble the Puppet manifests for the involved servers, see an example JSON fragment below.

```javascript
{
	"eClass": "net.cloudml.core:InternalComponent",
	"name": "ODBC Connection",
	"puppetResources": [{
		"eClass": "net.cloudml.core:PuppetResource",
		"name": "puppet_odbc_dsn",
		"requireCredentials": false,
		"executeLocally": false,
		"masterEndpoint": "192.168.100.4",
		"repositoryEndpoint": "ssh://boc@192.168.100.4//etc/puppet/manifests//boc-nodes",
		"username": "boc",
		"manifestEntry": "modaclouds::odbc_dsn { &apos;acloud30&apos; : db_host => &apos;192.168.100.22&apos;}"
	}],
	"providedPorts": [{
		"eClass": "net.cloudml.core:ProvidedPort",
		"name": "ODBCProvided",
		"isLocal": false,
		"portNumber": "0",
		"component": "internalComponents[ODBC Connection]"
	}],
	"requiredExecutionPlatform": {
		"eClass": "net.cloudml.core:RequiredExecutionPlatform",
		"name": "OSRequired5",
		"owner": "internalComponents[ODBC Connection]",
		"demands": [{
			"eClass": "net.cloudml.core:Property",
			"name": "Windows",
			"value": "true"
		}]
	}
}
```

The deployment JSON representation shows the descriptor of an ODBC connection that is needed for accessing an SQL Server database. The essential element in the component description is the _PuppetResources_ array that specifies which resources need to be declared in the resulting manifest in order to have the ODBC connection in place. The Puppet DSL snippet must be provided in the _manifestEntry_ property. For the communication with the Puppet Master node the properties _masterEndpoint_ (Puppet Master IP address) and _repositoryEndpoint_ (the connection string for the mercurial client) are mandatory. Note that using the SSH protocol together with public key authentication is the preferred method for the interaction with the repositories on the Puppet Master.

To upload the Puppet enriched deployment model to M@R you need to direct your web browser to the CloudML web editor and connect it to the websocket server (M@R) which also resides on the CloudML workstation, typically listening on TCP port 9030. Then, using the _File_ menu entry _“LOAD A DEPLOYMENT MODEL”_ upload the JSON deployment descriptor into the web application. In order to have the VM provisioned, the Puppet Master re-configured and software components deployed, use the _“DEPLOY ...”_ entry in the _Server_ menu of the CloudML web editor.

![Connecting the Web Editor to M@R](https://raw.githubusercontent.com/boc-modaclouds/cloudml-with-puppet/master/img/connect.png)
![Connection dialog](https://raw.githubusercontent.com/boc-modaclouds/cloudml-with-puppet/master/img/connect_dlg.png)

###Steps 2-4 – infrastructure provisioning and software deployment
When M@R receives the command to deploy the model that has been uploaded in the first step, it starts by communicating with involved cloud APIs. This communication requires authentication with credentials that are authorized to enact infrastructure provisioning in the target cloud. These credentials have to be stored on the CloudML workstation in a properties file with the syntax shown below. The credentials properties file is referenced from the Provider object in the deployment JSON. 
```
login=proj-42@my-company.com
passwd=some#SECRET!
```

M@R triggers the creation of a storage drive based on the image that has been created as a preparation step. The base image is referenced from the imageId attribute of the corresponding VM object in the deployment JSON. Once the drive is available the virtual machine is created according to the specification in the model.

M@R then connects to the machine with the user account that has been created during the preparation of the base image. The credentials need to be provided in the login and passwd properties of the VM object inside the deployment JSON. The host name of the newly created machine is set so that it matches the node directive of the Puppet manifest as this is where the association between a server and its manifest is established.

Setting the host name requires a restart of the machine and after this has been completed M@R reconnects again to it and makes sure that the Puppet agent software is installed and properly configured so that it can connect to the Puppet Master. Finally the Puppet Agent is invoked which requests its catalogue from the Puppet Master and performs all required steps needed to reach the server state described in the catalogue. The actions to be performed are taken from the Puppet modules that need to be available for the resources declared in the Puppet manifest.

