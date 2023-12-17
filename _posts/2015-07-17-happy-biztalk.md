# Simple PowerShell BizTalk Status Report
Have you ever felt the need to check all BizTalk ports and hosts instances are running before you leave work each day? If you don't have the budget for the excellent [BizTalk360](http://www.biztalk360.com), then a little scripting and the wonderful [BizTalk Powershell Provider](https://psbiztalk.codeplex.com/) could be just what you're looking for!

After creating the following Powershell script I setup a windows scheduled task to execute at 5pm each day. The result is an email with a :) or :(

    Function BizTalk: {Set-Location BizTalk: }
    Function BizTalk:\ {Set-Location BizTalk:\ }
    
    Write-Host "Loading PowerShell provider for BizTalk snap-in"
    
    $InitializeDefaultBTSDrive = $false
    
    Add-PSSnapin BizTalkFactory.Powershell.Extensions
    
    New-PSDrive -Name BizTalk -Root BizTalk:\ -PSProvider BizTalk -Instance li-biztalk-db -Database BizTalkMgmtdb
    
    $names = Get-ChildItem BizTalk:\Applications | Foreach-Object {$_.Name} | Sort-Object {$_.Name};
    $statuses = Get-ChildItem BizTalk:\Applications | Foreach-Object {$_.status} | Sort-Object {$_.Name};
    $index = 0;
    $sb = New-Object -TypeName "System.Text.StringBuilder";
    $status = "";
    $unexpected = 0;
    $line = "";
    
    $sb.AppendLine("Application Status");
    $sb.AppendLine("==================");
    
    foreach ($app in $names)
    {
        if ($app.ToString().StartsWith("BTS"))
        {
            $status = $statuses[$index].ToString();
            $line = "Status of $app is $status";
            if ($status -ne "Started")
            {
                $line = $line + " UNEXPECTED STATUS!";
                $unexpected = $unexpected + 1;
                $sb.AppendLine($line);
                $sb.AppendLine("*** Receive Locations ***");
    
                $rcvLocStatuses = Get-ChildItem "BizTalk:\Applications\$app\Receive Locations";
    
                foreach ($rcvLoc in $rcvLocStatuses | Where-Object {!$_.Enable})
                {
                    $sb.AppendLine($rcvLoc.Name + " is not enabled. `r`n ");
                }
    
                $sb.AppendLine("*** Send Ports ***");
    
                $sendPorts = Get-ChildItem "BizTalk:\Applications\$app\Send Ports";
    
                foreach ($sendPort in $sendPorts | Where-Object {$_.status -ne "Started"} )
                {
                    $sb.AppendLine($sendPort.Name + " is " +  $sendPort.Status);
                }
    
                $sb.AppendLine("*** Orchestrations ***");
    
                $orchestrations = Get-ChildItem "BizTalk:\Applications\$app\Orchestrations";
    
                foreach ($orchestration in $orchestrations | Where-Object {$_.status -ne "Started"} )
                {
                    $sb.AppendLine($orchestration.Name + " is not running");
                }
            }       
        }
        $index = $index + 1
    }
    
    $index = 0;
    
    $sb.AppendLine("");
    $sb.AppendLine("Host Instance Status");
    $sb.AppendLine("====================");
    
    $names = Get-ChildItem "BizTalk:\Platform Settings\Host Instances" | Foreach-Object {$_.Name} | Sort-Object {$_.Name};
    $statuses = Get-ChildItem "BizTalk:\Platform Settings\Host Instances" | Foreach-Object {$_."ServiceState"} | Sort-Object {$_.Name};
    
    foreach ($hi in $names)
    {
        if (!$hi.ToString().Contains("BizTalkServerIsolatedHost"))
        {
            $status = $statuses[$index].ToString();
            $line = "Status of $hi is $status";
            if ($status -ne "Running")
            {
                $line = $line + " UNEXPECTED STATUS!";
                $unexpected = $unexpected + 1;
            }
            $sb.AppendLine($line);
            $sb.AppendLine("");
        }
    
        $index = $index + 1
    }
    
    Write-Output $sb.ToString();
    $emailSmtpServer = "xxx"
    $emailSmtpServerPort = "25"
    $emailSmtpUser = "xxx"
    $emailSmtpPass = "xxx"
     
    $emailMessage = New-Object System.Net.Mail.MailMessage
    $emailMessage.From = "noreply@xxx.co.uk"
    $emailMessage.To.Add( "rob@biztalkers.com" )
    
    if ($unexpected -gt 0)
    {
        $emailMessage.Subject = "BizTalk Status Alert :("
    }
    else
    {
        $emailMessage.Subject = "BizTalk Status Update :)"
    }
    
    $emailMessage.IsBodyHtml = $false
    $emailMessage.Body = $sb.ToString()
     
    $SMTPClient = New-Object System.Net.Mail.SmtpClient( $emailSmtpServer , $emailSmtpServerPort )
    $SMTPClient.EnableSsl = $false
    $SMTPClient.Credentials = New-Object System.Net.NetworkCredential( $emailSmtpUser , $emailSmtpPass );
     
    $SMTPClient.Send( $emailMessage )