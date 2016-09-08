---
layout: post
title: SMRT-Link Integration with Slurm
tags: [linux, PacBio, Slurm]
---

[ACPFG](http://www.acpfg.com.au) recently purchased, as part of a mini-consortium, a PacBio [Sequel](http://www.pacb.com/products-and-services/pacbio-systems/sequel/)
machine. Prior to the installation taking place, we were required to have a small compute cluster setup and working as per
their site preparation guide. This is where the secondary data analysis takes place. The primary analysis (QC metrics, base-calling,
generating BAM files etc) is done in real-time on the hardware within the SEquel machine's enclosure. The specifications they provided
for the secondary analysis are based on whole Human genome sequencing and assembly.

Given we work on wheat, which has a genome size of
16Gbp (5x larger than the 3.2Gbp Human genome) it isn't so clear what the compute requirements might be. We didn't want to go out and
buy hardware in a hurry, only to find out we were an order of magnitude out! We do however operate a small [Slurm](http://slurm.schedmd.com/)
cluster which I hoped we could integrate into SMRT-Link. This would offer the possibility of plugging in new hardware over time as needed.

* Placeholder for Table of Contents (TOC) - this text will be scraped
{:toc}

<h2>SMRT-Link Support for Job Management Systems</h2>

PacBio only support [SGE](https://en.wikipedia.org/wiki/Oracle_Grid_Engine), [PBS](https://en.wikipedia.org/wiki/Portable_Batch_System) and
[LSF](https://en.wikipedia.org/wiki/Platform_LSF) Job Management Systems (JMS). The first time you see evidence of this is when running the
SMRT-Link installer which has a `--jmstype` command line argument with the following valid values: `SGE`, `PBS`, `LSF` and `NONE`. The
latter is for when you just want the server running SMRT-Link to also do the number crunching.

To figure out how to integrate SMRT-Link with Slurm, I had hoped to learn more about how SMRT-Link integrates with the other JMS' by just
specifying `SGE`, `PBS` or `LSF` during the install or reconfiguration of SMRT-Link. However, specifying any of these results in further
JMS-specific prompts which are followed by some validation steps. This makes sense, as you'd want to validate that SMRT-Link can talk to
the configured JMS. However, this means you can't simply force the install of one of these in an attempt to figure out what files are
created/modified for the JMS integration.

<h2>Slurm Integration</h2>

Here an overview of how I integrated [Slurm](http://slurm.schedmd.com/) into SMRT-Link:

  1. Install SMRT-Link
  2. Setup Slurm-specific template files
  3. Enable distribution mode for SMRT-Pipe
  4. Start the SMRT-Link services

<h3>Install SMRT-Link</h3>

Since our Slurm cluster is already up and running, we spun up a Virtual Machine (VM) to run the SMRT-Link services/web portal and performed
all our SMRT-Link install from there. Since this host will simply act as a web server, it doesn't need to be large in terms of resources.

The location into which SMRT-Linx is installed needs to be accessible (at the same path) to the Slurm nodes as well as the host which will
run the SMRT-Link services/web portal. This means some sort of shared/clustered filesystem like NFS - we're using the Oracle Cluster File
System 2 ([OCFS2](https://oss.oracle.com/projects/ocfs2/)).

We will install SMRT-Link into `/opt/shared_apps/pacbio/smrtlink` (this is accessible to our Slurm nodes using the same path) and the
user/group which will run our SMRT-Link services will be `smrtanalysis:smrtanalysis`. The version of SMRT-Link I performed the original
installation with is `3.1.0.180439`.

```bash
export SMRTLINK_VERSION=3.1.0.180439

# Setup some variables for use during installation/configuration
# ${SMRT_ROOT} needs to be accessible to the Slurm nodes at the same path
export SMRT_ROOT=/opt/shared_apps/pacbio/smrtlink
export SMRT_USER=smrtanalysis
export SMRT_GROUP=smrtanalysis

# This is the location where the Sequel instrument has been configured to rsync data into
# It also needs to be accessible to the Slurm nodes at the same path
export SMRT_DATA_ROOT='/mnt/sequel/data'
# This is where SMRT-Link job output goes
# It also needs to be accessible to the Slurm nodes at the same path
export SMRT_JOBS_ROOT='/mnt/sequel/jobs'
# This is local temporary space and needs to be fast but not shared between Slurm nodes
export SMRT_TMP_DIR='/tmp/smrtlink'

# Create the parent directory of the installation location
mkdir -p "${SMRT_ROOT%/*}"
chown "${SMRT_USER}:${SMRT_GROUP}" "${SMRT_ROOT%/*}"

# Change to the user who will be running the SMRT-Link services
su "${SMRT_USER}"
cd

#####
# Obtain the SMRT-Link installer from PacBio
#####

# Decompress the installer
tar xzf smrtlink_${SMRTLINK_VERSION}.tar.gz

#####
# Run the installer
#####
./smrtlink_${SMRTLINK_VERSION}/smrtlink_${SMRTLINK_VERSION}.run \
    --skip-hwcheck \
    --dnsname $(hostname -f) \
    --rootdir "${SMRT_ROOT}" \
    --smrtlink-ldap-disable \
    --dataroot ${SMRT_DATA_ROOT} --jobsroot ${SMRT_JOBS_ROOT} --tmpdir ${SMRT_TMP_DIR} \
    --jmstype NONE \
      --chunking true \
      --total-nproc 1000 \
      --nproc 9 \
      --maxchunks 16 \
      --nworkers 50
```

A few comments on some of the arguments I've shown:

  * I skip the hardware check (`--skip-hwcheck`) as SMRT-Link will complain my measly VM isn't up to scratch. I'll up the resources at 
    a later date if needs be.
  * `--dnsname` is the hostname of the machine through which users will access the SMRT-Link web portal. I'm just grabbing the
    Fully Qualified Domain Name (FQDN) of the host running this install, as it will be the one eventually running the SMRT-Link
    services.
  * Don't forget `--rootdir`, `--dataroot` and `--jobsroot` all need to be accessible from the Slurm nodes at the same location.
  * `--jmstype` is set to `NONE`, we'll make post-install modifications to integrate Slurm later.

<h3>Setup Slurm-Specific Template Files</h3>

Following the above install, SMRT-Link is configured (but not yet running) to perform all the analysis on the host which performed the
install. We want to change this, so that analysis are sent to our Slurm cluster.

During an installation of SMRT-Link with `--jmstype` set to `SGE`, `PBS` or `LSF` some JMS-specific template files are generated.
These files are used by SMRT-Pipe for starting and stopping jobs. For the supported
JMS', these template files will actually involve a call to a `runjmscmd` executable. It is `runjmscmd` that is responsible for
constructing the JMS-specific commands for starting and stopping jobs. Since SMRT-Link doesn't support Slurm, and hence `runjmscmd` doesn't
know how to construct a Slurm start/stop job command, we'll place the actual Slurm commands into these template files.

Lets start with the "start" template file:

```bash
#####
# Thing to customise:
#  * --partition, this is the Slurm partition on which you want SMRT-Link jobs run.
#    Here, I have specified the "unlimited" partition on my Slurm cluster
#  * Full paths to the Slurm commands "salloc" and "srun". This is on the host which
#    will run the SMRT-Link services/web portal.
#####
# NOTES:
#  * Variables used within the heredoc will be entered in the file as is. Since this is
#    a template file, SMRT-Link will parse this file, substituting in the values for known
#    JMS environmental variables.
#####
cat << "START_TMPL" > "${SMRT_ROOT}/userdata/generated/config/jms_templates/start.tmpl"
/opt/shared_apps/slurm/slurm-2-6-5-1/bin/salloc \
  --job-name="${JOB_ID}" \
  --partition=unlimited \
  --nodes=1 \
  --cpus-per-task=${NPROC} \
  /opt/shared_apps/slurm/slurm-2-6-5-1/bin/srun \
    --cpus-per-task=${NPROC} \
    --ntasks=1 \
    -o ${STDOUT_FILE} \
    -e ${STDERR_FILE} \
    /bin/bash -c "${CMD}"
START_TMPL
```

Next, lets create the "stop" template file:

```bash
#####
# Thing to customise:
#  * Full paths to the Slurm command "scancel". This is on the host which
#    will run the SMRT-Link services/web portal.
#####
cat << "STOP_TMPL" > "${SMRT_ROOT}/userdata/generated/config/jms_templates/stop.tmpl"
/opt/shared_apps/slurm/slurm-2-6-5-1/bin/scancel \
  ${JOB_ID}
STOP_TMPL
```

<h3>Enable Distribution Mode for SMRT-Pipe</h3>

SMRT-Pipe uses an XML file (`${SMRT_ROOT}/userdata/config/preset.xml`) for some of it's settings. This includes
whether it should run in distribution mode to run jobs on a configured JMS. Since we installed with
`--jmstype NONE`, distribution mode is correctly disabled. Enable it by changing the corresponding value from
`false` to `true`:

```bash
sed -i '/"pbsmrtpipe.options.distributed_mode"/{n; s/False/True/}' "${SMRT_ROOT}/userdata/config/preset.xml"
```

<h3>Start the SMRT-Link services</h3>

Once you start the SMRT-Link services, SMRT-Link will try to submit jobs to the Slurm cluster. This of course means
this host need to be configured as a Slurm Node, but not a member of any Slurm partitions (you probably don't want
Slurm to execute jobs on this host). This means having the host defined as a `NODE` in your `slurm.conf` on all nodes
of your Slurm cluster.

Once your host is configured as a node in your Slurm cluster, start the SMRT-Link services and run the Site
Acceptance Test (SAT):

```bash
# Start services
"${SMRT_ROOT}/admin/bin/services-start"
"${SMRT_ROOT}/admin/bin/services-status"

# Validate Install
"${SMRT_ROOT}/admin/bin/import-canneddata"
"${SMRT_ROOT}/admin/bin/run-sat-services"
```

<h2>Sources Used</h2>

There is some information out there on JMS integration with SMRT-Analysis (a similar product for the RSII), but it is often fragmented,
scattered and quite old. Here are a few I found useful:

  * [https://github.com/PacificBiosciences/SMRT-Analysis/issues/78]()
  * [https://github.com/PacificBiosciences/SMRT-Analysis/wiki/Distributed-Computing-Configuration]()
  * [https://github.com/PacificBiosciences/SMRT-Link/wiki/Distributed-Computing-Configuration]()
