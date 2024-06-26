# Ansible STIG Windows 2022

## Better way of creating a certificate
### Stolen from here 
https://woshub.com/powershell-remoting-over-https/  
https://woshub.com/how-to-create-self-signed-certificate-with-powershell/

```
$hostName = $env:COMPUTERNAME
$hostIP=(Get-NetAdapter| Get-NetIPAddress).IPv4Address|Out-String
$srvCert = New-SelfSignedCertificate -DnsName $hostName,$hostIP -CertStoreLocation Cert:\LocalMachine\My
$srvCert
```
## Create the new listener
```
New-Item -Path WSMan:\localhost\Listener\ -Transport HTTPS -Address * -CertificateThumbPrint $srvCert.Thumbprint -Force
```
## Create Firewall Rule
```
New-NetFirewallRule -Displayname 'WinRM - Powershell remoting HTTPS-In' -Name 'WinRM - Powershell remoting HTTPS-In' -Profile Any -LocalPort 5986 -Protocol TCP
```
## Restart the Service
```
Restart-Service WinRM
```
### You may need to reboot the server.


## Export the Certificate (optional)
```
Export-Certificate -Cert $srvCert -FilePath c:\PS\SSL_PS_Remoting.cer
```

### A way to create a subject alternative certificate for future use if needed
```
New-SelfSignedCertificate -TextExtension @("2.5.29.17={text}IPAddress=10.10.10.22&DNS=powerbi&DNS=powerbi.mydomain.com")

## Configure and Enable WMI on the Windows Server
Create Certificate for WMI in Powershell.  Update the hostname in the commands below by removing the <>.
```
New-SelfSignedCertificate -DnsName "<your hostname here>" -CertStoreLocation Cert:\LocalMachine\My
```
Import the Certificate in Powershell.  Replace the CertificateThumbprint with the output you get from above.
```
winrm create winrm/config/Listener?Address=*+Transport=HTTPS '@{Hostname="<your hostname here>"; CertificateThumbprint="<Thumbprint Goes Here>"}'
```
## Check your WMI configuration
```
winrm quickconfig -transport:https
```
Example output:
```
Listener
    Address = *
    Transport = HTTPS
    Port = 5986
    Hostname = powerbi
    Enabled = true
    URLPrefix = wsman
    CertificateThumbprint = B2614A66C1A6F5086D083F371602BB591B299A40
    ListeningOn = 10.255.255.5, 127.0.0.1, ::1, fe80::af52:d59d:8500:9ded%3
```
## Download the DOD Stig Ansible Code
Check under Ansible Content.  You will have to hit the next page button to see windows 2022.

https://public.cyber.mil/stigs/supplemental-automation-content/

## Install Powershell on Linux
You may need to search around to get the newest version and update this URL.
```
wget https://github.com/PowerShell/PowerShell/releases/download/v7.4.0/powershell-7.4.0-linux-x64.tar.gz
mkdir /opt/powershell
cp powershell-7.4.0-linux-x64.tar.gz /opt/powershell
cd /opt/powershell
tar -zxvf powershell-7.4.0-linux-x64.tar.gz
```
There are dependency conflicts so it required ```--allowerasing``` to resolve it.
```
sudo dnf install -y curl libunwind libicu libcurl openssl libuuid.x86_64 bash --allowerasing
sudo chmod +x pwsh
sudo ln -s /opt/powershell/pwsh /usr/bin/pwsh
```

## Test Commands
Check Basic Connectivity to the hosts.  
```ansible all -m ping -u Administrator -i hosts```  

There is a point where it wasn't finding the correct location for python3  
```ansible all -m ping -u Administrator -e 'ansible_python_interpreter=/usr/bin/python3' -i hosts```  

This runs a test to see if the playbook will run without making any changes.  
```ansible-playbook -v -b -i hosts --check site.yml```

## Execute the Playbook Command
```ansible-playbook -v -b -i hosts site.yml```
Fore more Verbose output use -vvv
```ansible-playbook -v -b -i hosts site.yml -vvv```

## Example ```hosts``` file
The other files did not need to be modified.
```
[windows]
10.255.255.5

[windows:vars]
ansible_user=Administrator
ansible_password=-<YOUR PASSWORD HERE> ## Update your password here
ansible_connection=winrm
ansible_winrm_server_cert_validation=ignore
ansible_winrm_transport=ntlm
ansible_python_interpreter=/usr/bin/python3
ansible_become_method=runas
ansible_runas_user=Administrator
```
