## Simple RIAK Cluster Infrastructure as Code

If you're not familiar with Ansible, it is a tool to automate, document and
systematise server provisioning a.k.a Infrastructure as Code. At the basic
level, one can write simple tasks in yaml files and Ansible will run them
sequentially via SSH. (by reinteractive.net)

    This is Version 0.1.0 of the RIAK Cluster Infrastructure.

    - Omne initium difficile est.
      Ovid (43 v. Chr. - 17 n. Chr.)

### Prerequisites

In order to use the playbooks and roles provided in this repository you need
a resent version on ansible (>=1.8.2) and some aws credentials.

    $ brew install ansible --HEAD
    ...
    $ export AWS_DEFAULT_REGION="eu-central-1"
    $ export AWS_ACCESS_KEY_ID="XXXXXXXXXXXXXXXXX"
    $ export AWS_SECRET_ACCESS_KEY="YYYYYYYYYYYYY"

### Deploying the cluster

00. Create the aws infrastructur.

    $ ansible-playbook -i inventories/testing play_aws-create-infrastructur.yml

00. Create the RIAK Cluster Nodes.

    $ ansible-playbook -i inventories/testing play_aws-create-nodes.yml \
      --extra-vars group=riak-nodes --extra-vars number=3
    ...
    -> "msg": "52.28.61.5"
    ...
    -> "msg": "52.28.62.43"
    ...
    -> "msg": "52.28.62.44"

00. Create the RIAK Client Nodes.

    $ ansible-playbook -i inventories/testing play_aws-create-nodes.yml \
      --extra-vars group=riak-clients --extra-vars number=1
    ...
    -> "msg": "52.28.61.89"

00. Add the IP's off all the newly created nodes to your inverntory file.

    $ cat inventories/testing
    node1   ansible_ssh_host=52.28.61.5  ansible_ssh_user=ubuntu
    node2   ansible_ssh_host=52.28.62.43 ansible_ssh_user=ubuntu
    node3   ansible_ssh_host=52.28.62.44 ansible_ssh_user=ubuntu
    client1  ansible_ssh_host=52.28.61.89 ansible_ssh_user=ubuntu
    ...

00. Deploy the cluster.
    $ ansible-playbook -i inventories/testing play_aws-deploy-cluster.yml


