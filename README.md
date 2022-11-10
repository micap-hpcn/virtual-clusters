# AWS CloudFormation template to automatically deploy virtual clusters on AWS Academy Learner Labs

> This template is part of the materials of the HPC in the Cloud subject of the [Master in High Performance Computing](https://estudos.udc.es/en/study/start/4473V02). It has been adapted from the [SLURM plugin for AWS](https://github.com/aws-samples/aws-plugin-for-slurm/tree/plugin-v2) to be used in AWS Academy Learner Labs.

[Slurm](https://slurm.schedmd.com/) is a popular HPC cluster management system. The plugin enables a Slurm head node to dynamically initiate and terminate compute nodes as needed, taking advantage of the elasticity and pay-per-use model of the cloud.

The template deploys a single EC2 instance (the SLURM head node) inside an specified VPC and private subnet. The SLURM stack and the AWS plugin are downloaded and installed in the head node. To allow running parallel jobs, OpenMPI is installed in all the cluster nodes and NFS is also setup to provide a shared directory (`/nfs/mpi`) between the cluster nodes.

## Table of Contents

* [Deployment with AWS CloudFormation](#tc_deployment)
* [Running an MPI program](#tc_mpi)
* [Initialization and termination of compute nodes](#tc_compute)
* [Pausing and destroying the stack](#tc_stack)

<a name="tc_deployment"/>

## Deployment with AWS CloudFormation

> BE AWARE that this template is configured to be used in AWS Academy Learner Labs. You need an AWS Academy Learner Lab account to deploy virtual clusters using it.

AWS CloudFormation is used to provision the pre-configured head node and the necessary network and security configurations. To proceed, from your AWS CloudFormation console create a new stack using the template (`template.yaml`) provided in this repository. You will need to specify several parameters:

* Whether or not to create IAM policies for the cluster nodes. These policies HAVE TO BE CREATED ONLY ONCE, the first time the template is used.
* An existing VPC and subnet where the head node will be launched.
* A list of subnets where the compute nodes will be launched. Currently, AWS Academy Learner Labs do not allow to launch instances inside a cluster placement group, so differences in network latency between HPC and HTC virtual clusters will be emulated by selecting only one or several -all available is recommended- subnets, respectively.
* The instance type you want to use for the head and compute nodes, and the number of vCPUs of the instance type selected for the compute nodes.
* An existing EC2 keypair that will be used to SSH to the head node.

The stack will create the following resources:

* Two IAM policies to grant the necessary permissions to the head and compute nodes. These policies are attached as inline policies to the predefined IAM role `LabRole`, which is generally available in AWS Academy Learner Labs. Due to the limitations of IAM in these labs, it is required to manually enable the creation of these policies ONLY THE FIRST TIME the template is used.
* A security group that allows SSH traffic from the Internet and traffic between Slurm nodes.
* A launch template that will be used to launch compute nodes.
* The head node. The stack returns the instance ID of the head node.

The plugin is configured with a single partition `aws` and a node group `<stack>NodeZone#` for each of the subnets selected to launch compute nodes on demand. There is a total limit of 8 instances that are distributed between the node groups. For example, if only one subnet is selected, there will be a node group `<stack>nodeZone1` with 8 nodes. But, if all subnets in the default VPC are selected (currently there are six in the North Virginia region), there will be six node groups with names `<stack>NodeZone1` to `<stack>NodeZone6` with 1 or 2 instances each, up to a total of 8.

Note that it is possible to have several stacks deployed at the same time, if more than one virtual cluster is needed.

<a name="tc_mpi"/>

## Running an MPI program

To run an MPI program in a virtual cluster:

1) SSH onto the head node using the specified EC2 keypair: `ssh -i "mykey.pem" ec2-user@<headnode-public-DNS>.compute.amazonaws.com`
2) Load the MPI module with `module load mpi`
3) Use the shared location `/nfs/mpi` as your working directory to edit and compile your MPI programs. You can upload an existing code using `scp`, download an existing repository or file in the Internet with `git clone` or `wget`, or edit a new file directly in the headnode using, for example, `vim` or `nano`.
4) Run the MPI program using a `sbatch` or `salloc` command to the `aws` partition. For example, if you have selected the `c5.large` instance type for compute nodes, when running `salloc -p aws -n 2 mpirun <myprogram>` you should see a new instance (`aws-<stack>NodeZone#-#`) being automatically launched in the EC2 console. For more information on how to run MPI jobs with SLURM, you can refer to [here](https://www-lb.open-mpi.org/faq/?category=slurm) and [here](https://support.ceci-hpc.be/doc/_contents/QuickStart/SubmittingJobs/SlurmTutorial.html).

> BE AWARE that in the virtual cluster OpenMPI is not configured to replace `mpirun` with `srun`, so submitting an MPI program with `srun` will fail. Use `mpirun` instead.

<a name="tc_compute"/>

## Initialization and termination of compute nodes

While the head node is running, it automatically initiates and terminates compute nodes as needed. Idle compute nodes are automatically terminated after a pre-configured time (around 5 minutes). They will be initiated again on demand when required by new job executions.

> BE AWARE that compute nodes will not be automatically terminated if the head node is paused or terminated. In that case, you must manually terminate them in the EC2 console.

<a name="tc_stack"/>

## Pausing and destroying the stack

To avoid being charged for the execution time of the head node when you are not using the virtual cluster, you can manually pause the head node in the EC2 console and resume it later to continue using the virtual cluster. Note that you will still be charged by the storage capacity used by the EBS volume of the instance, but it is a small amount.

To terminate all the resources of the virtual cluster, delete the stack in the AWS CloudFormation console.

> BE AWARE that destroying the stack does not automatically terminate the compute nodes, if any is running. In that case, you can wait for them to terminate automatically before destroying the stack or terminate them manually in the EC2 console after the stack is destroyed.
