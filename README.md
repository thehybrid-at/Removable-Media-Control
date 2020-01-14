# Removable Media Control using PowerShell script.

__Disclaimer:__ I didn’t re-invent the wheel, I built this script on top of the removable media listener that I found in some comment while googling long time ago, and I wish I can find it again so I can give this man credit.

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

**NOTE:** Please make sure to read the comments in the code as there is a line needs to be removed from the code.

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
##-------------------------The Listner Ends here ------------------
##for the pop-up message 
$wshell = New-Object -ComObject Wscript.Shell # To be used for a pop-up message to the user.
$driveconnected=$driveconnected+":"
$currentusername = Get-WMIObject -class Win32_ComputerSystem | select username | findstr "\"
$deviceid = (Get-WmiObject -Class Win32_Volume | select Name,  SerialNumber | findstr $driveconnected).Split('')[-1]
$initiallockstatus = $(manage-bde  -status $driveconnected | findstr  "Lock"  | findstr "Status").split(':')[1].replace(' ' , '')
$lockstatus=$initiallockstatus
$bitlocker="disabled"

if ($initiallockstatus -eq 'Locked')  # checks if the removeable media is encrypted
{
 $bitlocker = "enabled"
 # The below line just to be run standalone to get the password encoding, this shouldn't be a part of the script,
 # The below two lines are there just to make sure the password is not clear text in the code. You can change the approach with the you way you like.
 #[Convert]::ToBase64String([System.Text.Encoding]::Unicode.GetBytes('Password00')) --> to generate encoded value (UABhAHMAcwB3AG8AcgBkADAAMAA=) mentioned below
 # -------------The above line should not be left in the script ----------------
 Unlock-BitLocker -MountPoint $driveconnected -Password $(ConvertTo-SecureString $([System.Text.Encoding]::Unicode.GetString([System.Convert]::FromBase64String('UABhAHMAcwB3AG8AcgBkADAAMAA='))) -AsPlainText -Force) #tries to decrypt the removeable media with our password
 ##check if usb is still blocked
 $afterunlock = $(manage-bde  -status $driveconnected | findstr  "Unlocked"  | findstr "Status").split(':')[1].replace(' ' , '')
	if($afterunlock -eq 'Unlocked') # Means that the removable media was encrypted the our passord
	{
		$LOG = "Authorized USB has been inserted by user "+$currentusername+" and deviceID "+$deviceid+" access granted"
		Write-EventLog -LogName Application -Source "USBINSERTED" -EntryType Information -EventID 3333  -Message $LOG  #Writes to windows event log
		
	}
	else # Means that the removable media was not encrypted the our passord
	{
		(New-Object -comObject Shell.Application).Namespace(17).ParseName($driveconnected).InvokeVerb("Eject")
		$LOG = "Unauthorized USB has been inserted by user "+$currentusername+" and deviceID "+$deviceid+" access denied"
		Write-EventLog -LogName Application -Source "USBINSERTED" -EntryType Information -EventID 3334  -Message $LOG #Writes to windows event log
		$wshell.Popup("An unauthorized USB has been detected on your machine and it has been disabled. Please contact your IT in case you need to use it",0,"UNAUTHORIZED USB detected",0) # pop-up message to user

	}

}

else # Removable media is already unencrypted.

{
	(New-Object -comObject Shell.Application).Namespace(17).ParseName($driveconnected).InvokeVerb("Eject")
	$LOG = "Unencrypted USB has been inserted by user "+$currentusername+" and deviceID "+$deviceid+" access denied"
	Write-EventLog -LogName Application -Source "USBINSERTED" -EntryType Information -EventID 3335  -Message $LOG #Writes to windows event log
	$wshell.AppActivate('TTILE')
	$wshell.Popup("An unauthorized USB has been detected on your machine (unencrypted) and it has been disabled. Please contact your IT in case you need to use it",0,"Unencrypted USB detected",0) #pop-up message to user
	
	
}




 } -ArgumentList $driveletteronly

}
Remove-Event -SourceIdentifier volumeChange
} while (1-eq1) #Loop until next event
Unregister-Event -SourceIdentifier volumeChange
```


