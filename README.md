# Datadog + Ansible

### Brief History of Everything

The systems administrator’s role has been evolving rapidly in the last decade and with it the number of tools that we use has also grown exponentially.  Trying to stay on top of it has ushered the era of both monitoring and configuration management, the latest iterations of which are  Datadog and Ansible.

Datadog is a SaaS monitoring platform for cloud-scale applications that brings together data from infrastructure, applications, databases and services to present a unified view of an entire stack. Datadog can collect data directly from your cloud provider of choice, but its full power lies in its agent and the many application integrations that surround it. Datadog provides an Ansible role that's available in the galaxy and this is what we're using in this project.

Unlike many other configuration management solutions Ansible follows the push method. It does not require an agent on the managed nodes and there's no concept of “master”. Its dependencies are SSH and python 2.7, which are available out-of-the-box on most modern Linux distributions. Its configuration files (playbooks) are in YAML and follow simple top to bottom flow. All this translates to ease of use and quick setup time and has made it a favorite for many SRE teams. Ansible also provides the notion of "roles" which are intended to be used as shippable modules. Think Puppet modules or Chef cookbooks. Often times the work you need to get done, say installing a particular database and configuring its service, has already been done by someone in the open source community and shared out via the Ansible Galaxy (the ansible repository of roles - see https://galaxy.ansible.com/explore#/). This greatly reduces the amount of time you need to get started. The example above, installing a database, is then reduced to installing that particular role (or pulling its repository into your Ansible ./roles directory) and writing a simple playbook to call this role. 

We're now going to demonstrate how easy it is to deploy the Datadog agent with the help of Ansible. We will also see our progress as we do this via the Datadog events stream.

## Installation

Installing Ansible is as easy as `pip install ansible`, but it can also be installed via your distribution's package manager. This is done on the system you're going to be using to push from (control machine). You don't have to install Ansible on the systems you intend to manage. For more installation options on your specific platform see http://docs.ansible.com/ansible/latest/intro_installation.html. Installing Ansible roles is usually achieved with the `ansible-galaxy` command, but here we have the datadog role already included in the repository so that won't be required for this tutorial. 

Our intended use case is deploying Datadog agents with Ansible as well as posting to our Datadog events stream the results of Ansible playbook runs, which we'll achieve by using the datadog callback Ansible plug-in, also included in the repo. This would not only deploy the Datadog agents for us, but also allow us to see successes and failures of any playbook runs within our Datadog Events Stream. To do this, we also want to install the datadog python library with `pip install datadog` and pyyaml with `pip install pyyaml`.

Once you have done that you can clone this example repo with `git clone git@github.com:ncracker/dd_ansible.git`.

The repository contains both a local copy of the Datadog role as well as the plugin that will post events to Datadog, located in `./roles/Datadog.datadog` and `./playbooks/callback_plugins/` respectively. 

## Getting Started
1. First we source our setenv file, which will tell Ansible where our host file lives (the file that contains the nodes we want to manage `./hosts`), where to look for installed roles as well as our ansible configuration file (`./etc/ansible.cfg`)
```
cd dd_ansible_example
source setenv
```
2. We also want to tell Ansible what the FQDNs or IPs of the nodes we want to manage are. In our example we use a single Ubuntu 14 instance as a node. We add the instance fqdn to the ./hosts file with echo
```
echo "yourinstance.fqdn.name" >> ./hosts
```
3. Replace the example Datadog api key in the playbook `./playbooks/dd_agent.yml` with a valid key from your Datadog account - you can find it in https://app.datadoghq.com/account/settings#api. Do the same for `./playbooks/callback_plugins/datadog_callback.yml`, which will allow the plugin to post the results of your playbook runs into your Datadog events stream - here you have 3 ways to set your API key, but I've opted in for the YAML (see the callback plug-in documentation for details listed in the references below). 

4. You're now ready to execute your first Ansible command, but please make sure you can access the node you want to push to via ssh and you've loaded the ssh-key providing you that access (usually done with `ssh-add`). Once you're certain your access works go ahead and run 
```
ansible -m ping -i hosts mynodes
```
You should see the following output, e.g.
```
yourinstance.fqdn.name | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```
This indicated that you were able to successfully run Ansible against the your node.

5. The only thing left to do is run an actual Ansible playbook, which it turn will call the Datadog role and install the dd-agent on your node(s)
```
cd playbooks
ansible-playbook dd_agent.yml
```
The output of the command will give you details of the individual tasks being performed. It will also generate two events in your Datadog events stream, one indicating the a playbook run was started and one for its completion. Any errors will also be displayed if they occur during the run of the playbook.

<img src="https://farm5.staticflickr.com/4760/39598949111_18d4810868_c.jpg" width="800" height="122" alt="Ansible events in Datadog">

This indicates Ansible began the execution of the playbook and completed it. Our datadog agent was successfully deployed.

6. (Optional) To deploy the agent and enable the NGINX datadog integration, which will enable the agent to collect metrics from the NGINX status page, as soon as it's installed, edit `./playbooks/dd_agent_nginx.yml` to match your /nginx_status configuration and run
```
ansible-playbook dd_agent_nginx.yml
```
This assumes you have installed and running NGINX on the target system with running /nginx_status location. See the "How to collect NGINX metrics" in the references below for details.

## How it happened
Opening our example playbook (dd_agent.yml) in your prefered editor will show you how we achieved the agent push.
```
---
- hosts: mynodes
  remote_user: ubuntu
  become: yes
  roles:
      - role: Datadog.datadog
  vars:
    datadog_api_key: <Your API KEY>
    datadog_agent_version: 1:5.20.2-1
```
`- hosts:` defines the group of hosts we're pushing to. This matches the group name "mynodes" we had predefined for you in the `./hosts` file. All you did was to echo the FQDN at the end of the hosts file. 
`remote_user: ubuntu` indicates the username we've told ansible to use for this push. `become: yes` allows the playbook to elevate to sudo if necessary (required for package installs and service reboots). The next two lines tell our playbook to call the datadog role we have in our ./roles/ directory. That's then followed by two variables we pass to the datadog role, namely our api key we placed there earlier as well as the version of the agent we would like to push (Specifying version number is optional, but highly recommended. Examples: 1:5.20.2-1 on apt-based distributions, 5.20.2-1 on yum-based distributions). Less than 10 lines of yaml playbook deployed the datadog agent for us. We could have pushed to many more systems if we had added those to our ansible hosts file.

Something worth pointing out is that you can call multiple roles from a single playbook, thereby installing your database while also deploying the datadog agent. The Datadog ansible role provides more advanced functionality as well - such as loading configuration files for your datadog integrations (like for that database you just deployed). Please refer to the role's documentation in the link below if you want to take advantage of these features.

## The Visuals
A key benefit to monitoring with Datadog, among other things, is the fact that all of the provided integrations are paired with an out-of-the-box dashboard. This is also true for the Ansible callback integration. Assuming you've already run the above ansible commands you can find that dashboard under https://app.datadoghq.com/screen/integration/ansible

You should see something like:
<img src="https://datadog-docs.imgix.net/images/integrations/ansible/ansibledashboard-24481bf9.png?fit=max&w=850?fm=jpg&q=10" alt="Ansible Dashboard">

The Dashboard gives you all the essential information related to your Ansible pushes. The average playbook execution time, percentage of changes resulting from those pushes as well as the individual push events over time. As with all other Datadog default dashboards you can clone it and modify it to further benefit your particular use case.

Another popular usecase is to overlay events triggered by configuration management or CI/CD tools over your time series widgets in dashboards. This is frequently done when troubleshooting and investigating application or infrastructure issues. This enables you to look for correlation between deploys and pushes, and your metrics. We do this by using the "Search events to overlay..." search box in timeboards or the "Add Events" button when edition individual widgets. 

This is an example of Ansible event correlation with the use of the overlay search function:
<img src="https://farm5.staticflickr.com/4713/39567104722_ba9a513ce8_c.jpg" width="800" height="420" alt="correlation">

You'll notice the vertical red bands appearing over all time series widgets, each of them represeting an individual Ansible push event as defined in the search box. The Ansible events stream also appears to the left. As you hoover over the red bands in the timeseries widgets the events stream will scroll to and highlight the corresponding event message. This functionality is invaluable and greatly reduces time to resolution when investigating during an incident.

## Final thoughts
Our example node was running Ubuntu, but you can easily deploy to other distributions. The only changes you would need to make are in `./etc/ansible.cfg` and in `./playbooks/dd_agent.yml` where we've specified the user Ansible is to use and well as the package version (`1:5.17.2-1` for apt-based platforms, use a `5.17.2-1` format on yum-based platforms). Again, please refer to The Official Datadog Ansible role in the references below for more information.

I hope you found the information and example useful. Please do not hesitate to reach out with comments or suggestions.
Package dependencies can also be installed with `pip install -r requirements.txt`.

All the information provided here is purely for reference. Please test extensively before using in production environments.

## References
For more information on both Datadog and Ansible please refer to the sources below
Datadog - https://www.datadoghq.com/
Monitor your automation, automate your monitoring - https://www.datadoghq.com/blog/ansible-datadog-monitor-your-automation-automate-your-monitoring/
The official Datadog Ansible role - https://github.com/DataDog/ansible-datadog (included here in `./roles/datadog`)
Datadog's Ansible callback repo - https://github.com/DataDog/ansible-datadog-callback (included here in `./playbooks/callback_plugins`)
Ansible's official documentation page - http://docs.ansible.com/ansible/latest/index.html
How to collect NGINX metrics - https://www.datadoghq.com/blog/how-to-collect-nginx-metrics/
