## News
Pachyderm v0.8 is out. v0.8 includes a brand new pipelining system. [Read more](pachyderm.io/pps) about it or check out our [web scraper demo](https://medium.com/pachyderm-data/build-your-own-wayback-machine-in-10-lines-of-code-99884b2ff95c)

WE'RE HIRING! Love Docker, Go and distributed systems? Email us at jobs@pachyderm.io

## What is Pachyderm?
Pachyderm is a complete data analytics solution that lets you efficiently store and analyze your data using containers. We offer the scalability and broad functionality of Hadoop, with the ease of use of Docker.

## Key Features
- Complete version control for your data
- Jobs are containerized, so you can use any languages and tools you want
- Both batched and streaming analytics
- One-click deploy on AWS without data migration 

## Is Pachyderm enterprise production ready?
No, Pachyderm is in beta, but can already solve some very meaningful data analytics problems.  [We'd love your help. :)](#how-do-i-hack-on-pfs)

## What is a commit-based file system?
Pfs is implemented as a distributed layer on top of btrfs, the same
copy-on-write file system that powers Docker. Btrfs already offers
[git-like semantics](http://zef.me/6023/who-needs-git-when-you-got-zfs/) on a
single machine; pfs scales these out to an entire cluster. This allows features such as:
- Commit-based history: File systems are generally single-state entities. Pfs,
on the other hand, provides a rich history of every previous state of your
cluster. You can always revert to a prior commit in the event of a
disaster.
- Branching: Thanks to btrfs's copy-on-write semantics, branching is ridiculously
cheap in pfs. Each user can experiment freely in their own branch without
impacting anyone else or the underlying data. Branches can easily be merged back in the main cluster.
- Cloning: Btrfs's send/receive functionality allows pfs to efficiently copy
an entire cluster's worth of data while still maintaining its commit history.

## What are containerized analytics?
Rather than thinking in terms of map or reduce jobs, pps thinks in terms of pipelines expressed within a container. A pipeline is a generic way expressing computation over large datasets and it’s containerized to make it easily portable, isolated, and easy to monitor. In Pachyderm, all analysis runs in containers. You can write them in any language you want and include any libraries. 

### Deploying a Pachyderm cluster
Pachyderm is designed to run on CoreOS so we'll need to deploy a CoreOs cluster. We've created an AWS cloud template to make this insanely easy.
- [Deploy on Amazon EC2](https://console.aws.amazon.com/cloudformation/home?region=us-west-1#/stacks/new?stackName=Pachyderm&templateURL=https:%2F%2Fs3-us-west-1.amazonaws.com%2Fpachyderm-templates%2Ftemplate) using cloud templates (recommended)
- [Amazon EC2](https://coreos.com/docs/running-coreos/cloud-providers/ec2/) (manual)
- [Google Compute Engine](https://coreos.com/docs/running-coreos/cloud-providers/google-compute-engine/) (manual)
- [Vagrant](https://coreos.com/docs/running-coreos/platforms/vagrant/) (requires setting up DNS)

### Deploy Pachyderm manually
If you chose any of the manual options above, you'll neeed to SSH in to one of your new CoreOS machines and start Pachyderm.

```shell
$ curl pachyderm.io/deploy | sh
```
The startup process takes a little while the first time you run it because
each node has to pull a Docker image.

###  Settings
By default the deploy script will create a cluster with 3 shards and 3
replicas. However you can pass it flags to change this behavior:

```shell
$ ./deploy -h
Usage of /go/bin/deploy:
  -container="pachyderm/pfs": The container to use for the deploy.
  -replicas=3: The number of replicas of each shard.
  -shards=3: The number of shards in the deploy.
```

### Integrating with s3
If you'd like to populate your Pachyderm cluster with your own data, [jump ahead](https://github.com/pachyderm/pachyderm#using-pfs) to learn how. If not, we've created a public s3 bucket with chess data for you and we can run the chess pipeline in the full cluster.

As of v0.4 pfs can leverage s3 as a source of data for pipelines. Pfs also
uses s3 as the backend for its local Docker registry. To get s3 working you'll
need to provide pfs with credentials by setting them in etcd like so:

```
etcdctl set /pfs/creds/AWS_ACCESS_KEY_ID <AWS_ACCESS_KEY_ID>
etcdctl set /pfs/creds/AWS_SECRET_ACCESS_KEY <AWS_SECRET_ACCESS_KEY>
etcdctl set /pfs/creds/IMAGE_BUCKET <IMAGE_BUCKET>
```

### Checking the status of your deploy
The easiest way to see what's going on in your cluster is to use `list-units`,
this is what a healthy 3 Node cluster looks like.
```
UNIT                            MACHINE                         ACTIVE          SUB
router.service      8ce43ef5.../10.240.63.167   active  running
router.service      c1ecdd2f.../10.240.66.254   active  running
router.service      e0874908.../10.240.235.196  active  running
shard-0-3:0.service e0874908.../10.240.235.196  active  running
shard-0-3:1.service 8ce43ef5.../10.240.63.167   active  running
shard-0-3:2.service c1ecdd2f.../10.240.66.254   active  running
shard-1-3:0.service c1ecdd2f.../10.240.66.254   active  running
shard-1-3:1.service 8ce43ef5.../10.240.63.167   active  running
shard-1-3:2.service e0874908.../10.240.235.196  active  running
shard-2-3:0.service c1ecdd2f.../10.240.66.254   active  running
shard-2-3:1.service 8ce43ef5.../10.240.63.167   active  running
shard-2-3:2.service e0874908.../10.240.235.196  active  running
storage.service     8ce43ef5.../10.240.63.167   active  exited
storage.service     c1ecdd2f.../10.240.66.254   active  exited
storage.service     e0874908.../10.240.235.196  active  exited
```

## The Pachyderm HTTP API
Pfs exposes a "git-like" interface to the file system -- you can add files and then create commits, branches, etc.

### Creating files
```shell
# Write <file> to <branch>. Branch defaults to "master".
$ curl -XPOST <hostname>/file/<file>?branch=<branch> -T local_file
```

### Reading files
```shell
# Read <file> from <master>.
$ curl <hostname>/file/<file>

# Read all files in a <directory>.
$ curl <hostname>/file/<directory>/*

# Read <file> from <commit>.
$ curl <hostname>/file/<file>?commit=<commit>
```

### Deleting files
```shell
# Delete <file> from <branch>. Branch defaults to "master".
$ curl -XDELETE <hostname>/file/<file>?branch=<branch>
```

### Committing changes
```shell
# Commit dirty changes to <branch>. Defaults to "master".
$ curl -XPOST <hostname>/commit?branch=<branch>

# Getting all commits.
$ curl -XGET <hostname>/commit
```

### Branching
```shell
# Create <branch> from <commit>.
$ curl -XPOST <hostname>/branch?commit=<commit>&branch=<branch>

# Commit to <branch>
$ curl -XPOST <hostname>/commit?branch=<branch>

# Getting all branches.
$ curl -XGET <hostname>/branch
```
##Containerized Analytics

###Creating a new pipeline with a Pachfile

Pipelines are described as Pachfiles. The Pachfile specifies a Docker image, input data, and then analysis logic (run, shuffle, etc). Pachfiles are somewhat analogous to how Docker files specify how to build a Docker image. 

```shell
{
  # Specify the Docker image you want to run your analsis in. You can pull from any registry you want. 
  image <image_name> 
  # Example: image ubuntu
  
  # Specify the input data for your analysis.  
  input <data directory>
  # Example: input my_data/users
  
  # Specify Your analysis logic and the output directory for the results. You can use they keywords `run`, `shuffle` 
  # or any shell commands you want.
  run <output directory>
  run <analysis logic>
  # Example: see the wordcount demo:                   https://github.com/pachyderm/pachyderm/examples/WordCount.md#step-3-create-the-wordcount-pipeline
}
```

###POSTing a Pachfile to pfs

POST a text-based Pachfile with the above format to pfs:

```sh
$ curl -XPOST <hostname>/pipeline/<pipeline_name> -T <name>.Pachfile
```

**NOTE**: POSTing a Pachfile doesn't run the pipeline. It just records the specification of the pipeline in pfs. The pipeline will get run when a commit is made.

### Running a pipeline
Pipelines are only run on a commit. That way you always know exactly the state of
the data that is used in the computation. To run all pipelines, use
the `commit` keyword.

```sh
$ curl -XPOST <hostname>/commit
```

Think of adding pipelines as constructing a
[DAG](http://en.wikipedia.org/wiki/Directed_acyclic_graph) of computations that
you want performed. When you call `/commit`, Pachyderm automatically
schedules the pipelines such that a pipeline isn't run until the pipelines it depends on have
completed.

###Getting the output of a pipelines
Each pipeline records its output in its own read-only file system. You can read the output of the pipeline with:

```sh
$ curl -XGET <hostname>/pipeline/<piplinename>/file/*?commit=<commit>
```
or get just a specific file with:
```sh
$ curl -XGET <hostname>/pipeline/<piplinename>/file/<filename>?commit=<commit>
```

**NOTE**: You don't  need to  specify the commit you want to read from. If you use `$ curl -XGET <hostname>/pipeline/<piplinename>/file/<filename>` Pachyderm will return the most recently completed output of that pipeline. If the current pipeline is still in progress, the command will wait for it to complete before returning. We plan to update this API soon to handle these situations better. 

### Deleting pipelines

```shell
# Delete <pipelinename>
$ curl -XDELETE <hostname>/pipeline/<pipelinename>
```

### Getting the Pachfile

```shell
# Get the Pachfile for <pipelinename>
$ curl -XGET <hostname>/pipeline/<pipelinename>
```

## How do I hack on pfs?
We're hiring! If you like ambitious distributed systems problems and think there should be a better alternative to Hadoop, please reach out.  Email jobs@pachyderm.io

### Want to hack on pfs for fun?
You can run pfs locally using:

```shell
make container-launch
```

This will build a docker image from the working directory, tag it as `pfs` and
launch it locally using `bin/launch`.  The only dependencies are Docker >=
1.5 and btrfs-tools >= 3.14.
