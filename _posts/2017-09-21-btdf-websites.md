---
title: BTDF IIS
category: BTDF
tags:
    - BizTalk
    - BTDF
---
# BizTalk Deployment Framework and IIS Websites
Tom Abraham recently published v5.7 of the BTDF to Codeplex and Github. The main win of this release is full support for BizTalk 2016. Aside from that (and the fancy new toolbar icons!), another great benefit for this release is enhanced functionality for creation of IIS Websites and App Pools.

An example ItemGroup uses to create an app pool can be seen below:

    <ItemGroup>
      <IISAppPool Include="BtsWasteAppPool">
        <DeployAction>Create</DeployAction>
        <DotNetFrameworkVersion>v4.0</DotNetFrameworkVersion>
        <IdentityType>SpecificUser</IdentityType>
        <PipelineMode>Integrated</PipelineMode>
        <UserName>$(BtsIsoHostUser)</UserName>
        <Password>$(BtsIsoHostPassword)</Password>
      </IISAppPool>
    </ItemGroup>

My problem was where to get $(BtsIsoHostUser) and $(BtsIsoHostPassword) from. These are stored in the environmentsettings.xml Excel spreadhseet but that's not processed in time. The solution that my colleague came up with (thanks Cameron) was to add this to a custom target, so at the end of the .btdfproj file I have:

    <Target Name="CustomDeployTarget">
    <CallTarget Targets="CreateBizTalkAppPool" />
  </Target>

      <Target Name="CreateBizTalkAppPool">
    <ItemGroup>
      <IISAppPool Include="BtsWasteAppPool">
        <DeployAction>Create</DeployAction>
        <DotNetFrameworkVersion>v4.0</DotNetFrameworkVersion>
        <IdentityType>SpecificUser</IdentityType>
        <PipelineMode>Integrated</PipelineMode>
        <UserName>$(BtsIsoHostUser)</UserName>
        <Password>$(BtsIsoHostPassword)</Password>
      </IISAppPool>
    </ItemGroup>
  </Target>
</Project>

Then, prior to this in the same btdfproj file, the following is required - to ensure BtsIsoHostUser;BtsIsoHostPassword variables are pulled from the settings

    <ItemGroup>
    <PropsFromEnvSettings Include="SsoAppUserGroup;SsoAppAdminGroup;BtsIsoHostUser;BtsIsoHostPassword" />
  </ItemGroup>

Deployment of the actual website is achieved as follows:

    <ItemGroup>
    <IISApp Include="*">
      <AppPoolName>BtsWasteAppPool</AppPoolName>
      <DeployAction>CreateOrUpdate</DeployAction>
      <PhysicalPath>..\LCC.Waste.Adapter.Orchestrations_Proxy</PhysicalPath>
      <UndeployAction>Delete</UndeployAction>
      <VirtualPath>LCC.Waste.Adapter.Orchestrations_Proxy</VirtualPath>
    </IISApp>
  </ItemGroup>



