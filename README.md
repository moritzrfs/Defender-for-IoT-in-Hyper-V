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

## ⚙️ Configuration
```powershell
$VMName = "sensor"
$VHDPath= "C:\local\Defender for IoT\sensor_disk.vhdx"
$ISOPath= "C:\local\Defender for IoT\image.iso"
$DiskSize= 40GB
$MemorySize= 4GB
$CoreCount= 4

$ManagementPort = "Ethernet"
$vSwitchManagementName = "vSwitch_Management"
$ManagementAdapterName = "Management Adapter"

$MonitoringPort = "Ethernet 2"
$vSwitchMonitoringName = "vSwitch_Monitoring_1"
$MonitoringAdapterName = "Monitoring Adapter 1"
```

## 1. Creation of the VHDX disk for the sensor VM
```powershell
New-VHD -Path $VHDPath -SizeBytes $DiskSize -Fixed
```

## 2. Creation of the VM including application of all required configurations
```powershell
New-VM -Name $VMName -MemoryStartupBytes $MemorySize -Generation 2 -VHDPath $VHDPath

# Set cores
Set-VMProcessor -VMName $VMName -Count $CoreCount

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
$PhysicalMonitoringAdapter = Get-NetAdapter | Where-Object { $_.Name -eq $MonitoringPort -and $_.Status -eq 'Up' -and $_.MediaType -eq '802.3' }
if ($PhysicalMonitoringAdapter) {
	$PhysicalMonitoringAdapterName = $PhysicalMonitoringAdapter.Name
	New-VMSwitch -Name $vSwitchMonitoringName -NetAdapterName $PhysicalMonitoringAdapterName -AllowManagementOS $true
	Write-Host "✅ Virtual switch '$vSwitchMonitoringName' was created successfully and connected to '$PhysicalMonitoringAdapterName'."
} else {
	Write-Host "⛔ Adapter '$MonitoringPort' could not be found. The virtual switch was not created."
}
```

## 5. Adding the networks to the VM

```powershell
$VMNetworkAdapters = Get-VMNetworkAdapter -VMName $VMName

# Remove initial adapters
Remove-VMNetworkAdapter -VMName $VMName -Name "*"

if ($VMNetworkAdapters) {
    # Add the management network adapter
    Add-VMNetworkAdapter -VMName $VMName -SwitchName $vSwitchManagementName -Name $ManagementAdapterName
    Write-Host "✅ '$vSwitchManagementName' was successfully added to '$VMName'."

    # Add the monitoring network adapter
    Add-VMNetworkAdapter -VMName $VMName -SwitchName $vSwitchMonitoringName -Name $MonitoringAdapterName
    Write-Host "✅ '$vSwitchMonitoringName' was successfully added to '$VMName'."

    # Disable Virtual Machine Queue (VMQ) for the monitoring adapter
    Set-VMNetworkAdapter -VMName $VMName -Name $MonitoringAdapterName -VmqWeight 0
    
    # Enable Port Mirroring
    Set-VMNetworkAdapter -VMName $VMName -Name $MonitoringAdapterName -PortMirroring Destination
} else {
    Write-Host "⛔ No network adapter found for '$VMName'. Virtual switch '$vSwitchMonitoringName' could not be added."
}

$ExtPortFeature=Get-VMSystemSwitchExtensionPortFeature -FeatureName "Ethernet Switch Port Security Settings"
$ExtPortFeature.SettingData.MonitorMode=2
Add-VMSwitchExtensionPortFeature -ExternalPort -SwitchName $vSwitchMonitoringName -VMSwitchExtensionFeature $ExtPortFeature

Write-Host "✅ Configuration successfull! Start the VM to continue"
```

## 6. Optional: Adding additional monitoring ports

```powershell
$VMName= "sensor"
$MonitoringPort = "Ethernet n"
$vSwitchMonitoringName = "vSwitch_Monitoring_n"
$MonitoringAdapterName = "Monitoring Adapter n"

# Create external switch and enable "Allow management operating system"
$PhysicalMonitoringAdapter = Get-NetAdapter | Where-Object { $_.Name -eq $MonitoringPort -and $_.Status -eq 'Up' -and $_.MediaType -eq '802.3' }

if ($PhysicalMonitoringAdapter) {
	$PhysicalMonitoringAdapterName = $PhysicalMonitoringAdapter.Name
	New-VMSwitch -Name $vSwitchMonitoringName -NetAdapterName $PhysicalMonitoringAdapterName -AllowManagementOS $true
	Write-Host "✅ Virtual switch '$vSwitchMonitoringName' was created successfully and connected to '$PhysicalMonitoringAdapterName'."
} else {
	Write-Host "⛔ Adapter '$MonitoringPort' could not be found. The virtual switch was not created."
}

# Add the monitoring network adapter
Add-VMNetworkAdapter -VMName $VMName -SwitchName $vSwitchMonitoringName -Name $MonitoringAdapterName
Write-Host "✅ '$vSwitchMonitoringName' was successfully added to '$VMName'."

# Disable Virtual Machine Queue (VMQ) for the monitoring adapter
Set-VMNetworkAdapter -VMName $VMName -Name $MonitoringAdapterName -VmqWeight 0

# Enable Port Mirroring
Set-VMNetworkAdapter -VMName $VMName -Name $MonitoringAdapterName -PortMirroring Destination

$ExtPortFeature=Get-VMSystemSwitchExtensionPortFeature -FeatureName "Ethernet Switch Port Security Settings"
$ExtPortFeature.SettingData.MonitorMode=2
Add-VMSwitchExtensionPortFeature -ExternalPort -SwitchName $vSwitchMonitoringName -VMSwitchExtensionFeature $ExtPortFeature

Write-Host "✅ Successfully added additional monitoring adapter!"
```