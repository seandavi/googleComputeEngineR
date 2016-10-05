# googleComputeEngineR
An R interface to the Google Cloud Compute Engine API, for launching virtual machines.

[![CRAN](http://www.r-pkg.org/badges/version/googleComputeEngineR)](http://cran.r-project.org/package=googleComputeEngineR)
[![Build Status](https://travis-ci.org/cloudyr/googleComputeEngineR.png?branch=master)](https://travis-ci.org/cloudyr/googleComputeEngineR)
[![Coverage Status](https://img.shields.io/codecov/c/github/cloudyr/googleComputeEngineR/master.svg)](https://codecov.io/github/cloudyr/googleComputeEngineR?branch=master)

## TL;DR

1. Configure a Google Cloud Project with billing
2. Download a service acount key JSON file
3. Put your default project, zone and JSON file location in your `.Renviron`
4. Run `library(googleComputeEngineR)` and auto-authenticate
5. Run `vm <- gce_vm_template("rstudio", name = "rstudio-server", predefined_type = "f1-micro", username = "mark", password = "mark1234")` (or other credentials) to start up an RStudio Server.
6. Wait for it to install, login via the returned URL.

## Thanks to:

Scott Chamberlin for the [analogsea](https://github.com/sckott/analogsea) package for launching Digital Ocean VMs, which inspired the SSH connector functions for this one.

## Install

Its not on CRAN yet:

```r
devtools::install_github("MarkEdmondson1234/googleComputeEngineR")
```

## Setup

Google Compute Engine lets you create and run virtual machines on Google infrastructure.  See the [documentation here](https://cloud.google.com/compute/docs/).

Before you begin, you will need to set up a Cloud Platform project, and enable billing by adding a credit card.  

A quickstart guide is [available here](https://cloud.google.com/compute/docs/quickstart-linux) for making your first VM via the web interface, if you are not familiar with GCE then its best to start there.

Pricing is [available here](https://cloud.google.com/compute/pricing).

For `googleComputeEngineR` you will need:

* Your project ID e.g. `my-project-name`
* Your preferred geographical zone to launch VMs e.g. `europe-west1-a`
* [Optional] A `Service account key` json file, downloaded from the API Manager > Credentials > Create credentials > Service account key > Key type = JSON
* [Optional] If not using service account key, you will need the `client-id` and `client-secret` for your project.

> The recommended method to authenticate is using the JSON service account key via auto-authentication.

## Authentication

### OAuth2

Authentication can be carried out via OAuth2 each session via `gce_auth()`.  The first time you run this you will be sent to a Google login prompt in your browser to allow the `googleAuthR` project access (or preferably the Google project you configure via client ID). 

Once authenticated a file named `.httr-oauth` is saved to your working directory.  On subsequent authentication this file will hold your authentication details, and you won't need to go via the browser.  Deleting this file, or setting `new_user=TRUE` will start the authentication flow again.

```r
## set your project ID and secret if not using service account JSON
options(googleAuthR.client_id = YOUR_CLIENT_ID)
options(googleAuthR.client_secret = YOUR_CLIENT_SECRET)

library(googleComputeEngineR)
## first time this will send you to the browser to authenticate
gce_auth()

## to authenticate with a fresh user, delete .httr-oauth or run with new_user=TRUE
gce_auth(new_user = TRUE)

...call functions...etc...

```

Each new R session will need to run `gce_auth()` to authenticate future API calls.

### Auto-authentication

Alternatively, you can specify the location of a service account JSON file taken from your Google Project, or the location of a previously created `.httr-oauth` token in a system environment:

        Sys.setenv("GCE_AUTH_FILE" = "/fullpath/to/auth.json")

You can set default projects, zone and authentication via an `.Renviron` file or `Sys.setenv()`.

Example entries:

```
GCE_AUTH_FILE="/Users/mark/xxxxx/auth.json"
GCE_DEFAULT_PROJECT="mark-xxxxxxx"
GCE_DEFAULT_ZONE="europe-west1-a"
```

This file will then used for authentication via `gce_auth()` when you load the library:

```r
## GCE_AUTH_FILE set so auto-authentication
> library(googleComputeEngineR)
Successfully authenticated via /Users/mark/auth.json
Set default project name to 'mark-xxxxx'
Set default zone to 'europe-west1-a'

## no need for gce_auth()
> gce_get_project()
$kind
[1] "compute#project"

$id
[1] "43534234234324324"

$creationTimestamp
[1] "2015-05-08T15:22:38.416-07:00"

$name
[1] "mark-xxxxx"

...etc.... 
```

## Launch a Virtual Machine

To launch an existing VM, use `gce_vm_start()`. It returns a job operation object with a name, which you can check with `gce_get_zone_op(job$name)`, or use `gce_check_zone_op(job$name)` for it to retry and return when it has finished.

```r
library(googleComputeEngineR)

## auto auth, project and zone pre-set

## list your VMs in the project/zone
the_list <- gce_list_instances()

## start an existing instance
job <- gce_vm_start("markdev")
  
## check the job status until its finished
gce_check_zone_op(job$name)
  
## get the instance metadata
inst <- gce_get_instance("markdev")
inst$status
[1] "RUNNING"
``` 

You can also reset and start instances/VMs:

```r  
## reset instance
job <- gce_vm_reset("markdev")
  
## check job until its finished
gce_check_zone_op(job$name, wait = 20)
  
## stop VM
job <- gce_vm_stop("markdev")
  
## check job until finished
gce_check_zone_op(job$name, wait = 20)
  
inst <- gce_get_instance("markdev")
inst$status
[1] "TERMINATED"  
```

### External IP

You can view the external IP for an instance via `gce_get_external_ip()`

```r
> ip <- gce_get_external_ip("xxxxx")
 External IP for instance xxxxxx  :  146.1xx.24.xx 
```

## Creating an instance

To create an instance you need to specify:

* Name
* Project [if not default]
* Zone [if not default]
* Machine type - either a predefined type or custom CPU and memory
* Network - usually default, specifies open ports etc.
* Image - a source disk image containing the operating system, that may come from another image project or a snapshot

### Default settings

The default settings let you create a VM like so:

```r
## create a VM
> vm <- gce_vm_create(name = "test-vm")

## pause until operation is done
> gce_check_zone_op(vml$name, wait = 20)

## see VM created
> gce_get_instance("test-vm")
$kind
[1] "compute#instance"

$id
[1] "7425434871478204241"

$creationTimestamp
[1] "2016-09-28T11:19:42.826-07:00"

$name
[1] "test-vm"

..etc..
```

The defaults are:

* `predefined_type = "f1-micro"`
* `image_project = "debian-cloud"`
* `image_family = "debian-8"`
* `network = "default"`

### Custom settings

You can examine different options via the various list commands:

#### Machine type

A list of the predefined machine types:
```r
gce_list_machinetype()
```

#### Images

A list of the image projects and families available is here: `https://cloud.google.com/compute/docs/images`
```r
gce_list_images(image_project = "debian-cloud")
```

#### Network

Most of the time you will want to leave network to the default, at present you can only configure this in the UI.

#### Disks

You can also create another disk to attach to the VM via:

```r
gce_make_disk("my-disk")
```

By default it will be a 500GB disk unless you specify otherwise. You can then attach this disk to the instance upon creation using the `disk_source` argument set to the disk resource URL.

#### Metadata

You can add custom metadata by passing a named list to the instance.  More details from Google documentation is here `https://cloud.google.com/compute/docs/storing-retrieving-metadata`

```r
vm <- gce_vm_create(name = "test-vm2", 
                      predefined_type = "f1-micro",
                      metadata = list(start_date = as.character(Sys.Date())))
```

This includes useful utilities such as `startup-script` and `shutdown-script` that you can use to run shell scripts.  In those cases the named list should include the script as its value.

### Container based VMs

There is also support for launching VMs from a docker container, as configured via a [cloud-init](https://cloudinit.readthedocs.io/en/latest/topics/format.html) configuration file.

Here is the example from the [Google documentation](https://cloud.google.com/compute/docs/containers/vm-image/) - save this file locally:

```yaml
#cloud-config

users:
- name: cloudservice
  uid: 2000

write_files:
- path: /etc/systemd/system/cloudservice.service
  permissions: 0644
  owner: root
  content: |
    [Unit]
    Description=Start a simple docker container

    [Service]
    Environment="HOME=/home/cloudservice"
    ExecStartPre=/usr/share/google/dockercfg_update.sh
    ExecStart=/usr/bin/docker run --rm -u 2000 --name=mycloudservice gcr.io/google-containers/busybox:latest /bin/sleep 3600
    ExecStop=/usr/bin/docker stop mycloudservice
    ExecStopPost=/usr/bin/docker rm mycloudservice

runcmd:
- systemctl daemon-reload
- systemctl start cloudservice.service
```

If the above is saved as `example.yaml` you can then launch a VM using its configuration via the `gce_vm_container()` function:

```r
 vm <- gce_vm_container(cloud_init = "example.yml",
                        name = "test-container",
                        predefined_type = "f1-micro")

```

### Templated Container based VMs

There is support for RStudio, Shiny and OpenCPU docker images using the above to launch configurations.  The configurations are located in the [`/inst/cloudconfig`](https://github.com/MarkEdmondson1234/googleComputeEngineR/tree/master/inst/cloudconfig) package folder.

To launch those, use the `gce_vm_template()` function:

```r
> vm <- gce_vm_template("rstudio",
                        name = "rstudio-server",
                        predefined_type = "f1-micro",
                        username = "mark", password = "mark1234")

Checking job....
Job running:  0 /100
Job running:  0 /100
Operation complete in 22 secs
 External IP for instance rstudio  :  130.211.62.2 

##  rstudio running at 130.211.62.2:8787 

 You may need to wait a few minutes for the inital docker container to download and install before logging in.

```

You can then use `gce_vm_stop`, `gce_vm_start` etc. for your server.  You are only charged for when the VM is running, so you can stop it until you need it.

## Logging in to your instance

Google Cloud comes with a browser based SSH application, which you can launch via `gce_ssh_browser` to set it up further to your liking.

```r
library(googleComputeEngineR)
gce_ssh_browser("my-server")
```

![](http://g.recordit.co/TpM4IfRLgf.gif)

## SSH commands

You can also send ssh commands to your running instance from R via the `gce_ssh()` commands.

For this you will need to either connect first via the `gcloud compute ssh` command that generates SSH-keys, or generate them yourself following this guide.

Once you have generated for your username, the public and private key, you can connect via:

```r
library(googleComputeEngineR)

gce_ssh_setup(username = "mark", 
              instance = "your-instance", 
              key.pub = "filepath.to.public.key",
              key.private = "filepath.to.private.key")
              
gce_ssh("your-instance", "cd", user = "mark")
```

You can also call `gce_ssh` directly which will call `gce_ssh_setup` if it has not been run already.


