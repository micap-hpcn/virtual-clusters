# AWS CloudFormation template to automatically deploy virtual clusters on AWS Academy Learner Labs

> This template is part of the materials of the HPC in the Cloud subject of the [Master in High Performance Computing](https://estudos.udc.es/en/study/start/4473V02). Although originally it was adapted from the [SLURM plugin for AWS](https://github.com/aws-samples/aws-plugin-for-slurm/tree/plugin-v2), due to the limitations on the permissions of the 
    predefined IAM role `LabRole` in AWS Academy Learner Labs, the `aws-plugin-for-slurm` is no longer used and, as a consequence,
    the cluster is not elastic.

The template deploys a virtual cluster inside an specified VPC using EC2 instances. The cluster is composed of a head node and a configurable number of compute nodes. During the deployment, [Slurm](https://slurm.schedmd.com/) (the popular HPC cluster management system) is downloaded and configured with a single partition `aws` of up to 8 nodes. Also, to run parallel jobs, OpenMPI is installed and NFS is setup to provide a shared directory (`/nfs/mpi`) between the cluster nodes.

## Table of Contents

* [Deployment with AWS CloudFormation](#tc_deployment)
* [Running an MPI program](#tc_mpi)
* [Pausing and resuming nodes](#tc_compute)
* [Destroying the stack](#tc_stack)

<a name="tc_deployment"/>

## Deployment with AWS CloudFormation

> NOTE that this template is configured to be used in AWS Academy Learner Labs. You need an AWS Academy Learner Lab account to deploy virtual clusters using it.

AWS CloudFormation is used to provision the pre-configured head and compute nodes and the necessary network and security configurations. To proceed, from your AWS CloudFormation console create a new stack using the template (`template.yaml`) provided in this repository. You will need to specify several parameters:

* The VPC and subnet where the head node will be launched.
* The type of cluster, HPC or HTC, to be deployed. AWS Academy Learner Labs do not allow to launch instances inside a cluster placement group, so differences in network latency between HPC and HTC virtual clusters are emulated by deploying all compute nodes in the same AZ of the head node (HPC), or  deploying compute nodes in different AZs following a round-robin policy (HTC). The policy can be changed editing the `AvailabilityZones` mapping in the `Mappings` section of the template.
* The instance type to use for the head and compute nodes, and the number of compute nodes.
* An existing EC2 keypair that will be used to SSH to the head node.

The stack creates the following resources:

* A security group that allows SSH traffic from the Internet and traffic between cluster nodes.
* The head node. The stack returns the instance ID of the head node.
* The configured number of compute nodes.

> Note that it is possible to have several stacks deployed at the same time, if more than one virtual cluster is needed.

<a name="tc_mpi"/>

## Running an MPI program

To run an MPI program in a virtual cluster:

1) SSH onto the head node using the specified EC2 keypair (e.g. `ssh -i "mykey.pem" ec2-user@<headnode-public-DNS>.compute.amazonaws.com`)
2) Load the MPI module with `module load mpi`
3) Use the shared location `/nfs/mpi` as your working directory to edit and compile your MPI programs. You can upload an existing code using `scp`, download an existing repository or file in the Internet with `git clone` or `wget`, or edit a new file directly in the head node using, for example, `vim` or `nano`.
4) Run the MPI program in the `aws` partition using a `sbatch` or `salloc` command (e.g. `salloc -p aws -n 2 mpirun <myprogram>`). For more information on how to run MPI jobs with SLURM, you can refer to [here](https://www-lb.open-mpi.org/faq/?category=slurm) and [here](https://support.ceci-hpc.be/doc/_contents/QuickStart/SubmittingJobs/SlurmTutorial.html).

> NOTE that OpenMPI is not configured in the virtual cluster to replace `mpirun` with `srun`, so submitting an MPI program with `srun` will fail. Use `mpirun` instead.

<a name="tc_compute"/>

## Pausing and resuming nodes

Slurm is configured with support for [dynamic nodes](https://slurm.schedmd.com/dynamic_nodes.html). Once deployed, compute nodes can be paused and resumed in the EC2 console as required. As long as the head node is running, the state of running and stopped compute nodes is automatically updated to `idle` or `down`. 

The head node can also be paused. The state of the cluster is recovered when the head node is resumed.

<a name="tc_stack"/>

## Destroying the stack

To terminate all the resources of the virtual cluster, delete the stack in the AWS CloudFormation console.
