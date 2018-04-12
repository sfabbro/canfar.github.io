---
layout: pages_left_nav

namespace: docs.quick_start
lang: en
permalink: /en/docs/quick_start/
---
# Quick Start: Build a star detection pipeline

This quick start guide will demonstrate how to:

* create, configure, and interact with OpenStack cloud and CANFAR Virtual Machines (VMs)
* store files with the CANFAR VOSpace storage
* launch batch processing jobs from the CANFAR batch host, using VMs created in the previous step

## Resource Allocation

Before starting this example, you will need to have a CADC account ([register](http://apps.canfar.net/canfar/auth/request.html)) and you will need to request a CANFAR resource allocation. The CANFAR team will take care of your registration to Compute Canada infrastructure.
Send an email to [CANFAR support](mailto:support@canfar.net) and include your CADC account name, a rough amount of required resources (storage capacity and processin capabilitiesg), and a a few sentences of justification.  Your request will be reviewed and you will be contacted by our support group.

Once registered, make note of your:
* ```[CADC username]```: your CADC username
* ```[VOSpace]```: the short name of the VOSpace you have access to
* ```[tenant]```: the OpenStack tenant name you have access to 

## Create your interactive Virtual Machine

To access and manage your VMs with OpenStack, we suggest using the  dashboard at Compute Canada. [Log into the dashboard](https://west.cloud.computecanada.ca). Provide your ```[CADC username]```, adding a ```-canfar``` suffix, e.g, ```janesmith-canfar```, and your usual CADC password.

Each resource allocation corresponds to an OpenStack **tenant** or **project** (these two names are used interchangeably). A user may be a member of multiple tenants, and a tenant usually has multiple users. A pull-down menu near the top-left allows you to select between the different tenants that you are a member of.

### Allow ssh access to your VM

* Click on **Access & Security** (left column of page), and then the **Security Groups** tab.
* Click on the **Manage Rules** button next to the default group. If you see a rule with **Ingress** direction, **22(SSH)** Port Range and **0.0.0.0/0 (CIDR)** , then that means someone in your tenant already  opened the ssh port for you. If you don't see it, add a new rule following step.
* Click on the **+ Add Rule** button near the top-right. Select **SSH** at the bottom of the **Rule** pull-down menu, and then click on **Add** at the bottom-right. **This operation is only required once for the initial setup of the tenant**.

### Import an ssh public key

Access to VMs is facilitated by SSH key pairs rather than less secure user name / password. A private key resides on your own computer, and the public key is copied to all machines that you wish to connect to.

* If you have not yet created a key pair on your system, run **ssh-keygen** on this local machine to generate one or follow this [documentation](https://help.github.com/articles/generating-ssh-keys/) for example.
* Click on **Access & Security**, switch to the **Key Pairs** tab and click on the **Import Key Pair** button at the top-right.
* Choose a meaningful name for the key, and then copy and paste the contents of ```~/.ssh/id_rsa.pub``` from the machine you plan to ssh from into the **Public Key** window.

### Allocate a public IP address

You will need to connect to your VM via a public IP.

* Click on the **Floating IPs** tab. If there are no IPs listed, click on the **Allocate IP to Project** button at the top-right.

Each tenant/project will typically have one public IP. If you have already allocated all of your IPs, this button will read "Quota Exceeded".

### Launch a VM

We will now launch a VM with Ubuntu 16.04. We support two distributions: Ubuntu LTS and CentOS VMs.

* Switch to the **Images** window (left-hand column), and then click on the **Public** button at top right (it might be already selected).
* Select ```canfar-ubuntu-16.04``` and click on the **Launch Instance** button on the right.
* In the **Details** tab choose a meaningful **Instance Name**. **Flavor** is the hardware profile for the VM. ```c2-7.5gb-80``` provides the minimal requirements of 2 cores, 7.5GB or RAM for most VMs. Note that it provides an 80 GB *ephemeral disk* that will be used as scratch space for batch processing. **Availability Zone** should be left empty, and **Instance Count** 1.
* In the **Access & Security** tab ensure that your public key is selected, and the ```default``` security group (with ssh rule added) is selected.
* Finally, click the **Launch** button.

### Connect to the VM

After launching the VM you are returned to the **Instances** window. You can check the VM status once booted by clicking on its name (the **Console** tab of this window provides a basic console in which you can monitor the boot process).
Before being able to ssh to your instance, you will need to attach the public IP address to it.

* Select **Associate Floating IP** from the **More** pull-down menu.
* Select the address that was allocated and the new VM instance in the **Port to be associated** menu, and click on **Associate**.

Your ssh public key will have been injected into a **generic account** with a name like ```centos```, or ```ubuntu```, depending on the Linux distribution.

<div class="shell">

{% highlight bash %}
ssh ubuntu@[floating ip]
{% endhighlight %}

</div>

### Install software on the VM

The default operating system has only a minimal set of packages. For this example, we will use:

* the [SExtractor](http://www.astromatic.net/software/sextractor) package to create catalogues of stars and galaxies.
* We also need to read FITS images. Most FITS images from CADC come Rice-compressed with an `fz` extension. SExtractor only reads uncompressed images, so we also need the ```funpack``` utility to uncompress the incoming data. The ```funpack``` executable is included in the package ```libcfitsio-bin``` in Ubuntu.

Let's install them both system wide after a fresh update of the Ubuntu repositories:

<div class="shell">

{% highlight bash %}
sudo apt update -y
sudo apt install -y sextractor libcfitsio-bin
{% endhighlight %}

</div>

### Test the Software on the VM

We are now ready to do a simple test. Let’s download a FITS image to our scratch space. When we instantiated the VM we chose a flavour with an ephemeral disk, which was automatically mounted on ```/mnt```. You want to access the ephemeral disk with a non-root user and create a scratch directory to mimic the batch processing environment (note that this will be done automatically for batch jobs):

<div class="shell">

{% highlight bash %}
sudo mkdir /mnt/scratch
sudo chmod 777 /mnt/scratch
{% endhighlight %}

</div>

Next, enter the directory, copy an astronomical image there, and run SExtractor on it:

<div class="shell">

{% highlight bash %}
cd /mnt/scratch
cp /usr/share/sextractor/default* .
echo 'NUMBER
MAG_AUTO
X_IMAGE
Y_IMAGE' > default.param
curl -L http://www.canfar.phys.uvic.ca/data/pub/CFHT/1056213p | funpack -O 1056213p.fits -
sextractor 1056213p.fits -CATALOG_NAME 1056213p.cat
{% endhighlight %}

</div>

The image `1056213p.fits` is a Multi-Extension FITS file with 36 extensions, each containing data from one CCD from the CFHT Megacam camera.

## Store results on the CANFAR VOSpace

All data stored on the the ephemeral disk since the last time it was saved are normally **wiped out** when the VM shuts down). We will use the [CANFAR VOSpace](/docs/storage/) to store the result.
We want to store the output catalogue `1056213p.cat` at a persistent, externally-accessible location. We will use the CANFAR VOSpace for this purpose. To store anything on the CANFAR VOSpace from a command line, you will need the CANFAR VOSpace client which is already installed.

For an automated procedure to access VOSpace on your behalf, your proxy authorization must be present on the VM. You can accomplish this using a `.netrc` file that contains your CANFAR user name and password, and the command **getCert** can obtain an *X509 Proxy Certificate* using that username/password combination without any further user interaction. The commands below will create the file.

<div class="shell">

{% highlight bash %}
canfar_dotnetrc
{% endhighlight %}

</div>

Let's check that the VOSpace client works by copying the results to your VOSpace:

<div class="shell">

{% highlight bash %}
vcp 1056213p.cat vos:[VOSpace]
{% endhighlight %}

</div>

Verify that the file is properly uploaded by pointing your browser to the [VOSpace browser interface](http://www.canfar.phys.uvic.ca/vosui).

If you do not need batch processing, you can snapshot your VM (see below) and you are done. If you need batch, read along.

### Install HTCondor for Batch

Batch jobs are launched using a scheduler software  called [HTCondor](http://www.htcondor.org). HTCondor will dynamically launch jobs on the VMs (workers), connecting to the batch processing head node (the central manager). Your worker VM needs a HTCondor agent to connect the central manager. Run this script which will install HTCondor and tune the network on your VM instance:

<div class="shell">

{% highlight bash %}
sudo canfar_batch_prepare
{% endhighlight %}

</div>

### Snapshot (save) the VM Instance

Now you want to save the VM with your software. Return to your browser on the OpenStack dashboard.

* Save the state of your VM by navigating to the **Instances** window of the dashboard, and click on the **Create Snapshot** button to the right of your VM instance's name. After selecting a name for the snapshot of the VM instance, e.g., ```my_vm_image```, click the **Create Snapshot** button. It will eventually be saved and listed in the VM **Images** window, and will be available next time you launch an instance of that VM image.
While the system is taking a snapshot of your VM, avoid doing anything on your VM.

## Automate all the above and run it in batch

Now we want to automate the whole procedure as a batch script. Log into the batch head node:

<div class="shell">

{% highlight bash %}
ssh [CADC username]@batch.canfar.net
{% endhighlight %}

</div>

Paste the following commands into one BASH script named ```~/do_catalog.bash```:

<div class="shell">

{% highlight bash %}
#!/bin/bash
id=$1
curl -L http://www.canfar.phys.uvic.ca/data/pub/CFHT/${id} | funpack -O ${id}.fits -
cp /usr/share/sextractor/default* .
echo 'NUMBER
MAG_AUTO
X_IMAGE
Y_IMAGE' > default.param
sextractor ${id}.fits -CATALOG_NAME ${id}.cat
vcp ${id}.cat vos:[VOSpace]
{% endhighlight %}

</div>

Remember to substitute ```[VOSpace]``` with your CANFAR VOSPace name that you requested, often this is your ```[CADC username]```.

### Write your batch processing job submission

Let's write a submission file that will transfer the `do_catalog.bash` script to the execution host. The execution host will be an instance of your snapshot VM image with 4 cores, and for each given CADC CFHT file id, will run a job on one of the core. The job will consist of running your script for 4 CFHT images with the file IDs 1056215p, 1056216p, 1056217p, and 1056218p. For this tutorial you will modify the configuration file listed below. Fire up your favorite editor and paste the following text into a submission file:

<div class="shell">

{% highlight text %}

Executable = do_catalog.bash

Arguments = 1056215p
Log = 1056215p.log
Output = 1056215p.out
Error = 1056215p.err
Queue

Arguments = 1056216p
Log = 1056216p.log
Output = 1056216p.out
Error = 1056216p.err
Queue

Arguments = 1056217p
Log = 1056217p.log
Output = 1056217p.out
Error = 1056217p.err
Queue

Arguments = 1056218p
Log = 1056218p.log
Output = 1056218p.out
Error = 1056218p.err
Queue
{% endhighlight %}

</div>

Save the submission file as `myjobs.sub`.

### Submit the batch jobs

Source the OpenStack RC tenant file, and enter your CANFAR password. This sets environment variables used by OpenStack (only required once per login session):

<div class="shell">

{% highlight bash %}
. canfar-[tenant]-openrc.sh
{% endhighlight %}

<code class="output">
Please enter your OpenStack Password:
</code>
</div>

You can then submit your jobs to the condor job pool:

<div class="shell">

{% highlight bash %}
canfar_submit myjobs.sub my_vm_image c4-15gb-83
{% endhighlight %}

</div>

```my_vm_image``` is the name of the snapshot you used during the VM configuration above, and ```c4-15gb-83``` is the flavor for the VM(s) that will execute the jobs. If you wish to use a different flavor, they are visible through the dashboard when [launching an instance](#launch-a-vm-instance), or using the [nova command-line client](../cli/#launch-the-instance): ```nova flavor-list```.

After submitting, wait a couple of minutes. Check where your jobs stand in the queue:

<div class="shell">

{% highlight bash %}
condor_q [CADC username]
{% endhighlight %}

</div>

Check the status of the VMs and jobs running on the cloud:

<div class="shell">

{% highlight bash %}
condor_status -submitter
{% endhighlight %}

</div>

Once you have no more jobs in the queue, check the logs and output files `myjobs.*` on the batch host, and check on your VOSpace browser. All 4 of the generated catalogues should have been uploaded.

## Helpful CANFAR commands and VM maintenance

Once your VM is built, the responsibility is yours to maintain and update software. There are a few scripts you can run, all of them come with a ```--help```:

* ```canfar_batch_prepare```: prepare a VM for batch by installing HTCondor and some network tuning
* ```canfar_setup_scratch```: simple script to authorize a scratch space on ```/mnt/scratch```
* ```canfar_create_user [username]```: create a user with ```[username]``` on the VM and propage ssh public key.
* ```canfar_cert -u [CADC username]```: try hard to get a CANFAR proxy certificate by various methods
* ```canfar_dotnetrc```: update / create a ```${HOME}/.netrc```` file to help with VOSpace access
* ```canfar_update```: update all the CANFAR scripts and VOSpace clients

You can find all these commands on ```/usr/local/bin```, or at the [GitHub](https://github.com/canfar/canfarproc/tree/master/worker/bin) source.


If you do need your VM anymore, you can terminate it from the OpenStack dashboard.
You are done!



