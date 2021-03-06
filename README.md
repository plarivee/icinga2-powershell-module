Icinga 2 Powershell Module Documentation
==============

Introduction
--------------

What is this PowerShell module for?
The idea was to create a basic module, providing a framework to install new Icinga 2 Agents in Windows more easily in combination with the Icinga Director.
Depending on the configuration of the module, you can install / update the Icinga 2 Agent, sign the certificates on your CA (the Icinga 2 master for example) and create the required configuration to operate the Agent on Windows correctly.

Installing the Module
--------------

In general there are two ways to use this module (or script).

### Install the module as real PowerShell module

At first we need to obtain folders in which we can install the module. For this type the following into your PowerShell
```powershell
    echo $env:PSModulePath
```    

Depending on your requirements who will be allowed to execute functions from the module, install it inside one of the provided folders. On demand you can also modify your path variable inside the Windows settings to add another module location.

While inside the modules folder, simply clone this repository or create a folder, named exactly like the .psm1 file and place it inside.
Once done restart your PowerShell and validate the correct installation of the module by writing
```powershell
    Get-Module -ListAvailable
```    

You might require to scroll a bit to locate your module directory with the installed Icinga 2 module. Once it's there, the installation was successfull

### Use the module as simple PowerShell script

The module is designed to work as simple PowerShell script which can either be executed from the PowerShell or by parsing the entire content into a PowerShell console.
To make the script executable from within the PowerShell, rename it from .psm1 to .ps1

Using the Module
--------------

To use the module (or as script) you will at first have to initialise a variable with the function including parameters

```powershell
    $icinga = Icinga2AgentModule
```    

Now you will have all members and variables availalbe inside $icinga, which can be used to install the Agent for example. As we require some parameters to ensure the module knows what you are intending to do, here is a full example for installing an Agent:

```powershell
    $icinga = Icinga2AgentModule `
              -AgentName        'windows-host-name' `
              -Ticket           '3459843583450834508634856383459' `
              -ParentZone       'icinga-master' `
              -ParentEndpoints  'icinga2a', 'icinga2b' `
              -CAServer         'icinga-master'

    exit $icinga.installIcinga2Agent()          
```    

Available Parameters
--------------

##### -AgentName
This is in general the name of your Windows host. It will have to match with your Icinga configuration, as it is part of the Icinga 2 Ticket and Certificate handling to ensure a valid certificate is generated

##### -Ticket
The Ticket you will receive from your Icinga 2 CA. In combination with the Icinga Director, it will tell you which Ticket you will require for your host

##### -InstallAgentVersion
You can either leave this parameter or add it to allow the module to install or update the Icinga 2 Agent on your system. It has to be a string value and look like this for example:
```powershell
    -InstallAgentVersion '2.4.10'
```    

##### -ParentZone
Each Icinga 2 Agent is in general forwarding it's check results to a parent master or satellite zone. Here you will have to specify the name of the parent zone

##### -AcceptConfig
Icinga 2 internals to make it configurable if the Agent is accepting configuration from the Icinga config master.

Default value: **$TRUE**

##### -ParentEndpoints
This parameter requires an array of string values, to which endpoints the Agent should in general connect to. If you are only having one endpoint, only add one. You will have to specify all endpoints the Agent requires to connect to. Examples:
```powershell
    -ParentEndpoints 'icinga2a'
    -ParentEndpoints 'icinga2a', 'icinga2b'
    -ParentEndpoints 'icinga2a', 'icinga2b', 'icinga2c'
```    

##### -DownloadUrl
With this parameter you can define a download Url from which the module will download a specific Icinga 2 Agent MSI Installer package. Please ensure to only define the base download Url, as the Module will generate the file to download based on your operating system architecture and the version to install.
The Icinga 2 MSI Installer name is internally build as follows: *Icinga2-v[InstallAgentVersion]-[OSArchitecture].msi*

Full example: Icinga2-v2.4.10-x86_64.msi

Default value: **https://packages.icinga.org/windows/**

##### -AllowUpdates
In case the Icinga 2 Agent is already installed on the system, this parameter will allow you to configure if you wish to upgrade / downgrade to a specified version with the *-InstallAgentVersion* parameter as well.
If none of both parameters is defined, the module will not update or downgrade the agent.

Default value: **$FALSE**

##### -InstallerHashes
To ensure downloaded packages are build by the Icinga Team and not compromised by third parties, you will be able to provide an array of SHA1 hashes here. In case you have defined any hashses, the module will not continue with updating / installing the Agent in case the SHA1 hash of the downloaded MSI package is not matching one of the provided hashes of this parameter.
Example:
```powershell
    -InstallerHashes 'hash1'
    -InstallerHashes 'hash1', 'hash2'
    -InstallerHashes 'hash1', 'hash2', 'hash3'
```    

##### -FlushApiDirectory
In case the Icinga Agent will accept configuration from the parent Icinga 2 system, it will possibly write data to /var/lib/icinga2/api/*
By setting this parameter to *$TRUE*, all content inside the api directory will be flushed once a change is detected by the module which requires a restart of the Icinga 2 Agent

Default value: **$FALSE**

##### -CAServer
Here you can provide a string to the Icinga 2 CA or any other CA responsible to generate the required certificates for the SSL communication between the Icinga 2 Agent and it's parent

##### -CAPort
Here you can specify a custom port in case your CA Server is not listening on 5665

Default value: **5665**

##### -ForceCertificateGeneration
The module will generate the certificates in general only if one of the required files is missing. By setting this parameter to $TRUE, the module will force the re-creation of the certificates.

Default value: **$FALSE**

##### -DebugMode
This is simply for development purposes and is barely used inside the module. Might change in the future.

Default value: **$FALSE**

Writing custom extensions
--------------
The module itself is designed to provide a basic framework which is designed to be extended on demand, without modifying this module at all. The easiest way for this is to install it as PowerShell module.
Once done, an example to write a custom function could look like this:

```powershell
function extendIcinga {

    #
    # Parameters you require to specify for this function
    #
    param(
        # Agent setup
        [string]$AgentName,
        [string]$Ticket,
        [string]$InstallAgentVersion,
        [array]$ParentEndpoints
    )

    #
    # Initialse an object with the Icinga2AgentModule
    # and add parameters including values for initialising
    #
    $agent = Icinga2AgentModule `
             -AgentName $AgentName `
             -Ticket $Ticket `
             -ParentEndpoints $ParentEndpoints `
             -InstallAgentVersion $InstallAgentVersion `
             -ParentZone 'icinga-parent-zone';
    
    #
    # Add a custom function doing the installation
    # and update of the icinga 2 Agent
    #
    $agent | Add-Member -membertype ScriptMethod -name 'customInstallFunction' -value {
        
        if ($this.isAgentInstalled()) {
            if (-Not $this.isAgentUpToDate()) {
                $this.updateAgent();
                $this.cleanupAgentInstaller();
            } else {
                $this.info('The Icinga Agent is up-to-date. No custom actions required.');
            }
        } else {
            $this.installAgent();
            $this.cleanupAgentInstaller();
        }
        
        $this.generateIcingaConfiguration();
        $this.applyPossibleConfigChanges();
        $this.flushIcingaApiDirectory();        
        
        # Some custom function calls
        $this.validateAgainstInternalSystem();
        
        return 0;
    }

    #
    # Another custom function we defined
    #
    $agent | Add-Member -membertype ScriptMethod -name 'validateAgainstInternalSystem' -value {
        # do some stuff
    }

    #
    # Return our extended Icinga object
    #
    return $agent;
}

$icinga = extendIcinga `
          -AgentName           'windows-host' `
          -Ticket              '43543534253453453453453435435' `
          -InstallAgentVersion '2.4.10' `
          -ParentEndpoints     'icinga2a', 'icinga2b';


exit $icinga.customInstallFunction()
```    