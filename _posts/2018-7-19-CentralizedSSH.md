---
layout: post
title: Centralized SSH Key Management For Windows/Linux Environments
thumbnail: "assets/img/thumbnails/CentralSSHBanner.png"
tags: [Windows, PowerShell, PowerShell Core, Core, pwsh, Docker, Hashicorp, Vault, ssh, key, certificate]
excerpt_separator: <!--more-->
---

![Hansel Inside the Computer]({{ site.baseurl }}/assets/img/pexels/CentralSSH.png){: .center-image }

***"SSH is the future and everyone should use it."***
{: style="text-align: center"}
\- Joey Aiello, PowerShell PM, [PowerShell Conference EU](https://youtu.be/cLsLFOB3i0A?t=738), April 2018
{: style="text-align: center"}
<!--more-->

<div style='position:relative;padding-bottom:54%'><iframe src='https://gfycat.com/ifr/GiddyDarkAlaskankleekai' frameborder='0' scrolling='no' width='100%' height='100%' style='position:absolute;top:0;left:0' allowfullscreen></iframe></div><p> <a href="https://gfycat.com/gifs/detail/GiddyDarkAlaskankleekai"></a></p>

100% agree! But most people run into a pretty big problem soon after they start down this road:

*How do I effectively manage thousands of SSH keys for my organization?*
{: style="text-align: center"}

Well, I spent the past few of weeks working out a pretty exciting solution that leverages LDAP (Active Directory) and [Hashicorp's Vault Server](https://www.vaultproject.io/).

**Key Benefits:**
- SSH Authentication and Authorization is tied to Active Directory Accounts and Security Groups
- No need to ever revoke SSH keys
- Users can generate their own SSH keys (that grant them access to resources as defined by Active Directory) whenever they want

**Things to Keep In Mind**
- This is a tutorial that aspires to be Production-Ready, but isn't quite there in many respects (for instance, you probably won't want to use a Vagrant Box as your Vault Server, but I do in the tutorial just to make things easier)
- This tutorial uses 3 PowerShell Modules that I wrote to make things a lot easier: [MiniLab](https://www.powershellgallery.com/packages/minilab), [WinSSH](https://www.powershellgallery.com/packages/winssh), and [VaultServer](https://www.powershellgallery.com/packages/vaultserver).
- The WinSSH Module uses Microsoft's port of OpenSSH - [OpenSSH-Win64](https://github.com/powershell/Win32-OpenSSH) - which is still very much in beta.
- There are **many** different ways to configure a Hashicorp Vault Server. Some functions (specifically *'Configure-VaultServerForLDAPAuth'* and *'Configure-VaultServerForSSHManagement'*) within my VaultServer Module represent **my** preferred configurations. Your organization should update these functions as appropriate to meet your security/policy guidelines. Luckily, the Vault Server's HTTP API makes it *very* easy to configure/reconfigure.
- Finishing this tutorial should take 30 minutes to an hour.

# Let's Get This Show On The Road

**1)** Log into your localhost Windows 10, Windows 2012 R2, or Windows 2016 workstation as a Domain Admin

**2)** Certain steps in this tutorial have different commands for Windows PowerShell 5.1 (WinPS) versus PowerShell Core 6.X (PSCore). At this point, please launch your prefered version of PowerShell (Run As Administrator).

**SIDE NOTE:** If you don't have PSCore installed yet, or if you would like to update to the latest version of PSCore, launch WinPS and use my ProgramManagement Module...

```powershell
Install-Module ProgramManagement
Import-Module ProgramManagement
$InstallPwshResult = Install-Program -ProgramName powershell-core -CommandName pwsh.exe -PreRelease -ExpectedInstallLocation "C:\Program Files\PowerShell"
```

**3)** Install and import the WinSSH, VaultServer, and MiniLab PowerShell Modules.

**IMPORTANT NOTE:** Since these Modules were originally written for WinPS (as opposed to PSCore), importing them in PSCore can take around 5-10 minutes. (This is due to how I made them compatible with PSCore and not representative of PSCore performance in general.)

**NOTE:** The *'-AllowClobber'* switch is used with *'Install-Module'* because these 3 Modules have some functions in common (such as *'New-Runspace'*). They will not override any other function names from other Modules in your environment.

```powershell
$ModulesToInstallAndImport = @("WinSSH","VaultServer","MiniLab")
foreach ($ModuleName in $ModulesToInstallAndImport) {
    Install-Module $ModuleName -AllowClobber
    Import-Module $ModuleName
}
```

**4)** Install OpenSSH-Win64. The below command installs ssh binaries (ssh.exe, scp.exe, etc), the ssh-agent service, and the sshd server/service. It also configures sshd such that when someone tries to ssh to your workstation (localhost), their default shell will be PSCore (pwsh.exe). Please note that if PSCore is not installed on your localhost, it will be installed. (If you do **not** want PSCore installed or used as the DefaultShell for the localhost sshd server, use *'-DefaultShell powershell'*). Also, please note that if you have any other version of ssh.exe on your localhost, the OpenSSH-Win64 version will take precedence.

```powershell
$InstallWinSSHResult = Install-WinSSH -GiveWinSSHBinariesPathPriority -ConfigureSSHDOnLocalHost -DefaultShell pwsh
```

**5)** Next, let's deploy a CentOS 7 Virtual Machine to our hypervisor. If you are using Hyper-V like me, you can use my *'Deploy-HyperVVagrantBoxManually'* function from the MiniLab Module. Be sure to update *'$VMDestDir'* with a value appropriate for your environment. 

**IMPORTANT NOTE:** If you do not already have Hyper-V installed, below *'Deploy-HyperVVagrantBoxManually'* function will install it for you. If a restart is required, the function will halt and ask you to restart before proceeding. After restart, re-run the function.

```powershell
$VMName = "CentOS7Vault"
$VMDestDir = "H:\VirtualMachines"
$DeployHyperVVagrantBoxSplatParams = @{
    VagrantBox              = "centos/7"
    VagrantProvider         = "hyperv"
    VMName                  = $VMName
    VMDestinationDirectory  = $VMDestDir
    Memory                  = 2048
    CPU                     = 1
}
```

If your localhost **IS** your Hyper-V hypervisor, do the following...

```powershell
$DeployCentOS7Result = Deploy-HyperVVagrantBoxManually @DeployHyperVVagrantBoxSplatParams
```

**OTHERWISE**, run the function on your remote Hyper-V hypervisor via...

```powershell
$HyperVFQDN = "ZeroHyper01.zero.lab"
$HyperVCreds = [pscredential]::new("zero\zeroadmin",$(Read-Host "Enter Password" -AsSecureString))
$MiniLabModuleFunctions = $(Get-Module MiniLab).Invoke({$FunctionsForSBUse})
$DeployCentOS7Result = Invoke-Command -ComputerName $HyperVFQDN -ScriptBlock {
    # Load the MiniLab Module in the Remote Session
    $using:MiniLabModuleFunctions | foreach { Invoke-Expression $_ }

    $SplatParams = $using:DeployHyperVVagrantBoxSplatParams
    Deploy-HyperVVagrantBoxManually @SplatParams
}
```

**6)** Take note of the CentOS 7 VM's IP Address

```powershell
$VaultServerIP = $DeployCentOS7Result.VMIPAddress
```

**7)** Make sure we have the Vagrant Unsecure Private Key so that we can ssh into our CentOS 7 VM

```powershell
if (!$(Test-Path "$HOME\.ssh\vagrant_unsecure_private_key")) {
    Invoke-WebRequest -Uri "https://raw.githubusercontent.com/hashicorp/vagrant/master/keys/vagrant" -OutFile "$HOME\.ssh\vagrant_unsecure_private_key"
}
```

**8)** Change the hostname of our CentOS 7 VM to match the VM Name 'CentOS7Vault'

**IMPORTANT NOTE:** If you would like to be able to resolve 'CentOS7Vault' via HostName/FQDN (as opposed to IP), you will need to update your DNS Records (which is covered later on). If you intend to update your DNS Records, ensure that the relevant domain is appended to the hostname. This tutorial will assume that you intend to update your DNS Records.

```powershell
$CentOS7LocalUser = "vagrant"
$DomainName = "zero.lab"
$VMNameLowerCase = $VMName.ToLowerInvariant()
$DomainNameLowerCase = $DomainName.ToLowerInvariant()
$RenameCentOSHostScript = @"
sudo hostnamectl set-hostname $VMNameLowerCase.$DomainNameLowerCase --static
sudo sed -i '/localhost/ s/`$/ $VMNameLowerCase $VMNameLowerCase.$DomainNameLowerCase/' /etc/hosts
"@
ssh -o "StrictHostKeyChecking=no" -o "IdentitiesOnly=yes" -i "$HOME\.ssh\vagrant_unsecure_key" -t $CentOS7LocalUser@$VaultServerIP "$RenameCentOSHostScript"
```

**9)** Next, let's upgrade the kernel for 'CentOS7Docker'. This will make some additional Docker features available to us. Note that this will cause CentOS7Vault to reboot, so be sure to wait for it to become available on the network again before moving to the next step.

```powershell
$UpgradeKernelScript = @'
sudo yum -y update
sudo yum -y install yum-plugin-fastestmirror
sudo rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
sudo rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
sudo yum -y --enablerepo=elrepo-kernel install kernel-ml
sudo grub2-set-default 0
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
sudo reboot
'@
ssh -o "StrictHostKeyChecking=no" -o "IdentitiesOnly=yes" -i "$HOME\.ssh\vagrant_unsecure_key" -t $CentOS7LocalUser@$VaultServerIP "$UpgradeKernelScript"
```

**10)** Let's turn our CentOS7 VM into a Docker Machine

```powershell
$InstallDockerScript = @'
sudo yum install net-tools -y
sudo curl -fsSL https://get.docker.com/ | sh
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker vagrant
'@
ssh -o "StrictHostKeyChecking=no" -o "IdentitiesOnly=yes" -i "$HOME\.ssh\vagrant_unsecure_key" -t $CentOS7LocalUser@$VaultServerIP "$InstallDockerScript"
```

**11)** Install PowerShell Core on CentOS7Vault to help streamline future operations

```powershell
$InstallPowerShellScript = @'
curl https://packages.microsoft.com/config/rhel/7/prod.repo | sudo tee /etc/yum.repos.d/microsoft.repo
sudo yum install powershell -y
pscorepath=$(which pwsh)
subsystemline=$(echo \"Subsystem powershell $pscorepath -sshs -NoLogo -NoProfile\")
sudo sed -i \"s|sftp-server|sftp-server\n$subsystemline|\" /etc/ssh/sshd_config
sudo systemctl restart sshd
'@
ssh -o "StrictHostKeyChecking=no" -o "IdentitiesOnly=yes" -i "$HOME\.ssh\vagrant_unsecure_key" -t $CentOS7LocalUser@$VaultServerIP "$InstallPowerShellScript"
```

**12)** Update your DNS Records on your DNS Server. Assuming you're using a Windows DNS Server and depending on your particular DNS implementation, you can use PowerShell to create a new A entry and corresponding reverse lookup via...

(NOTE: If you're already logged in as a Domain Admin or user with permissions to Remote into the DNS Server, you don't need the below *'$DNSServerCreds'* or the *'-Credential'* parameter)

```powershell
$FQDNOfDNSServer = "ZeroDC01.$DomainName"
$DNSServerCreds = [pscredential]::new("zero\zeroadmin",$(Read-Host "Enter Password" -AsSecureString))
Invoke-Command -ComputerName $FQDNOfDNSServer -Credential $DNSServerCreds -ScriptBlock {
    $AddDNSServerRecordSplatParams = @{
        Name            = $using:VMName
        ZoneName        = $using:DomainName
        AllowUpdateAny  = $True
        IPv4Address     = $using:VaultServerIP
        CreatePtr       = $True
    }
    Add-DnsServerResourceRecordA @AddDNSServerRecordSplatParams
}
```

**13)** Take note of the Vault Server's FQDN

```powershell
$VaultServerFQDN = "$VMName.$DomainName"
```

**14)** Remove any old entries from *'$HOME\.ssh\known_hosts'* referencing the same IP Address or FQDN as CentOS7Vault

```powershell
[System.Collections.ArrayList][array]$KnownHostsContent = Get-Content "$HOME\.ssh\known_hosts"
$LineToRemoveA = $KnownHostsContent -match $VaultServerIP
if ($LineToRemoveA) {
    if ($KnownHostsContent.Count -gt 1) {
        $LineNumberToRemoveA = $KnownHostsContent.IndexOf($LineToRemoveA)
        $KnownHostsContent.RemoveAt($LineNumberToRemoveA)
    }
    else {
        $KnownHostsContent = $null
    }
}
$LineToRemoveB = $KnownHostsContent -match $VaultServerFQDN
if ($LineToRemoveB) {
    if ($KnownHostsContent.Count -gt 1) {
        $LineNumberToRemoveB = $KnownHostsContent.IndexOf($LineToRemoveB)
        $KnownHostsContent.RemoveAt($LineNumberToRemoveB)
    }
    else {
        $KnownHostsContent = $null
    }
}
if ($LineNumberToRemoveA -or $LineNumberToRemoveB) {Set-Content -Path "$HOME\.ssh\known_hosts" -Value $KnownHostsContent}
```

**15)** **If you are using PSCore on your localhost**, create a PSSession to CentOS7Vault **(otherwise, skip this step)**

```powershell
$CentOS7PSSession = New-PSSession -HostName $VaultServerFQDN -KeyFilePath "$HOME\.ssh\vagrant_unsecure_key" -UserName $CentOS7LocalUser
```

**16)** Set the Docker Storage Driver and apply the Docker daemon.json config to CentOS7Docker

If you're using PSCore on your localhost...

```powershell
## BEGIN PSCore Version ##
$DockerDaemonJson = @'
{
  "storage-driver": "overlay2"
}
'@
$EncodedPwshCmdPrep = @"
Set-Content -Path /etc/docker/daemon.json -Value @'`n$DockerDaemonJson`n'@
`$null = systemctl restart docker *>/dev/null
Start-Sleep -Seconds 5
systemctl status docker | grep 'Active:'
"@
$EncodedPwshCmd = [Convert]::ToBase64String([System.Text.Encoding]::Unicode.GetBytes($EncodedPwshCmdPrep))

# Apply the Docker daemon.json config to CentOS7Vault
Invoke-Command -Session $CentOS7PSSession -ScriptBlock {sudo pwsh -EncodedCommand $using:EncodedPwshCmd}

## END PSCore Version ##
```

If you're using WinPS 5.1 on your localhost...

```powershell
## BEGIN WinPS 5.1 Version ##

$SetDockerDaemonJson = @'
sudo bash -c 'cat << EOF > /etc/docker/daemon.json
{
  \"storage-driver\": \"overlay2\"
}
EOF'

sudo systemctl restart docker
sleep 5
systemctl status docker | grep 'Active:'
'@
ssh -o "StrictHostKeyChecking=no" -o "IdentitiesOnly=yes" -i "$HOME\.ssh\vagrant_unsecure_key" -t $CentOS7LocalUser@$VaultServerIP "$SetDockerDaemonJson"

## END WinPS 5.1 Version ##
```

# Setting Up TLS

Since we're trying our best to mock a Production scenario, we need to setup TLS so that all communications with our Vault Server are encrypted over the network.

If you happen to have a Microsoft Certificate Authority available on your network, you can use the MiniLab Module's *'Generate-Certificate'* function to request a Certificate for the Vault Server.

If you do NOT have a Microsoft Certificate Authority available, but you would like to setup new Microsoft Root and Issuing CAs, please see one of my older blog posts: [https://pldmgg.github.io/2018/06/02/MiniLab.html](https://pldmgg.github.io/2018/06/02/MiniLab.html)

If you would like to setup new Linux Root and Issuing CAs (which takes significantly less time), please see my previous post: [https://pldmgg.github.io/2018/07/10/CFSSL.html](https://pldmgg.github.io/2018/07/10/CFSSL.html)

**17)** Assuming you have a Microsoft Certificate Authority available, generate key/certificate files for the Vault Server using the MiniLab Module's *'Generate-Certificate'* function 

```powershell
$VaultTLSCertsDir = "$HOME\Downloads\VaultServerTLSCerts"
$GenCertSplatParams = @{
    CertGenWorking              = $VaultTLSCertsDir
    BasisTemplate               = "WebServer"
    CertificateCN               = "VaultServer"
    Organization                = "pldmgg"
    OrganizationalUnit          = "DevOps"
    Locality                    = "Philadelphia"
    State                       = "PA"
    Country                     = "US"
    CertFileOut                 = "VaultServer.cer"
    PFXFileOut                  = "VaultServer.pfx"
    CertificateChainOut         = "VaultServerChain.p7b"
    AllPublicKeysInChainOut     = "VaultServerChain.pem"
    ProtectedPrivateKeyOut      = "VaultServerPwdProtectedPrivateKey.pem"
    UnProtectedPrivateKeyOut    = "VaultServerUnProtectedPrivateKey.pem"
    SANObjectsToAdd             = @("IP Address","DNS")
    IPAddressSANObjects         = @("$VaultServerIP","0.0.0.0")
    DNSSANObjects               = @($VMName,$VaultServerFQDN)
}
$GenVaultCertResult = Generate-Certificate @GenCertSplatParams
```

**NOTE:** We need *'0.0.0.0'* as a SAN for the Vault Server Certificate because we're going to be running the Vault Server
as a docker container on CentOS7Vault. This will make more sense later.

**18)** Get the File Paths of all of the files we need copied to the CentOS7Vault

```powershell
$VaultServerUnProtectedPrivateKey = $GenVaultCertResult.FileOutputHashTable.EndPointUnProtectedPrivateKey
$VaultServerPublicCertInPemFormat = $GenVaultCertResult.FileOutputHashTable.EndPointPublicCertFile
$RootCAPublicCertInPemFormat = $GenVaultCertResult.FileOutputHashTable.RootCAPublicCertFile
$IntermediateCAPublicCertInPemFormat = $GenVaultCertResult.FileOutputHashTable.IntermediateCAPublicCertFile
$CertChain = $GenVaultCertResult.FileOutputHashTable.AllPublicKeysInChainOut

$CertFilesToTransfer = @(
    $VaultServerUnProtectedPrivateKey
    $VaultServerPublicCertInPemFormat
    $RootCAPublicCertInPemFormat
    $IntermediateCAPublicCertInPemFormat
    $CertChain
)
```

**19)** **IMPORTANT STEP:** If you **didn't use a Microsoft Certificate Authority** (i.e. Active Directory Certificate Services (ADCS)) on the same Domain as your Windows workstation (localhost) to generate the Vault Server key/certificate files, or if you used the Linux Certificate Authority solution, you **MUST** import the Root CA Public Key and Subordinate/Issuing/Intermediate CA Public Key in the Vault Server Certificate Chain into the appropriate Local Machine Certificate Stores so that TLS works between your Windows workstation and the Vault Server. **You must do this on every workstation that you intend to use to interact with the Vault Server's HTTP API**

For Windows workstations, you can do the following...

```powershell
Import-Certificate -FilePath $RootCAPublicCertInPemFormat -CertStoreLocation Cert:\LocalMachine\CA
Import-Certificate -FilePath $RootCAPublicCertInPemFormat -CertStoreLocation Cert:\LocalMachine\Root
Import-Certificate -FilePath $IntermediateCAPublicCertInPemFormat -CertStoreLocation Cert:\LocalMachine\CA
```

For Linux workstations, the exact commands can vary, but for the most part (on modern distributions), you would do something like this:

```bash
sudo cp rootca.pem /usr/local/share/ca-certificates/rootca.pem
sudo cp issuingca.pem /usr/local/share/ca-certificates/issuingca.pem
sudo update-ca-certificates
```

**20)** Copy *'$CertFilesToTransfer'* to CentOS7Vault

```powershell
# Create a $TransferDirectory so that we can transfer the entire directory all at once
$TransferDirectory = "$VaultTLSCertsDir\$($VaultTLSCertsDir | Split-Path -Leaf)"
if (!$(Test-Path $TransferDirectory)) {
    $null = New-Item -ItemType Directory -Path $TransferDirectory -Force
} else {
    if ([array]$(Get-ChildItem $TransferDirectory).Count -gt 0) {
        Remove-Item "$TransferDirectory\*" -Recurse -Force
    }
}

# Copy the files to the $TransferDirectory
foreach ($CertFilePath in $CertFilesToTransfer) {
    Copy-Item -Path $CertFilePath -Destination $TransferDirectory
}
```

If you're using PSCore on your localhost...

```powershell
Copy-Item -ToSession $CentOS7PSSession -Path $TransferDirectory -Destination "/home/$CentOS7LocalUser" -Recurse -Force
```

If you're using WinPS 5.1 on your localhost...

```powershell
scp -i "$HOME\.ssh\vagrant_unsecure_private_key" -r "$TransferDirectory" $CentOS7LocalUser@$($VaultServerFQDN):/home/$CentOS7LocalUser/
```

**21)** SSH Into the Vault Server

```powershell
ssh -i "$HOME\.ssh\vagrant_unsecure_private_key" $CentOS7LocalUser@$VaultServerFQDN
```

**22)** Start an elevated (root) pwsh session

```bash
sudo pwsh
```

**23)** Create some docker volumes that will contain important Vault Data

```powershell
docker volume create vaultconfig
docker volume create vaultsecrets
docker volume create vaultaudit

# The above docker commands created the three directories /var/lib/docker/volumes/vault<config|secrets|audit>/_data
$VaultConfigDirectory = $($(docker volume inspect vaultconfig) | ConvertFrom-Json).MountPoint
$VaultSecretsDirectory = $($(docker volume inspect vaultsecrets) | ConvertFrom-Json).MountPoint
$VaultAuditDirectory = $($(docker volume inspect vaultaudit) | ConvertFrom-Json).MountPoint
```

**24)** Move the Vault Cert/Key Files to *'$VaultConfigDirectory'*

```powershell
$VaultServerTLSCertFiles = Move-Item -Path /home/vagrant/VaultServerTLSCerts/* -Destination $VaultConfigDirectory -Passthru
```

**25)** Redefine all of the Cert files we are working with in the CentOS7Vault root pwsh session

```powershell
$VaultTLSPrivKeyFileItem = $VaultServerTLSCertFiles | Where-Object {$_.Name -match "UnProtectedPrivateKey\.pem"}
$VaultTLSPubCertFileItem = $VaultServerTLSCertFiles | Where-Object {$_.Name -match "VaultServer[\w]+Public_Cert\.pem"}
$CertChainFileItem = Get-ChildItem -Path $VaultConfigDirectory -File -Filter "*Chain.pem"
# The smallest 'Public_Cert' file will be the Root CA
$RootAndIssuingCABySize = $VaultServerTLSCertFiles | Where-Object {
    $_.BaseName -notmatch "VaultServer" -and $_.BaseName -match "Public_Cert"
} | Sort-Object -Property Length
$RootCAPubCertFileItem = $RootAndIssuingCABySize[0]
$IssuingCAPubCertFileItem = $RootAndIssuingCABySize[-1]
```

**26)** Write the Vault Server Config File (vault.hcl) to *'$VaultConfigDirectory'*. Note that the file extension *'.hcl'* refers to a Hashicorp structured data format called HCL

**IMPORTANT NOTE:** The below *'address'* property must match one of Subject Alternate Names (SANs) on the Vault Server Certificate. Since we are using a docker container, it must be wildcard *'0.0.0.0'*

```powershell
$VaultConfigHCL = @"
ui = true

storage "file" {
    path = "/vault/file"
}

listener "tcp" {
    address = "0.0.0.0:8200"
    tls_disable = 0
    tls_cert_file = "/vault/config/$($CertChainFileItem.Name)"
    tls_key_file = "/vault/config/$($VaultTLSPrivKeyFileItem.Name)"
}
"@
Set-Content -Path "$VaultConfigDirectory\vault.hcl" -Value $VaultConfigHCL
```

**27)** Start the Vault Server Docker Container

```powershell
docker run --cap-add=IPC_LOCK --mount source=vaultconfig,target=/vault/config --mount source=vaultsecrets,target=/vault/file --mount source=vaultaudit,target=/var/log -p 8200:8200 -d --name=vaultprod-jump vault server
```
**28)** After the container is up and running (check with *'docker ps -a'*), fix permissions on the audit log directory (within the docker container) so that we can actually enable to audit log. At the same time, add the Root and Intermediate/Issuing Certificate Authority Public Certs to *'/etc/ssl/certs'* (within the docker container) and *'update-ca-certificates'*

```powershell
$FixAuditLogsAndUpdateCertsWithinContainerAshScript = @"
chmod 755 /vault/logs
chown vault:vault /vault/logs
cp /vault/config/$($RootCAPubCertFileItem.Name) /etc/ssl/certs/
cp /vault/config/$($IssuingCAPubCertFileItem.Name) /etc/ssl/certs/
update-ca-certificates
exit
"@
docker exec -i vaultprod-jump /bin/ash -c $FixAuditLogsAndUpdateCertsWithinContainerAshScript
```

**Congratulations!** You can now navigate to *'https://CentOS7Vault.zero.lab:8200'* and continue the configuration via GUI, but we will be sticking with the command line.

**29)** Still within our ssh session on CentOS7Vault in a root pwsh session, create the Vault Server Master Key Shards

```powershell
$MasterKeyShardsPrep = docker exec -it --env "VAULT_ADDR=https://0.0.0.0:8200" vaultprod-jump /bin/ash -c 'vault operator init'
$MasterKeyShardsNoColorCodes = $MasterKeyShardsPrep[0..6] -replace '\x1b\[[0-9;]*[a-z]',''
$KeyShardInfo = [ordered]@{}
$MasterKeyShardsNoColorCodes[0..6] | Where-Object {$_ -match "Unseal|Initial"} | foreach {
    $SplitResult = $_ -split ": "
    $KeyShardInfo.Add($($SplitResult[0] -replace "[\s]","").Trim(),$SplitResult[1].Trim())
}
```

*'$KeyShardInfo'* will contain something like...

```powershell
Name                           Value
----                           -----
UnsealKey1                     y3ksCwPqsBYnMJBVD1jLvtmo6y3H4sZEJW/6riaQoZky
UnsealKey2                     LEH6O95xbO09rdigFXgfM47iqh2SXgDc0VXasBPRyE9j
UnsealKey3                     7FDtGfVgjjJATaXA3Sp1p5KNyd4YWQaksKCsssRfyNQD
UnsealKey4                     s5EFEAmSipwH0d8wvqLsI43tS2sioHUlZCAZxXrRyyqJ
UnsealKey5                     ZA6Xd4fltPx3DgZYUTFQIyJ2I5LzSJrAezDimdNaUNKo
InitialRootToken               99034eb0-cf42-45c3-cfaa-4345588141e5
```

**30)** Next let's prepare to move *'$KeyShardInfo'* to our localhost Windows machine. First, write the hashtable object to *'$HOME/KeyShardInfo.xml'*

```powershell
$KeyShardInfo | Export-Clixml "$HOME/KeyShardInfo.xml"
```

**31)** Encrypt *'$HOME/KeyShardInfo.xml'* with openssl

**NOTE:** openssl will prompt you for a password. You'll need it for the decryption operation after we move *'KeyShardInfoEnc.xml'* to our localhost Windows machine. After decryption on Windows, you won't need the password ever again.

```powershell
openssl aes-256-cbc -salt -a -e -in "$HOME/KeyShardInfo.xml" -out "$HOME/KeyShardInfoEnc.xml"
```

**32)** Remove the original unencrypted *'$HOME/KeyShardInfo.xml'*

```powershell
Remove-Item "$HOME/KeyShardInfo.xml" -Force
```

**33)** Move the encrypted file from *'/root/KeyShardInfoEnc.xml'* to *'/home/vagrant/KeyShardInfoEnc.xml'* and change permissions so that we can transfer over PSSession (or via scp if you're using WinPS 5.1 on your localhost Windows workstation)

```powershell
$NonRootUserPath = "/home/vagrant/KeyShardInfoEnc.xml"
Move-Item -Path "$HOME/KeyShardInfoEnc.xml" -Destination $NonRootUserPath
chown vagrant:vagrant $NonRootUserPath
chmod 600 $NonRootUserPath
```

**34)** Exit the elevated (root) pwsh session

```powershell
exit
```

**35)** Exit the ssh session

```bash
exit
```

**36)** Back on your localhost Windows workstation, let's transfer *'/home/vagrant/KeyShardInfoEnc.xml'* to *'$VaultTLSCertsDir'* (just to keep everything together...the key shards have nothing to do with TLS certs)

If you're using PSCore on your localhost...

```powershell
Copy-Item -FromSession $CentOS7PSSession -Path /home/$CentOS7LocalUser/KeyShardInfoEnc.xml -Destination $VaultTLSCertsDir
```

If you're using WinPS 5.1 on your localhost...

```powershell
scp -i "$HOME\.ssh\vagrant_unsecure_private_key" $CentOS7LocalUser@$($VaultServerFQDN):/home/$CentOS7LocalUser/KeyShardInfoEnc.xml "$VaultTLSCertsDir\KeyShardInfoEnc.xml"
```

**37)** Let's also remove *'/home/vagrant/KeyShardInfoEnc.xml'* from the 'CentOS7Vault' filesystem

If you're using PSCore on your localhost...

```powershell
Invoke-Command -Session $CentOS7PSSession -ScriptBlock {Remove-Item /home/$using:CentOS7LocalUser/KeyShardInfoEnc.xml -Force}
```

If you're using WinPS 5.1 on your localhost...

```powershell
ssh -o "StrictHostKeyChecking=no" -o "IdentitiesOnly=yes" -i "$HOME\.ssh\vagrant_unsecure_key" -t $CentOS7LocalUser@$VaultServerIP "rm /home/$CentOS7LocalUser/KeyShardInfoEnc.xml"
```

**38)** Make sure you have the latest Long Term Support (LTS) openssl.exe available on your localhost Windows workstation (because this is what CentOS 7 uses). To do so, you can use the MiniLab Module's *'Get-WinOpenSSL'* function. The function will also add openssl.exe to your *'$env:Path'*

```powershell
Get-WinOpenSSL
```

**39)** Decrypt *'$VaultTLSCertsDir\KeyShardsEnc.xml'* using the password from earlier.

```powershell
openssl.exe aes-256-cbc -salt -a -d -in "$VaultTLSCertsDir\KeyShardInfoEnc.xml" -out "$VaultTLSCertsDir\KeyShardInfo.xml"
```

**40)** Load the hashtable object *'$KeyShardinfo'* into your current PowerShell session via...

```powershell
$KeyShardInfo = Import-Clixml "$VaultTLSCertsDir\KeyShardInfo.xml"
```

**41)** \*\***ULTRA MEGA IMPORTANT STEP:**\*\* The Vault Server Key Shards are used to decrypt the Vault Server's underlying storage. At least 3 of the 5 key shards are needed to 'Unseal' the Vault Server (i.e. decrypt its underlying storage). If you do not Unseal the Vault Server, you cannot use it. The idea is to give 5 different employees a single key shard, so that at least 3 employees are required to Unseal the Vault Server. This means that *'$VaultTLSCertsDir\KeyShardInfo.xml'* and *'$VaultTLSCertsDir\KeyShardInfoEnc.xml'* should be removed from the filesystem. But before doing so, make sure you have a **physical paper backup** of the key shards and Initial Root Token (hint: don't send it to a printer...write it down with pen and paper). I recommend placing the physical piece of paper in a physical safe that only an employee at CIO level (or equivalent level of trust) has access to.

**42)** Remove *'$VaultTLSCertsDir\KeyShardInfo.xml'* and *'$VaultTLSCertsDir\KeyShardInfoEnc.xml'* from the file system

```powershell
Remove-Item "$VaultTLSCertsDir\KeyShardInfo.xml" -Force
Remove-Item "$VaultTLSCertsDir\KeyShardInfoEnc.xml" -Force
```

# Configuring and Managing the Vault Server via HTTP API

From here on, we'll be using PowerShell on our localhost Windows machine to manage the Vault Server via its HTTP API. Note that since we setup TLS, all network traffic will be encrypted.

**43)** Unseal (i.e. decrypt underlying storage) the Vault Server

**IMPORTANT NOTE:** You must submit 3 of the 5 Master Key shards in order to Unseal Vault

**IMPORTANT NOTE:** Everytime the Vault Docker Container is restarted, Vault will need to be Unsealed again

```powershell
$VaultServerNetworkLocation = $VaultServerFQDN
$VaultServerPort = "8200"
$VaultServerBaseUri = "https://$VaultServerNetworkLocation" + ":$VaultServerPort/v1"
$VaultAuthToken = $KeyShardInfo.InitialRootToken
$HeadersParameters = @{
    "X-Vault-Token" = $VaultAuthToken
}
$UnsealKeys = $($KeyShardInfo.GetEnumerator() | Where-Object {$_.Name -notmatch "RootToken"}).Value

[System.Collections.ArrayList]$KeyShardsSubmitted = @()
[System.Collections.ArrayList]$KeyShardTracker = @()
foreach ($KeyShard in $UnsealKeys[0..2]) {
    $jsonRequest = @"
{
    "key": "$KeyShard"
}
"@
    # Validate JSON
    try {
        $JsonRequestAsSingleLineString = $jsonRequest | ConvertFrom-Json -ErrorAction Stop | ConvertTo-Json -Compress -ErrorAction Stop
    }
    catch {
        Write-Error "There was a problem with the JSON for Key Shard $KeyShardCounter! Halting!"
        if ($KeyShardTracker.Count -gt 0) {
            Write-Warning "The following Key Shards have been submitted and do not need to be resubmitted:`n$($KeyShardTracker -join "`n")"
        }
        else {
            Write-Warning "No Key Shards have been submitted to the Vault Server thus far."
        }
        break
    }
    $IRMSplatParams = @{
        Uri         = "$VaultServerBaseUri/sys/unseal"
        Headers     = $HeadersParameters
        Body        = $JsonRequestAsSingleLineString
        Method      = "Put"
    }
    $SubmitKeyShard = Invoke-RestMethod @IRMSplatParams
    $null = $KeyShardsSubmitted.Add($SubmitKeyShard)
    $null = $KeyShardTracker.Add($KeyShard)
}
```

**44)** Confirm Vault is Unsealed

```powershell
$ConfirmVaultUnsealed = Invoke-RestMethod -Uri "$VaultServerBaseUri/sys/seal-status" -Headers $HeadersParameters -Method Get

# The below should return '$false'
$ConfirmVaultUnsealed.sealed
```

**45)** Let's Enable Vault's LDAP Authentication Method so that Active Directory users can interact with the Vault Server via its HTTP API. For this, I've created the *'Configure-VaultServerForLDAPAuth'* function (part of the VaultServer Module).

**IMPORTANT NOTE:** Besides configuring Vault for LDAP Authentication, the *'Configure-VaultServerForLDAPAuth'* function also enables the Vault audit log so that there is a record of who/what/when under *'/var/lib/docker/volumes/vaultaudit/_data'* on CentOS7Vault

**IMPORTANT NOTE:** The parameters used in this function check to ensure that the values for *'-BindUserDN'*, *'-LDAPUserOUDN'*, *'-LDAPGroupOUDN'*, *'-LDAPUsersSecurityGroupDN'*, and *'-LDAPVaultAdminsSecurityGroupDN'* do, in fact, exist in Active Directory for your domain. **Your Active Directory implementation is most likely organized differently**, so please be sure to update these values (as well as the values for *'-LDAPServerHostNameOrIP'* and '*-LDAPServicePort'*) as appropriate

```powershell
# Create a pscredential for an existing non-privileged LDAP/AD account ('zero\vault')
# whose sole purpose is allowing the Vault Server to read the LDAP Database. Do not
# use this account for anything else in your Domain. If an account like this doesn't
# exist in AD, please create it.
$LDAPBindCredentials = [pscredential]::new("zero\vault",$(Read-Host -Prompt "Enter Password" -AsSecureString))

# SIDE NOTE: You can configure Vault to allow $null bind, but my function is
# specifically written to not allow that

# NOTE: The below '-PerformOptionalSteps' switch does the following:
#   - Creates backup Vault root token (other than the Initial Root Token) with
#   username 'backupadmin' (this is a Vault Server user, NOT an AD user)
#   - Creates a Vault Policy called 'custom-root' and applies it to the "VaultAdmins"
#   LDAP Security Group (all permissions)
#   - Creates a Vault Policy called 'vaultUsers' and applies it to the "VaultUsers"
#   LDAP Security Group (all permissions except 'delete' and 'sudo')

$ConfigureVaultLDAPSplatParams = @{
    VaultServerNetworkLocation      = $VaultServerFQDN
    VaultServerPort                 = 8200
    VaultAuthToken                  = $VaultAuthToken
    LDAPServerHostNameOrIP          = "ZeroDC01.zero.lab"
    LDAPServicePort                 = 636
    LDAPBindCredentials             = $LDAPBindCredentials
    BindUserDN                      = "cn=vault,ou=OrgUsers,dc=zero,dc=lab"
    LDAPUserOUDN                    = "ou=OrgUsers,dc=zero,dc=lab"
    LDAPGroupOUDN                   = "ou=Groups,dc=zero,dc=lab"
    PerformOptionalSteps            = $True
    LDAPVaultUsersSecurityGroupDN   = "cn=VaultUsers,ou=Groups,dc=zero,dc=lab"
    LDAPVaultAdminsSecurityGroupDN  = "cn=VaultAdmins,ou=Groups,dc=zero,dc=lab"
}
$ConfigureVaultLDAPResult = Configure-VaultServerForLDAPAuth @ConfigureVaultLDAPSplatParams
```

**SIDE NOTE:** You can view information about all current Vault Users by using the VaultServer Module's *'Get-VaultAccessorLookup'* function...

```powershell
Get-VaultAccessorLookup -VaultServerBaseUri $VaultServerBaseUri -VaultAuthToken $VaultAuthToken
```

**46)** Now that we have LDAP Authentication Configured, let's "Login" to the Vault Server as a member of the 'VaultAdmins' Security Group so that we can stop using the Initial Root Token for HTTP API requests

**NOTE:** The term "Login" in this context basically means that we use the 'zero\zeroadmin' AD Account (which is a member of the 'VaultAdmins' Security Group) to request a new Vault Authentication Token. We then use this Authentication Token to make subsequent HTTP API calls as 'zero\zeroadmin' (as opposed to the Initial Root Token that we've been using that was created at the same time the Key Shards were created). This token is valid for 30 days by default, at which point 'zero\zeroadmin' can choose to renew that specific token, or simply request a new one via the same method outlined below (i.e. the *'Get-VaultLogin'* function from the VaultServer PowerShell Module).

```powershell
$DomainCreds = [pscredential]::new("zero\zeroadmin",$(Read-Host -Prompt "Enter Password" -AsSecureString))
$ZeroAdminToken = Get-VaultLogin -VaultServerBaseUri $VaultServerBaseUri -DomainCredentialsWithAccessToVault $DomainCreds
```

**47)** Confirm that *'$ZeroAdminToken'* works by trying and access a Vault HTTP API Endpoint that requires a Token

```powershell
$ConfirmLogin = Invoke-RestMethod -Uri "$VaultServerBaseUri/sys/auth" -Headers @{"X-Vault-Token" = "$ZeroAdminToken"} -Method Get
```

**48)** Next, I recommend saving your Vault Auth Token (i.e. *'$ZeroAdminToken'*) to the Windows Credential Manager via the *'Manage-StoredCredentials'* function from the VaultServer Module

```powershell
Manage-StoredCredentials -AddCred -Target $VaultServerBaseUri -User "zero\zeroadmin" -Pass $ZeroAdminToken -Comment "Vault Authentication Token for zero\zeroadmin"

# NOTE: This method of storing credentials ensures that only zero\zeroadmin
# can view the Vault Auth Token in the Windows Credential Manager on the localhost.

# NOTE: Retrieve the Vault Auth Token form the Windows Credential Manager
# by doing the following:
$MyVaultAuthToken = $(Manage-StoredCredentials -GetCred -Target $VaultServerBaseUri).Password
```

**IMPORTANT NOTE:** From now on, we'll be using the token assigned to 'zero\zeroadmin' (who effectively has root access on the Vault Server since the account is part of the VaultAdmins Security Group in AD) to perform all subsequent operations. This is more desirable than continuing to use the Initial Root Token.

# Configuring the Vault Server for SSH Key Management

I've created the *'Configure-VaultServerForSSHManagement'* function (part of the VaultServer Module) to execute all of the HTTP API requests needed to configure the Vault Server for SSH Key Management. You can use *'Get-Help Configure-VaultServerForSSHManagement'* for details on exactly what it does, but to summarize:

1. The Vault SSH User/Client Key Signer is enabled
2. A Certificate Authority (CA) for the SSH User/Client Key Signer is created
3. The Vault SSH Host/Machine Key Signer is enabled
4. A Certificate Authority (CA) for the SSH Host/Machine Key Signer is created
5. The Vault the SSH User/Client Signer Role Endpoint is configured
6. The Vault the SSH Host/Machine Signer Role Endpoint is configured

```powershell
$ConfigureVaultSSHMgmt = Configure-VaultServerForSSHManagement -VaultServerBaseUri $VaultServerBaseUri -VaultAuthToken $ZeroAdminToken
```

**Now our Vault Server is configured for SSH Key Management!**

# Configuring Workstations For Public Cert Authentication

Next, we need to configure our Windows Servers/Workstations to use the Vault Server for SSH Key Management. To do so, I've created the function *'Add-CAPubKeyToSSHAndSSHDConfig'* (part of the VaultServer PowerShell Module).

**NOTE:** The below *'-AuthorizedPrincipalsUserGroup'* parameter makes it so that any user within the specified Groups is allowed to ssh into the Windows machine that the *'Add-CAPubKeyToSSHAndSSHDConfig'* function is run on. So far, the function can only handle four groups:

1. "LocalAdmins" : Corresponds to the "Administrators" Security Group on the Local Machine
2. "LocalUsers" : Corresponds to the "Users" Security Group on the Local Machine
3. "DomainUsers" : Corresponds to the "Domain Users" Security Group in Active Directory
4. "DomainAdmins" : Corresponds to the "Domain Admins" Security Group in Active Directory

Currently, the *'Add-CAPubKeyToSSHAndSSHDConfig'* function does not have functionality that allows for specifying Security Groups other than the ones listed above. HOWEVER, there is another available parameter called *'-AuthorizedUserPrincipals'* that allows you to grant specific users ssh access. For example, using the following string array with *'-AuthorizedUserPrincipals'* will grant the Local User Account *'$env:ComputerName\pdadmin'* and the Domain Account 'zero\zeroadmin' ssh access to *'$env:ComputerName'*:

```powershell
@("zeroadmin@zero","pdadmin@$env:ComputerName")
```

Please note that the strings in the string array for the *'-AuthorizedUserPrincipals'* parameter **MUST** follow the format \<User\>@\<DomainShortName\> or \<User\>@\<LocalHostName\>

**NOTE:** The below *'-VaultSSHHostSigningUrl'* parameter is optional, but is highly recommended. Use it to have the localhost's SSH Host Key (i.e. *'C:\ProgramData\ssh\ssh_host_rsa_key.pub'*) signed by the Vault Server SSH Host Signer (outputs *'C:\ProgramData\ssh\ssh_host_rsa_key-cert.pub'*)

Now, let's run the following on our localhost Windows machine that we've been using this entire time...

```powershell
$AddCAPubKeyToSSHAndSSHDConfigSplatParams = @{
    PublicKeyOfCAUsedToSignUserKeysAsString     = $ConfigureVaultSSHMgmt.SSHClientSignerCAPublicKey
    PublicKeyOfCAUsedToSignHostKeysAsString     = $ConfigureVaultSSHMgmt.SSHHostSignerCAPublicKey
    AuthorizedPrincipalsUserGroup               = @("LocalAdmins","LocalUsers","DomainAdmins","DomainUsers")
    VaultSSHHostSigningUrl                      = "$VaultServerBaseUri/ssh-host-signer/sign/hostrole"
    VaultAuthToken                              = $ZeroAdminToken
}
$AddCAPubKeysResult = Add-CAPubKeyToSSHAndSSHDConfig @AddCAPubKeyToSSHAndSSHDConfigSplatParams
```

**NOTE:** *'$AddCAPubKeysResult'* is a PSCustomObject with Properties 'SignSSHHostKeyResult' and 'FilesUpdated', so you can review what the *'Add-CAPubKeyToSSHAndSSHDConfig'* function did via...

```powershell
$AddCAPubKeysResult.SignSSHHostKeyResult
$AddCAPubKeysResult.FilesUpdated
```

**NOTE:** If this is your first time running *'Add-CAPubKeyToSSHAndSSHDConfig'* on the Windows machine, *'$AddCAPubKeysResult'* **MUST** have output. If you run this function multiple times, it may or may not need to change/update things, so *'$AddCAPubKeysResult'* may or may not have any output.

On Windows workstations other than the one you've been using this entire time, you won't have the PSCustomObject *'$ConfigureVaultSSHMgmt'* that contains the SSHClientSignerCAPublicKey and SSHHostSignerCAPublicKey. So, you can use the following parameters to query public Vault Server HTTP API endpoints that contains this information (no Vault Login/Authentication needed)...

```powershell
$AddCAPubKeyToSSHAndSSHDConfigSplatParams = @{
    PublicKeyOfCAUsedToSignUserKeysVaultUrl     = "$VaultServerBaseUri/ssh-client-signer/public_key"
    PublicKeyOfCAUsedToSignHostKeysVaultUrl     = "$VaultServerBaseUri/ssh-host-signer/public_key"
    AuthorizedPrincipalsUserGroup               = @("LocalAdmins","LocalUsers","DomainAdmins","DomainUsers")
    VaultSSHHostSigningUrl                      = "$VaultServerBaseUri/ssh-host-signer/sign/hostrole"
    VaultAuthToken                              = $ZeroAdminToken
}
$AddCAPubKeysResult = Add-CAPubKeyToSSHAndSSHDConfig @AddCAPubKeyToSSHAndSSHDConfigSplatParams
```

Now that we ran *'Add-CAPubKeyToSSHAndSSHDConfig'* our localhost Windows workstation, we're ready to create SSH Credentials that will allow us to ssh to any Windows machine that also ran the *'Add-CAPubKeyToSSHAndSSHDConfig'* function. I've created the function *'New-SSHCredentials'* for this purpose.

Let's say my localhost Windows machine is 'Win10Jump.zero.lab' and the remote Windows machine that I want to ssh to is 'Win16Serv.zero.lab'. Let's assume that I already ran the *'Add-CAPubKeyToSSHAndSSHDConfig'* function on both machines. And let's also assume that I'm logged into 'Win10Jump' with a Active Directory Account that has access to the Vault Server (i.e. an account in the 'VaultAdmins' or 'VaultUsers' Security Groups in AD). From a fresh WinPS or PSCore session on 'Win10Jump', do the following:

```powershell
# Install and Import the WinSSH and VaultServer PowerShell Modules if they aren't already
$ModulesToInstallAndImport = @("WinSSH","VaultServer")
foreach ($ModuleName in $ModulesToInstallAndImport) {
    if (![bool]$(Get-Module -ListAvailable $ModuleName)) {
        Install-Module $ModuleName
    }
    if (![bool]$(Get-Module $ModuleName)) {
        Import-Module $ModuleName
    }
}

# Create Splat Parameters that will be used by the New-SSHCredentials function
# (The below $NewSSHKeyName can be whatever you want, but I recommend the following)
$NewSSHKeyName = $($(whoami) -split "\\")[-1] + "_" + $(Get-Date -Format MMddyy)
$NewSSHCredentialsSplatParams = @{
    VaultServerBaseUri      = $VaultServerBaseUri
    NewSSHKeyName           = $NewSSHKeyName
    BlankSSHPrivateKeyPwd   = $True
    AddToSSHAgent           = $True
}
```

**IMPORTANT NOTE:** The above *'-BlankSSHprivateKeyPwd'* switch is necessary, otherwise OpenSSH-Win64 has issues. If you are concerned about having an unprotected private key on the filesystem, I recommend adding the *'-RemovePrivateKey'* switch (as long as you also include the *'-AddToSSHAgent'* switch)

If you already have your Vault Authentication Token available (like via the Windows Credential Manager outlined earlier), then add it to the Splat Parameters...

```powershell
$MyVaultAuthToken = $(Manage-StoredCredentials -GetCred -Target $VaultServerBaseUri).Password
$NewSSHCredentialsSplatParams.Add("VaultAuthToken",$MyVaultAuthToken)
```

...otherwise use your Domain Credentials...

```powershell
$DomainCredentialsWithAccessToVault = [pscredential]::new("$(whoami)",$(Read-Host -Prompt "Enter Password for $(whoami)" -AsSecureString))
$NewSSHCredentialsSplatParams.Add("DomainCredentialsWithAccessToVault",$DomainCredentialsWithAccessToVault)
```

Finally, create your new SSH keys...

```powershell
$NewSSHCredsResult = New-SSHCredentials @NewSSHCredentialsSplatParams
```

The PSCustomObject *'$NewSSHCredsResult'* should give you output like...

```powershell
PublicKeyCertificateAuthShouldWork : True
FinalSSHExeCommand                 : ssh zeroadmin@zero@<RemoteHost>
PrivateKeyPath                     : C:\Users\zeroadmin\.ssh\zeroadmin_071918
PublicKeyPath                      : C:\Users\zeroadmin\.ssh\zeroadmin_071918.pub
PublicCertPath                     : C:\Users\zeroadmin\.ssh\zeroadmin_071918-cert.pub
```

**IMPORTANT NOTE:** If *'$NewSSHCredsResult.FinalExeCommand'* gives you output like...

```powershell
PS C:\Users\zeroadmin> $NewSSHCredsResult.FinalSSHExeCommand
ssh -o "IdentitiesOnly=true" -i "C:\Users\zeroadmin\.ssh\zeroadmin_071718" -i "C:\Users\zeroadmin\.ssh\zeroadmin_071718-cert.pub" zeroadmin@zero@<RemoteHost>
```

...then you most likely have too many identities loaded in your ssh-agent (you should have 5 max since most remote sshd servers have a limit on how many identities can be attempted in a single connection attempt). Double-check the identities loaded in your ssh-agent by doing...

```powershell
ssh-add -L
```

To remove ALL identities in your ssh-agent, you can do...

```powershell
ssh-add -D
```

To remove a SPECIFIC identity in your ssh-agent, you can do...

```powershell
ssh-add -D <FullPathToCorrespondingSSHPrivateKey>
```

...then add your new ssh private key to the ssh-agent...

```powershell
ssh-add "$($NewSSHCredsResult.PrivateKeyPath)"
```

Finally, you should now be able to use the SSH Command outlined by the *'FinalSSHExeCommand'* property in order to ssh to 'Win16Serv.zero.lab' without any extra ssh.exe parameters...

```powershell
ssh zeroadmin@zero@Win16Serv.zero.lab
```

**SIDE NOTE:** If you ever need to check what the ssh command **should** work for a particular ssh key/cert, you can use the VaultServer Module's *'Get-SSHClientAuthSanity'* function on the private key, public key, or public cert...

```powershell
PS C:\Users\zeroadmin> Get-SSHClientAuthSanity -SSHKeyFilePath "C:\Users\zeroadmin\.ssh\zeroadmin_071918"

PublicKeyCertificateAuthShouldWork FinalSSHExeCommand
---------------------------------- ------------------
                              True ssh zeroadmin@zero@<RemoteHost>


PS C:\Users\zeroadmin> Get-SSHClientAuthSanity -SSHKeyFilePath "C:\Users\zeroadmin\.ssh\zeroadmin_071918.pub"

PublicKeyCertificateAuthShouldWork FinalSSHExeCommand
---------------------------------- ------------------
                              True ssh zeroadmin@zero@<RemoteHost>


PS C:\Users\zeroadmin> Get-SSHClientAuthSanity -SSHKeyFilePath "C:\Users\zeroadmin\.ssh\zeroadmin_071918-cert.pub"

PublicKeyCertificateAuthShouldWork FinalSSHExeCommand
---------------------------------- ------------------
                              True ssh zeroadmin@zero@<RemoteHost>
```

# Manage Vault Tokens, Not SSH Keys

So, what happens if an employee leaves the company or I otherwise need to revoke ssh access for a particular user?

Well, the *'Configure-VaultServerForSSHManagement'* function that we used earlier configures Vault such that any ssh key pair generated via the *'New-SSHCredentials'* function is only valid for 24 hours. So there's really no need to manually revoke SSH Keys (at least not directly).

What we should be focused on managing is the Vault Token assigned to the Active Directory user with access to the Vault Server.

If you disable an account in AD, then the user will no longer be able to request a Token from the Vault Server, which means they will not be able to create SSH Keys that have access to anything. So, let's say John Smith ('zero\jsmith') left the company. As s VaultAdmin (member of the 'VaultAdmins' Security Group in AD), you can use the *'Revoke-VaultToken'* function from th VaultServer Module to revoke jsmith's token...

```powershell
Revoke-VaultToken -VaultServerBaseUri $VaultServerBaseUri -VaultAuthToken $ZeroAdminToken -VaultUserToDelete "jsmith"
```

Just make sure the AD Account that runs the *'Revoke-VaultToken'* function is a member of the 'VaultAdmins' Security Group, and you shouldn't have any issues.

The last thing you should keep in mind with regards to Vault Tokens is that they are good for 30 days by default. On or before day 30, the user can either renew the same token, or the user can request and receive a new Token via the 'Get-VaultLogin' function (using his/her Domain credentials).

# Conclusion

Hashicorp's Vault Server is a great tool for managing secrets in general (not just SSH keys) - so much so that I believe that it should be a part of everyone's infrastructure. (I don't work for Hashicorp, I just really like this product). I've been using Vault to manage SSH remoting in a hybrid Windows/Linux environment for about a month now, and things have been working well.

One thing that this tutorial didn't cover is how to configure Linux workstations to generate new SSH credentials using the Vault Server. I have plans to extend the functionality of *'New-SSHCredentials'* so that it can be used with PSCore on Linux, but I haven't done so just yet.

The last thing I want to mention is that while this tutorial focused on using LDAP/AD as the Vault Server's backend authentication mechanism, Vault supports a bunch of other authentication backends (like GitHub or Azure accounts). I'm planning on writing some more PowerShell functions for my VaultServer Module that can configure Vault to use those backend auth mechanisms. You can read more about these auth mechanisms here: [https://www.vaultproject.io/api/auth/index.html](https://www.vaultproject.io/api/auth/index.html)

If anyone would like to contribute to my VaultServer Module, you'd be very welcome!
