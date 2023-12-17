# Biztalk Deployment Framework - Local Bindings
For new BizTalk solutions I use the BizTalk Deployment Framework (BTDF). In the early stages of development, I find that I run through the following cycle very frequently:

* deploy
* from the BizTalk admin console, identify a problem with the bindings
* update portbindingsmaster.xml and / or environmentsettings.xlsx
* cycle...

The BTDF toolbar provides a "quick deploy" option and whilst this is very useful in some scenarios, it doesn't deploy the bindings. The "full deploy" option covers all bases but it is rather slow.

I've created the following script which simply re-creates a portbindings.xml file from the portbindingsmaster.xml and environmentsettings.xlsx, then deploys this into the local BizTalk environment. 

The script requires a single parameter - the path to your repo folder that contains the solution folder. Before running, you will need to change the three variables *Set* at the top of the script to give values relevant to your solution.  Extra params could be used to replace the need for the Set commands.

I typically then keep a copy of the script in the SolutionName\Deployment folder.

Hope somebody finds the script helpful!

    REM Requires single param of the base folder for repo, eg to run "DeployLocalBindings C:\Tfs_BizTalkAdmin"
    Set repoBase=%1
    if not defined repoBase set repoBase=C:\Tfs_BizTalkAdmin
    Set PATH_TO_DEPLOYMENT_FOLDER=%repoBase%\BizTalk.Unit4\XXX.Unit4.HRPersonnel\XXX.Unit4.HRPersonnel.Deployment"
    Set "SOLUTION_ROOT=%repoBase%\BizTalk.Unit4\XXX.Unit4.HRPersonnel"
    Set "APPLICATION_NAME=XXX.Unit4.HRPersonnel"


    REM export setting from excel
    "C:\Program Files (x86)\Deployment Framework for BizTalk 5.6\Framework\DeployTools\EnvironmentSettingsExporter.exe" %PATH_TO_DEPLOYMENT_FOLDER%"\SettingsFileGenerator.xml" %PATH_TO_DEPLOYMENT_FOLDER%"\EnvironmentSettings"

    REM clear read only attrib on port bindings.xml
    "C:\Program Files (x86)\Deployment Framework for BizTalk 5.6\Framework\DeployTools\xmlpreprocess.exe" /v /c /noDirectives /i:%PATH_TO_DEPLOYMENT_FOLDER%"\PortBindingsMaster.xml" /o:%PATH_TO_DEPLOYMENT_FOLDER%"\PortBindings.xml" /d:CurDir=%SOLUTION_ROOT% /s:%PATH_TO_DEPLOYMENT_FOLDER%"\EnvironmentSettings\Exported_LocalSettings.xml"

    REM Element tunnel
    "C:\Program Files (x86)\Deployment Framework for BizTalk 5.6\Framework\DeployTools\ElementTunnel.exe" /i:%PATH_TO_DEPLOYMENT_FOLDER%"\PortBindings.xml" /o:%PATH_TO_DEPLOYMENT_FOLDER%"\PortBindings.xml" /x:"C:\Program Files (x86)\Deployment Framework for BizTalk 5.6\Framework\DeployTools\adapterXPaths.txt" /encode+

    REM copy port bindings to temp file
    copy %PATH_TO_DEPLOYMENT_FOLDER%"\PortBindings.xml" %PATH_TO_DEPLOYMENT_FOLDER%"\TempPortBindings.xml"

    REM BTS Deploy - add bindings as a resource
    BTSTask.exe AddResource -Type:BizTalkBinding -Overwrite -Source:%PATH_TO_DEPLOYMENT_FOLDER%"\TempPortBindings.xml" -ApplicationName:%APPLICATION_NAME%
    del %PATH_TO_DEPLOYMENT_FOLDER%"\TempPortBindings.xml"

    REM Import bindings
    BTSTask.exe ImportBindings -Source:%PATH_TO_DEPLOYMENT_FOLDER%"\PortBindings.xml" -ApplicationName:%APPLICATION_NAME%
