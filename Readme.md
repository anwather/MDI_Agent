# Instructions

**Prereqs**
- You must execute the code below in PowerShell 7 as it is the only version which support the GuestConfiguration module ;)

1. Download the Microsoft Defender for Identity package and get the key
2. Upload the package to a storage account - generate a read only SAS token for this
3. Use the code below to upload the configuration zip file to a storage account (can be the same one in the same container)

```
$ctx = (Get-AzStorageAccount -ResourceGroupName <<rg>> -Name <<storage accountname>>).Context
Set-AzStorageBlobContent -File .\MDIAgentInstall.zip -Container <<container>> -Blob MDIAgentInstall.zip -Context $ctx -Force
```

4. Use the code below to generate a SAS token that the policy will use to pull the configuration package.

```
$StartTime = Get-Date
$EndTime = $startTime.AddYears(3)
$contenturi = New-AzStorageBlobSASToken -StartTime $StartTime -ExpiryTime $EndTime -Container "mdi" -Blob "MDIAgentInstall.zip" -Permission r -Context $Ctx -FullUri 
```

5. Use the code below to create a new configuration policy.

```
Install-Module GuestConfiguration -Force -Verbose
$PolicyParameterInfo = @(
  @{
    # Policy parameter name (mandatory)
    Name                 = 'AgentBlobUrl'
    # Policy parameter display name (mandatory)
    DisplayName          = 'Agent Blob Url'
    # Policy parameter description (optional)
    Description          = 'Agent download blob URL with read SAS token'
    # DSC configuration resource type (mandatory)
    ResourceType         = 'FileDownload'
    # DSC configuration resource id (mandatory)
    ResourceId           = 'download'
    # DSC configuration resource property name (mandatory)
    ResourcePropertyName = 'Url'
  },
  @{
    # Policy parameter name (mandatory)
    Name                 = 'AgentDownloadPath'
    # Policy parameter display name (mandatory)
    DisplayName          = 'Agent Download Path'
    # Policy parameter description (optional)
    Description          = 'Agent download path'
    # DSC configuration resource type (mandatory)
    ResourceType         = 'FileDownload'
    # DSC configuration resource id (mandatory)
    ResourceId           = 'download'
    # DSC configuration resource property name (mandatory)
    ResourcePropertyName = 'Path'
    DefaultValue         = 'C:\Temp\agent.zip'
  },
  @{
    # Policy parameter name (mandatory)
    Name                 = 'ArchiveFile'
    # Policy parameter display name (mandatory)
    DisplayName          = 'Archive FileName'
    # Policy parameter description (optional)
    Description          = 'Name of archive to unpack'
    # DSC configuration resource type (mandatory)
    ResourceType         = 'Archive'
    # DSC configuration resource id (mandatory)
    ResourceId           = 'unzip'
    # DSC configuration resource property name (mandatory)
    ResourcePropertyName = 'Path'
    DefaultValue         = 'C:\Temp\agent.zip'
  },
  @{
    # Policy parameter name (mandatory)
    Name                 = 'ArchiveDestination'
    # Policy parameter display name (mandatory)
    DisplayName          = 'Archive Destination'
    # Policy parameter description (optional)
    Description          = 'Destination of archive'
    # DSC configuration resource type (mandatory)
    ResourceType         = 'Archive'
    # DSC configuration resource id (mandatory)
    ResourceId           = 'unzip'
    # DSC configuration resource property name (mandatory)
    ResourcePropertyName = 'Destination'
    DefaultValue         = 'C:\Temp'
  },
  @{
    # Policy parameter name (mandatory)
    Name                 = 'AccessKey'
    # Policy parameter display name (mandatory)
    DisplayName          = 'Access Key'
    # Policy parameter description (optional)
    Description          = 'Access key for sensor'
    # DSC configuration resource type (mandatory)
    ResourceType         = 'MDIInstall'
    # DSC configuration resource id (mandatory)
    ResourceId           = 'install'
    # DSC configuration resource property name (mandatory)
    ResourcePropertyName = 'AccessKey'
  },
  @{
    # Policy parameter name (mandatory)
    Name                 = 'InstallerLocation'
    # Policy parameter display name (mandatory)
    DisplayName          = 'Installer Location'
    # Policy parameter description (optional)
    Description          = 'Location of Installer'
    # DSC configuration resource type (mandatory)
    ResourceType         = 'MDIInstall'
    # DSC configuration resource id (mandatory)
    ResourceId           = 'install'
    # DSC configuration resource property name (mandatory)
    ResourcePropertyName = 'Path'
    DefaultValue         = 'C:\Temp'
  })

$PolicyParam = @{
  PolicyId      = '813aed4c-9d10-4f41-8982-8d91e13ac5c5'
  ContentUri    = $contentUri
  DisplayName   = 'Deploy MDI Agent'
  Description   = "Deploy MDI Agent"
  Path          = '.\policies\'
  Parameter     = $PolicyParameterInfo
  PolicyVersion = '1.0.0'
}

New-GuestConfigurationPolicy -Mode ApplyAndAutoCorrect @PolicyParam
```

6. Use the code below to deploy the policy definition (this just goes the current subscription at the moment)

```
New-AzPolicyDefinition -Name 'Deploy MDI Agent' -Policy '.\policies\MDIAgentInstall_DeployIfNotExists.json' ## This path may be wrong - but it should point to the policy JSON file
```

7. Assign the policy using the SAS token from Step 2 for the 'Agent Blob Url' and the Access Key. If you want it to hit Arc machines ensure you untick the box in the policy assignment.

Wait for remediation...... Or do a remediation task. - By default it does the following

1. Downloads the agent zip file to C:\Temp
2. Extract to the same folder
3. Install using the key