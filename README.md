# Fully local MAS (Core+Manage) deployment using CRC (aka OpenShift local)

This is a modification of Alex Donatelli (@evilADevil) scripts for deploying MAS (v8.11) on CRC. In this version everything is installed in cluster without need to connect to external UDS.


# MAS Local
 A few scripts to bring up a MAS Manage on OCP Local with DB2 in the cluster (there is also a version with an external MS SQL). If you want to watch a recording of the installation, it is available on YouTube [here](https://youtu.be/LrbPGrxxAvo). If you would like to use this MAS Manage on OCP Local to demo Maximo Mobile using a local emulator, check [this blog](https://community.ibm.com/community/user/asset-facilities/blogs/alex-donatelli/2022/12/22/maximo-mobile-demo-env-connected-to-manage-local).
 Please note that the current version works from CRC 2.19.0 (based on OCP 4.12.13) on.

## Summary
This is modified version of @evilADevil scripts for bringing up the MAS Manage on CRC. It does not require any external elements - everything is setup within CRC.

## What you would need to run OpenShift Local
First of all, you'll need a decent Laptop. I used a Lenovo ThinkPad P15 with 64 GiB of memory. In the end, you will need the availability of 14 vCPU and 30 GiB of memory in the virtual environment you will use that is Hyper-V for Windows.
If you have a smaller PC that has only 32 GB of memory but still at least 12 vCPU, you may want to try the deployment using an external MS SQL Server described [here](https://github.com/evilADevil/mas-local/tree/main/mssql)
Then you will need a locally running OCP (also know previously as CRC, i.e. Code Ready Container). Normally you would download it from the Red Hat Console page, but the version in that page is always the latest, while we need the one supported by MAS. You can download it from https://developers.redhat.com/content-gateway/rest/mirror/pub/openshift-v4/clients/crc/2.12.0. For Windows, the file is `crc-windows-installer.zip`. Unzip it and install it. Then register to Red Hat if you haven't already, go to the [Red Hat Console for OpenShift](https://console.redhat.com/openshift), click on the *Create Cluster* button, click on the *Local* tab. In that page, there is your pull secret that you'll need to copy and use during setup
Now you are ready for the next step. Open a command prompt and run `crc setup`. Before staring a new OpenShift Local, we want to configure it so that it will allow MAS to fit. We will need to use a trick because CRC seems to have a bug in expanding the disk. We need to start it a first time, then stop it, set the new disk size and start it again.
This is the set of commands to configure it correctly and start it
```
crc config set consent-telemetry no
crc config set cpus 14
crc config set memory 30720
crc start
crc stop
crc config set disk-size 200
crc start
```
At this point you should have an OpenShift running and ready to host MAS Core + Manage. From the last ourput of `crc start` you may want to record the password to log into the cluster.

## What you would need to run MAS Manage
Now that you have OpenShift Local running, you are almost ready to install MAS Core and Manage, but before doing that, you'll need a few files and info.
Open a Windows command prompt, and `cd` to a directory in which you will want to create the MAS Local working directory.
As a firt step, let's grab the content of this repository. Issue the following command from the directory you have chosen for this work.
```
git clone https://github.com/evilADevil/mas-local
cd mas-local
```
Then you need to procure yourself a few files to add to this directory and some important information:
1. The **Entitled Registry (ER) key**. This key will have to be enabled to get the MAS and CloudPak for Data images and you can get it by logging into [My IBM](https://myibm.ibm.com/dashboard/) and click on *Container Software & Entitlement key*
2. A **MAS license file**. Put this file called `license.dat` in the `mas-local` directory.
3. A **license id** matching the MAS license file. You can find out what this is by open the license file in an editor, and check the first line. The license id will be the second-last number. For example, if your first line is `SERVER sls-rlks-0.rlks 0272bc344002 27000` then your license id is `0272bc344002`.


The next step is to customize the file `masocpl.yml` using the information you collected. Specifically:
- Replace `<<your ER key>>` with your ER key from step 0 above.
- Replace `<<your license id>>` with the license id you obtained from step 2 above

We are ready to proceed to install MAS Core and Manage

## How you would install MAS Core and MAS Manage
At this point, your working directory should include the following files:
```
local-path-storage-mod.yaml
masdevops.yaml
masocpl.yml
uds.crt
license.dat
masinst.bat
```
other files may be present there (like this `README.md`), but these are the important ones for the installation.
Before starting the MAS installation, you need to login to the OpenShift Local instance. The `crc start` command you have issued should have ended with some messages like these that include the admin credentials:
```
Started the OpenShift cluster.

The server is accessible via web console at:
  https://console-openshift-console.apps-crc.testing

Log in as administrator:
  Username: kubeadmin
  Password: H2rDA-GXB82-dSdTA-cAAYu

Log in as user:
  Username: developer
  Password: developer

Use the 'oc' command line interface:
  > @FOR /f "tokens=*" %i IN ('crc oc-env') DO @call %i
  > oc login -u developer https://api.crc.testing:6443
```
At the command prompt, run the following commands using the password that your environment provided:
```
@FOR /f "tokens=*" %i IN ('crc oc-env') DO @call %i
oc login -u kubeadmin -p H2rDA-GXB82-dSdTA-cAAYu https://api.crc.testing:6443
```
At this point you are ready to install MAS with Manage. Run `masinst` at the prompt and wait it to finish.
As the log proceed, pay attention to record the userid and password of the MAS superuser, that should look like this:
```
ok: [localhost] => {
    "msg": [
        "Maximo Application Suite is Ready, use the superuser credentials to authenticate",
        "Admin Dashboard ... https://admin.masdemo.apps-crc.testing",
        "Username .......... DIZv7X2eavITxb3vKtf3XRsY85UYj7FV",
        "Password .......... 58Wi9n9U4yVgZ7AhXVRS4eIqEQSnMhsq"
    ]
}
```
In case you don't have the log anymore, you can always retrieve them from the `masdemo-credentials-superuser` secret in the `mas-masdemo-core` namespace.
After the installation completes, you may want to log into MAS administration using the MAS superuser credentials. You can fing the urls to use in the `Networking -> Routes` of the `mas-masdemo-core` namespace. First of all click on the `masdemo-api` url and accept the self-signed certificate. In this way your browser won't have a problem in accessing the MAS APIs later on. Ignore the execption you will get and close that window. Then go back to the cluster UI and click the `masdemo-admin` url. Once in MAS, navigate to the `Users` section, filter by `wilson`, then click on the `wilson` user and reset his password. Wait for the syncronization to finish. Log out and login as `wilson`. In the MAS Navigator you should see the Manage tile and you should be able to launch into it.
When you are done with your environment, you can stop it using the `crc stop` command. Of course, you can restart it when needed, using the `crc start` command.
