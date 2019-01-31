$VerbosePreference = 'silentlycontinue'
$DebugPreference = 'silentlycontinue'
# Craig Tolley - 05 August 2016 
    # - Changed formatting to allow form to be wider, better for longer ReportPath values
[xml]$xaml = @"
<Window
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
    x:Name = 'OptionsWindow' Title="PoshPAIG Options" Height="325" Width="500"  
    WindowStartupLocation="CenterScreen" WindowStyle="SingleBorderWindow" FontWeight="Bold">
        <Window.Background>
        <LinearGradientBrush StartPoint='0,0' EndPoint='0,1'>
            <LinearGradientBrush.GradientStops> <GradientStop Color='#C4CBD8' Offset='0' /> <GradientStop Color='#E6EAF5' Offset='0.2' /> 
            <GradientStop Color='#CFD7E2' Offset='0.9' /> <GradientStop Color='#C4CBD8' Offset='1' /> </LinearGradientBrush.GradientStops>
        </LinearGradientBrush>
    </Window.Background>     
    <Grid Name="Grid1" ShowGridLines="False">
        <Grid.RowDefinitions>
            <RowDefinition />
            <RowDefinition />
            <RowDefinition />
            <RowDefinition />
            <RowDefinition />
        </Grid.RowDefinitions>
        <Grid.ColumnDefinitions>
            <ColumnDefinition Width="110" />
            <ColumnDefinition Width="20" />
            <ColumnDefinition Width="340" />
        </Grid.ColumnDefinitions>
        <Label Grid.ColumnSpan="3" Name="OptionsLabel" FontSize="24" VerticalAlignment="Center">PoshPAIG Options</Label>
        <Label Grid.Row="1" Name="MAxJobs_lbl" VerticalAlignment="Center" HorizontalAlignment="Center">MaxJobs</Label>
        <TextBox Grid.Column="2" Grid.Row="1" Name="MaxJobs_txtBx" VerticalAlignment="Center" />
        <TextBox Name="MaxRebootJobs_txtbx" Grid.Column="2" Grid.Row="2" VerticalAlignment="Center" />
        <Label Name="MaxRebootJobs_lbl" Grid.Row="2" HorizontalAlignment="Center" VerticalAlignment="Center">MaxRebootJobs</Label>
        <Label Name="ReportPath_lbl" Grid.Row="3" HorizontalAlignment="Center" VerticalAlignment="Center">Report Path</Label>
        <TextBox Name="ReportPath_txtbx" Grid.Column="2" Grid.Row="3" VerticalAlignment="Center" />
        <Grid Grid.Column="2" Grid.Row="4" Name="Grid2" ShowGridLines="False">
            <Grid.ColumnDefinitions>
                <ColumnDefinition />
                <ColumnDefinition Width="5" />
                <ColumnDefinition />
            </Grid.ColumnDefinitions>
            <Button Name="Cancel_btn" Grid.Column="0" VerticalAlignment="Center" Width = "50">Cancel</Button>
            <Button Name="Save_btn" Grid.Column="2" VerticalAlignment="Center" Width = "50">Save</Button>
        </Grid>
    </Grid>
</Window>
"@

$reader=(New-Object System.Xml.XmlNodeReader $xaml)
$Global:Window=[Windows.Markup.XamlReader]::Load( $reader )
Set-Location $(Split-Path $MyInvocation.MyCommand.Path)
$Global:Path = $(Split-Path $MyInvocation.MyCommand.Path)

##Connect to Controls
$MaxJobs_txtBx = $Window.FindName('MaxJobs_txtBx')
$ReportPath_txtbx = $Window.FindName('ReportPath_txtbx')
$MaxRebootJobs_txtbx = $Window.FindName('MaxRebootJobs_txtbx')
$Cancel_btn = $Window.FindName('Cancel_btn')
$Save_btn = $Window.FindName('Save_btn')

##Event Handlers
#Cancel Button
$Cancel_btn.Add_Click({
    $Window.Close()
})

$Window.Add_Loaded({
    # Craig Tolley - 05 August 2016 
    # - Copied Options logic from the main form for consistency

    # If the Options.xml file exists, then use it, if not then set default option values
    If (Test-Path (Join-Path $Global:Path 'options.xml')) {
        Write-Debug "Options.xml file found"
        $Optionshash = Import-Clixml -Path (Join-Path $Path 'options.xml')
    } Else {
        Write-Debug "Options.xml file not present. Setting default values"
        $optionshash = @{
            MaxJobs = 5
            MaxRebootJobs = 5
            ReportPath = [Environment]::GetFolderPath("Desktop")
        }
    }

    # Validate the MaxJobs Option
    <#If ($Optionshash['MaxJobs'])
    {
        If ([int]$Optionshash['MaxJobs'] -lt 1) {
            $Optionshash['MaxJobs'] = 5
        }
    } Else {
        $Optionshash['MaxJobs'] = 5
    } #>

    # Validate the MaxRebootJobs Option
    <#If ($Optionshash['MaxRebootJobs'])
    {
        If ([int]$Optionshash['MaxRebootJobs'] -lt 1) {
            $Optionshash['MaxRebootJobs'] = 5
        }
    } Else {
        $Optionshash['MaxRebootJobs'] = 5
    } #>   
        
    # Validate the ReportPath Option
    If ($Optionshash['ReportPath']) {
        If (Test-Path $Optionshash['ReportPath']) {
            Write-Debug "Stored ReportPath option found and is valid"
        } Else {
            Write-Debug "Stored ReportPath option is invalid. Reverting to default"
            $Optionshash['ReportPath'] = [Environment]::GetFolderPath("Desktop")
        }
    
    } Else {
        Write-Debug "ReportPath option not found in imported file. Reverting to default"
        $Optionshash['ReportPath'] = [Environment]::GetFolderPath("Desktop")
    }

    # Load the values from the Hashtable into the form
    $MaxRebootJobs_txtbx.Text = $Optionshash['MaxRebootJobs']
    $MaxJobs_txtBx.Text = $Optionshash['MaxJobs']
    $ReportPath_txtbx.Text = $Optionshash['ReportPath']
    
    Write-Verbose ("Current Path: {0}" -f $Global:Path)
})

#Save Button
$Save_btn.Add_Click({
    # Craig Tolley - 05 August 2016 
    # - Validate path now uses Test-Path, so UNC paths are now accepted
    # - Export-CliXML updated to use $Path instead of $pwd
    $optionshash = @{
            MaxJobs = ""
            MaxRebootJobs = ""
            ReportPath = ""
        }

    $i = 0
    #Validate option data is valid
    If ($MaxRebootJobs_txtbx.Text -notmatch "^\d+$") {
        $MaxRebootJobs_txtbx.ForeGround = 'Red'
        $i++
    } Else {
        $MaxRebootJobs_txtbx.Foreground = 'Black'
        $Optionshash['MaxRebootJobs'] = $MaxRebootJobs_txtbx.Text
    }

    If ($MaxJobs_txtBx.Text -notmatch "^\d+$") {
        $MaxJobs_txtBx.ForeGround = 'Red'
        $i++
    } Else {
        $MaxJobs_txtBx.Foreground = 'Black'
        $Optionshash['MaxJobs'] = $MaxJobs_txtBx.Text
    }    

    If ((Test-Path ($ReportPath_txtbx.Text.Trim())) -eq $false) {
        $ReportPath_txtbx.ForeGround = 'Red'
        $i++
    } Else {
        $ReportPath_txtbx.Foreground = 'Black'
        $Optionshash['ReportPath'] = $ReportPath_txtbx.Text.Trim()
    }      

    #Save update options to XML file
    If ($i -eq 0) {

        $optionshash | Export-Clixml -Path (Join-Path $Global:Path 'options.xml') -Force
        $Window.Close()
    }
})

#Used for debugging
$Window.Add_KeyUp({
    If ($_.Key -eq 'F5') {
        Write-Verbose ("MaxJobs_txtBx.Text: {0};{1}" -f $MaxJobs_txtBx.Text,($MaxJobs_txtBx.Text -notmatch "^\d+$"))
        Write-Verbose ("MaxRebootJobs_txtbx: {0};{1}" -f $MaxRebootJobs_txtbx.Text,($MaxRebootJobs_txtbx.Text -notmatch "^\d+$"))
        Write-Verbose ("ReportPath_txtbx: {0}" -f $ReportPath_txtbx.Text)
        Write-Verbose ("I: {0}" -f $i)
    }
})

$Window.Showdialog() | Out-Null
