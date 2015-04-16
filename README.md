## Simple RIAK Cluster Infrastructure as code

If you're not familiar with Ansible, it is a tool to automate, document and systematise server
provisioning a.k.a Infrastructure as Code. At the basic level, one can write simple tasks in yaml
files and Ansible will run them sequentially via SSH. (by reinteractive.net)

    This is Version 0.2.1 of the RIAK Cluster Infrastructure.

    - Omne initium difficile est.
      Ovid (43 v. Chr. - 17 n. Chr.)

### Prerequisites

In order to use the playbooks and roles provided in this repository you need a resent version
on ansible (>=2.0.0) and some aws credentials with the 'AdministratorAccess' Policy enabled.
If you're on a Mac, this should have you covered:

    $ brew install ansible --HEAD
    ...
    $ export AWS_DEFAULT_REGION="eu-central-1"
    $ export AWS_ACCESS_KEY_ID="XXXXXXXXXXXXXXXXX"
    $ export AWS_SECRET_ACCESS_KEY="YYYYYYYYYYYYY"

### Deploying the cluster

00. Create the aws infrastructure.

The `play_aws-create-infrastructure.yml` playbook adds you default public key
to the aws console and creates two restricted security groups for the
riak-nodes and the riak-clients. Run it like this:

    $ ansible-playbook -i inventories/testing play_aws-create-infrastructure.yml

00. Create the RIAK Cluster Nodes.

The next step creates some (at the moment only 3 are possible) cluster-nodes
to deploy the riak-nodes on. Run it like this:

    $ ansible-playbook -i inventories/testing play_aws-create-nodes.yml \
      --extra-vars group=riak-nodes --extra-vars number=3
    ...
    -> "msg": "52.28.61.5"
    ...
    -> "msg": "52.28.62.43"
    ...
    -> "msg": "52.28.62.44"

When the command finishes please adapt the node ip's at the top of your
inventory file:

    $ cat inventories/testing
    node1   ansible_ssh_host=52.28.61.5  ansible_ssh_user=ubuntu
    node2   ansible_ssh_host=52.28.62.43 ansible_ssh_user=ubuntu
    node3   ansible_ssh_host=52.28.62.44 ansible_ssh_user=ubuntu

00. Create the RIAK Client Nodes.

Now create an client node, to later access the riak cluster. At the moment
only one client is supported. Run it like this:

    $ ansible-playbook -i inventories/testing play_aws-create-nodes.yml \
      --extra-vars group=riak-clients --extra-vars number=1
    ...
    -> "msg": "52.28.61.89"

Also adapt the client ip at the top of your inventory file:

    $ cat inventories/testing
    node1   ansible_ssh_host=52.28.61.5  ansible_ssh_user=ubuntu
    node2   ansible_ssh_host=52.28.62.43 ansible_ssh_user=ubuntu
    node3   ansible_ssh_host=52.28.62.44 ansible_ssh_user=ubuntu
    client1  ansible_ssh_host=52.28.61.89 ansible_ssh_user=ubuntu
    ...

00. Deploy the cluster.

This is the last step but the main on. This step applies the ansible roles
found within the `roles` folder to the aws nodes. The mapping from the roles
to the nodes is done by the group definitions in the inventory file and the
plays in the `play_aws-deploy-cluster.yml` file. Run it like this:

    $ ansible-playbook -i inventories/testing play_aws-deploy-cluster.yml

### Notes & Hints

What happens when you deploy the cluster can be told by looking at the individual roles:

    roles/mhubig.aws:

      This role basicly tags the nodes so they can be easily indentified in the
      aws console. It can also be used to e.g. add the nodes to route53 or
      attach some volumes.

    roles/mhubig.common

      This role takes care of the usual house cleaning on a new server like
      preping the resolf.conf file and installing an ntp server, adding the
      swap volume ...

    roles/mhubig.docker

      This role installes docker and docker-compose, deals with apparmor and
      mounts an extra volume for the docker container, images and volums.

    roles/mhubig.ovs-nodes

      This role downloads, compiles and installes the openvswitch package and
      kernel module. It also creates the internal ans external bridges for the
      VXLAN network to connecting the riak conainers.

    roles/mhubig.riak-nodes

      This role starts the riak-containers (hectcastro/docker-riak-cs) and
      attaches the vlan interface to them.

While debugging and testing this set of ansible playbooks a handy feature are
the tags assosiated with each role and task:

    # Just play the mhubig.riakcs role
    $ ansible-playbook -i inventories/testing play_aws-deploy-cluster.yml \
      --tags mhubig.riakcs

    # Just play the mhubig.riakcs role on node1
    $ ansible-playbook -i inventories/testing play_aws-deploy-cluster.yml \
      --tags mhubig.riakcs --limit node1

    # Just play the the roles/mhubig.common/tasks/ntp.yml task on client1
     $ ansible-playbook -i inventories/testing play_aws-deploy-cluster.yml \
      --tags mhubig.common.ntp --limit client1

### Security

The security of the cluster infrastructure is enforsed by strictly separating the node-groups
based on there position inside the cluster. This separation is done by usind aws security groups
and applying a `DENY_ALL` policy and only the needed ports.

So while the riak-nodes can communicate with each other, there are only reachable from the
outside by the riak-clients on port 8080 and per ssh.

The riak-clients can only be reached on port 443 (HTTPS) and with ssh.

All ssh access is keypair based and there are no passwords involved.

### TODO's and Problems

00. At the Moment the riak-cs cluster is not reachable, do to the the creation
    of the network interface **after** starting the containers/riak-nodes, somehow
    the exported ports are lost ...

00. Ideally there would be an haproxy note between the riak-nodes and the client node.

00. Some of the tasks are not "repeating-save", so the `play_aws-deploy-cluster.yml`
    playbook can not be started easily a second time. To fix this, especially the
    ovs stuff needs some more love ...

00. One way to fix the port issues would be to use the [socketplane][1] docker
    network wrapper. Socketplane basicly starts ovs in a special container and
    allows you to set up the ovs-vxlan-bridge stuff inside the container. You
    can than reuse that network when starting new containers. Using the socketplane
    auto-discovery feature would also be a good way to make this cluster
    infrastructure scaleable.

00. Another tool to look at would be [kubernetes][2] ...

00. Of cource the nodes need backup and monitoring ...

[1]: https://github.com/socketplane/socketplane
[2]: http://kubernetes.io
