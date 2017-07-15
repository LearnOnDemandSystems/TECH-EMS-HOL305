
<!---
Version: 1.0 
-->
# Lingering Object Fundamentals
>LODSProperties
>* IntroductionUri = https://lodmanuals.blob.core.windows.net/manuals/Ignite 2016/Videos/IDL3088/Exercise 1- Lingering Object Fundamentals.mp4

## INTRODUCTION MESSAGE
In this exercise, you will review lingering object terminology, prevention methods and use ADREPLStatus, repadmin.exe and the Directory Service event log to identify symptoms of lingering objects.   
  
1. Lingering Object terminology  
2. How to prevent a lingering object problem.   
3. Understand the cause and identify the symptoms of Lingering Objects
## COMPLETION MESSAGE
We reviewed lingering object fundamentals: core concepts and terminology  
Lingering object symptoms:  
a. For strict replication consistency:  
 AD replication status 8606 and event 1988  
b. For loose replication consistency:  
 Event 1388   
  
In this exercise:   
1. We began by getting a forest-wide AD replication status report. In the report, we found that replication was failing on all DCs in almost all partitions with error 8606, "Insufficient attributes were given to create an object…"   
2. We then went to one DC and found a single lingering object reported in event 1988. We dug into the details of the event and identified all DCs with the lingering object.   
3. We then used repadmin to discover that there were actually many more lingering objects than just the one reported.   
4. Finally, we checked for lingering objects on the DC that was not displaying any symptoms, and discovered that it actually had more lingering objects than the DC with the symptoms.
### Task 0: Lingering object terminology
This lab is jargon intense, so a Lingering Object Glossary is provided for your reference.  
  
Refer to the Lingering Object Glossary document on the desktop of the Win8Client as needed for a description of the various terms mentioned throughout the lab.





### How to prevent a lingering object problem
The root cause of most lingering object problems are long term AD replication failures that have been allowed to persist beyond the tombstone lifetime number of days. The best way to avoid and prevent lingering object issues:  
1. Proactively monitor AD replication with a tool like ADReplStatus.  
2. Correct AD replication problems within the tombstone lifetime number of days  
3. Prevent large jumps in system time from occurring on domain controllers

#### :warning: ALERT
• Resolve replication failures within TSL \# of days  
• Ensure Strict Replication Consistency is enabled  
• Ensure large jumps in system time are blocked via registry key or policy  
• Don't remove replication quarantine with the "allowDivergent" setting without removing LOs first  
• Don't restore system backups that are near TSL number of days old  
• Don't bring DCs back online that haven't replicated within TSL  
• Do not allow a server to replicate that has experienced a USN rollback  
• Ensure originating changes are replicated out to other DCs in the same domain before forcefully demoting a DC or restoring a VM checkpoint of a Windows Server 2012 DC VM guest





### Task 1: Lingering object symptoms and id
AD replication status 8606 and event ID 1988 are good indicators of lingering objects \)when the DCs are configured for Strict Replication Consistency). It is important to note, however, that AD replication may complete successfully \)and not log an error) from a DC containing lingering objects since replication is based on changes. If there are no changes to any of the lingering objects, there is no reason to replicate them and they will continue to exist without logging any noticeable errors. For this reason, when cleaning up lingering objects, do not just clean up the DCs logging the errors; instead, assume that all DCs may contain them, and clean them up as well.

#### :bulb: KNOWLEDGE
Scenario  
• AD replication of the Root partition from DC1 to DC2 fails with error, "Insufficient attributes were given to create an object".  
• All DCs have lingering objects in almost all partitions  
• DC2 reports error 8606 replicating from DC1

#### :camera: SCREENSHOT
>LODSProperties
>* Uri = screens/613962.jpg





### Open AD Replication Status tool
Connect to Win8Client  
  
Double click the **AD Replication Status Tool 1.0** shortcut on the Desktop

#### :warning: ALERT
• The ROOT\\Administrator account is already logged on to this machine.  
• Note: Domain admin privileges are not needed for this task, but these privileges are required in later exercises.




#### :computer: ACTIONS
>LODSProperties
>* VM = Win8Client



### Retrieve replication status
Within the AD Replication Status Tool, click Refresh Replication Status.  
  
• The tool will take one to two minutes to check the AD replication status.   
• You will know data collection is complete when the Status: prompt changes from Running to Ready and the focus is switched to the Replication Status Viewer tab.  
  
Verify that replication status information was collected from all DCs.  If you are missing data from ChildDC1 and ChildDC2, then perform the steps in the knowledge section.

#### :bulb: KNOWLEDGE
If AD Replication Status fails to return replication status information from ChildDC1 and ChildDC2:  
  
Select the Configuration/Scope Settings tab and then Select the Environment Discovery tab  
  
From Win8Client use repadmin /bind to try to connect to ChildDC1  
repadmin /bind childdc1 fails with  LDAP Error 49\)0x31): Invalid Credentials  
  
   
  
c:\\Files>repadmin /bind childdc1  
  
Repadmin can't connect to a "home server", because of the following error.  Try  
  
specifying a different  
  
home server with /homeserver:\[dns name\]  
  
Error: An LDAP lookup operation failed with the following error:  
  
   
  
    LDAP Error 49\)0x31): Invalid Credentials  
  
    Server Win32 Error 0\)0x0):  
  
    Extended Information:  
  
   
  
Cause: The trust password between the Child and Root domain is bad on one or more DCs \)likely DC2)  
  
Solution:  
  
Stop the KDC service on DC2 for the remainder of the lab \)It needs to replicate the updated trust password information \)TDO) from DC1 but it can’t because replication has been blocked due to a lingering object problem.  
  
On DC2, open a command prompt and type the following command and press ENTER  
  
Net stop KDC  
Verify you are able to use repadmin /bind now from Win8Client  
Switch to Win8Client and attempt to bind to ChildDC1 using repadmin /bind  
  
Repadmin /bind childdc1  
  
Close and reopen AD Replication Status tool and click Refresh Replication status  
If it still fails, you will need to reset the trust password between the Root and Child domain  
Switch to childdc1 and run the following command:  
  
netdom trust child.root.contoso.com /Domain:root.contoso.com /UserD:root\\Administrator /PasswordD:adrepl123! /UserO:child\\Administrator /PasswordO:adrepl123! /Reset /TwoWay  
  
To vet the trust password:  
  
Repadmin /showobjmeta \* CN=child.root.contoso.com,CN=System,DC=root,DC=contoso,DC=com >childTDO.txt  
  
Repadmin /showobjmeta \* CN=root.contoso.com,CN=System,DC=child,DC=root,DC=contoso,DC=com >RootTDO.txt  
  
Compare TrustAuthOutgoing and TrustAuthIncoming attribute modify times between child and parent DCs -they should be relatively close to the same time  
  
So in childTDO.txt look at trustAuthIncoming timestamp -verify DC1 and DC2 are the same  
  
In RootTDO.txt look at trustAuthIncoming timestamp -verify ChildDC1 and ChildDC2 are the same and were not modified more than 30 or 60 days ago.  
  
Repadmin /bind fails with LDAP 49 invalid credentials when we use an out of date KDC for tickets  
  
   
  
If it’s outdated everywhere, then reset it:  
  
netdom trust child.root.contoso.com /Domain:root.contoso.com /UserD:root\\Administrator /PasswordD:adrepl123! /UserO:child\\Administrator /PasswordO:adrepl123! /Reset /TwoWay  
  
  






### View replication errors
Click the Errors Only menu option on the Data section of the ribbon to see a detailed view of all replication errors in the forest.

#### :bulb: KNOWLEDGE
The Replication Status Viewer is highly customizable.   
o Drag different columns to the top for different pivot options.   
o Add and remove columns of interest.  
Later on, we will be investigating this one failure:   
DC2 is failing to replicate the root partition from DC1.

#### :camera: SCREENSHOT
>LODSProperties
>* Uri = screens/613965.jpg





### Review the error summary
Click the Replication Error Guide tab for a quick summary view of all errors.

#### :camera: SCREENSHOT
>LODSProperties
>* Uri = screens/613966.jpg





### Select all "Insufficient attribute" errors
Select the message text, "Insufficient attributes were given to create an object…" to see a sortable list of all DCs with this replication error.  
  
If you click on error 8606 in the Error Code column, our latest troubleshooting content for that issue loads up in the tool.

#### :bulb: KNOWLEDGE
The DCs listed in the Source DC column have at least one lingering object for the partition in the Naming Context column.

#### :camera: SCREENSHOT
>LODSProperties
>* Uri = screens/613967.jpg





### Find symptoms for a single DC
Switch to DC2

#### :bulb: KNOWLEDGE
For ease of command entry: There is a file on Win8Client in the D:\\files directory, called fix\_lab.txt that contains all necessary commands needed for this lab. There is a mixture of both CMD-line and PowerShell commands in the file. To execute the commands:   
1. Open an elevated PowerShell prompt on Win8Client.  
2. Copy the commands for the step you are working on, and paste them into the PowerShell window.  
3. It is best to copy the Files directory to the root of the C:\\ drive before executing any commands. Some commands attempt to output files to the current working directory \)which will fail for D:\\Files because it is a read-only ISO file attached to the VM guest.  
Alternately, you can copy them from the lab manual.




#### :computer: ACTIONS
>LODSProperties
>* VM = DC2



### Initiate Replication
Initiate replication between DC1 and DC2 \)have DC1 pull from DC2)  
Repadmin /replicate dc1 dc2 "dc=root,dc=contoso,dc=com"  
  
Replication completes successfully:  
Sync from dc2 to dc1 completed successfully



#### :calling: COMMAND
```TypeText
Repadmin /replicate dc1 dc2 "dc=root,dc=contoso,dc=com"
```


### Check replication in the other direction
Check replication the other way \)have DC2 pull from DC1)  
Repadmin /replicate dc2 dc1 "dc=root,dc=contoso,dc=com"  
  
Replication fails with the following error:  
DsReplicaSync\)) failed with status 8606 \)0x219e):  
Insufficient attributes were given to create an object. This object may not exist because it may have been deleted and already garbage collected.  
  
Event 1988 is logged in the Directory Service event log on DC2.



#### :calling: COMMAND
```TypeText
Repadmin /replicate dc2 dc1 "dc=root,dc=contoso,dc=com"
```


### Review the errors in the event log
Review the Directory Services event log on DC2 for event 1988 using event viewer \)eventvwr.msc) or PowerShell  
Get-WinEvent -LogName "Directory Service" -MaxEvents 5 | fl  
  
To just return the last 10 event ID 1988s from DC2's Directory Service event log:  
get-winevent -LogName "Directory Service" -ComputerName dc2 -MaxEvents 10 | Where-Object \{$\_.ID -eq

#### :bulb: KNOWLEDGE
Event 1988 only reports the first lingering object encountered during the replication attempt. There are usually many more lingering objects present on the source DC. Use repadmin /removelingeringobjects with the /advisory\_mode switch to have all lingering objects reported for that partition.

#### :camera: SCREENSHOT
>LODSProperties
>* Uri = screens/613971.jpg



#### :calling: COMMAND
```TypeText
get-winevent -LogName "Directory Service" -ComputerName dc2 -MaxEvents 10 | Where-Object {$_.ID -eq
```


### Identify key information
Identify the following from event 1988 \)they are needed later in the exercise):  
• Object GUID: e44b0379-382a-43e2-9e95-92f53c403002  
• Source DC: DC1.root.contoso.com  
• Partition DN: DC=root,DC=contoso,DC=com  
  
Use the events to help you answer Knowledge questions. Answers can be found in the downloadable appendix located on the desktop of the Win8Client.

#### :bulb: KNOWLEDGE
How can you translate the DNS alias provided in the event to the host name of the source DC?  
  
Is DC2 configured for Strict or Loose Replication Consistency?  
  
What event is logged on the destination DC when there is an attempt to send changes for a lingering object when strict replication consistency is enabled?  
  
What event is logged on the destination DC when there is an attempt to send changes for a lingering object when loose replication consistency is enabled?





### Task 2: Lingering object analysis
In this task, you will use repadmin to return replication metadata for the lingering object identified in event ID 1988. The repadmin output will allow you to identify DCs containing the lingering object reported in the event.  
  
These next few steps are performed on either DC2 or DC1.





### Get the ObjectGUID
Obtain the ObjectGUID reported in the event on DC2. \)see the previous step for location of the ObjectGUID).





### Identify all affected DCs
Identify all DCs that have a copy of this object using repadmin /showobjmeta  
Repadmin /showobjmeta \* "<GUID=e44b0379-382a-43e2-9e95-92f53c403002>" >emp2.txt



#### :calling: COMMAND
```TypeText
Repadmin /showobjmeta * "<GUID=e44b0379-382a-43e2-9e95-92f53c403002>" >emp2.txt
```


### Review the replication metadata
Open emp2.txt. Any DC that returns replication metadata for this object are DCs containing one or more lingering objects. DCs that do not have a copy of the object report status 8439, "The distinguished name specified for this replication operation is invalid".  
  
This is a good method to conduct a quick spot check of DCs containing the lingering object reported in the event. It is NOT a good method to discover all lingering objects. For more information, see the Lingering Object discovery section of the appendix.  
  
  


#### :bulb: KNOWLEDGE
Which DCs return replication metadata for the object?  
See the Answers section in the Appendix if needed.  
  
Is the EMP2 user account the only lingering object present on DC1?  
It is likely there are many more. We will use repadmin in the next step to check for more objects in the Root partition on DC1.





### Retrieve DC2 replication information
Obtain DC2's DSA ObjectGUID and use repadmin /removelingeringobjects with the /advisory\_mode parameter to identify all lingering objects in the ROOT partition on DC1.

#### :bulb: KNOWLEDGE
In order to use the /removelingeringobjects command you need to know three things:  
1. You need to know which DCs contain lingering objects  
2. Which partition the lingering object resides in  
3. The DSA Object GUID of a good reference DC that hosts that partition that does not contain lingering objects





### Retrieve the DSA object GUID on DC2
Since DC2 is the only other DC in the ROOT partition, we will have to use it as the reference DC. Obtain the DSA object GUID on DC2:  
Repadmin /showrepl DC2 >DC2\_showrepl.txt  
  
The DSA object GUID is at the top of the output and will look like this:  
DSA object GUID: 3fe45b7f-e6b1-42b1-bcf4-2561c38cc3a6



#### :calling: COMMAND
```TypeText
Repadmin /showrepl DC2 >DC2_showrepl.txt
```


### Verify lingering objects on DC1
Verify the existence of lingering objects on DC1 by comparing its copy of the ROOT partition with DC2.  
  
Run the following repadmin command \)ensure you use the /advisory\_mode parameter)  
Repadmin /removelingeringobjects DC1 3fe45b7f-e6b1-42b1-bcf4-2561c38cc3a6 "dc=root,dc=contoso,dc=com" /Advisory\_Mode  
  
RemoveLingeringObjects successful on dc1.



#### :calling: COMMAND
```TypeText
Repadmin /removelingeringobjects DC1 3fe45b7f-e6b1-42b1-bcf4-2561c38cc3a6 "dc=root,dc=contoso,dc=com" /Advisory_Mode
```


### Retrieve event log entries from DC1
Connect to DC1. Review the Directory Service event log on DC1. If there are any lingering objects present, each one will be reported in its own event ID 1946. The total count of lingering objects for the partition is reported in event 1942.

#### :bulb: KNOWLEDGE
We compared DC1 against DC2 for the root partition. How do we know that DC2 is clean? Earlier we saw that AD replication completes successfully from DC2. As mentioned in the exercise introduction, you cannot assume a DC is clean just because replication completes without error; we will now check DC2 against DC1.

#### :camera: SCREENSHOT
>LODSProperties
>* Uri = screens/613980.jpg



#### :calling: COMMAND
```TypeText
get-winevent -LogName "Directory Service" -ComputerName dc1 -MaxEvents 10 | Where-Object {$_.ID -eq "1942"} | fl >DC1_DSevents1942.txt
```


### Get DSA objectGUID from DC2
Use repadmin to check for the existence of lingering objects in the root partition on DC2  
  
Obtain DSA object GUID from the only other DC in the domain DC1  
Repadmin /showrepl DC1  
  
DSA object GUID: 70ff33ce-2f41-4bf4-b7ca-7fa71d4ca13e





### Check for lingering objects in the root on DC2
Use repadmin /removelingeringobjects with the /Advisory\_Mode parameter  
repadmin /removelingeringobjects dc2 70ff33ce-2f41-4bf4-b7ca-7fa71d4ca13e dc=root,dc=contoso,dc=com /Advisory\_Mode  
  
On DC2, review event ID 1942 logged in the Directory Service event log  
  
  


#### :bulb: KNOWLEDGE
Event 1942 indicates that DC2 also contains lingering objects in the root partition. This is notable because we saw no indication of a lingering object problem for this partition from the AD replication status report.

#### :camera: SCREENSHOT
>LODSProperties
>* Uri = screens/613982.jpg



#### :calling: COMMAND
```TypeText
repadmin /removelingeringobjects dc2 70ff33ce-2f41-4bf4-b7ca-7fa71d4ca13e dc=root,dc=contoso,dc=com /Advisory_Mode
```


### Optional: PowerShell Method
PowerShell method to view forest-wide replication results \)Optional steps)  
Perform these steps on DC2 if desired. The steps can be found in the Knowledge item for this task.  
  
  


#### :bulb: KNOWLEDGE
1. Open a PowerShell prompt and type the following commands, and then press ENTER:  
 PowerShell: Repadmin /showrepl \* /csv | convertfrom-csv | out-gridview  
  
a. Select Add criteria and check Last Failure Status. Select Add.  
b. Type 8606 in the text box.

#### :camera: SCREENSHOT
>LODSProperties
>* Uri = screens/613983.jpg






# Lingering Object Diagnosis and Documentation
>LODSProperties
>* IntroductionUri = https://lodmanuals.blob.core.windows.net/manuals/Ignite 2016/Videos/IDL3088/Exercise 2- Lingering Object Discovery.mp4

## INTRODUCTION MESSAGE
AD replication status 8606 and event ID 1988 are good indicators of lingering objects \)when the DCs are configured for Strict Replication Consistency).   
  
As you saw in the prior lesson, however, lingering objects can be present on a DC without any noticeable symptoms. AD replication is based on change notifications; if there are no changes to an object that is lingering, it is not replicated, and therefore there are no symptoms of the condition. For this reason, when cleaning up lingering objects, do not just clean up the DCs logging the errors; instead, assume that all DCs may contain them, and clean them up as well.  
  
Use several tools to identify the full scope of a lingering object problem.  
Documenting all lingering objects has traditionally been a challenging problem. The new Lingering Object Liquidator tool makes this a simple task, as you will discover in this exercise.   
  
Important:  
When lingering objects are discovered, assume they are present on all DCs in all partitions. Do not just clean up the DCs reporting the errors. Repldiag automates the majority of the cleanup work. See the Lingering Object discovery and cleanup section for more information.
## COMPLETION MESSAGE
In this exercise, we explored alternate lingering object discovery methods. Using a tool that does a forest-wide discovery of lingering objects is preferred over picking individual DCs and individual partitions.  
Lingering Object Liquidator, repldiag and repadmin /removelingeringobjects all leverage the same function for lingering object discovery. Replfix.exe uses a different mechanism for lingering object discovery and is useful for discovery of abandoned deleted objects because it compares two DCs against each other both ways; the tool's usage is complicated and should be leveraged only when there is a need to do a reverse comparison.
### Task 0: Review discovery and cleanup process
Read the attached Knowledge article to better understand the process of Lingering Object discovery and cleanup, as well as some of the tools involved.

#### :bulb: KNOWLEDGE
**Lingering Object discovery and cleanup**  
Repadmin /removelingeringobjects /advisory\_mode is a good method to conduct a spot check of lingering objects on an individual DC, per partition basis.   
  
However, lingering objects may exist on DCs without any noticeable symptoms. For that reason, checking and cleaning up just the DCs that report errors is not a good method to ensure all lingering objects are removed from the environment.  
  
**To remove lingering objects**  
1. Determine the root cause of the lingering object issue and prevent it from occurring again  
2. Assume all DCs contain lingering objects in all partitions and clean up everyone  
  
Those that clean up just the source DCs reported with AD replication status 8606 usually find they have more objects to clean up later.  
  
To accomplish the above using repadmin, you need to do the following:  
1. Identify one DC per partition to use as a reference DC  
2. Clean up each DC identified in step 1 against all other DCs that host a writeable copy of the same partition. This DC is now considered "clean" and suitable to use as a reference DC.  
3. Clean up all other DCs against the reference DCs  
In the simple, five DC, three-domain lab environment, this requires 30 separate executions of the repadmin command. In a real-word production environment, the count of repadmin executions is usually in the hundreds or thousands.  
  
 **More: For more information, see:**  
Clean that Active Directory Forest of Lingering Objects  
http://blogs.technet.com/b/glennl/archive/2007/07/26/clean-that-active-directory-forest-of-lingering-objects.aspx   
  
The good news is that Lingering Object Liquidator and repldiag /removelingeringobjects automates the above for you.  
• Repldiag requires just one execution: Repldiag /removelingeringobjects   
• With Lingering Object Liquidator, you just click the RemoveAll button  
  
Blindly removing all objects is fine for most; however, some people like to know what is going to be removed from their Active Directory ---especially if the problem is widespread like it is in the lab environment. The domain controllers in this environment have not replicated with each other in over a year. It is usually wise to forcefully demote a DC that has not replicated in that length of time. However, this is not an option here since all DCs fall into this category. For that reason, we need to identify and then remove all objects before replication is enabled. It is generally a good idea to document what you are going to delete before you delete it.





### Task 1: Lingering Object Discovery
In this task, we will use several additional tools that help with lingering object discovery. Featured here is the new and improved Lingering Object Liquidator tool. We will explore a brand new version of this tool. We will touch on repldiag and use another tool called Replfix to round out our lingering object discovery options.





### Introducing the Lingering Object Liquidator
Connect to DC1 for this task.  
  
Launch Lingering Object Liquidator from the shortcut on the desktop of DC1.  
  
If you get a Windows protected your PC SmartScreen prompt, click More info and then click Run anyway.

#### :bulb: KNOWLEDGE
Lingering Object Liquidator automates the discovery and removal of lingering objects by using the DRSReplicaVerifyObjects method used by repadmin /removelingeringobjects and repldiag combined with the removeLingeringObject rootDSE primitive used by LDP.EXE. Tool features include:  
• Combines both discovery and removal of lingering objects in one interface  
• Is available via the Microsoft Connect site  
o See post on blogs.technet.com/askds for instructions  
• The version of the tool at the Microsoft Connect site is an earlier version of the tool.  The version used in this lab is a complete new rewrite and will be available for download soon.  
• Feature improvements beyond what you see in this version are under consideration

#### :camera: SCREENSHOT
>LODSProperties
>* Uri = screens/613986.jpg




#### :computer: ACTIONS
>LODSProperties
>* VM = DC1



### Detect AD Topology
Click the Detect AD Topology button.

#### :bulb: KNOWLEDGE
Fast detection populates the Naming Context, Reference DC and Target DC lists by querying the local DC. Thorough detection does a more exhaustive search of all DCs and leverages DC Locator and DSBind calls. Note that Thorough detection will likely fail if one or more DCs are unreachable.





### Explore the UI
The "Naming Context" control allows you to choose which AD partition you want to operate against.  
  
The "Reference DC" control allows you to choose which DC is "authoritative" for the session. The reference DC hosts a writeable copy of the partition.  
  
The "Target DC" is the DC that lingering objects are to be removed from.

#### :warning: ALERT
The version of the tool in this lab is newer than the one currently available from the Connect site.  It will be available for download soon.





### Scan the entire environment
Select the following options to have the entire environment scanned,   
  
Naming Context: \[Scan All NCs\}  
  
Reference DC \[Scan Entire Forest\]  
  
Target DC \[Target All DCs\]  
  
and then click Detect Lingering Objects.  
  
  
The tool does a comparison amongst all DCs for all partitions in a pairwise fashion with these options selected. In a large environment, this comparison will take a great deal of time as the operation targets \)n \* \)n-1)) number of DCs in the forest for all locally held partitions. For shorter, targeted operations, select a naming context, reference DC and target DC. The reference DC must hold a writable copy of the selected naming context.

#### :warning: ALERT
Lingering Object Liquidator discovery method   
• Leverages DRSReplicaVerifyObjects method in Advisory Mode  
• Runs for all DCs and all Partitions  
• Collects lingering object event ID 1946s and displays objects in main content pane  
• List can be exported to CSV for offline analysis \)or modification for import)  
• Supports import and removal of objects from CSV import \)leverage for objects not discoverable using DRSReplicaVerifyObjects)  
• Supports removal of objects by DRSReplicaVerifyObjects and LDAP rootDSE removeLingeringobjects modification

#### :bulb: KNOWLEDGE
When the scan is complete, the buttons are re-enabled and the total execution time and the count of lingering objects detected is displayed. The bottom pane updates dynamically with the status of the detection. During this execution phase, the tool is running in an advisory mode and reading the event log data reported on each target DC.  
   
When the scan is complete, the status bar updates, buttons are re-enabled and total count of lingering objects is displayed. The log pane at the bottom of the window updates with any errors encountered during the scan.   
  
Error 1396 is logged if the tool incorrectly uses an RODC as a reference DC.   
  
Error 8440 is logged when the targeted reference DC doesn't host a writable copy of the partition.  
  
The tool leverages the Advisory Mode method exposed by DRSReplicaVerifyObjects that both repadmin /removelingeringobjects /Advisory\_Mode and repldiag /removelingeringobjects use. In addition to the normal Advisory Mode related events logged on each DC, it displays each of the lingering objects within the main content pane.  
  
Details of the scan operation are logged in the LOL\_Date\_timestamp.log file in the same directory as the tool's executable.  
  
The Export button allows you to export a list of all lingering objects listed in the main pane into a CSV file. View the file in Excel, modify if necessary and use the Import button later to view the objects without having to do a new scan. The Import feature is also useful if you discover abandoned objects \)not discoverable with DRSReplicaVerifyObjects) that you need to remove.  
  


#### :camera: SCREENSHOT
>LODSProperties
>* Uri = screens/613988.jpg





### Remove individual lingering objects
Select two or three objects \)hold down the Ctrl key to select multiple objects, or the SHIFT key to select a range of objects) and then select Remove.  
  
When prompted to confirm, click Yes.  
  


#### :bulb: KNOWLEDGE
The details pane logs the removal of the objects.  This is also logged in the  linger<Date-TimeStamp>.log.txt file

#### :camera: SCREENSHOT
>LODSProperties
>* Uri = screens/613989.jpg





### Repldiag discovery
An alternate methods of lingering object discovery is to use repldiag.exe with the /AdvisoryMode switch.  
Repldiag /removelingeringobjects /AdvisoryMode

#### :bulb: KNOWLEDGE
• Leverages DRSReplicaVerifyObjects method in Advisory Mode \)Like the Lingering Objects Liquidator)  
• Run against almost all DCs \)does not support RODCs), all partitions sans Schema  
• Event ID 1946s are logged on each DC in the forest   
• Need separate method to collect event message text from each DC for lingering object identification \)can leverage PowerShell)



#### :calling: COMMAND
```TypeText
Repldiag /removelingeringobjects /AdvisoryMode
```


### Replfix discovery
The Lingering Objects tool and repldiag do an excellent job of lingering object discovery. However, they do not identify one class of lingering objects, "Abandoned delete / Live lingering objects ".

#### :warning: ALERT
Replfix is an unsupported tool that can be leveraged for lingering object discovery and removal. In order to use it, you must first get LDIFDE dumps of the partition from DCs you want replfix to analyze, then you use the tool to compare the two ldifde files. The tool leverages the LDAP rootDSE removeLingeringObject modification for lingering object removal.

#### :bulb: KNOWLEDGE
**Abandoned delete / Live lingering objects**   
An object deleted on one DC that was never replicated to other DCs hosting a writable copy of the NC for that object. The deletion replicates to DCs/GCs hosting a read-only copy of the NC. The DC that originated the object deletion goes offline prior to replicating the change to other DCs hosting a writable copy of the partition.  
Replfix.exe does a good job of discovery of this lingering object type.  
  
  






### LDIFDE dumps from each DC
Perform the following task on Win8Client.  
  
LIDFDE dumps of the root partition from each DC  
Copy the following LDIFDE commands and paste into a command prompt on Win8Client.  
  
Ldifde -f dc1\_root.ldf -d "dc=root,dc=contoso,dc=com" -p subtree -r "\)objectclass=\*)" -l "replPropertyMetadata,objectGUID,replUptodateVector" -x -1 -s dc1.root.contoso.com  
  
Ldifde -f dc2\_root.ldf -d "dc=root,dc=contoso,dc=com" -p subtree -r "\)objectclass=\*)" -l "replPropertyMetadata,objectGUID,replUptodateVector" -x -1 -s dc2.root.contoso.com  
  
Ldifde -f trdc1\_root.ldf -d "dc=root,dc=contoso,dc=com" -p subtree -r "\)objectclass=\*)" -l "replPropertyMetadata,objectGUID,replUptodateVector" -x -1 -s trdc1.treeroot.fabrikam.com -t 3268   
  
Ldifde -f childdc1\_root.ldf -d "dc=root,dc=contoso,dc=com" -p subtree -r "\)objectclass=\*)" -l "replPropertyMetadata,objectGUID,replUptodateVector" -x -1 -s childdc1.child.root.contoso.com -t 3268   
  
Ldifde -f childdc2\_root.ldf -d "dc=root,dc=contoso,dc=com" -p subtree -r "\)objectclass=\*)" -l "replPropertyMetadata,objectGUID,replUptodateVector" -x -1 -s childdc2.child.root.contoso.com -t 3268

#### :bulb: KNOWLEDGE
For ease of command entry: There is a file on Win8Client in the D:\\files directory, called fix\_lab.txt that contains all necessary commands needed for this lab. There is a mixture of both CMD-line and PowerShell commands in the file. To execute the commands:   
1. Open an elevated PowerShell prompt on Win8Client.  
2. Copy the commands for the step you are working on, and paste them into the PowerShell window.  
3. It is best to copy the Files directory to the root of the C:\\ drive before executing any commands. Some commands attempt to output files to the current working directory \)which will fail for D:\\Files because it is a read-only ISO file attached to the VM guest.  
Alternately, you can copy them from the lab manual.



#### :calling: COMMAND
```TypeText
Ldifde -f dc1_root.ldf -d "dc=root,dc=contoso,dc=com" -p subtree -r "(objectclass=*)" -l "replPropertyMetadata,objectGUID,replUptodateVector" -x -1 -s dc1.root.contoso.com

Ldifde -f dc2_root.ldf -d "dc=root,dc=contoso,dc=com" -p subtree -r "(objectclass=*)" -l "replPropertyMetadata,objectGUID,replUptodateVector" -x -1 -s dc2.root.contoso.com

Ldifde -f trdc1_root.ldf -d "dc=root,dc=contoso,dc=com" -p subtree -r "(objectclass=*)" -l "replPropertyMetadata,objectGUID,replUptodateVector" -x -1 -s trdc1.treeroot.fabrikam.com -t 3268 

Ldifde -f childdc1_root.ldf -d "dc=root,dc=contoso,dc=com" -p subtree -r "(objectclass=*)" -l "replPropertyMetadata,objectGUID,replUptodateVector" -x -1 -s childdc1.child.root.contoso.com -t 3268 

Ldifde -f childdc2_root.ldf -d "dc=root,dc=contoso,dc=com" -p subtree -r "(objectclass=*)" -l "replPropertyMetadata,objectGUID,replUptodateVector" -x -1 -s childdc2.child.root.contoso.com -t 3268
```

#### :computer: ACTIONS
>LODSProperties
>* VM = Win8Client



### Copy replfix.exe to the new LDIF file location
replfix.exe is located on the C:\\files on the win8client. Ensure it is in the same directory as the files created in the prior step. \)there is also a copy of replfix on DC1)





### Use replfix to compare the LDIFDE files
Copy the replfix commands below to perform the comparison  
replfix dc1\_root.ldf dc2\_root.ldf -lingering dc1\_root\_lingering.ldf dc2\_root\_lingering.ldf -log root\_dc1\_dc2.log -domaindn "dc=root,dc=contoso,dc=com" -rootdn "dc=root,dc=contoso,dc=com"  
  
replfix dc1\_root.ldf childdc1\_root.ldf -lingering dc1\_root\_lingering\_childdc1.ldf childdc1\_root\_lingering.ldf -log root\_dc1\_childdc1.log -domaindn "dc=root,dc=contoso,dc=com" -rootdn "dc=root,dc=contoso,dc=com"  
  
replfix dc1\_root.ldf childdc2\_root.ldf -lingering dc1\_root\_lingering\_childdc2.ldf childdc2\_root\_lingering\_dc1.ldf -log root\_dc1\_childdc2.log -domaindn "dc=root,dc=contoso,dc=com" -rootdn "dc=root,dc=contoso,dc=com"  
  
replfix dc1\_root.ldf trdc1\_root.ldf -lingering dc1\_root\_lingering\_trdc1.ldf trdc1\_root\_lingering\_dc1.ldf -log root\_dc1\_trdc1.log -domaindn "dc=root,dc=contoso,dc=com" -rootdn "dc=root,dc=contoso,dc=com"  
  
Output should look similar to the attached screenshot

#### :bulb: KNOWLEDGE
**Replfix syntax**  
replfix <dc1.ldf> <dc2.ldf> -lingering <lingering1.ldf lingering2.ldf> \[-log <log.txt>\] \[-debug\] -domaindn "domaindn" \[-rootdn "rootdn"\]-bloom <id>

#### :camera: SCREENSHOT
>LODSProperties
>* Uri = screens/614034.JPG





### Review the replfix log
Review one of the .log files created by the various replfix commands to see a list of lingering objects present  
  
You can also view the screen output from the replfix commands for a quick overview of the level of divergence between the two DCs.  
  
See the attached screenshot for sample output.

#### :warning: ALERT
Pay attention to the scenarios where DCs hosting a writeable copy of the NC are compared against GCs - the second check in the examples above. Replfix.exe is currently the only tool that supports this reverse comparison. However, the objects discovered could simply be flagged due to AD replication latency. For that reason, investigate the replication metadata for each object to determine if it is truly a lingering object.

#### :bulb: KNOWLEDGE
In this example, DC1 \)which hosts a writeable copy of the root partition) has nine lingering objects according to TRDC1 \)which hosts a read-only copy of the partition).  
  
In this task, we used replfix.exe for discovery of lingering objects only \)not removal. The tool created importable LDIFDE files that could be leveraged for object cleanup. We will not be using this removal method. We will look at various removal methods in the next exercise.

#### :camera: SCREENSHOT
>LODSProperties
>* Uri = screens/614035.JPG






# Lingering Object Removal Methods
## INTRODUCTION MESSAGE
In this exercise, you will use LDP, Repadmin, Repldiag and Lingering Objects.exe to remove lingering objects.  
You will see the benefits of each method in order to help you understand which cleanup method to use.  
  
There are many methods to remove lingering objects. This lab presents:   
• Lingering Object Liquidator  
• Repldiag /removelingeringobjects  
• Repadmin /removelingeringobjects  
• RemoveLingeringObject rootDSE modification variants  
  
Other removal options include Repadmin /rehost | Repadmin /unhost with Repadmin /add \)for GC read-only partitions)  
  
A more detailed chart comparing methods can be found in the Appendix.
## COMPLETION MESSAGE
In this exercise, we removed lingering objects using LDP, repadmin, repldiag and Lingering Object Liquidator. We also identified and re-animated live lingering objects.
### Task 1: Remove lingering objects using LDP
Perform this task on Win8Client and ChildDC1

#### :bulb: KNOWLEDGE
In this task, you will discover lingering objects using the Lingering Object Liquidator, but you will remove one using LDP. LDP leverages the LDAP RemoveLingeringObject rootDSE modification. You could also use another LDAP tool to perform the same object removal procedure \)such as LDIFDE). The task is covered here so that a thorough review of lingering object removal methods are demonstrated in this exercise.  
  
In this task, you will remove a DNS record in the ForestDnsZones partition from ChildDC1 using LDP.





### Copy the Lingering Object Liquidator files
Connect to Win8Client.   
  
Copy the d:\\Files directory to the root of the c:\\ drive \)if you have not already in a prior exercise)




#### :computer: ACTIONS
>LODSProperties
>* VM = Win8Client



### Open the Lingering Objects Liquidator tool
Open the Lingering Objects Liquidator: "C:\\files\\LoL.exe"   
If the tool is open from a prior step, close it and reopen it





### Choose the Naming Context
Choose Naming Context and select dc=forestdnszones,dc=root,dc=contoso,dc=com





### Select the Reference D
Choose Reference DC and then select DC1.root.contoso.com





### Choose the Target DC and Detect
Choose Target DC and then select ChildDC1.child.root.contoso.com  
  
Select **Detect Lingering Objects**





### Review the Results
Two lingering objects should have been discovered.

#### :camera: SCREENSHOT
>LODSProperties
>* Uri = screens/614478.jpg





### Use LDP.exe to view a lingering object
Perform these steps from Win8Client  
  
Open LDP, connect and bind to the DC that has the lingering object  
a. From the Connection menu, choose Connect  
b. In the Server name field, type childdc1, ensure the port used is 389 and then choose OK

#### :bulb: KNOWLEDGE
**Get Object and Reference DC details for lingering object removal**  
We just used the Lingering Objects tool to discover a lingering object on ChildDC1 that does not exist on DC1.  
In order to remove an object using LDP, you need:  
  
 The objectGUID for the object \)We will use LDP to get this, but there are certainly many other methods)  
  
The DSA object GUID for a DC that hosts a writeable copy of the partition that does not have the object in the partition \)DC1 for this example).   
  
Next we will use LDP to view the DC93 object in order to get the objectGUID of the object

#### :camera: SCREENSHOT
>LODSProperties
>* Uri = screens/614483.jpg




#### :computer: ACTIONS
>LODSProperties
>* VM = Win8Client



### Choose the Connection menu
Select the Connection menu again, select Bind and then OK. \)Ctrl \+ B is the keyboard shortcut)





### Select the ForestDNSZones partition
From the View menu, select Tree, from the BaseDN menu, select DC=ForestDnsZones,DC=root,DC=contoso,DC=com and then select OK.





### Select the _msdcs zone
Expand DC=ForestDNSZones…, expand CN=MicrosoftDNS…, expand DC=\_msdcs.root.contoso.com…

#### :camera: SCREENSHOT
>LODSProperties
>* Uri = screens/614489.jpg





### Copy the objectGUID for DC93
Double click DC=93 and copy the objectGUID for the DC93 object  
  
Leave LDP open as it is needed after the following step

#### :camera: SCREENSHOT
>LODSProperties
>* Uri = screens/614609.jpg





### Get the DSA objectGUID for DC1
Get the DSA object GUID for DC1  
repadmin /showrepl DC1  
\)one of many ways to get the DSA object GUID for DC1 is via repadmin /showrepl)

#### :bulb: KNOWLEDGE
We now have everything we need to remove this object:  
1. The objectGUID of the lingering object:  
3e873993-982b-47e8-8f20-5c50a5860ba8  
2. The DSA object GUID from a DC that is a good reference DC \)hosts a writable copy of the partition and does not contain the object)  
70ff33ce-2f41-4bf4-b7ca-7fa71d4ca13e

#### :camera: SCREENSHOT
>LODSProperties
>* Uri = screens/614610.JPG



#### :calling: COMMAND
```TypeText
repadmin /showrepl DC1
```


### Use LDP to remove the Lingering Object
Switch back to LDP, from the Browse menu, select**Modify**  
  
In the Attribute box, type **RemoveLingeringObject.**





### Build the RemoveLingeringObject value
Type **<GUID=** as the value.  
  
Append the DSA object GUID of the reference domain controller  
**<GUID=70ff33ce-2f41-4bf4-b7ca-7fa71d4ca13e**  
  
Append **> : <GUID=**. Do not omit the spaces.  
  
It should look similar to this:  
<GUID=70ff33ce-2f41-4bf4-b7ca-7fa71d4ca13e> : <GUID=





### Continue building the value
Append the ObjectGUID of the lingering object.  
  
Append **>**.  
  
The complete value should look similar to:  
<GUID=70ff33ce-2f41-4bf4-b7ca-7fa71d4ca13e> : <GUID=3e873993-982b-47e8-8f20-5c50a5860ba8>





### Commit the RemoveLingeringObject value
Click the Replace operation, and then click Enter on the interface. Now the command appears in the Entry list.

#### :camera: SCREENSHOT
>LODSProperties
>* Uri = screens/614614.jpg





### Remove the object
Click Run to have the object removed. The main content pane of LDP contains the result of the request. It will look like this if the operation was successful.  
\*\*\*Call Modify...  
ldap\_modify\_s\)ld, '\)null)',\[1\] attrs);  
Modified "".





### Use LoL to verify the lingering object removal
Switch back to the Lingering Object Liquidator tool and Select the Detect Lingering Objects button.  There should be only one lingering object left now.

#### :camera: SCREENSHOT
>LODSProperties
>* Uri = screens/689294.png





### Task 2: Remove lingering objects using repadmin
In the last task, we removed one object from the ForestDNSZones partition from ChildDC1. However, one or more lingering objects still remain, so replication is still blocked. In this task, we will use repadmin /removelingeringobjects to remove the remaining objects from this partition \)as compared with DC1).

#### :warning: ALERT
Lingering Object removal using repadmin /removelingeringobjects  
The command's syntax is:  
repadmin /removelingeringobjects LingeringDC ReferenceDC\_DSA\_GUID PartitionDN  
Where:  
 LingeringDC: FQDN of DC that has the lingering objects  
 ReferenceDC\_DSA\_GUID: The DSA GUID of a domain controller that hosts a writeable copy of the partition  
 PartitionDN: The distinguished name of the directory partition where the lingering objects exist

#### :bulb: KNOWLEDGE
In this task, you will remove lingering objects using repadmin /removelingeringobjects.  
• Repadmin /removelingeringobjects  
Remove objects from one partition on one DC per command line execution  
• Note that may also cleanup lingering objects in GC read-only partitions by rehosting the partition using repadmin  
o Repadmin /rehost  
o Repadmin /unhost followed up with repadmin /add





### Delete the objects with repadmin
Run the following command to clean up the remaining object\)s) in the ForestDNSZones partition on childdc1  
Repadmin /removelingeringobjects childdc1.child.root.contoso.com 70ff33ce-2f41-4bf4-b7ca-7fa71d4ca13e "dc=forestdnszones,dc=root,dc=contoso,dc=com"



#### :calling: COMMAND
```TypeText
Repadmin /removelingeringobjects childdc1.child.root.contoso.com 70ff33ce-2f41-4bf4-b7ca-7fa71d4ca13e "dc=forestdnszones,dc=root,dc=contoso,dc=com"
```


### Confirm the removal request
Review the Directory Service event log on ChildDC1 for the results of the lingering object removal request. Review the details of event ID 1939, which reports the status of the lingering object removal process.  
  
You can use Event Viewer or the following PowerShell command:  
get-winevent -LogName "Directory Service" -ComputerName childdc1 -MaxEvents 10 | Where-Object \{$\_.ID -eq "1939"\} | fl

#### :bulb: KNOWLEDGE
At this point, the ForestDNSZone partition is clean on childdc1 as compared to DC1. Thoroughly cleaning this partition requires that you compare childdc1 against everyone else and then compare all of them against childdc1. Also, keep in mind, that if there are lingering objects in one partition, there are usually lingering objects in the other partitions.

#### :camera: SCREENSHOT
>LODSProperties
>* Uri = screens/614618.jpg



#### :calling: COMMAND
```TypeText
get-winevent -LogName "Directory Service" -ComputerName childdc1 -MaxEvents 10 | Where-Object {$_.ID -eq "1939"} | fl
```


### Task 3: Remove lingering objects using Repldiag
In the last task, you cleaned up one partition on one DC. There is still a lot of work to do if you want to do a thorough job of lingering object removal though. In this task, you will leverage a tool that automates the majority of the lingering object removal work needed for most environments.  
  
Repldiag will run commands to remove lingering objects from all partitions.

#### :bulb: KNOWLEDGE
• Repldiag requires a well-connected topology. It will fail to run in environments that suffer from poor network connectivity \*.  
• Always check for the latest version on CodePlex:  
http://activedirectoryutils.codeplex.com/  
\* There is a hidden parameter that allows the tool to continue in spite of topology issues, but do not use it without recognizing the ramifications: Use of the /BypassStabilityCheck parameter will likely result in a failure to fully clean up the environment.  
  
When lingering objects are discovered, assume they are present on all DCs in all partitions. Do not just clean up the DCs reporting the errors. Repldiag automates the majority of the cleanup work. See the Lingering Object discovery and cleanup section more information.





### Remove lingering objects from most DCs
The following command will check for and remove lingering objects from most DCs \)RODCs are not checked) for all partitions \)except Schema)  
  
From Win8Client, run the following from an elevated command prompt  
Repldiag /removelingeringobjects



#### :calling: COMMAND
```TypeText
Repldiag /removelingeringobjects
```

#### :computer: ACTIONS
>LODSProperties
>* VM = Win8Client



### Detect remaining objects using LoL
Close and Reopen the Lingering Object tool \)if already opened) ), Detect AD Topology and   
  
Are all objects removed from the environment?  
  
Notice the RODC in the child domain still contains lingering objects.

#### :warning: ALERT
At the time of this writing, Replidag \)v 2.0.4947.18978) does not remove lingering objects from RODCs. \)It was developed prior to the existence of RODCs.) This functionality will likely never be implemented. 

#### :bulb: KNOWLEDGE
There are alternate steps for deleting lingering objects using RepdAdmin in the Appendix .   
  
You can find them in the Exercise 3 section.





### Task 4: Use Lingering Object Liquidator
In Exercise 2, you leveraged LOL to remove individual objects. You also have a two-button option to remove all objects present in the environment. In this task, you will remove the remaining lingering objects visible in the tool.  
  
On Win8client, open the Lingering Object Liquidator \)LoL) if not already open.




#### :computer: ACTIONS
>LODSProperties
>* VM = Win8Client



### Detect remaining lingering objects
Use LoL to do another forest-wide scan to see if repldiag already removed all objects from the environment.

#### :camera: SCREENSHOT
>LODSProperties
>* Uri = screens/614623.jpg





### Remove the lingering objects
Click the **Select All** button, and then click the **Remove Selected Lingering Objects** button. The status of object removal is shown in the bottom pane. It’s normal to see lingering object removal failures in this scenario as the same object may be displayed on the same target may be listed multiple times since we’ve scanned against multiple references.  Multiple attempts to remove the same object fail with “An operation error occurred”





### See if any lingering objects remain using LoL
Click the **Detect Lingering Objects** button again.  If any remain, click **Select All** and **Remove Selected Lingering Objects** again.  Repeat if necessary.

#### :camera: SCREENSHOT
>LODSProperties
>* Uri = screens/614625.jpg





### Initiate replication on all DCs
Initiate replication on all DCs by running the following commands  
  
Repadmin /syncall dc1 /Aed  
Repadmin /syncall dc2 /Aed  
Repadmin /syncall childdc1 /Aed  
Repadmin /syncall childdc2 /Aed  
Repadmin /syncall trdc1 /Aed



#### :calling: COMMAND
```TypeText
Repadmin /syncall dc1 /Aed
Repadmin /syncall dc2 /Aed
Repadmin /syncall childdc1 /Aed
Repadmin /syncall childdc2 /Aed
Repadmin /syncall trdc1 /Aedb
```


### Check forest-wide AD replication
Check forest-wide AD replication using ADReplstatus or repadmin /showrepl \* /csv  
   
Open a PowerShell prompt if using the following command:   
  
repadmin /showrepl \* /csv | convertfrom-csv | out-gridview



#### :calling: COMMAND
```TypeText
repadmin /showrepl * /csv | convertfrom-csv | out-gridview
```


### Review the results
The only replication error that remains is error 8606 for the Child, Root and TreeRoot partitions.   
  
Why are there still symptoms of lingering objects in the environment?  
  
In the next lesson, we will explore a special class of lingering objects not detected via the DRSReplicaVerifyObjects method: abandoned objects.

#### :camera: SCREENSHOT
>LODSProperties
>* Uri = screens/614628.jpg





### Task 5: "Live" lingering object remediation
After a thorough removal of lingering objects in the last task, we discovered there are still symptoms of lingering objects in the environment. In this exercise, we explore a special class of lingering object, called a "live" lingering object.

#### :warning: ALERT
**"Live" lingering object / Abandoned deleted object**  
An object deleted on one DC that never replicated to other DCs hosting a writable copy of the NC for that object. The deletion replicates to DCs/GCs hosting a read-only copy of the NC. The DC that originated the object deletion goes offline prior to replicating the change to other DCs hosting a writable copy of the partition. The lingering object remains "live" on the remaining DCs due to the abandoned delete.

#### :bulb: KNOWLEDGE
**Scenario:**  
Destination DC/GCs report that source DCs have lingering objects in source DC partition:  
 Root.contoso.com: DC1 and DC2  
 Child.root.contoso.com: ChildDC1 and ChildDC2  
 ChildDC1 replicates Root partition from DC1 and replication fails with error 8606  
  
Event 1988 can identify one object for us, but as discussed earlier, event 1988 only reports the first object encountered and there are usually many more. We will use replfix.exe to identify the rest.





### Generate required ldif exports
From win8client, switch to the C:\\Files directory \)folder copied from the D drive in an earlier exercise)  
  
Execute ldifde\_replfixCMDs.bat

#### :bulb: KNOWLEDGE
• The contents of the ldifde\_replfixCMDs batch file are also included in the Appendix.  
• This batch file initiates all of the ldifde exports that replfix.exe needs for its analysis.



#### :calling: COMMAND
```TypeText
ldifde_replfixCMDs.bat
```

#### :computer: ACTIONS
>LODSProperties
>* VM = Win8Client



### Run replfix_cmds.bat
Execute the Replfix\_cmds.bat file \)also included in the Appendix).

#### :bulb: KNOWLEDGE
• This runs replfix against each of the LDIF files in a pairwise fashion so that all DCs are checked for their respective partitions.   
• There are two LDIF files and one log generate for each commands execution.   
• The summary output for all command execution is in the file, run.log.



#### :calling: COMMAND
```TypeText
Run replfix_cmds.bat
```


### Review the results
Open the run.log file and examine the output to help determine the scope of the problem

#### :bulb: KNOWLEDGE
We can see from the example that the lingering objects in the root partition on ChildDC1 have been removed \)from our repldiag and LingeringObjects.exe removal cleanup steps in Tasks 3 and 4), but DC1 \)which hosts a writeable copy of the root partition)still has 9 lingering objects according to ChildDC1 \)which hosts a read-only copy of the partition).

#### :camera: SCREENSHOT
>LODSProperties
>* Uri = screens/614632.JPG





### Review the DC to DC  comparison logs
A review of the logs for the root partition reveals that DC1 and DC2 have two user objects, "Carl Woodbury" and "Cassie McKenzie" and child objects associated with the same two user objects while the GCs do not.  
  
The next step is to determine if these objects are not present on the GCs due to AD replication latency or if they are perhaps live lingering objects.





### Collect replication metatdata for each object
Obtain the Object GUID for the Carl Woodbury user object. This is displayed in the root\_dc1\_childdc1.log file.  
Repadmin /showobjmeta \* "<GUID=2fb239db-eb8c-47ad-9b1d-67323e01c112>" >carl\_obj.txt  
  
Analyze the data and determine if this is an AD replication latency issue, or if these are live lingering objects.



#### :calling: COMMAND
```TypeText
Repadmin /showobjmeta * "<GUID=ObjectGUID>" >carl_obj.txt
```


### Retrieve USN and DSA GUIDs
Look at attribute metadata for an attribute that was stamped at time of object creation \)such as objectClass) and is still version 1. In other words, identify the following metadata for the earliest timestamp so you can determine when the object was created.  
a. Originating DSA GUID  
b. Originating USN





### Find the object's partition
Look at the /ShowUTDVec output for the object's partition  
  
a. Open utdvec\_root.txt  
b. For the objects that replfix discovered: correlate the replication metadata with the /showutdvec output from each GC to determine if that GC should have seen the object creation USN.  
c. If the GC has a USN value for the DSA GUID higher than the one used for object creation by the same DSA, then this GC should have seen this object creation. This data indicates a live lingering object / abandoned delete scenario. If the USN in the showutdvec output is lower than the value used for object creation then this is AD replication latency.

#### :camera: SCREENSHOT
>LODSProperties
>* Uri = screens/614636.JPG





### Review the data for lingering object information
The data reveals that ChildDC1 has seen all changes up to and including USN 40967 from the DC that originated the Carl Woodbury object.  The DC that originated the object used USN 32942.  So that means that ChildDC1 should have replicated / instantiated this object, so this is not a case of AD replication latency. A review of the data reveals that the rest of the DCs also have "live" lingering objects in their partitions according to the GCs.

#### :camera: SCREENSHOT
>LODSProperties
>* Uri = screens/614637.JPG





### Compare remediation scenarios
There is a Lingering Object Scenarios example in the Exercise 3 section of the Appendix. The Lab Appendix is on the desktop of the Win8Client. Review the scenarios and consider which one would be most effective.  
  
To save lab time, we go with the easiest / fastest method. However, weigh the Pros and Cons of each scenario for your customer's environment. I prefer the authoritative restore of each object method since that option poses the least amount of risk to the environment.





### Replicate the lingering objects to all GCs
Use repadmin /replicate with the /full parameter to have the GCs get a copy of the live lingering object\)s), then update replication status.  
  
repadmin /replicate dc2 dc1 dc=root,dc=contoso,dc=com /full  
repadmin /replicate dc1 dc2 dc=root,dc=contoso,dc=com /full  
repadmin /replicate \* dc1 dc=root,dc=contoso,dc=com /full  
repadmin /replicate \* childdc1 dc=child,dc=root,dc=contoso,dc=com /full  
repadmin /replicate \* trdc1 dc=treeroot,dc=fabrikam,dc=com /full  
repadmin /syncall dc1 /Aed  
repadmin /syncall dc2 /Aed  
repadmin /syncall childdc1 /Aed  
repadmin /syncall childdc2 /Aed  
repadmin /syncall trdc1 /Aed



#### :calling: COMMAND
```TypeText
repadmin /replicate dc2 dc1 dc=root,dc=contoso,dc=com /full
repadmin /replicate dc1 dc2 dc=root,dc=contoso,dc=com /full
repadmin /replicate * dc1 dc=root,dc=contoso,dc=com /full
repadmin /replicate * childdc1 dc=child,dc=root,dc=contoso,dc=com /full
repadmin /replicate * trdc1 dc=treeroot,dc=fabrikam,dc=com /full
repadmin /syncall dc1 /Aed
repadmin /syncall dc2 /Aed
repadmin /syncall childdc1 /Aed
repadmin /syncall childdc2 /Aed
repadmin /syncall trdc1 /Aed
```


### Check forest-wide replication status
Use AD Replication Status or another method to check forest-wide AD replication health.  
  
AD Replication now completes successfully for each partition. However, there are still data divergence issues in this Active Directory environment. In the next optional Exercise, we will leverage a tool called Oabvalidate to aid in the discovery of the data divergence.

#### :bulb: KNOWLEDGE
Basic Data collection to identify abandoned objects  
• Sample object is present in the Engineering OU  
• Repadmin /showattr \* "<GUID=ObjectGuid>" /gc >show.txt  
• Repadmin /showobjmeta \* "<GUID=ObjectGUID>" >>show.txt  
• Identify Originating DSA for object creation from showobjmeta output  
• Use Repadmin /showutdvec to determine highest USN received by RW replicas from originator of this object






# OPTIONAL: Lingering Link identification and cleanup
>LODSProperties
>* IntroductionUri = https://lodmanuals.blob.core.windows.net/manuals/Ignite 2016/Videos/IDL3088/Exercise 4- Lingering Link Identification and Cleanup-.mp4

## INTRODUCTION MESSAGE
**Time permitting. This exercise is not fully documented due to time constraints. Try this exercise if you have adeaquate time.**  
  
During this exercise, you will identify all lingering-linked values in the environment. You will leverage a tool called Oabvalidate.exe that was originally written for Microsoft Exchange Offline Address Book generation failure troubleshooting. Further development went into the tool recently to help in the discovery of other AD data inconsistency issues. It is not a requirement to have Exchange in the environment \)if you execute the tool from a command-line and pass an LDAP filter as an argument). The tool scans for a variety of AD data inconsistencies and logs the data to the user's Documents directory.
## COMPLETION MESSAGE

### Scenario: Group membership consistency issues
Perform these tasks on Win8Client




#### :computer: ACTIONS
>LODSProperties
>* VM = Win8Client



### Run oabvalidate
Open an elevated command prompt and run oabvalidate.exe against DC1  
Oabvalidate dc1 "\)Objectclass=\*)"  
  
• Ignore the Oabvalidate window that opens and closes  
• Output is logged in the Documents directory in a folder named data\_timestamp-<DC Name>



#### :calling: COMMAND
```TypeText
Oabvalidate dc1 "(Objectclass=*)"
```

#### :computer: ACTIONS
>LODSProperties
>* VM = Win8Client



### Check DC2 in the same way
Next check DC2  
Oabvalidate dc2 "\)Objectclass=\*)"  
  
If the command appears to hang without returning to a command prompt, open a new command prompt window and run the remaining commands one at a time



#### :calling: COMMAND
```TypeText
Oabvalidate dc2 "(Objectclass=*)"
```


### Check ChildDC1
Check ChildDC1  
Oabvalidate childdc1 "\)Objectclass=\*)"



#### :calling: COMMAND
```TypeText
Oabvalidate childdc1 "(Objectclass=*)"
```


### Check ChildDC2
Check ChildDC2  
Oabvalidate childdc2 "\)Objectclass=\*)"



#### :calling: COMMAND
```TypeText
Oabvalidate childdc2 "(Objectclass=*)"
```


### Check TRDC1
Finally, check TRDC1  
Oabvalidate trdc1 "\)Objectclass=\*)"



#### :calling: COMMAND
```TypeText
Oabvalidate trdc1 "(Objectclass=*)"
```


### Open the problemattributes text file
Open problemattributes.txt in Excel \)tab and semicolon delimited). Steps for this are in the Appendix in the Open problemattributes.txt in Excel section.  
  
Problem attributes.txt from each DC reveals the following scenario:  
• There are many lingering links in the member attribute of several group objects.  
• The group membership inconsistencies are all for read-only copies of the group.

#### :bulb: KNOWLEDGE
A consolidated copy of this data is present in the c:\\files\\ALL\_DCs\_ProblemAttributes.xlsx file to speed up data analysis for this lab.





### Identify the problem group
Identify one object on DC1: LLGroup1 is listed with two member attributes listed as lingeringLink

#### :camera: SCREENSHOT
>LODSProperties
>* Uri = screens/614798.JPG





### Collect object metadata
Collect repadmin /showattr and repadmin /showobjmeta data for this \)and all other objects) reported in the problemattributes.txt files.  
  
A batch file that collects this data is located in the c:\\Files directory.  
repadmin\_cmds.bat

#### :camera: SCREENSHOT
>LODSProperties
>* Uri = screens/614799.JPG



#### :calling: COMMAND
```TypeText
repadmin /showattr * "<GUID=8a6efacc-bc38-4431-b577-2b3207f90155>" /filter:"(objectclass=*)" /deleted /atts:member /long /allvalues /gc >obj_8a6efacc-bc38-4431-b577-2b3207f90155.txt
repadmin /showobjmeta * "<GUID=8a6efacc-bc38-4431-b577-2b3207f90155>" /linked  >>obj_8a6efacc-bc38-4431-b577-2b3207f90155.txt
repadmin /showattr * "<GUID=0974a6d0-8a75-4f9b-bb83-be236c1e43f7>" /filter:"(objectclass=*)" /deleted /long /allvalues /gc >0974a6d0-8a75-4f9b-bb83-be236c1e43f7.txt
repadmin /showobjmeta * "<GUID=0974a6d0-8a75-4f9b-bb83-be236c1e43f7>" /linked >>0974a6d0-8a75-4f9b-bb83-be236c1e43f7.txt
```


### Review group membership differences
Review group membership differences for object LLGroup1. This data is collected in the repadmin\_cmds.bat file: obj\_8a6efacc-bc38-4431-b577-2b3207f90155.txt

#### :bulb: KNOWLEDGE
DCs in the child domain host a writable copy of this object. ChildDC1 is the authoritative source for this object since the only other DC in the Child domain is an RODC.  
DC1, DC2 and TRDC1 list four users in the member attribute in LLGroup1. ChildDC1 only reports two users in this group.

#### :camera: SCREENSHOT
>LODSProperties
>* Uri = screens/614800.JPG





### Review the object replication metadata
Review the replication metadata for these objects.  
  
Look at the object's attribute values on DCs containing a writable copy of the object and compare them to GCs.  
  
Several of the members for these group objects do not exist on the DCs that hosts a writable copy of the partition.

#### :camera: SCREENSHOT
>LODSProperties
>* Uri = screens/614801.JPG





### Use Repadmin /showobject to view metadata
From repadmin /showobjmeta output, we can see that the user Object was created on a DC with this DSAGUID 606f5d34-7202-4073-83fb-aac8bb109868 at 2013-05-10 04:36:04. We will use this information in the next step when we look at the up-to-dateness vector on each DC.

#### :camera: SCREENSHOT
>LODSProperties
>* Uri = screens/614802.JPG





### Review Repadmin /showeutdvec data
Review the repadmin /showutdvec data for each of these partitions and compare with the replication metadata for each of the objects found in the prior step.  
  
repadmin /showutdvec \* "dc=child,dc=root,dc=contoso,dc=com" /latency /nocache >utdvec\_child.txt

#### :bulb: KNOWLEDGE
From this output, we can see that highest change that ChildDC1 received from the replication partner that created the Brackish Waters account is 152523. However, the USN used by the DC that created the object is higher than the one in the up-to-dateness vector for ChildDC1.  
  
Therefore ChildDC1 never received the originating create for this object.  
• Showutdvec from other DCs does show that they received this and other changes: 152695 @ Time 2013-05-10 05:05:19  
  
This is an abandoned object.

#### :camera: SCREENSHOT
>LODSProperties
>* Uri = screens/614803.JPG



#### :calling: COMMAND
```TypeText
repadmin /showutdvec * "dc=child,dc=root,dc=contoso,dc=com" /latency /nocache >utdvec_child.txt
```


### ID abandoned objects
Identify the abandoned objects based on the Oabvalidate and replication metadata output.  
  
Leverage the consolidated Problem Attributes Excel file.   
  
Abandoned objects can be removed with the LDAP RemoveLingeringObject rootDSE modify procedure. Perhaps the easiest way to do all these objects in bulk is to remove them all from all GCs.

#### :warning: ALERT
To save lab time, the full analysis is done for you. It is documented in the Data Analysis tab in the All\_DCs\_ProblemAttributes.xlsx file. The process used for the analysis is detailed in the Abandoned object identification using conditional formatting section in the Appendix.





### Create a Lingering Objects CSV file
Create a Lingering Objects tool importable CSV file to make light work of the abandoned object removal.   
  
o You can also leverage one that has been created for you in the C:\\files directory: abandoned.csv  
o Once you have the file, open the Lingering Objects tool and select the Import button, browse to the file and choose Open.  
  


#### :bulb: KNOWLEDGE
The following format is required for the CSV file:  
FQDN of RWDC,CNAME of RWDC,FQDN of DC to remove object from, DN of the object, Object GUID of the object, DN of the object's partition

#### :camera: SCREENSHOT
>LODSProperties
>* Uri = screens/614805.jpg





### Import the CSV file
Open the Lingering Objects tool and select the Import button, browse to the file C:\\Files\\abandoned.csv and choose Open.

#### :camera: SCREENSHOT
>LODSProperties
>* Uri = screens/614809.jpg





### Remove the objects
Select all objects and then choose Remove.

#### :camera: SCREENSHOT
>LODSProperties
>* Uri = screens/614810.jpg





### Review replication metadata
Review replication metadata to verify the objects were removed.  
  
What impact does this have on the group membership issues for the same objects?  
  
All issues related to these four objects are cleared up except one: Brackish Waters is still listed as a member on Oabvalidate output from childdc1.





### Address group membership issues
This is one of the easier scenarios to correct because you can simply add the user back to the group and then remove them again.  
  
CN=Tabatha Acosta,OU=Sales,DC=child,DC=root,DC=contoso,DC=com  
CN=Juliette Lancaster,OU=SingleSignOn,DC=root,DC=contoso,DC=com  
CN=Ulysses Breland,OU=SingleSignOn,DC=root,DC=contoso,DC=com  
  
Are lingering links in this group:  
CN=LLinkGroup1,OU=LLinkgroups,DC=treeroot,DC=fabrikam,DC=com  
  
Further investigation will discover similar objects in other groups.

#### :warning: ALERT
The files needed to fix group membership for the absent users have already been created for you to save lab time.  
1. Import FixAbsent.ldf  
ldifde -i -f c:\\files\\FixAbsent.ldf -s trdc1.treeroot.fabrikam.com  
2. Import DeletaAbsentUsers.ldf  
ldifde -i -f c:\\files\\deleteAbsentusers.ldf -s trdc1.treeroot.fabrikam.com

#### :bulb: KNOWLEDGE
To correct group membership for objects that no longer exist you need to create new objects with the same extended DN as the phantom members and then delete them again  
  
1. Review membership enable extended DN control  
2. Copy GUID out of the member attribute \)from extended DN in member attribute) for all missing members  
3. Convert the objectGUID portion to base64  
4. Create LDIFDE importable files to add users  
5. Change DSHeuristics to allow ObjectGUID specify on add  
6. Import ldifde  
7. Remove objects



#### :calling: COMMAND
```TypeText
ldifde -i -f c:\files\FixAbsent.ldf -s trdc1.treeroot.fabrikam.com

ldifde -i -f c:\files\deleteAbsentusers.ldf -s trdc1.treeroot.fabrikam.com
```



# NOT REQUIRED: Troubleshoot and resolve AD replication error 8614
## INTRODUCTION MESSAGE
**8614 | The directory service cannot replicate with this server because the time since the last replication with this server has exceeded the tombstone lifetime.**  
 Important: This exercise is needed only if error 8614 is logged in showrepl or adreplstatus output.  
  
Error 8614 is logged when a destination DC has not replicated with a source DC over an existing replication connection for longer than tombstone lifetime.   
 **Warning:**  
• This quarantine is put in place on a per-replica, per-partition basis so that replication with an out of date DC does not introduce lingering objects into the environment.   
• If this issue occurs in a production environment, careful consideration should be made prior to removing the replication safeguard.   
• In some cases, forceful demotion of the source DC makes more sense. See the content linked in the appendix for more information.  
• Large jumps in system time \)forward or backward) are common causes of this issue  
  
In this exercise, you will use repadmin to resolve AD replication error 8614 in a supported manner.
## COMPLETION MESSAGE

### Get replication status
Run the AD Replication Status tool or repadmin /showrepl \* /csv. Review the output.   
**If AD replication error 8614 is not present, then do not do this exercise.**




#### :computer: ACTIONS
>LODSProperties
>* VM = Win8Client



### Set strict replication consistency
Ensure Strict Replication consistency is set on all DCs  
Repadmin /regkey \* \+strict  
  
In the output of the above command, verify status for all DCs: registry key set  
"Strict Replication Consistency" REG\_DWORD 0x0000001 \)1)



#### :calling: COMMAND
```TypeText
Repadmin /regkey * +strict
```


### Remove lingering objects
Remove lingering objects if present using repldiag \)skip if already performed in exercise 4).  
Repldiag /removelingeringobjects



#### :calling: COMMAND
```TypeText
Repldiag /removelingeringobjects
```


### Prepare to fix failed replications
Run the following command on destination DCs that fail to replicate from source DCs with error 8614: \)replace DestinationDCName with the actual DC name)  
Repadmin /regkey DestinationDCName \+AllowDivergent  
  
In this lab environment, it is safe to just temporarily set the registry value on all DCs  
Repadmin /regkey \* \+AllowDivergent  
  
Verify status from all DCs:  
"Allow Replication With Divergent and Corrupt Partner" REG\_DWORD 0x0000001 \)1)

#### :warning: ALERT
Do not run the following command without first verifying that Strict replication consistency is enabled.



#### :calling: COMMAND
```TypeText
Repadmin /regkey * +AllowDivergent
```


### Initiate replication
Initiate replication to all destination DCs from all source DCs where replication failed with status 8614





### Review the results of latest replication
Use repadmin /showrepl \* /csv or the AD Replication Status tool to verify error 8614 is no longer logged in the environment





### Remove replication quarantine safeguards
Delete the registry value so that the replication quarantine safeguards are back in place  
Repadmin /regkey \* -AllowDivergent\\  
  
Your lab is now complete  




#### :calling: COMMAND
```TypeText
Repadmin /regkey * -AllowDivergent
```



