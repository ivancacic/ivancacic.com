---
id: 124
title: 'NetScaler 10.5 Upgrade - High Availability and Custom Theme'
date: 2015-05-10T20:45:55+00:00
author: Ivan Cacic
layout: post
comments: true
guid: https://ivancacic.com/?p=124
permalink: /netscaler-10-5-upgrade-high-availability-and-custom-theme
categories:
  - Citrix
  - NetScaler
tags:
  - HA
  - Theming
---
Upgrading NetScaler appliances should be part and parcel of the administration and maintenance process. The process is reasonably straight forward, however in this post I would like to address two complications which can cause hiccups to this procedure:

  * 2 or more NetScalers in a High Availability configuration and how to apply upgrades with minimal end user impact
  * Customisations applied to the NetScaler Gateway and how upgrades affect them

It should be stated that the update process in a High Availability pair can result in an almost undetectable down time, basically the only downtime that should occur will be when the pair is failed over from the primary to secondary node.

Before we begin, it's a good idea to read the short [reference article](http://support.citrix.com/proddocs/topic/netscaler-migration-10-5/ns-instpk-upgrd-high-avail-pair-tsk.html) provided by Citrix.

## Upgrading the first NetScaler

Let's begin the process by logging into both of our NetScaler appliances from the web UI, by expanding the System menu item and clicking High Availability we can see all the nodes in the pair and which Master State they are in:

[![HA Node status]({{ site.baseurl }}/assets/2015/05/2015-05-10-22_24_41-Citrix-NetScaler-VPX-Configuration.png)]({{ site.baseurl }}/assets/2015/05/2015-05-10-22_24_41-Citrix-NetScaler-VPX-Configuration.png)

Switch to the node in the Primary state, you can see the current firmware version at the top of the screen, clicking the arrow gives some more detail, my NetScaler is currently running an 'Enhanced' release:

![NetScaler Enhanced version]({{ site.baseurl }}/assets/2016/05/2015-05-10-22_32_31-Citrix-NetScaler-VPX-Configuration.png)

By going back to the High Availability section we can now move our NetScaler to _Stay Primary_ mode, as the name suggests this forces the NetScaler HA pair to keep the node as Primary unless it goes down. Select the Primary node and click the Edit button, from the Configure HA Node menu switch the Status of the node to STAY PRIMARY:

![Configure HA status]({{ site.baseurl }}/assets/2016/05/2015-05-10-22_25_17-Citrix-NetScaler-VPX-Configuration.png)

The NetScalers shouldn't switch Nodes at this stage, we are now ready to go back to our Secondary node and begin the upgrade process.

At this point jump back into the Secondary node and switch the custom theme to the Default or Green Bubbles theme, you can do this by opening the NetScaler Gateway tab, selecting Global Settings and click the Change Global Settings link:

[![Change Global Settings]({{ site.baseurl }}/assets/2016/05/2015-05-10-22_48_30-Citrix-NetScaler-VPX-Configuration.png)]({{ site.baseurl }}/assets/2016/05/2015-05-10-22_48_30-Citrix-NetScaler-VPX-Configuration.png) [![Set UI theme]({{ site.baseurl }}/assets/2016/05/2015-05-10-22_48_58-Citrix-NetScaler-VPX-Configuration.png)]({{ site.baseurl }}/assets/2016/05/2015-05-10-22_48_58-Citrix-NetScaler-VPX-Configuration.png)

The above steps are important because the Custom theme, contains the entire `ns_gui` folder, part of this folder is the Access Gateway VPN client and EPA folders which would not be upgraded if we kept the old custom theme, but more importantly the old `admin_ui` folder would remain which would break any new admin pages in the new release of the firmware. For this reason we need to upgrade our custom theme - more on this later in the post.

Hit the save button (so all of your running config is saved to `ns.conf`) at this point and fire up WinSCP or your favourite SFTP client and copy across your customtheme.tar.gz and the `/flash/nsconfig` folder. Once you have a backup, jump back into the web UI and select the _Upgrade Wizard_ button from the System Information menu:

![Upgrade Wizard]({{ site.baseurl }}/assets/2016/05/2015-05-10-22_27_41-Citrix-NetScaler-VPX-Configuration.png)

As the wizard is still written in Java, if you run into errors make sure your NetScaler URL is white listed in Java settings:

![Java - Exception Site List]({{ site.baseurl }}/assets/2016/05/2015-05-10-22_28_41-Citrix-NetScaler-VPX-Configuration.png)

I still had issues with compatibility so I switched from Chrome to Internet Explorer. Once the wizard launches, click Next to continue onto the Upload Software screen, select Local Computer and browse to the gzip file you downloaded from Citrix.com, click Next:

[![Upgrade Wizard - Upload Software]({{ site.baseurl }}/assets/2016/05/2015-05-10-22_30_06-Citrix-NetScaler-VPX-Configuration-Internet-Explorer.png)]({{ site.baseurl }}/assets/2016/05/2015-05-10-22_30_06-Citrix-NetScaler-VPX-Configuration-Internet-Explorer.png)

The next screen allows you to change the licenses currently on the NetScaler, licensing is applied on reboot so any changes here will be reflected once the upgrade is complete, click Next when ready:

[![Upgrade Wizard - Manage Licenses]({{ site.baseurl }}/assets/2016/05/2015-05-10-22_30_14-Citrix-NetScaler-VPX-Configuration-Internet-Explorer.png)]({{ site.baseurl }}/assets/2016/05/2015-05-10-22_30_14-Citrix-NetScaler-VPX-Configuration-Internet-Explorer.png)

On the next screen select the Reboot option to have the NetScaler reboot itself after a successful installation, which saves you rebooting it manually. You can also elect to move some old kernel files and configuration files during this process, be warned however that I have had issues in the past with this process. As a general rule of thumb, I do not run the clean-up operation.

[![Upgrade Wizard - Clean-up/Reboot]({{ site.baseurl }}/assets/2016/05/2015-05-10-22_31_42-Citrix-NetScaler-VPX-Configuration-Internet-Explorer.png)]({{ site.baseurl }}/assets/2016/05/2015-05-10-22_31_42-Citrix-NetScaler-VPX-Configuration-Internet-Explorer.png)

Click Next when ready and Finish on the next screen to begin uploading the firmware:

[![Upgrade Wizard - Summary]({{ site.baseurl }}/assets/2016/05/2015-05-10-22_31_56-Citrix-NetScaler-VPX-Configuration-Internet-Explorer.png)]({{ site.baseurl }}/assets/2016/05/2015-05-10-22_31_56-Citrix-NetScaler-VPX-Configuration-Internet-Explorer.png)

The firmware will begin extracting and installing:
  
![Upgrading...]({{ site.baseurl }}/assets/2016/05/2015-05-10-22_33_14-Upgrade-Wizard.png)

If you're using an MPX appliance you will be prompted to enable the [CallHome feature](http://support.citrix.com/proddocs/topic/ns-faq-map-10-5/ns-faq-call-home-ref.html). If you've selected the reboot option the NetScaler will reboot once the script has completed. On our Primary node we can see the Secondary node is not responding:

[![Secondary node UNKNOWN]({{ site.baseurl }}/assets/2016/05/2015-05-10-22_40_32-Citrix-NetScaler-VPX-Configuration.png)]({{ site.baseurl }}/assets/2016/05/2015-05-10-22_40_32-Citrix-NetScaler-VPX-Configuration.png)

## Adjusting the Custom Theme

When the node comes back up re-connect your SFTP session and also launch an SSH session to the device. At this point we will repackage our custom theme, firstly unpack the vpn folder from the backup you took earlier, you will need to use a program similar to [7-Zip](http://www.7-zip.org/):

![7zip customtheme]({{ site.baseurl }}/assets/2016/05/2015-05-10-23_16_35-C__temp_customtheme.tar.gz_customtheme.tar_ns_gui_.png)

Our NetScaler should be using the Default or Green Bubbles theme. By having either of these themes selected, the NetScaler will have all of the updates applied to the `epa` and `admin_ui` folders. Open your SSH session and run the following commands to delete the existing custom theme gzip:

`shell`

`rm /var/ns_gui_custom/customtheme.tar.gz`

Let's tar up the current Default/Green Bubbles theme to use as a base for our new custom theme:

`cd /netscaler`

`tar -cvzf /var/ns_gui_custom/customtheme.tar.gz ns_gui/*`

We can jump back into our /var/ns\_gui\_custom folder and check our new customtheme file is in there and also go ahead and delete the old ns_gui folder which the NetScaler uses to display the custom theme:

`cd /var/ns_gui_custom/`

`rm -rf ns_gui/`

At this point we can switch the NetScaler back to the custom theme (we can switch the theme from the console as per below or using the web UI), we should see the ns_gui folder get recreated now:

![PuTTY]({{ site.baseurl }}/assets/2016/05/2015-05-10-23_15_01-netscaler2.cacic_.local-PuTTY.png)

![WinSCP]({{ site.baseurl }}/assets/2016/05/2015-05-10-23_15_56-ns_gui_custom-netscaler2-WinSCP.png)

We can take the vpn folder we extracted earlier and copy it over the top of our current ``/var/ns_gui_custom/ns_gui/vpn/`` folder, you will see lots of warnings about overwriting files, click _Yes to All_:

![WinSCP - Copy]({{ site.baseurl }}/assets/2016/05/2015-05-10-23_17_23-Copy.png)

![WinSCP - Confirm overwrite]({{ site.baseurl }}/assets/2016/05/2015-05-10-23_17_36-Confirm.png)]({{ site.baseurl }}/assets/2016/05/2015-05-10-23_17_36-Confirm.png)

Once complete we can tar up our new theme, the following commands delete the existing customtheme file and tar up our latest creation (new `epa` and `admin_ui` folders with the replaced `vpn` folder from our custom theme):

`cd /var/ns_gui_custom/`

`rm customtheme.tar.gz`

`tar -cvzf /var/ns_gui_custom/customtheme.tar.gz ns_gui/*`

Once complete we can copy the file back to our local machine so it can be applied to the Primary NetScaler once it has been upgraded. You should see a slight size difference between the two:

![customtheme filesize differences]({{ site.baseurl }}/assets/2016/05/2015-05-10-23_20_21-ns_gui_custom-netscaler2-WinSCP.png)

## Upgrading the second NetScaler

Our Secondary node should now be up to date and ready to move into a Hot state so that we can perform the upgrade on the current Primary. We need to log onto the current Primary node and check the HA state, the Secondary node should be up. Select the Primary node, click the Edit button and switch out the HA Status back to Enabled, click Apply to return to the High Availability screen:

![Configure HA nodes]({{ site.baseurl }}/assets/2016/05/2015-05-10-22_25_17-Citrix-NetScaler-VPX-Configuration.png)

We force fail over by clicking the _Edit_ button, selecting the _Force Failover_ option and clicking _Proceed_. At this point the NetScalers will switch Primary/Secondary Node status, using ARP/GARP traffic will be switched across to the upgraded node, it is at this point a very short outage occurs. TLS/SSL sessions are dropped and reset (which may cause back end servers to request re-authentication from connected clients). ICA sessions will drop out and reconnect if [Session Reliability](http://support.citrix.com/proddocs/topic/xenapp-xendesktop-76/xad-sessions-maintain.html) is in use and configured correctly.

[![HA - Force Failover]({{ site.baseurl }}/assets/2016/05/2015-05-10-22_59_22-Citrix-NetScaler-VPX-Configuration.png)]({{ site.baseurl }}/assets/2016/05/2015-05-10-22_59_22-Citrix-NetScaler-VPX-Configuration.png)

We should see the new node status:

[![HA - node status]({{ site.baseurl }}/assets/2016/05/2015-05-10-22_59_40-Citrix-NetScaler-VPX-Configuration.png)]({{ site.baseurl }}/assets/2016/05/2015-05-10-22_59_40-Citrix-NetScaler-VPX-Configuration.png)

The following actions should now be taken as per the instructions above:

  * The upgraded node should now be set to Stay Primary;
  * The secondary node needs to have the theme changed to Default or Green Bubble;
  * The configuration should be saved and the `/flash/nsconfig` folder should be backed up;
  * The NetScaler can then be upgraded from the web UI;
  * Once rebooted, the customtheme.tar.gz file and `/var/ns_gui_custom/ns_gui/` folder can be deleted.
  * The new customtheme file can be uploaded to the NetScaler and the Custom theme enabled;
  * Stay Primary can be removed from the Primary NetScaler and the configuration saved;
  * The upgrade process is complete.

## Troubleshooting - Invalid argument [ns]

![Invalid argument [ns]]({{ site.baseurl }}/assets/2016/05/2015-05-10-22_50_25-Citrix-Login.png)

If you forgot to switch out the Custom theme for the Default/Green Bubbles themes before the upgrade you will likely receive and error such as Invalid argument [ns]. This occurs because the old `admin_ui` has been copied over the top of the new `admin_ui` folder, we need to switch the theme back to Default/Green Bubbles and then apply the Custom theme upgrade as per the above instructions. Without being able to log onto the web UI we need to connect the either the console of the NetScaler or SSH to it and issue the following command:

`set vpn paramater -UITHEME DEFAULT`

![set vpn paramter -UITHEME]({{ site.baseurl }}/assets/2016/05/2015-05-10-22_54_04-2015-05-10-22_31_42-Citrix-NetScaler-VPX-Configuration-Internet-Explorer.png.png)