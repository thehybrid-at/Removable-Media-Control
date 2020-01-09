# Removable Media Control using PowerShell script.

__Disclaimer:__ I didn’t re-invent the wheel, I saw the MAIN component of this script- the media listener – in some comment while googling long time ago, and I wish I can find it again so I can give this man credit.

## Business case:

1. Few USBs are allowed to be used within our network and should not be used outside.   
2. No other USBs can be used within our network.     

## Business needs:

1.	USBs should be encrypted using BitLocker by the IT/any other team.    
2.	This team only knows the password.    
3.	USBs to be distributed to the required teams/personnel   
4.	All our encrypted USBs should auto-unlock if inserted in our network.   
5.	All un-encrypted USBs should be eject automatically.   
6.	All any other encrypted USBs with different passwords should be ejected automatically.    
7.  Windows Event is logged.    

```powershell
#Requires -version 2.0
Register-WmiEvent -Class win32_VolumeChangeEvent -SourceIdentifier volumeChange
New-EventLog -LogName Application -Source "USBINSERTED"
write-host (get-date -format s) " Beginning script..."
do{
$newEvent = Wait-Event -SourceIdentifier volumeChange
$eventType = $newEvent.SourceEventArgs.NewEvent.EventType
$eventTypeName = switch($eventType)
{
1 {"Configuration changed"}
2 {"Device arrival"}
3 {"Device removal"}
4 {"docking"}
}
write-host (get-date -format s) " Event detected = " $eventTypeName
if ($eventType -eq 2)
{
$driveLetter = $newEvent.SourceEventArgs.NewEvent.DriveName
$driveLabel = ([wmi]"Win32_LogicalDisk='$driveLetter'").VolumeName
write-host (get-date -format s) " Drive name = " $driveLetter
write-host (get-date -format s) " Drive label = " $driveLabel
$driveletteronly=$driveLetter.split(':')[0]
Start-Job -ScriptBlock { 

 Param(

   [Parameter(Mandatory=$true)]
   
   $driveconnected=""
) #end param
##for the pop-up message 
$wshell = New-Object -ComObject Wscript.Shell
$driveconnected=$driveconnected+":"
$currentusername = Get-WMIObject -class Win32_ComputerSystem | select username | findstr "\"
$deviceid = (Get-WmiObject -Class Win32_Volume | select Name,  SerialNumber | findstr $driveconnected).Split('')[-1]
$initiallockstatus = $(manage-bde  -status $driveconnected | findstr  "Lock"  | findstr "Status").split(':')[1].replace(' ' , '')
$lockstatus=$initiallockstatus
$bitlocker="disabled"

if ($initiallockstatus -eq 'Locked')
{
 $bitlocker = "enabled"
 #[Convert]::ToBase64String([System.Text.Encoding]::Unicode.GetBytes('Password00')) --> to generate encoded value (UABhAHMAcwB3AG8AcgBkADAAMAA=) mentioned below
 Unlock-BitLocker -MountPoint $driveconnected -Password $(ConvertTo-SecureString $([System.Text.Encoding]::Unicode.GetString([System.Convert]::FromBase64String('UABhAHMAcwB3AG8AcgBkADAAMAA='))) -AsPlainText -Force)
 ##check if usb is still blocked
 $afterunlock = $(manage-bde  -status $driveconnected | findstr  "Unlocked"  | findstr "Status").split(':')[1].replace(' ' , '')
	if($afterunlock -eq 'Unlocked')
	{
		$LOG = "Authorized USB has been inserted by user "+$currentusername+" and deviceID "+$deviceid+" access granted"
		Write-EventLog -LogName Application -Source "USBINSERTED" -EntryType Information -EventID 3333  -Message $LOG
		
	}
	else
	{
		(New-Object -comObject Shell.Application).Namespace(17).ParseName($driveconnected).InvokeVerb("Eject")
		$LOG = "Unauthorized USB has been inserted by user "+$currentusername+" and deviceID "+$deviceid+" access denied"
		Write-EventLog -LogName Application -Source "USBINSERTED" -EntryType Information -EventID 3334  -Message $LOG
		$wshell.Popup("An unauthorized USB has been detected on your machine and it has been disabled. Please contact your IT in case you need to use it",0,"UNAUTHORIZED USB detected",0)

	}

}

else 

{
	(New-Object -comObject Shell.Application).Namespace(17).ParseName($driveconnected).InvokeVerb("Eject")
	$LOG = "Unencrypted USB has been inserted by user "+$currentusername+" and deviceID "+$deviceid+" access denied"
	Write-EventLog -LogName Application -Source "USBINSERTED" -EntryType Information -EventID 3335  -Message $LOG
	$wshell.AppActivate('TTILE')
	$wshell.Popup("An unauthorized USB has been detected on your machine (unencrypted) and it has been disabled. Please contact your IT in case you need to use it",0,"Unencrypted USB detected",0)
	
	
}




 } -ArgumentList $driveletteronly

}
Remove-Event -SourceIdentifier volumeChange
} while (1-eq1) #Loop until next event
Unregister-Event -SourceIdentifier volumeChange
```


