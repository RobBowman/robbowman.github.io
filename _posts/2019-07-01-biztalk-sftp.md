# BizTalk 2016 sFTP 
I recently worked on a project that required BizTalk to collect files from a 3rd party's server using the SFTP adapter. I've had to do this many times over the years but each time I've had to refer back to my notes. Also, the process for current versions has changed a little recently, so I thought the subject worthy of a new blog post.

Since BizTalk 2016, the SFTP adapter provided with BizTalk is implemented as a wrapper around  WinSCP. This introduces the dependency that WinSCP must be installed on the BizTalk server before the adapter can be used, as described in the Microsoft doc: https://docs.microsoft.com/en-us/biztalk/core/sftp-adapter

## Version Problems - Only if Pre CU5
There have been problems aligning the version of the WinSCP that's installed with the version required by the BizTalk adapter. This is documented in the following post: http://microsoftintegration.guru/2016/12/19/biztalk-2016-sftp-adapter-how-to-set-it-up-properly/

## Feature Pack 3
I was using BizTalk Server 2016 with Feature Pack 3 CU5 and found that the version of winscp.exe and dll required was v5.13.1

## Creating a Local SFTP Server
It is useful to have a local SFTP server that can be used to test BizTalk SFTP Receive or Send Ports. 

Windows Server 2019 ships with an sFTP server called OpenSSH. This is hosted on GitHub and it seems to be maintained by Microsoft staff. The following describes how to configure a BizTalk development VM as a sFTP server:
 
OpenSSH can be installed from chocolatey (www.chocolatey.org) by running the following from an admin command prompt: "choco install openssh"

This should install to c:\program files\openssh-win64

Run  .\install-sshd.ps1 from c:\program files\openssh-win64. This creates two windows services:

OpenSSH Authentication Agent
OpenSSH SSH Server
 Set both of these services to Automatic and start them

 Run .\ssh-keygen.exe -A Note: if an error check that OpenSSH SSH Server service is running

 Run the following to add firewall rule

netsh advfirewall firewall add rule name=sshd dir=in action=allow protocol=TCP localport=22

 Create a local user called sFtpTest

 Make an addition to the following config file: C:\ProgramData\ssh\sshd_config (note the file has no extension):

 At the foot of the file I added the line: "AllowGroups users"

 This just enables any member of the local group "users" to connect - the new sFtpTest user is a member of this group

## Testing Connectivity from the WinSCP Client
Before configuring the BizTalk receive location I decided to check connectivity to the 3rd Party's site using the WinSCP client application. 

## Multi-Factor
In my case, the external site had been secured using what's known as multi-factor authentication. This meant that in order to connect, my client had to provide a private key (provided to me by the 3rd Party) in addition to a username & password.

### Config for Private Key not Obvious - Pageant
The following screenshot illustrates the config for my WinSCP connection:
![](/images/biztalk-sftp/WinSCP1.png)

Notice that the text box for "Private key file" is empty. Instead a tool called "Pagent" (https://winscp.net/eng/docs/ui_pageant) is used to provide the private key.

I had to:
run the Pageant.exe
click "Add Key"
browse to the private key pfx file
enter the passphrase associated with the private key

![](/images/biztalk-sftp/Pageant.png)

## BizTalk Config
Fortunately, configuration of the BizTalk adapter did not incur the complexity of having to use Pageant. I was able to simply enter the username, password and path to private key directly into the config dialog for the BizTalk receive location, as shown in the following screenshot:

![](/images/biztalk-sftp/WinSCP2.png)

