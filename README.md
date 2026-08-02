# Active Directory Home Lab: Domain Controller and Domain Join in Azure

A hands-on home lab where I built a Windows Server domain controller in Microsoft Azure, set up Active Directory, created organisational units, groups and users, and joined a Windows client PC to the domain. It also documents a real networking problem I ran into and how I diagnosed and fixed it.

**Environment**
- Microsoft Azure virtual machines (built and managed from a MacBook over Remote Desktop)
- Windows Server as the domain controller
- Windows client PC as the machine joined to the domain
- Domain: `lab.test`

---

## Why I built this

I am a data analyst moving into IT support and cybersecurity. I wanted to understand, hands-on, how a corporate environment manages identity and devices: how staff accounts are created, how a computer is joined to a domain, and how central control actually works. I have used Azure for testing in my current role, so I built the whole lab on Azure VMs rather than local virtualisation.

Everything documented here is something I did myself. Where I got stuck, I have written down what went wrong and how I solved it, because that troubleshooting is the part I learned the most from.

---

## Part 1: Installing the server roles

On the Windows Server VM, I used Server Manager to add the roles needed for a domain controller and a small branch network.

![Server Manager add roles](images/image1.png)
![Selecting roles](images/image2.png)
![Role selection continued](images/image3.png)
![Role selection continued](images/image4.png)

Roles selected:
- Active Directory Domain Services
- DHCP Server
- DNS Server
- Print and Document Services
- Web Server (IIS)

I made sure **Group Policy Management** was included, since that is the tool used to push settings out to the domain later.

![Confirm role installation](images/image5.png)

The installation took around 5 to 10 minutes. After it finished, I rebooted the server.

## Part 2: Promoting the server to a domain controller

With the roles installed, I promoted the server to a domain controller and created a new forest for the domain `lab.test`.

![Promote to domain controller](images/image6.png)
![Deployment configuration](images/image7.png)
![Domain controller options](images/image8.png)
![DNS options](images/image9.png)
![Additional options](images/image10.png)
![Paths](images/image11.png)
![Review options](images/image12.png)
![Prerequisites check](images/image13.png)
![Installation](images/image14.png)
![Installation complete](images/image15.png)

This step also takes 5 to 10 minutes, and the virtual machine automatically reboots when it finishes. After the reboot the server is a working domain controller for `lab.test`.

## Part 3: Building the organisational structure

I created an organisational unit (OU) called **branch1** under `lab.test`.

![Create branch1 OU](images/image16.png)

Under **branch1**, I created three more OUs to keep things organised: **Users**, **Computers**, and **Groups**.

![Sub-OUs](images/image17.png)
![Sub-OUs](images/image18.png)
![Sub-OUs](images/image19.png)
![Sub-OUs](images/image20.png)
![Sub-OUs](images/image21.png)

Organising a domain into OUs matters because you can apply different policies to different OUs later. It mirrors how a real organisation separates branches, departments or campuses.

## Part 4: Groups and users

I created a security group, then created a user, then created a second user by **copying** the first.

![Create group](images/image22.png)

When you create a user by copying an existing one, the new user is automatically added to the same groups as the original. So copying a member of the IT Workers group produces a new user who is also in IT Workers, without adding the group by hand.

![Copy user inherits group](images/image23.png)

I also practised **unlocking** and **disabling** accounts.

![Unlock or disable account](images/image24.png)

An important real-world note I took here: before unlocking an account you should verify with HR that the person is still employed, and disabling an account is how you cut off access when someone leaves. In a real environment these are security actions, not just clicks.

## Part 5: Moving a user between branches

I moved a user (Liam) from **branch1** to **branch2**.

![Move user between OUs](images/image25.png)

Moving a user between OUs is mostly about which Group Policy and which resources (like the printers for each branch) apply to them. When someone changes campus or department, this is the action you take.

![Branch structure](images/image26.png)
![Branch structure](images/image27.png)
![Org chart in Teams](images/image28.png)

The organisation structure set here is also what can be reflected in an org chart in Microsoft Teams.

## Part 6: Case study, a name change with the old email kept

Scenario: Emma Chen changes her last name to Emma Bell, but still needs to receive email sent to her old address.

![Name change case study](images/image29.png)
![Adding a proxy address](images/image30.png)
![Result](images/image31.png)

This is a common real request. The user's display name and primary address change to the new name, while the old address is kept as an additional address so nothing sent to it is lost.

## Notes: finding a user quickly

When someone calls in to reset their password, you need to find them in Active Directory fast.

![Find user path](images/image32.png)
![Search for user](images/image33.png)

Knowing how to locate a user by name and see their OU path is a basic but constant help desk task.

## Part 7: Joining a PC to the domain

On the Windows client VM, I joined the machine to the `lab.test` domain so it could be managed centrally and log in with domain accounts.

![Join PC to domain](images/image34.png)
![System properties](images/image35.png)
![Enter domain](images/image36.png)
![Domain credentials](images/image37.png)

---

## Troubleshooting: when DNS or networking is not working

This is the part I learned the most from. After joining, I hit the error that the domain controller could not be contacted. Here is how I worked through it.

I opened the network adapter settings on the client using `ncpa.cpl`.

![Network adapter settings](images/image38.png)

I checked that the client and the domain controller were on the same virtual network and subnet.

![Checking VNet and subnet](images/image39.png)

I manually set the client's DNS server to the domain controller's IP address.

![Manually set DNS to DC IP](images/image40.png)

**It still did not work in my case.** After troubleshooting, it turned out the client's virtual network and subnet were **not the same as the domain controller's**. The two machines could not talk to each other even though I had specified the DNS IP address, because in Azure each VM had been placed on its own separate virtual network. Once I put both machines on the **same virtual network and subnet**, it worked.

![Fixing the network](images/image41.png)
![Confirming connectivity](images/image42.png)
![Domain join succeeds](images/image43.png)
![Client visible in AD](images/image44.png)
![Working result](images/image45.png)

The root cause was that the DNS server was not resolving to the domain controller, because the client was not on the same network as the domain controller in the first place. Setting the DNS IP alone was not enough when the two VMs were on isolated networks.

---

## What I learned

- A domain controller centralises identity and control: one account lets a user log in on a joined PC, and disabling that one account cuts off their access everywhere.
- Organisational units are how you separate branches or departments so different policies and resources apply to each.
- Creating a user by copying another one inherits the original's group memberships, which saves time when onboarding into an existing team.
- Account actions like unlock and disable are security actions. In the real world you verify with HR before unlocking, and you disable accounts when people leave.
- **In Azure, two VMs must be on the same virtual network and subnet to communicate.** Setting the DNS server to the domain controller's IP does nothing if the machines are on separate networks. This was the main problem I solved, and understanding it taught me how Azure networking underpins everything else.

## How this relates to the cloud

This lab is on-premises style Active Directory, running on a server I manage. Modern cloud-first organisations often do the equivalent with Microsoft Entra ID and Intune instead, managing identity and devices over the internet rather than through a local domain controller. The goals are the same, central identity and central control, and building this lab helped me understand both the traditional and the cloud approach.

