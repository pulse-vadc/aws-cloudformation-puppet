# Brocade vADC + example web server + config by Puppet

## What's in the box

* `vADC-Deploy-Puppet-EIP.template` - main template. Download and deploy from your computer via CLI or CloudFormation UI, or use "Launch Stack" button below.
* `example.pp` - Example parameterised Puppet manifest for the vADC cluster config.
* `QueryWebServers.sh` - script to update the cluster-specific manifest file with the IP addresses of the backend pool servers. It is run periodically to adjust backend pool config if/when necessary.
* `gencerts.sh` - A script to generate self-signed certs and convert them into comma-delimited format suitable for pasting into template / parameter input box.

## What does the template do

At the high level, it builds this:

![Diagram](https://raw.githubusercontent.com/brocade-vadc/aws-cloudformation-puppet/master/images/vADC_with_Puppet_and_Web_Servers.png "High level diagram")

The first part of the template builds the cluster of 2 vADCs. Note:

- vADCs do not use Elastic IPs, because otherwise this template would require more than AWS gives you by default (5 per region). Instead, they are allocated public IPs dynamically.
- The template does not use ENIs for vADCs.
- vADCs get 2 secondary private IPs each, to accommodate the 2 EIPs for our Traffic IP Group.
- The REST API is enabled on the vADCs, to allow Puppet to push the configuration.

The second part of the template is all concerned with two additional parts:

- 2 web servers, representing our "application", started through an AutoScale Group; and
- An EC2 instance with Puppet that (a) pushes the initial config to the vADC cluster, and (b) watches for any changes in Web servers' IPs, for example if an AutoScale event happens, and updates vADC cluster config accordingly.

Since the template creates an Internet-facing "web app", it also creates two EIPs that will be used for a vADC Traffic Group. The vADC cluster will be responsible for keeping the EIPs raised.

It also creates a Route53 DNS zone, by default for domain `corp.local` and creates A records for `www.corp.local` and ALIAS for `corp.local` pointing to the two EIPs. If you would like to access your demo app at http(s)://www.corp.local, you'll need to delegate the NS for corp.local to the AWS Route53 servers associated with your newly created zone. To find out what these are, you'll need to open AWS Console UI, go to Route53, and look for the NS records for your hosted zone (`corp.local` if you used the defaults).

Alternatively you can simply create the A records on your DNS server directly. `Outputs` section in the stack prints out the domain name and public IP addresses that vADCs will be listening on, for example:

```
URL: corp.local, IP1: 52.65.69.185, IP2: 52.62.54.195
```

Public/Private SSL certs included with the template are self-signed for *.corp.local. - please note that they need to be in comma-delimited format. To convert your cert files into comma-delimited format, you can use something similar to the following command:

`awk 1 ORS=',' < myfile.crt > myfile-comma.crt`

You will also notice the template creates a NAT gateway - this is there purely to provide our web servers sitting in the private subnets ability to reach to the internet and install Apache.

## How is this supposed to work

This template is a demo designed to provide building blocks for roughly the following real life workflow:

1. You have an application that requires vADC.
2. You deploy the rest of your application.
3. Using vADC UI and CLI, as necessary, you configure vADC cluster as your application requires.
4. Once the above is in place, you run [`genNodeConfig`](https://forge.puppet.com/tuxinvader/brocadevtm#tools-gennodeconfig) to create a Puppet manifest from your running configured cluster.
5. You then edit the resulting Puppet manifest, replacing the deployment-specific bits inside the manifest, such as IP addresses, DNS names, logins/passwords, SSL certs and so on with mustache-formatted variables, e.g., {{AdminPass}}. See what this looks like in the [example.pp](https://github.com/brocade-vadc/aws-cloudformation-puppet/example.pp) manifest file in this repo. Once done, you upload the resulting parameterised manifest somewhere the `Puppet` resource from your application stack can access it, e.g. Git, S3, etc.
6. Then take the `Puppet`, `vADC1` and `vADC2` resources from this template with their dependencies, add them to yours, adjusting the `/root/example.pp` section of the `Puppet` resource such that it points to the URL of your parametrised manifest, and has the appropriate variables in the `context` section.

For demo / test purposes, you can use this template directly. Make note of the Route53 bit described above.

## Miscellanea

- The original Puppet manifest (as per step 4 above) was generated using the following command, after configuring vADC cluster through the UI:

`/etc/puppet/modules/brocadevtm/bin/genNodeConfig -h 10.8.1.123 -U admin -P Password123 -v 3.9 -sn -o example-raw.pp`

- Template doesn't add a license key or link your cluster to a Services Director.

- Tested with vADC version 11.0, 11.1, and 17.1.

## How to use

You can deploy this template directly through AWS console or CLI, by downloading it to your computer first. All inputs are labelled, and should present no trouble. Once the deployment is complete, connect to your new vADC cluster on the URL displayed in the Output, and login as admin with the password that you've supplied, or "Password123" if you accepted the default. Your example web app should be accessible through HTTP or HTTPS on the domain you've specified (if you've delegated the DNS zone to Route53), or on either of the WebAppIPs in the Outputs section after deployment.

You can also launch this template into us-east-1 region by clicking the "Launch Stack" button below:

<a href="https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=Brocade-vADC-webapp&templateURL=https://s3.amazonaws.com/zxtm-cloudformation/vADC-Deploy-Puppet-EIP.template"><img src=https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png></a>

**Note**: You will need to activate either hourly or annual subscription to Brocade vADC software through the AWS Marketplace before you'll be able to deploy this template successfully. To do this:

1. Visit [AWS Marketplace](https://aws.amazon.com/marketplace/)
2. Type "brocade virtual traffic manager" into the "Search AWS Marketplace" box and click "Go"
3. Click the "Brocade Virtual Traffic Manager Developer Edition" in the results (or select an appropriate different one if you've selected a different SKU)
4. Select **Hourly** or **Annual** in the *"Pricing Details"* box
5. Click "Continue" button
6. On the next screen, first click "Manual Launch" tab away from pre-selected "1-Click Launch", then click yellow "Accept Software Terms" button in the "Price for your selections" box.

## Q&A

**Q**: My deployment sits forever at *"CREATE\_IN\_PROGRESS"* of vADC instances, then fails and rolls back. In the "Status reason" of CloudFormation "Events" log I see a message similar to *"In order to use this AWS Marketplace product you need to accept terms and subscribe. To do so please visit http://aws.amazon.com/marketplace/pp?sku=30zvsq8o1jmbp6jvzis0wfgdt"* against the vADC1 AWS::EC2::Instance.  
**A**: You haven't subscribed to Brocade vADC software through the AWS Marketplace yet. Please visit the link CloudFormation has shown (which may be different from the one above), which will take you to the step #4 in the subscription instructions above the Q&A. Once subscribed, select "Delete Stack", and try deploying again. Please note that there are many SKUs available; so make sure you subscribe to the one that your template is trying to deploy. The easiest way to get to the right one is to visit the URL that CloudFormation tells you in the error message.

**Q**: I don't need the vADC anymore. How can I cancel my subscription?  
**A**: Visit [Your Software Subscriptions](https://aws.amazon.com/marketplace/library/) in AWS Marketplace, and click "Cancel Subscription" against the "Brocade Virtual Traffic Manager Developer Edition", or the appropriate other SKU you may have chosen.

**Q**: I selected a vADC version "100", and deployment has failed. I'm subscribed to the product. I also see something about product being no longer available from Marketplace in the CloudFormation Events log.  
**A**: You are attempting to use an inactive version of the vADC software. Refer to the marketplace for the current supported versions.
