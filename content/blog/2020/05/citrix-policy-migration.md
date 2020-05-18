+++
author = "Scott Caster	"
author_url = "https://github.com/scaster513"
categories = ["Scott Caster", "Citrix", "Policy", "Migrate"]
date = "2020-05-14"
description = "Simple process for policy migration from xenapp 6.5 to Citrix 7.xx"
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
title = "Migrate Citrix Policies from XenApp 6.5 to >7.6"
type = "post"

+++

## Migrate Citrix Policies from XenApp 6.5 to >7.6

In this post I will outline the process I used to overcome the limitations of existing Citrix policy migrating tools. Per Citrix, there is no direct patch to migrate versions 6.5 to 7.7 or above, you must first install and migrate to 7.6, then do an in-place upgrade to the desired version. ( source: https://docs.citrix.com/en-us/xenapp-and-xendesktop/7-15-ltsr/upgrade-migrate/xenapp-worker-upgrade.html ). Unfortunatly for me, I discovered this only after my brand new 7.19 farm was stood up. Before tearing it all down, I wanted to dig a bit deeper to see if there was a better way. If you are in the same boat, or if you would like to try a more streamlined approach, the following steps will help you save some time. They are quick and relatively painless. You will need to use and modify Powershell modules. If you are uncomfortable with PowerShell this is a great learning opportunity. I will break the process down into 4 parts. 

# Export
1.	Download the Citrix Migration Tool from the xenapp 7.6 download site: https://www.citrix.com/downloads/xenapp/product-software/xenapp-76-enterprise-edition.html
     * Important note, be sure to download the migration "scripts", not the migration "tool". The files we need are ReadIMA.zip and ImportFMA.zip. If you see XDExport.zip and XDImport.zip, you have the the actual tool and it will not work. 
	 ![migration_script_download](/img/2020/05/migration_script_download.png)
2.	On a 6.5 farm server ensure you have windows management framework 3 installed, I used wmf 4. You can download that here: https://www.microsoft.com/en-us/download/details.aspx?id=40855
3.	Extract ReadIMA.zip to c:\temp
4.	Open and administrator command prompt and execute the following:
     * Note if you get security warnings press "R" to ignore and load scripts
     ![script_warning](/img/2020/05/script_warning.png)

```bash
cd c:\temp\ReadIMA
import-module .\exportpolicy.psd1
import-module .\exportxafarm.psd1
export-policy -XmlOutputFile .\policies.xml -LogFile .\policy-export.log
```

5.	If you have a large number of policies you will want to output a file now that we can use to compare the to the new farm later on. Run the below commands: 

```bash
if (!(Get-PSDrive | where name -eq LocalFarmGpo)) { 
asnp citrix*
import-module .\exportpolicy.psd1
import-module .\exportxafarm.psd1
}
cd LocalFarmGpo:\
$userpolicies=Get-ChildItem .\User |Select-Object name
$outfile="c:\temp\ctxpolicy-old.txt"
foreach ($p in $userpolicies) {
    Get-ItemProperty -path .\User\$($p.name)  |Out-File $outfile -Append
    Get-ChildItem .\User\$($p.name) -Recurse |where state -eq enabled |Out-File $outfile -Append
    Get-ChildItem .\User\$($p.name) -Recurse |where enabled -eq true |Out-File $outfile -Append
}
```

6.	This will create a file c:\temp\ctxpolicy.txt (feel free to rename it) that contains all of policies on the farm. Save this for the verification process later. 
7.	After the policy export completes, copy the policies.xml file from the 6.5 server to a 7.xx delivery controller and place in c:\temp\ImportFMA

# Import


1.	On the 7.xx delivery controller, ensure the Citrix Group Policy Management and Powershell snappins are installed. 
2.	Extract ImportFMA.zip to c:\temp\ImportFMA 
3.	Right click on the ImportPolicy.psm1 file, select edit and remove or comment out the below

```bash
    $p = [Reflection.Assembly]::LoadWithPartialName("Citrix.GroupPolicy.PowerShellProvider")
    if ($p -eq $null)
    {
        Write-Error ([string]::Format("{0}`n{1}",
                                      "Citrix Group Policy Provider Snapin is not installed",
                                      "You must have Citrix Group Policy Provider Snapin installed to use this script."))
        return
    }

    Write-LogFile ('Import-Module -Assembly [Reflection.Assembly]::LoadWithPartialName("Citrix.GroupPolicy.PowerShellProvider")') 1
    [void](Import-Module -Assembly ([Reflection.Assembly]::LoadWithPartialName("Citrix.GroupPolicy.PowerShellProvider")))
```
4.	Add the following code in its place

```bash
    if (!$(Get-PSSnapin |where name -Match Citrix.Common.GroupPolicy)) {
        Write-Output "Adding Citrix snap-in"
        asnp Citrix*
    } 
```	

5.	Save the file and open Powershell as administrator
6.	Execute the following commands. If you receive an error about unapproved verbs, ignore it.
    ![verb_warning](/img/2020/05/psverb_warning.png)
```bash
	cd c:\temp\ImportFMA
	Import-Module .\ImportPolicy.psm1
	Import-Module .\ImportXAFarm.psm1
	Import-Module .\LogUtilities.psm1
	Import-Module .\XmlUtilities.psm1
```
 
7.	Test policies import with the following command 
```bash
	Import-Policy -XmlInputFile .\policies.xml -LogFile .\policy-import.log -preview
```
    
8	Examine the output for any errors. If you see warnings about valid settings these can usually be ignored as tehre are some differences between how 6.5 and 7.xx utilize policies for configuration.

9.	Remove the -Preview switch to import for real this time
```bash
	Import-Policy -XmlInputFile .\policies.xml -LogFile .\policy-import.log
```
10.	Generate your verification file as you did before, this script will be a bit different due to the way we need to create the PSDrive. 

```bash
if (!(Get-PSDrive | where name -eq LocalFarmGpo)) { New-PSDrive LocalFarmGpo –PSProvider CitrixGroupPolicy –Root \ -Controller localhost }
cd LocalFarmGpo:\
$userpolicies=Get-ChildItem .\User |Select-Object name
$outfile="c:\temp\ctxpolicy-new.txt"
foreach ($p in $userpolicies) {
    Get-ItemProperty -path .\User\$($p.name)  |Out-File $outfile -Append
    Get-ChildItem .\User\$($p.name) -Recurse |where state -eq enabled |Out-File $outfile -Append
    Get-ChildItem .\User\$($p.name) -Recurse |where enabled -eq true |Out-File $outfile -Append
}
```	
# Resolve Merge Discrepancies with Citrix Studio
Citrix 7.xx Studio handles policy filtering in a differently than 6.x. Separate user and computer policies no longer exists inside the GUI, instead it uses filters that can apply to either users or computer. After an import, studio attempts to merge user and computer policies with the same name into one policy and applies filters for the various users and computers. Read more on this here https://support.citrix.com/article/CTX138509. 
1.	Open Citrix studio and click on policies, If you added several policies, there is a good chance you will be presented with the below screen 
    *	Note the name of the policy given in the error message. There is no priority interference with the “unfiltered” policy as indicated in the message. While this message is misleading, it does give us a valuable bit of information, the other policy mentioned is called “HCG Best Practices”. Make note of this policy name. 
    ![ctx_studio_error](/img/2020/05/ctx_studio_error.png)

2.	To resolve this. Return to your admin powershell session and execute the below commands

```bash
asnp Citrix*
New-PSDrive Site -PSProvider CitrixGroupPolicy -Root \ -Controller localhost
cd Site:\User
ls  | where name -eq "HCG Best Practices”
cd ..\computer
ls  | where name -eq "HCG Best Practices”
 ```

3.  Examine the output. 
    ![merge_conflict_compare](/img/2020/05/merge_conflict_compare.png)
4.  Note the priority on the same policy is different between the user and computer policies. We need to adjust one of these so that it matches the other. In this c ase I want the priority to be 4.
Since ware already in the computer directory, we can change to the “HCG Best Practices" policy and execute the following commands 
    * `cd ".\HCG Best Practices"` 
    * If you wish to switch to the user policy run: `cd "..\user\HCG Best Practices"`
    * Then run where the priority value matches the other policy: `Set-ItemProperty . -Name Priority -value 4`
4.	Refresh the policy view. If you see your policies you are done with this step. Otherwise you may need to repeat this process for the next policy mentioned in the error message. 


# Verify Policy Migration 
If you followed all of the steps from the export and import process you should have 2 new files. 
On the old server: "c:\temp\ctxpolicy-old.txt"
On the new server "c:\temp\ctxpolicy-new.txt"
The diff command on windows is not great. I will be using the open source tool “WinMerge” to diff these files.
Open these files in Winmerge (https://sourceforge.net/projects/winmerge/) and it will highlight any discrepancies between the old/new Citrix environments. 
It is important to point out that this is not a perfect process. There will be differences due to the varying versions of powershell and Citrix snap ins. You will likely some extra lines on the new farm. These can usually be ignored but use your best judgement. 
If all the policies look correct you can declare success. If some policies are incorrect, take a closer look in Citrix Studio and fix or add as necessary. In my case I had 160+ policies and did not need to recreate any. 

## Related links 
* Citrix migrate tool doc: https://docs.citrix.com/en-us/xenapp-and-xendesktop/7-15-ltsr/upgrade-migrate/xenapp-worker-upgrade.html
