# ---------------------------
# STA mode is required for WPF drag and drop.
# If not already in STA, relaunch in STA and exit.
# ---------------------------
if ([System.Threading.Thread]::CurrentThread.ApartmentState -ne 'STA') {
    Start-Process powershell.exe -ArgumentList "-STA -NoProfile -ExecutionPolicy Bypass -File `"$PSCommandPath`""
    exit
}

Add-Type -AssemblyName PresentationFramework, PresentationCore, WindowsBase

# ---------------------------
# Allow drag & drop through UIPI when running as Administrator.
# Without this, Windows blocks drag messages from lower-privilege
# processes (like Explorer) reaching an elevated PowerShell window.
# ---------------------------
Add-Type -TypeDefinition @"
using System;
using System.Runtime.InteropServices;
public class DragDropHelper {
    [DllImport("user32.dll")]
    public static extern bool ChangeWindowMessageFilterEx(
        IntPtr hWnd, uint msg, uint action, IntPtr changeInfo);

    public const uint WM_DROPFILES = 0x0233;
    public const uint WM_COPYDATA = 0x004A;
    public const uint WM_COPYGLOBALDATA = 0x0049;
    public const uint MSGFLT_ALLOW = 1;
}
"@

# ---------------------------
# GUI XAML
# ---------------------------
$xaml = @"
<Window xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="Active Directory Scan Tool"
        Height="600" Width="950"
        Background="#FF0B1E3A"
        WindowStartupLocation="CenterScreen"
        ResizeMode="CanResize">
    <Window.Resources>
        <Style TargetType="TextBlock">
            <Setter Property="Foreground" Value="White"/>
        </Style>
        <Style TargetType="Label">
            <Setter Property="Foreground" Value="White"/>
        </Style>
        <Style TargetType="Button">
            <Setter Property="Margin" Value="5"/>
            <Setter Property="Padding" Value="8,4"/>
        </Style>
    </Window.Resources>
    <Grid Margin="10">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
            <RowDefinition Height="2*"/>
            <RowDefinition Height="Auto"/>
        </Grid.RowDefinitions>

        <!-- Header -->
        <StackPanel Grid.Row="0" Orientation="Vertical" Margin="0,0,0,10">
            <StackPanel Orientation="Horizontal">
                <TextBlock Text="Active Directory Scan Tool" FontSize="20" FontWeight="Bold" Margin="0,0,10,0"/>
                <TextBlock Text="(CSV: No headers, Column A only)" FontSize="12" VerticalAlignment="Bottom" Opacity="0.8"/>
            </StackPanel>
            <TextBlock Text="by: Gary Langley" FontSize="13" FontWeight="Bold" Opacity="0.8" Margin="0,2,0,0"/>
        </StackPanel>

        <!-- File selection -->
        <Grid Grid.Row="1">
            <Grid.ColumnDefinitions>
                <ColumnDefinition Width="*"/>
                <ColumnDefinition Width="Auto"/>
            </Grid.ColumnDefinitions>

            <StackPanel Grid.Column="0">
                <Label Content="CSV File:"/>
                <TextBox x:Name="txtCsvPath"
                         AllowDrop="True"
                         Height="26"
                         Background="#FF102447"
                         Foreground="White"
                         BorderBrush="#FF3C5A8A"
                         BorderThickness="1"
                         Margin="0,0,0,5"
                         ToolTip="Drag and drop a CSV file here or use Browse..."/>
            </StackPanel>

            <StackPanel Grid.Column="1" VerticalAlignment="Bottom" Orientation="Horizontal">
                <Button x:Name="btnBrowse" Content="Browse..." Width="90" Margin="5,0,0,5"/>
                <Button x:Name="btnStart" Content="Start Scan" Width="100" Margin="5,0,0,5"/>
            </StackPanel>
        </Grid>

        <!-- DataGrid -->
        <GroupBox Grid.Row="2" Header="Results" Margin="0,10,0,5" Foreground="White" BorderBrush="#FF3C5A8A">
            <DataGrid x:Name="dgResults"
                      AutoGenerateColumns="True"
                      IsReadOnly="True"
                      Background="#FF102447"
                      Foreground="White"
                      GridLinesVisibility="Horizontal"
                      HeadersVisibility="Column"
                      BorderBrush="#FF3C5A8A"
                      BorderThickness="1"
                      AlternatingRowBackground="#FF1A315C"
                      RowBackground="#FF102447"/>
        </GroupBox>

        <!-- Log -->
        <GroupBox Grid.Row="3" Header="Log" Margin="0,5,0,5" Foreground="White" BorderBrush="#FF3C5A8A">
            <TextBox x:Name="txtLog"
                     Background="#FF050F22"
                     Foreground="White"
                     BorderBrush="#FF3C5A8A"
                     BorderThickness="1"
                     IsReadOnly="True"
                     TextWrapping="Wrap"
                     VerticalScrollBarVisibility="Auto"
                     HorizontalScrollBarVisibility="Auto"/>
        </GroupBox>

        <!-- Bottom bar -->
        <Grid Grid.Row="4" Margin="0,5,0,0">
            <Grid.ColumnDefinitions>
                <ColumnDefinition Width="*"/>
                <ColumnDefinition Width="Auto"/>
            </Grid.ColumnDefinitions>

            <StackPanel Grid.Column="0" Orientation="Vertical">
                <ProgressBar x:Name="pbProgress" Height="18" Minimum="0" Maximum="100" Margin="0,0,0,3"/>
                <TextBlock x:Name="lblStatus" Text="Idle" FontSize="12" Opacity="0.8"/>
            </StackPanel>

            <StackPanel Grid.Column="1" Orientation="Horizontal" VerticalAlignment="Center">
                <Button x:Name="btnExport" Content="Export Results..." Width="130" Margin="5,0,0,0" IsEnabled="False"/>
                <Button x:Name="btnHelp" Content="Help" Width="60" Margin="5,0,0,0"/>
                <Button x:Name="btnClose" Content="Close" Width="80" Margin="5,0,0,0"/>
            </StackPanel>
        </Grid>
    </Grid>
</Window>
"@

# ---------------------------
# Load XAML
# ---------------------------
$reader = New-Object System.Xml.XmlNodeReader ([xml]$xaml)
$window = [Windows.Markup.XamlReader]::Load($reader)

# Controls
$txtCsvPath = $window.FindName('txtCsvPath')
$btnBrowse = $window.FindName('btnBrowse')
$btnStart = $window.FindName('btnStart')
$btnExport = $window.FindName('btnExport')
$btnHelp = $window.FindName('btnHelp')
$btnClose = $window.FindName('btnClose')
$dgResults = $window.FindName('dgResults')
$txtLog = $window.FindName('txtLog')
$pbProgress = $window.FindName('pbProgress')
$lblStatus = $window.FindName('lblStatus')

# Results collection
$script:Results = New-Object System.Collections.ObjectModel.ObservableCollection[object]
$dgResults.ItemsSource = $script:Results

# ---------------------------
# Help modal
# ---------------------------
function Show-HelpModal {
    $helpXaml = @"
<Window xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="Help - Active Directory Scan Tool"
        Height="480" Width="560"
        Background="#FF0B1E3A"
        WindowStartupLocation="CenterOwner"
        ResizeMode="NoResize">
    <Window.Resources>
        <Style TargetType="TextBlock">
            <Setter Property="Foreground" Value="White"/>
            <Setter Property="TextWrapping" Value="Wrap"/>
        </Style>
    </Window.Resources>
    <Grid Margin="20">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
            <RowDefinition Height="Auto"/>
        </Grid.RowDefinitions>

        <!-- Title -->
        <StackPanel Grid.Row="0" Margin="0,0,0,15">
            <TextBlock Text="Active Directory Scan Tool" FontSize="18" FontWeight="Bold"/>
            <TextBlock Text="by: Gary Langley" FontSize="12" FontWeight="Bold" Opacity="0.7" Margin="0,3,0,0"/>
            <Rectangle Height="1" Fill="#FF3C5A8A" Margin="0,10,0,0"/>
        </StackPanel>

        <!-- Help content -->
        <ScrollViewer Grid.Row="1" VerticalScrollBarVisibility="Auto">
            <StackPanel>

                <TextBlock Text="Overview" FontSize="14" FontWeight="Bold" Margin="0,0,0,5"/>
                <TextBlock Margin="0,0,0,12">
This tool queries Active Directory for computer objects listed in a CSV file and displays logon timestamps, operating system, description, location, and distinguished name for each machine.
                </TextBlock>

                <TextBlock Text="CSV File Format" FontSize="14" FontWeight="Bold" Margin="0,0,0,5"/>
                <TextBlock Margin="0,0,0,12">
The CSV must contain no header row. Computer names should be listed one per line in Column A only. Example:

  DESKTOP-001
  LAPTOP-FINANCE-02
  WS-HELPDESK-07

Blank rows are skipped automatically.
                </TextBlock>

                <TextBlock Text="How to Run a Scan" FontSize="14" FontWeight="Bold" Margin="0,0,0,5"/>
                <StackPanel Margin="0,0,0,12">
                    <TextBlock Text="1. Click Browse... or drag and drop your CSV file into the file path box." Margin="0,0,0,3"/>
                    <TextBlock Text="2. Click Start Scan. Progress is shown in the bar at the bottom." Margin="0,0,0,3"/>
                    <TextBlock Text="3. Results populate the grid in real time as each computer is queried." Margin="0,0,0,3"/>
                    <TextBlock Text="4. Computers not found in AD will show NOT FOUND across all fields." Margin="0,0,0,3"/>
                </StackPanel>


                <TextBlock Text="Exporting Results" FontSize="14" FontWeight="Bold" Margin="0,0,0,5"/>
                <TextBlock Margin="0,0,0,12">
Once a scan completes, click Export Results... to save the result grid as a CSV file. The export button is disabled until at least one scan has been run.
                </TextBlock>

                <TextBlock Text="Requirements" FontSize="14" FontWeight="Bold" Margin="0,0,0,5"/>
                <TextBlock Margin="0,0,0,6">- PowerShell 5.1 or later<LineBreak/>- Active Directory module (RSAT or domain-joined machine)<LineBreak/>- Sufficient AD permissions to read computer object properties</TextBlock>

            </StackPanel>
        </ScrollViewer>

        <!-- Close button -->
        <Button Grid.Row="2"
                x:Name="btnHelpClose"
                Content="Close"
                Width="90"
                HorizontalAlignment="Right"
                Margin="0,15,0,0"
                Padding="8,4"/>
    </Grid>
</Window>
"@

    $helpReader = New-Object System.Xml.XmlNodeReader ([xml]$helpXaml)
    $helpWindow = [Windows.Markup.XamlReader]::Load($helpReader)
    $helpWindow.Owner = $window

    $helpWindow.FindName('btnHelpClose').Add_Click({ $helpWindow.Close() })
    $helpWindow.ShowDialog() | Out-Null
}

$btnHelp.Add_Click({ Show-HelpModal })

# Auto-launch help on startup + unblock drag & drop messages through UIPI
$window.Add_Loaded({
    # Allow drag-drop messages through Windows privilege boundary
    $hwnd = (New-Object System.Windows.Interop.WindowInteropHelper($window)).Handle
    [DragDropHelper]::ChangeWindowMessageFilterEx($hwnd, [DragDropHelper]::WM_DROPFILES, [DragDropHelper]::MSGFLT_ALLOW, [IntPtr]::Zero) | Out-Null
    [DragDropHelper]::ChangeWindowMessageFilterEx($hwnd, [DragDropHelper]::WM_COPYDATA, [DragDropHelper]::MSGFLT_ALLOW, [IntPtr]::Zero) | Out-Null
    [DragDropHelper]::ChangeWindowMessageFilterEx($hwnd, [DragDropHelper]::WM_COPYGLOBALDATA, [DragDropHelper]::MSGFLT_ALLOW, [IntPtr]::Zero) | Out-Null

    Show-HelpModal
})

# ---------------------------
# File picker
# ---------------------------
$btnBrowse.Add_Click({
    $ofd = New-Object Microsoft.Win32.OpenFileDialog
    $ofd.Filter = "CSV Files (*.csv)|*.csv|All Files (*.*)|*.*"
    if ($ofd.ShowDialog() -eq $true) {
        $txtCsvPath.Text = $ofd.FileName
        $time = (Get-Date).ToString("HH:mm:ss")
        $txtLog.AppendText("[$time] Selected CSV: $($ofd.FileName)`r`n")
        $txtLog.ScrollToEnd()
    }
})

# ---------------------------
# Drag & drop
# ---------------------------
$txtCsvPath.Add_DragEnter({
    if ($_.Data.GetDataPresent([Windows.DataFormats]::FileDrop)) {
        $_.Effects = [Windows.DragDropEffects]::Copy
    } else {
        $_.Effects = [Windows.DragDropEffects]::None
    }
    $_.Handled = $true
})

# DragOver must also set Effects â€” WPF resets it to None on every move event
# without this handler the drop is cancelled before it can register
$txtCsvPath.Add_DragOver({
    if ($_.Data.GetDataPresent([Windows.DataFormats]::FileDrop)) {
        $_.Effects = [Windows.DragDropEffects]::Copy
    } else {
        $_.Effects = [Windows.DragDropEffects]::None
    }
    $_.Handled = $true
})

$txtCsvPath.Add_Drop({
    if ($_.Data.GetDataPresent([Windows.DataFormats]::FileDrop)) {
        $files = $_.Data.GetData([Windows.DataFormats]::FileDrop)
        if ($files -and $files.Count -gt 0) {
            $txtCsvPath.Text = $files[0]
            $time = (Get-Date).ToString("HH:mm:ss")
            $txtLog.AppendText("[$time] Dropped CSV: $($files[0])`r`n")
            $txtLog.ScrollToEnd()
        }
    }
})

# ---------------------------
# Export results
# ---------------------------
$btnExport.Add_Click({
    if ($script:Results.Count -eq 0) {
        [System.Windows.MessageBox]::Show("No results to export.", "Export", 'OK', 'Information')
        return
    }

    $sfd = New-Object Microsoft.Win32.SaveFileDialog
    $sfd.Filter = "CSV Files (*.csv)|*.csv"
    $sfd.FileName = "ADLastLogons.csv"

    if ($sfd.ShowDialog() -eq $true) {
        $script:Results | Export-Csv -Path $sfd.FileName -NoTypeInformation -Encoding UTF8
        $time = (Get-Date).ToString("HH:mm:ss")
        $txtLog.AppendText("[$time] Exported results to: $($sfd.FileName)`r`n")
        $txtLog.ScrollToEnd()
    }
})

# ---------------------------
# Close button
# ---------------------------
$btnClose.Add_Click({ $window.Close() })

# ---------------------------
# Start Scan
# ---------------------------
$btnStart.Add_Click({

    $csvPath = $txtCsvPath.Text.Trim()
    if (-not (Test-Path $csvPath)) {
        [System.Windows.MessageBox]::Show("CSV file not found:`n$csvPath", "Error", 'OK', 'Error')
        return
    }

    try {
        # Headerless CSV â€” Column A mapped to Column1
        $computers = Import-Csv -Path $csvPath -Header Column1
    } catch {
        [System.Windows.MessageBox]::Show("Failed to read CSV:`n$($_.Exception.Message)", "Error", 'OK', 'Error')
        return
    }

    # Guard: nothing to do if CSV is empty
    if ($computers.Count -eq 0) {
        [System.Windows.MessageBox]::Show("The CSV file contains no rows.", "Error", 'OK', 'Warning')
        return
    }

    $script:Results.Clear()
    $pbProgress.Value = 0
    $lblStatus.Text = "Running..."
    $btnStart.IsEnabled = $false
    $btnExport.IsEnabled = $false

    $time = (Get-Date).ToString("HH:mm:ss")
    $txtLog.AppendText("[$time] Starting AD scan. Total rows: $($computers.Count)`r`n")
    $txtLog.ScrollToEnd()

    # Sync context for background runspace
    $sync = [hashtable]::Synchronized(@{
        Window = $window
        Results = $script:Results
        ProgressBar = $pbProgress
        StatusLabel = $lblStatus
        LogBox = $txtLog
        BtnStart = $btnStart
        BtnExport = $btnExport
        Computers = $computers
    })

    $ps = [PowerShell]::Create()
    $null = $ps.AddScript({
        param($sync)

        # ---------------------------
        # Helpers (background scope)
        # ---------------------------
        function Write-BGLog {
            param([string]$Message)
            $time = (Get-Date).ToString("HH:mm:ss")
            $sync.Window.Dispatcher.Invoke([action]{
                $sync.LogBox.AppendText("[$time] $Message`r`n")
                $sync.LogBox.ScrollToEnd()
            })
        }

        function Convert-ADTime {
            param([object]$Value)
            if (-not $Value) { return $null }
            try {
                if ($Value -is [long] -or $Value -is [int64]) {
                    return [DateTime]::FromFileTimeUtc($Value).ToLocalTime()
                }
                if ($Value -is [DateTime]) {
                    return $Value.ToLocalTime()
                }
                return $Value
            } catch {
                return $Value
            }
        }

        # Guard: ensure the AD module loaded
        Import-Module ActiveDirectory -ErrorAction SilentlyContinue
        if (-not (Get-Module -Name ActiveDirectory)) {
            Write-BGLog "ERROR: ActiveDirectory module could not be loaded. Aborting scan."
            $sync.Window.Dispatcher.Invoke([action]{
                $sync.StatusLabel.Text = "Error â€” AD module missing."
                $sync.BtnStart.IsEnabled = $true
            })
            return
        }

        $total = $sync.Computers.Count
        $index = 0

        foreach ($row in $sync.Computers) {

            $rawName = $row.Column1
            # Skip blank rows without counting them toward progress
            if ([string]::IsNullOrWhiteSpace($rawName)) { continue }

            $computerName = $rawName.Trim().ToUpper()
            $index++
            $percent = [int](($index / $total) * 100)

            $sync.Window.Dispatcher.Invoke([action]{
                $sync.ProgressBar.Value = $percent
                $sync.StatusLabel.Text = "Processing $computerName ($index of $total)..."
            })

            Write-BGLog "Querying AD for $computerName ..."

            try {
                $adComp = Get-ADComputer -Identity $computerName -Properties `
                    lastLogonTimestamp, lastLogonDate, `
                    msDS-LastSuccessfulInteractiveLogonTime, `
                    msDS-LastFailedInteractiveLogonTime, `
                    OperatingSystem, Description, DistinguishedName, Location `
                    -ErrorAction Stop
            } catch {
                $obj = [PSCustomObject]@{
                    ComputerName = $computerName
                    Location = "NOT FOUND"
                    LastLogonTimestamp = "NOT FOUND"
                    LastLogonDate = "NOT FOUND"
                    LastSuccessfulInteractiveLogonTime = "NOT FOUND"
                    LastFailedInteractiveLogonTime = "NOT FOUND"
                    OperatingSystem = "NOT FOUND"
                    Description = "NOT FOUND"
                    DistinguishedName = "NOT FOUND"
                }
                $sync.Window.Dispatcher.Invoke([action]{ $sync.Results.Add($obj) })
                Write-BGLog "AD object not found for $computerName."
                continue
            }

            $obj = [PSCustomObject]@{
                ComputerName = $computerName
                Location = $adComp.Location
                LastLogonTimestamp = Convert-ADTime $adComp.lastLogonTimestamp
                LastLogonDate = Convert-ADTime $adComp.lastLogonDate
                LastSuccessfulInteractiveLogonTime = Convert-ADTime $adComp.'msDS-LastSuccessfulInteractiveLogonTime'
                LastFailedInteractiveLogonTime = Convert-ADTime $adComp.'msDS-LastFailedInteractiveLogonTime'
                OperatingSystem = $adComp.OperatingSystem
                Description = $adComp.Description
                DistinguishedName = $adComp.DistinguishedName
            }

            $sync.Window.Dispatcher.Invoke([action]{ $sync.Results.Add($obj) })
            Write-BGLog "Done: $computerName"
        }

        $sync.Window.Dispatcher.Invoke([action]{
            $sync.ProgressBar.Value = 100
            $sync.StatusLabel.Text = "Completed."
            $sync.BtnStart.IsEnabled = $true
            $sync.BtnExport.IsEnabled = $true
        })

        Write-BGLog "AD scan completed."

        # Clean up the runspace
        $ps.Dispose()

    }).AddArgument($sync)

    # Pass $ps into the sync table so the script block can dispose it
    $sync.PS = $ps

    $ps.BeginInvoke() | Out-Null
})

# ---------------------------
# Show window
# ---------------------------
$window.ShowDialog() | Out-Null

