.. title:: Nutanix .Next Files HOL

.. toctree::
  :maxdepth: 2
  :caption: Appendix
  :name: _appendix
  :hidden:

  tools_vms/linux_tools_vm
  tools_vms/windows_tools_vm

.. _files:

-----
Files
-----

*The estimated time to complete this lab is 75 minutes.*

Overview
++++++++

Traditionally, file storage has been yet another silo within IT, introducing unnecessary complexity and suffering from the same issues of scale and lack of continuous innovation seen in SAN storage. Nutanix believes there is no room for silos in the Enterprise Cloud. By approaching file storage as an app, running in software on top of a proven HCI core, Nutanix Files  delivers high performance, scalability, and rapid innovation through One Click management.

**In this lab you will step through a Files deployment, manage SMB shares and NFS exports, scale out the environment, and explore upcoming Files features. The lab will provide key considerations around deployment, configuration, and use cases.**


Lab Setup
+++++++++

This lab requires applications provisioned as part of the :ref:`windows_tools_vm`.

If you have not yet deployed this VM, see the linked steps and deploy this VM after starting the installation of files in the next section.

Deploying Files
+++++++++++++++

#. In **Prism > File Server**, click **+ File Server** to open the **New File Server Pre-Check** dialogue.

   .. figure:: images/1.png

   For the purpose of saving time, the Files 3.6 package has already been uploaded to your cluster. Files binaries can be downloaded directly through Prism or uploaded manually.

   .. figure:: images/2.png

   Additionally, the cluster's **Data Services** IP Address has already been configured (*10.XX.YY.38*). In a Files cluster, storage is presented to the Files VMs as a Volume Group via iSCSI, hence the dependency on the Data Services IP.

   .. note::

     If staging your own environment, the Data Services IP can be easily configured by selecting :fa:`gear` **> Cluster Details**, specifying the **iSCSI Data Services IP**, and clicking **Save**. Currently, the Data Services IP must be in the same subnet as your CVMs.

   Lastly Files will ensure that at least 1 network has been configured on the cluster. A minimum of 2 networks are recommended to have segmentation between the client side and storage side networks.

#. Click **Continue**.

   .. figure:: images/3.png

#. Fill out the following fields:

   - **Name** - *Intials*-Files (e.g. XYZ-Files)
   - **Domain** - ntnxlab.local
   - **File Server Size** - 1 TiB

   .. figure:: images/4.png

   .. note::

     Clicking **Custom Configuration** will allow you to alter the scale up and scale out sizing of the Files VMs based on User and Throughput targets. It also allows for manual sizing of the Files cluster.

     .. figure:: images/5.png

#. Click **Next**.

#. Select the **Primary - Managed** VLAN for the **Client Network**.

   Each Files VM will consume a single IP on the client network.

   .. note::

     It is typically desirable in production environments to deploy Files with dedicated virtual networks for client and storage traffic. When using two networks, Files will, by design, disallow client traffic the storage network, meaning VMs assigned to the primary network will be unable to access shares.

   .. note::

     As this is an AHV managed network, configuration of individual IPs is not necessary. In an ESXi environment, or using an unmanaged AHV network, you would specify the network details and available IPs as shown below.

     .. figure:: images/6.png

#. Specify your cluster's **Domain Controller** VM IP (found in :ref:`stagingdetails`) as the **DNS Resolver IP** (e.g. 10.XX.YY.40). Leave the default (cluster) NTP Server.

   .. raw:: html

     <strong><font color="red">In order for the Files cluster to successfully find and join the NTNXLAB.local domain it is critical that the DNS Resolver IP is set to the Domain Controller VM IP FOR YOUR CLUSTER. By default, this field is set to the primary Name Server IP configured for the Nutanix cluster, this value is incorrect and will not work.</font></strong>

   .. figure:: images/7.png

#. Click **Next**.

#. Select the **Primary - Managed** VLAN for the Storage Network.

   Each Files VM will consume a single IP on the storage network.

   .. figure:: images/8.png

#. Click **Next**.

#. Fill out the following fields:

   - Select **Use SMB Protocol**
   - **Username** - Administrator@ntnxlab.local
   - **Password** - nutanix/4u
   - Select **Make this user a File Server admin**
   - Select **Use NFS Protocol**
   - **User Management and Authentication** - Unmanaged

   .. figure:: images/9.png

   .. note:: In unmanaged mode, users are only identified by UID/GID. In Files 3.5, Files supports both NFSv3 and NFSv4

#. Click **Next**.

   By default, Files will automatically create a Protection Domain to take daily snapshots of the Files cluster and retain the previous 2 snapshots. After deployment, the snapshot schedule can be modified and remote replication sites can be defined.

   .. figure:: images/10.png

#. Click **Create** to begin the Files deployment.

#. Monitor deployment progress in **Prism > Tasks**.

   Deployment should take approximately 10 minutes.

   .. figure:: images/11.png

   .. note::

     If you receive a warning regarding DNS record validation failure, this can be safely ignored. The shared cluster does not use the same DNS servers as your Files cluster, and as a result is unable to resolve the DNS entries created when deploying Files.

#. While waiting for the file server deployment, if you have not already done so deploy the Windows Tools VM.

#. Connect to the Windows Tools VM via RDP or console

#. Download the sample files for File Analytics to the Tools VM:

   - `https://peerresources.blob.core.windows.net/sample-data/SampleData_Small.zip <https://peerresources.blob.core.windows.net/sample-data/SampleData_Small.zip>`_

#. Upon completion, return to **Prism > File Server** and select the *Initials*\ **-Files** server and click **Protect**.

   .. figure:: images/12.png

#. Observe the default Self Service Restore schedules, this feature controls the snapshot schedule for Windows' Previous Versions functionality. Supporting Previous Versions allows end users to roll back changes to files without engaging storage or backup administrators. Note these local snapshots do not protect the file server cluster from local failures and that replication of the entire file server cluster can be performed to remote Nutanix clusters. Click **Close**.

   .. figure:: images/13.png

Using SMB Shares
++++++++++++++++

In this exercise you will create and test a SMB share, used to support home directories, user profiles, and other unstructured file data such as departmental shares commonly accessed by Windows clients.

Creating the Share
..................

#. In **Prism > File Server**, click **+ Share/Export**.

#. Fill out the following fields:

   - **Name** - Marketing
   - **Description (Optional)** - Departmental share for marketing team
   - **File Server** - *Initials*\ **-Files**
   - **Share Path (Optional)** - Leave blank. This field allows you to specify an existing path in which to create the nested share.
   - **Max Size (Optional)** - Leave blank. This field allows you to set a hard quota for the individual share.
   - **Select Protocol** - SMB

   .. figure:: images/14.png

#. Click **Next**.

#. Select **Enable Access Based Enumeration** and **Self Service Restore**.

   .. figure:: images/15.png

   Because this is a single node AOS cluster and therefore a single file server VM, all shares will be **Standard** shares. A Standard share means that all top level directories and files within the share, as well as connections to the share, are served from a single file server VM.

   If this were a three node Files cluster or larger you’d have an option to create a **Distributed** share.  Distributed shares are appropriate for home directories, user profiles, and application folders. This type of share shards top level directories across all Files VMs and load balances connections across all Files VMs within the Files cluster.

   **Access Based Enumeration (ABE)** ensures that only files and folders which a given user has read access are visible to that user. This is commonly enabled for Windows file shares.

   **Self Service Restore** allows users to leverage Windows Previous Version to easily restore individual files to previous revisions based on Nutanix snapshots.

#. Click **Next**.

#. Review the **Summary** and click **Create**.

   .. figure:: images/16.png

Testing the Share
.................

#. Connect to your *Initials*\ **-ToolsVM** via RDP or console.

   .. note::

     The Tools VM has already been joined to the **NTNXLAB.local** domain. You could use any domain joined VM to complete the following steps.

#. Open ``\\<Intials>-Files.ntnxlab.local\`` in **File Explorer**.

   .. figure:: images/17.png

#. Test accessing the Marketing share by extracting the SampleData_Small.zip files downloaded in the previous step into the share.

   .. figure:: images/18.png

   - The **NTNXLAB\\Administrator** user was specified as a Files Administrator during deployment of the Files cluster, giving it read/write access to all shares by default.
   - Managing access for other users is no different than any other SMB share.

#. Right-click **Marketing > Properties**.

#. Select the **Security** tab and click **Advanced**.

   .. figure:: images/19.png

#. Select **Users (**\ *Initials*\ **-Files\\Users)** and click **Remove**.

#. Click **Add**.

#. Click **Select a principal** and specify **Everyone** in the **Object Name** field. Click **OK**.

   .. figure:: images/20.png

#. Fill out the following fields and click **OK**:

   - **Type** - Allow
   - **Applies to** - This folder only
   - Select **Read & execute**
   - Select **List folder contents**
   - Select **Read**
   - Select **Write**

   .. figure:: images/21.png

#. Click **OK > OK > OK** to save the permission changes.

   All users will now be able to create folders and files within the Marketing share.

   It is common for shares utilized by many people to leverage quotas to ensure fair use of resources. Files offers the ability to set either soft or hard quotas on a per share basis for either individual users within Active Directory, or specific Active Directory Security Groups.

#. In **Prism > File Server > Share > Marketing**, click **+ Add Quota Policy**.

#. Fill out the following fields and click **Save**:

   - Select **Group**
   - **User or Group** - SSP Developers
   - **Quota** - 10 GiB
   - **Enforcement Type** - Hard Limit

   .. figure:: images/22.png

#. Click **Save**.

#. With the Marketing share still selected, review the **Share Details**, **Usage** and **Performance** tabs to understand the available on a per share basis, including the number of files & connections, storage utilization over time, latency, throughput, and IOPS.

   .. figure:: images/23.png

Using NFS Exports
+++++++++++++++++

In this exercise you will create and test a NFSv4 export, used to support clustered applications, store application data such as logging, or storing other unstructured file data commonly accessed by Linux clients.

Creating the Export
...................

#. In **Prism > File Server**, click **+ Share/Export**.

#. Fill out the following fields:

   - **Name** - logs
   - **Description (Optional)** - File share for system logs
   - **File Server** - *Initials*\ **-Files**
   - **Share Path (Optional)** - Leave blank
   - **Max Size (Optional)** - Leave blank
   - **Select Protocol** - NFS

   .. figure:: images/24.png

#. Click **Next**.

#. Fill out the following fields:

   - Select **Enable Self Service Restore**
      - These snapshots appear as a .snapshot directory for NFS clients.
   - **Authentication** - System
   - **Default Access (For All Clients)** - No Access
   - Select **+ Add exceptions**
   - **Clients with Read-Write Access** - *The first 3 octets of your cluster network*\ .* (e.g. 10.38.1.\*)

   .. figure:: images/25.png

   By default an NFS export will allow read/write access to any host that mounts the export, but this can be restricted to specific IPs or IP ranges.

#. Click **Next**.

#. Review the **Summary** and click **Create**.

Testing the Export
..................

You will first provision a CentOS VM to use as a client for your Files export.

.. note:: If you have already deployed the :ref:`linux_tools_vm` as part of another lab, you may use this VM as your NFS client instead.

#. In **Prism > VM > Table**, click **+ Create VM**.

#. Fill out the following fields:

   - **Name** - *Initials*\ -NFS-Client
   - **Description** - CentOS VM for testing Files NFS export
   - **vCPU(s)** - 2
   - **Number of Cores per vCPU** - 1
   - **Memory** - 2 GiB
   - Select **+ Add New Disk**
      - **Operation** - Clone from Image Service
      - **Image** - CentOS
      - Select **Add**
   - Select **Add New NIC**
      - **VLAN Name** - Primary
      - Select **Add**

#. Click **Save**.

#. Select the *Initials*\ **-NFS-Client** VM and click **Power on**.

#. Note the IP address of the VM in Prism, and connect via SSH using the following credentials:

   - **Username** - root
   - **Password** - nutanix/4u

#. Execute the following:

     .. code-block:: bash

       [root@CentOS ~]# yum install -y nfs-utils #This installs the NFSv4 client
       [root@CentOS ~]# mkdir /filesmnt
       [root@CentOS ~]# mount.nfs4 <Intials>-Files.ntnxlab.local:/ /filesmnt/
       [root@CentOS ~]# df -kh
       Filesystem                      Size  Used Avail Use% Mounted on
       /dev/mapper/centos_centos-root  8.5G  1.7G  6.8G  20% /
       devtmpfs                        1.9G     0  1.9G   0% /dev
       tmpfs                           1.9G     0  1.9G   0% /dev/shm
       tmpfs                           1.9G   17M  1.9G   1% /run
       tmpfs                           1.9G     0  1.9G   0% /sys/fs/cgroup
       /dev/sda1                       494M  141M  353M  29% /boot
       tmpfs                           377M     0  377M   0% /run/user/0
       *intials*-Files.ntnxlab.local:/             1.0T  7.0M  1.0T   1% /afsmnt
       [root@CentOS ~]# ls -l /filesmnt/
       total 1
       drwxrwxrwx. 2 root root 2 Mar  9 18:53 logs

#. Observe that the **logs** directory is mounted in ``/filesmnt/logs``.

#. Reboot the VM and observe the export is no longer mounted. To persist the mount, add it to ``/etc/fstab`` by executing the following:

     .. code-block:: bash

       echo 'Intials-Files.ntnxlab.local:/ /filesmnt nfs4' >> /etc/fstab

#. The following command will add 100 2MB files filled with random data to ``/filesmnt/logs``:

     .. code-block:: bash

       mkdir /filesmnt/logs/host1
       for i in {1..100}; do dd if=/dev/urandom bs=8k count=256 of=/filesmnt/logs/host1/file$i; done

#. Return to **Prism > File Server > Share > logs** to monitor performance and usage.

   Note that the utilization data is updated every 10 minutes.

Selective File Blocking
+++++++++++++++++++++++

In this exercise you will configure Files to block specific file extensions for the file server and the Marketing share. 

#. In **Prism** > **File Server** > Select your file server and click **Update** > then click **Blocked File Types**

   .. figure:: images/47.png
   
#. Under **Blocked File Types** enter a comma separated list of extensions like .flv,.mov and click **Save**

   .. figure:: images/48.png
   
#. Open a PowerShell window by clicking on the **PowerShell icon** on the taskbar. Enter the following command where you will see an access denied error message: 

   .. code-block:: bash
     
	 new-item \\xyz-files.ntnxlab.local\marketing\MyMovie.flv
   
   .. figure:: images/49.png
   
#. In **Prism** > **File Server** > **Share/Export** > click on the Marketing share and select **Update**

   .. figure:: images/50.png
	
#. Select **Next** to get to the **Settings** page.

#. Check **Blocked File Types** and enter .none as a file extension. 

   .. figure:: images/51.png
   
#. Select **Next** then **Save** on the **Summary** page to complete the update. 

#. Blocked file type settings at the share level override the server level setting.  Using PowerShell issue the same command as the previous step.  The command will now complete successfully.

   .. figure:: images/52.png

Multi-protocol
++++++++++++++

In this exercise you will configure an existing SMB share to also support NFS. Enabling multi-protocol access requires you to configure user mappings and define the native and non-native protocol for a share.  

Configure User Mappings
.......................

A Nutanix Files share has the concept of a native and non-native protocol.  All permissions are applied using the native protocol.  
Any access requests using the non-native protocol requires a user or group mapping to the permission applied from the native side.
There are several ways to apply user and group mappings including rule based, explicit and default mappings.  You will first configure a default mapping.

#. In **Prism** > **File Server** > Select your file server and click **Protocol Management** > then click **User Mapping**

   .. figure:: images/53.png
   
#. In the **User Mapping** dialog click **Next** at least two times, until you are on the **Default Mapping** page. 

#. From the **Default Mapping** page choose both **Deny access to NFS export** and **Deny access to SMB share** as the defaults for when no mapping is found.

   .. figure:: images/54.png
   
#. Complete the initial mapping by choosing **Next** and then **Save** on the **Summary** page. 

#. In **Prism** > **File Server** > **Share/Export** > click on the Marketing share and select **Update**.

#. From the **Basics** page check the box at the bottom which says **Enable multiprotocol access for NFS**.

   .. figure:: images/55.png
   
#. Click **Next** then from the **Settings* page check **Simultaneous access to the same files from both protocols**.

   .. figure:: images/56.png

#. Click **Next** and then **Save** from the **Summary** page. 

#. Connect via SSH to the *Initials*\ -NFS-Client VM.

#. Execute the following commands:

     .. code-block:: bash

       [root@CentOS ~]# mkdir /filesmnt/marketing
       [root@CentOS ~]# mount.nfs4 <Intials>-Files.ntnxlab.local:/Marketing /filesmnt/marketing
       [root@CentOS ~]# dir /filesmnt/marketing
       dir: cannot open directory /filesmnt/marketing: Permission denied
       [root@CentOS ~]#
	  
   .. note:: The mount operation is case sensitive.  

Because the default mapping is to deny access the Permission denied error is expected.  You will now add an explicit mapping to allow access to the non-native NFS protocol user. 
We will need to get the user ID (UID) to create the explicit mapping.

#. Execute the following command and take note of the UID:

     .. code-block:: bash

       [root@CentOS ~]# id
       uid=0(root) gid=0(root) groups=0(root) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
       [root@CentOS ~]#

#. In **Prism** > **File Server** > Select your file server and click **Protocol Management** > then click **User Mapping**

#. Click **Next** until you are on the **Explicit Mapping** page 

#. Click **+ Add one-to-one mapping**

#. Fill out the following fields:

   - **SMB Name** - ntnxlab\\administrator
   - **NFS ID** - UID from previous step (0 if root)
   - **User/Group** - User

   .. figure:: images/57.png

#. Click **Save** under the **Actions** column

#. Click **Next** until the **Summary** page and then click **Save**

#. Click **Close**

#. Go back to the NFS-Client VM and execute the following:

     .. code-block:: bash

       [root@CentOS ~]# dir /filesmnt/marketing
       MyMovie.flv
       [root@CentOS ~]#

File Analytics
++++++++++++++

In this exercise you will deploy the File Analytics VM and scan the existing shares to build out the dashboard.  You will also create anomaly alerts and view the audit details for your file server instance.

#. In **Prism** > **File Server** > click **Deploy File Analytics**

   .. figure:: images/31.png

#. Select **Deploy**

#. Choose **Download** for the 2.0.x version available

#. Fill out the details

   - **Name** - Initials
   - **Storage Container** – Will automatically select the container used by your file server instance
   - **Network List** – Primary - Managed
   
#. Select **Show Advanced Settings**

#. Ensure **DNS Resolver IP** is set to your Active Directory, ntnxlab.local, domain controller/DNS IP address and **ONLY** that address.

#. Choose **Deploy**

#. You can monitor the deployment from the **Tasks** page.  The Analytics VM deployment should take ~5 minutes.

#. In **Prism** > **File Server** > click **File Analytics**

   .. figure:: images/33.png

#. On the Enable File Analytics page enter your domain administrator which is also your file server administrator.

   - **Username**: administrator
   - **Password**: nutanix/4u

   .. figure:: images/34.png

#. Select **Enable**

#. Analytics will perform an initial scan of the existing shares which will take just a couple minutes.  You can see the scan by going to the gear icon within the Analytics UI and selecting **Scan File System**

   .. figure:: images/35.png

#. Choose **Cancel** to exit the scan details window

#. After viewing the scan details, refresh your browser.  You should see the **Data Age**, **File Distribution by Size** and **File Distribution by Type** dashboard panels update.

   .. figure:: images/36.png

#. Create some audit trail activity by going to the marketing share and opening one of the word files under **Sample Data** > **Documents**

   .. note:: You may need to complete a short wizard for OpenOffice if using that application to open a file.

#. Refresh the **Dashboard** page in your browser to see the **Top 5 active users**, **Top 5 accessed files** and **File Operations** panels update

   .. figure:: images/37.png

#. Click on your user under **Top 5 active users**.  This will take you to the audit trail of the user.

#. You can also click on the **Audit Trails** menu and search for either your user or a given file.  You can use wildcards for your search, for example **.doc**

   .. figure:: images/38.png

#. Next, create two anomaly rules by going to **Define Anomaly Rules** from under the gear icon

   .. figure:: images/39.png

#. Choose **Define Anomaly Rules** and create a rule with the following settings

   - **Events:** Delete
   - **Minimum Operation %:** 1
   - **Minimum Operation Count:** 10
   - **User:** All Users
   - **Type:** Hourly
   - **Interval:** 1

#. Choose **Save** for that anomaly table entry

#. Choose **+ Configure new anomaly** and create a second rule with the following settings

   - **Events**: Create
   - **Minimum Operation %**: 1
   - **Minimum Operation Count**: 10
   - **User**: All Users
   - **Type**: Hourly
   - **Interval**: 1

#. Choose **Save** for that anomaly table entry

   .. figure:: images/40.png

#. Select **Save** to exit the Define Anomaly Rules window

#. Go to the Sample Data folder in the Marketing share and copy, then paste that folder to the same share.

   .. figure:: images/42.png

#. Now delete the original Sample Data folder.

#. While waiting for the Anomaly Alerts to populate we’ll create a permission denial.

   .. note:: The Anomaly engine runs every 30 minutes.  While this setting is configurable from the File Analytics VM, modifying this variable is outside the scope of this lab.

#. Create a new directory called **RO** in the Marketing share

#. Create a text file in the **RO** directory with some text like “hello world” called **myfile.txt**

#. Go to the **Properties** of the **RO** folder and select the Security tab

#. Select **Advanced**

#. Choose **Disable inheritance** and select the **Convert…** option

#. Then add the **Everyone** permissions with the following:

   - Read & Execute
   - List folder contents
   - Read

   .. figure:: images/43.png

#. Choose **OK** then **OK** again

#. Open a PowerShell window as a specific user

   - Hold down **Shift** and **right click** on the **PowerShell icon** on the taskbar
   - Select **Run as different user**

   .. figure:: images/44.png

#. Enter the following

   - **User name**: Poweruser01
   - **Password**: nutanix/4u

#. Change Directories into the Marketing share and the **RO** directory

     .. code-block:: bash

        cd \\xyz-files.ntnxlab.local\marketing\RO

#. Execute the following commands, the first should succeed, the second should fail:

     .. code-block:: bash

        more .\myfile.txt
        rm .\myfile.txt

   .. figure:: images/45.png

#. After a minute or so you should see **Permission Denials** in both the dashboard and the **Audit Trails** view.  You may need to refresh your browser.

   .. figure:: images/46.png

   .. note:: The Capacity Trend dashboard panel updates every 24 hrs.

New with Files 3.6
++++++++++++

With the recent Files 3.6 release we have introduced:

- NearSync DR support
- In-flight Encryption for SMB
- SMB durable handle support
- Selective file blocking
- Windows 2019 domain and client support
- 120TB node support


Takeaways
+++++++++

What are the key things you should know about **Nutanix Files**?

- Files can be rapidly deployed on top of existing Nutanix clusters, providing SMB and NFS storage for user shares, home directories, departmental shares, applications, and any other general purpose file storage needs.
- Files is not a point solution. VM, File, Block, and Object storage can all be delivered by the same platform using the same management tools, reducing complexity and management silos.
- Files can scale up and scale out with One Click performance optimization.
- File Analytics helps you better understand how data is utilized by your organizations to help you meet your data auditing, data access minimization and compliance requirements.
