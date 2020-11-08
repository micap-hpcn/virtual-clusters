# AWS CloudFormation Template to automatically deploy virtual clusters on AWS

> The template is part of the materials of the HPCN subject and is based on the [SLURM plugin for AWS](https://github.com/aws-samples/aws-plugin-for-slurm/tree/plugin-v2).

[Slurm](https://slurm.schedmd.com/) is a popular HPC cluster management system. The plugin enables a Slurm head node to dynamically initiate and terminate compute nodes as needed, taking advantage of the elasticity and pay-per-use model of the cloud.

The template deploys a single EC2 instance (the SLURM head node) inside an specified VPC and private subnet. The SLURM stack and the AWS plugin will be downloaded and installed in the head node. To allow running parallel jobs, OpenMPI is installed in all the cluster nodes and NFS is also setup to provide a directory (`/nfs/mpi`) shared by all the nodes.

## Table of Contents

* [Deployment with AWS CloudFormation](#tc_deployment)
* [Running an MPI program](#tc_mpi)
* [Initialization and termination of compute nodes](#tc_compute)
* [Pausing and destroying the stack](#tc_stack)

<a name="tc_deployment"/>

## Deployment with AWS CloudFormation

AWS CloudFormation is used to provision the pre-configured head node on AWS. To proceed, from your AWS CloudFormation console create a new stack using the template (`template.yaml`) provided in this repository. You will need to specify several parameters:

* An existing VPC and subnet, and also to select whether you want to automatically create a placement group, where the head and the compute nodes will be launched.
* The instance type you want to use for head and compute nodes, and the number of vCPUs of the instance type specified for the compute nodes.
* An existing EC2 keypair that will be used to SSH to the head node.

The stack will create the following resources:

* A security group that allows SSH traffic from the Internet and traffic between Slurm nodes
* Two IAM roles to grant necessary permissions to the head and the compute nodes
* A launch template that will be used to launch compute nodes
* A placement group. The creation of this resource is optional. It can be used to launch all the compute nodes inside the same placement group.
* The head node. The stack returns the instance ID of the head node.

The plugin is configured with a single partition `aws` and a single node group `node` that contains up to 8 instances launched in on-demand mode.

Note that it is possible to have several stacks deployed at the same time, if more than one virtual cluster is needed.

<a name="tc_mpi"/>

## Running an MPI program

To run an MPI program in a virtual cluster:

1) SSH onto the head node using the specified EC2 keypair: `ssh -i "mykey.pem" ec2-user@<headnode-public-DNS>.compute.amazonaws.com`
2) Load the MPI module with `module load mpi`
3) Use the shared location `/nfs/mpi` as your working directory to edit and compile your MPI programs. You can upload an existing code using `scp`, download an existing repository or file in the Internet with `git clone` or `wget`, or edit a new file directly in the headnode using, for example, `vim` or `nano`.
4) Run the MPI program using a `sbatch` or `sallocate` command to the `aws` partition. For example, if you run `sallocate -p aws -n 2 mpirun <myprogram>`, you should see in the EC2 console a new instance (`aws-nodeX`) being automatically launched. For more information on how to run MPI jobs with SLURM, you can refer to [here](https://www-lb.open-mpi.org/faq/?category=slurm) and [here](https://support.ceci-hpc.be/doc/_contents/QuickStart/SubmittingJobs/SlurmTutorial.html).

> BE AWARE that in the virtual cluster OpenMPI is not configured to replace `mpirun` with `srun`, so submitting an MPI program with `srun` will fail. Use `mpirun` instead.

<a name="tc_compute"/>

## Initialization and termination of compute nodes

While the head node is running, it automatically initiates and terminates compute nodes as needed. Idle compute nodes are automatically terminated after a pre-configured time (around 5 minutes). They will be initiated again when required by new job executions.

> BE AWARE that compute nodes will not be automatically terminated if the head node is paused or terminated. In that case, you must manually terminate them in the EC2 console.

<a name="tc_stack"/>

## Pausing and destroying the stack

To avoid being charged for the execution time of the head node when you are not using the virtual cluster, you can manually pause the head node in the EC2 console and resume it later to continue using the virtual cluster. Note that you will still be charged by the storage capacity used by the EBS volume of the instance, but it will be a small amount.

To terminate all the resources of the virtual cluster, delete the stack in the AWS CloudFormation console.

> BE AWARE that destroying the stack does not automatically terminate the compute nodes, if any are running. In that case, you can wait for them to terminate automatically before destroying the stack or terminate them manually in the EC2 console after the stack is destroyed.
