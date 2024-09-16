# Defender-for-IoT-in-Hyper-V
Instructions for configuring Defender for IoT as a virtual appliance in Hyper-V.

The configuration is created in 5 steps.

1. Creation of the VHDX disk for the sensor VM
2. Creation of the VM including application of all required configurations
3. Creation of the management network
4. Creation of the monitoring network
5. Adding the networks to the VM
6. Optional: Adding additional monitoring ports

If several monitoring ports are to be used, step 6 can be carried out optionally.

Make sure to configure these settings properly.

⚙️ Configuration
```powershell
$VMName = "sensor"
$VHDPath= "C:\local\Defender for IoT\sensor_disk.vhdx"
$ISOPath= "C:\local\Defender for IoT\image.iso"
$DiskSize= 40GB
$MemorySize= 4GB

$ManagementPort = "Ethernet"
$vSwitchManagementName = "vSwitch_Management"

$MonitoringPort = "Ethernet 2"
$vSwitchMonitoringName = "vSwitch_Monitoring_1"

```

## 1. Creation of the VHDX disk for the sensor VM
```powershell
New-VHD -Path $VHDPath -SizeBytes $DiskSize -Fixed
```

## 2. Creation of the VM including application of all required configurations
```powershell
New-VM -Name $VMName -MemoryStartupBytes $MemorySize -Generation 2 -VHDPath $VHDPath

# Disable Dynamic Memory
Set-VMMemory -VMName $VMName -DynamicMemoryEnabled $false

# Add SCSI-Controller and DVD Drive
Add-VMScsiController -VMName $VMName
Add-VMDvdDrive -VMName $VMName -ControllerNumber 0 -Path $ISOPAth

# Change the Boot Order
$firmware = Get-VMFirmware -VMName $VMName
Set-VMFirmware -VMName $VMName -FirstBootDevice (Get-VMDvdDrive -VMName $VMName)

# Disable Secure Boot
Set-VMFirmware -VMName $VMName -EnableSecureBoot Off
```

## 3. Creation of the management network

⚠️If the remote session with the Hyper-V host is established via this connection, it is terminated and has to be re-established

```powershell
# Selecting the network adapter by it's name
$NetworkAdapter = Get-NetAdapter | Where-Object { $_.Name -eq $ManagementPort -and $_.Status -eq 'Up' -and $_.MediaType -eq '802.3' }

if ($NetworkAdapter) {
	$AdapterName = $NetworkAdapter.Name

	New-VMSwitch -Name $vSwitchManagementName -NetAdapterName $AdapterName -AllowManagementOS $true

	Write-Host "✅ Virtual Switch '$vSwitchManagementName' was created successfully and connected to '$AdapterName'."
} else {
	Write-Host "⛔ Adapter '$ManagementPort' could not be found. The virtual switch was not created."
}
```

## 4. Creation of the monitoring network

⚠️If the remote session with the Hyper-V host is established via this connection, it is terminated and has to be re-established

```powershell

# Create external switch and enable "Allow management operating system"
$MonitoringAdapter = Get-NetAdapter | Where-Object { $_.Name -eq $MonitoringPort -and $_.Status -eq 'Up' -and $_.MediaType -eq '802.3' }
if ($MonitoringAdapter) {
	$MonitoringAdapterName = $MonitoringAdapter.Name
	New-VMSwitch -Name $vSwitchMonitoringName -NetAdapterName $MonitoringAdapterName -AllowManagementOS $true
	Write-Host "✅ Virtual switch '$vSwitchMonitoringName' was created successfully and connected to '$MonitoringAdapterName'."
} else {
	Write-Host "⛔ Adapter '$MonitoringPort' could not be found. The virtual switch was not created."
}
```

## 5. Adding the networks to the VM

```powershell
$VMNetworkAdapters = Get-VMNetworkAdapter -VMName $VMName

# Remove initial adapters
Remove-VMNetworkAdapter -VMName $VMName -Name "Network Adapter"

if ($VMNetworkAdapters) {
    # Add the management network adapter
    Add-VMNetworkAdapter -VMName $VMName -SwitchName $vSwitchManagementName
    Write-Host "✅ '$vSwitchManagementName' was successfully added to '$VMName'."

    # Add the monitoring network adapter
    Add-VMNetworkAdapter -VMName $VMName -SwitchName $vSwitchMonitoringName
    Write-Host "✅ '$vSwitchMonitoringName' was successfully added to '$VMName'."

    # Configure monitoring network adapter
    $VMNetworkAdapter = Get-VMNetworkAdapter -VMName $VMName | Where-Object { $_.SwitchName -eq $vSwitchMonitoringName }
    if ($VMNetworkAdapter) {
        # Disable Virtual Machine Queue (VMQ)
        Set-VMNetworkAdapter -VMName $VMName -Name $VMNetworkAdapter.Name -EnableVMQ $false
        
        # Enable Port Mirroring
        Set-VMNetworkAdapterAdvancedFeature -VMName $VMName -Name $VMNetworkAdapter.Name -PortMirroring Destination
    } else {
        Write-Host "⛔ Found no network adapter connected to '$vSwitchMonitoringName'. Configuration is not complete."
    }
} else {
    Write-Host "⛔ No network adapter found for '$VMName'. Virtual switch '$vSwitchMonitoringName' could not be added."
}

$ExtPortFeature=Get-VMSystemSwitchExtensionPortFeature -FeatureName "Ethernet Switch Port Security Settings"
$ExtPortFeature.SettingData.MonitorMode=2
Add-VMSwitchExtensionPortFeature -ExternalPort -SwitchName $vSwitchMonitoringName -VMSwitchExtensionFeature $ExtPortFeature

Write-Host "✅ Configuration successfull! Start the VM to continue"
```