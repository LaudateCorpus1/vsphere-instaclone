# vsphere-instaclone

Use VMware instant clone technology to spawn TeamCity cloud agent quickly.

## Installation

1. Download the plugin .zip file from this repo's releases.
2. Upload it to TeamCity.

## Server Configuration

Create a new cloud profile in your project and set its type to vSphere Instaclone.
Fill in the path to the vSphere API, this is typically "https://my.vsphere.host/sdk".
Note that if your company is too lazy to get the real certificate for that URL,
you'll need to install the appropriate root certificate to cacerts store
of your TeamCity's JRE. I'm purposefully leaving out the detailed instructions
in the hopes that obtaining a trusted cert is easier than googling them.

Fill in username and password; the account must have sufficient privileges
to search for machines and perform the instant clone operation:

* System.View
* VirtualMachine.Provisioning.Clone
* VirtualMachine.Interact.PowerOn
* VirtualMachine.Inventory.CreateFromExisting
* Datastore.AllocateSpace
* Resource.AssignVMToPool

Finally, provide one or more image configurations.

    {
        "image-name": {
            "template": "datacenter/vm/tc-agent-template",
            "instanceFolder": "datacenter/vm/tc-agents",
            "maxInstances": 50,
            "network": "datacenter/network/agent-network"
        }
    }

For each image, `template` specifies path to the virtual machine to clone.
On vSphere, this path always starts with the name of the datacenter, followed by
"vm".

The key `instanceFolder` specifies the vSphere folder in which cloned machines should
be placed. This can be the same folder as your template.
The name of the image is used as a base for the names of the cloned images.

The plugin will not allow more than `maxInstances` instances to run simultaneously.

Optionally, you can set the network to which the cloned machine's ethernet card should
be connected. If your image contains multiple network cards, set this field to an
array of network names.

## Creating Agent Image

1. Prepare the machine to contain everything you need in your jobs.
2. Make sure VMware tools are installed.
3. Install the TeamCity agent and connect it to TeamCity.
4. *Let it upgrade*. This is very important, you don't want the agent upgrading post-clone.
   Also, this plugin must be installed on the server so that the agent can pick it up.
5. Once ready, modify agent's `build.properties` file by
   * deleting the authentication token (this ensures that each clone gets a new one) and by
   * setting `vmware.freeze.script` to a path to your freeze script (see below).
6. Wait a little. The agent will pick up the changes and restart. Just before registering
   to TeamCity server, the agent will execute the freeze script, which should freeze your VM.
   Once cloned, the freeze script continues executing on the new machine.

## Agent configuration

The agent needs access to `rpctool` from VMware tools.
Usually, the agent will find the tool automatically, unless it's installed in an unusual path.
In that case, set `teamcity.vmware.rpctool.path` property in `build.properties`
to the path to the `rpctool` executable.

Setting `vmware.freeze.script` will cause the agent to execute it during the next agent startup.

## The Freeze Script

The simplest freeze script only performs the freeze, but you probably also want
to restart networking, so as to have new clones pick up new IP address.
On Windows, the script might be a .bat file and look like this.

    netsh interface set interface "Ethernet0" disable
    "c:\Program Files\VMware\VMware Tools\rpctool.exe" "instantclone.freeze"
    netsh interface set interface "Ethernet0" enable

Adjust for other systems and your requirements.

