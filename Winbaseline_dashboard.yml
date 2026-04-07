---

- name: Collect Windows server baseline and build central dashboard

  hosts: windows

  gather_facts: no

  serial: 10

  vars:

    baseline_root: 'D:\ServerBaseline'

    local_collect_dir: './collected_baselines'

    dashboard_output_dir: './dashboard_output'

  tasks:

    - name: Ensure baseline folder exists on target

      ansible.windows.win_file:

        path: "{{ baseline_root }}"

        state: directory

    - name: Collect granular baseline and save JSON on target D drive

      ansible.windows.win_powershell:

        script: |

          $ErrorActionPreference = 'SilentlyContinue'

          $baselineRoot = "{{ baseline_root }}"

          $serverName = $env:COMPUTERNAME

          if (-not (Test-Path "D:\")) {

              throw "D: drive not found on $serverName"

          }

          if (-not (Test-Path $baselineRoot)) {

              New-Item -Path $baselineRoot -ItemType Directory -Force | Out-Null

          }

          $os   = Get-CimInstance Win32_OperatingSystem

          $cs   = Get-CimInstance Win32_ComputerSystem

          $bios = Get-CimInstance Win32_BIOS

          $cpu  = Get-CimInstance Win32_Processor

          $uptime = (Get-Date) - $os.LastBootUpTime

          $logicalDisks = Get-CimInstance Win32_LogicalDisk -Filter "DriveType=3"

          $partitions   = Get-CimInstance Win32_DiskPartition

          $physicalDisk = Get-CimInstance Win32_DiskDrive

          $diskInfo = foreach ($d in $logicalDisks) {

              [PSCustomObject]@{

                  Drive       = $d.DeviceID

                  Label       = $d.VolumeName

                  FileSystem  = $d.FileSystem

                  SizeGB      = [math]::Round($d.Size / 1GB, 2)

                  FreeGB      = [math]::Round($d.FreeSpace / 1GB, 2)

                  UsedGB      = [math]::Round(($d.Size - $d.FreeSpace) / 1GB, 2)

                  UsedPercent = if ($d.Size -gt 0) { [math]::Round((($d.Size - $d.FreeSpace) / $d.Size) * 100, 2) } else { 0 }

              }

          }

          $partitionInfo = foreach ($p in $partitions) {

              [PSCustomObject]@{

                  Name             = $p.Name

                  DiskIndex        = $p.DiskIndex

                  BootPartition    = $p.BootPartition

                  PrimaryPartition = $p.PrimaryPartition

                  SizeGB           = [math]::Round($p.Size / 1GB, 2)

                  Type             = $p.Type

              }

          }

          $physicalDiskInfo = foreach ($pd in $physicalDisk) {

              [PSCustomObject]@{

                  Model      = $pd.Model

                  Interface  = $pd.InterfaceType

                  MediaType  = $pd.MediaType

                  Serial     = $pd.SerialNumber

                  SizeGB     = [math]::Round($pd.Size / 1GB, 2)

                  Partitions = $pd.Partitions

              }

          }

          $netAdapters = Get-CimInstance Win32_NetworkAdapterConfiguration | Where-Object { $_.IPEnabled -eq $true }

          $networkInfo = foreach ($n in $netAdapters) {

              [PSCustomObject]@{

                  Description = $n.Description

                  IPAddress   = ($n.IPAddress -join ', ')

                  SubnetMask  = ($n.IPSubnet -join ', ')

                  Gateway     = ($n.DefaultIPGateway -join ', ')

                  DNSServers  = ($n.DNSServerSearchOrder -join ', ')

                  MACAddress  = $n.MACAddress

                  DHCPEnabled = $n.DHCPEnabled

              }

          }

          $routeInfo = Get-NetRoute -AddressFamily IPv4 | Sort-Object DestinationPrefix, RouteMetric |

              Select-Object -First 25 DestinationPrefix, NextHop, InterfaceAlias, RouteMetric

          $allServices = Get-Service

          $autoServicesNotRunning = $allServices |

              Where-Object { $_.StartType -eq 'Automatic' -and $_.Status -ne 'Running' } |

              Select-Object Name, DisplayName, Status, StartType

          $stoppedServices = $allServices |

              Where-Object { $_.Status -eq 'Stopped' } |

              Select-Object -First 25 Name, DisplayName, StartType

          $topProcessesByCPU = Get-Process | Sort-Object CPU -Descending |

              Select-Object -First 15 Name, Id, CPU,

              @{Name='MemoryMB';Expression={[math]::Round($_.WorkingSet64/1MB,2)}}

          $topProcessesByMemory = Get-Process | Sort-Object WorkingSet64 -Descending |

              Select-Object -First 15 Name, Id,

              @{Name='MemoryMB';Expression={[math]::Round($_.WorkingSet64/1MB,2)}},

              CPU

          $hotfixes = Get-HotFix | Sort-Object InstalledOn -Descending |

              Select-Object -First 25 HotFixID, Description, InstalledBy, InstalledOn

          $uninstallPaths = @(

              'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*',

              'HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*'

          )

          $software = foreach ($path in $uninstallPaths) {

              Get-ItemProperty -Path $path -ErrorAction SilentlyContinue |

                  Where-Object { $_.DisplayName } |

                  Select-Object DisplayName, DisplayVersion, Publisher, InstallDate

          }

          $software = $software | Sort-Object DisplayName -Unique | Select-Object -First 200

          $localUsers = @()

          try {

              $localUsers = Get-LocalUser | Select-Object Name, Enabled, LastLogon, PasswordRequired, PasswordExpires

          } catch {}

          $localAdmins = @()

          try {

              $localAdmins = Get-LocalGroupMember -Group "Administrators" |

                  Select-Object Name, ObjectClass, PrincipalSource

          } catch {}

          $shares = @()

          try {

              $shares = Get-SmbShare |

                  Where-Object { $_.Name -notin @('ADMIN$','C$','IPC$','PRINT$') } |

                  Select-Object Name, Path, Description, CurrentUsers

          } catch {}

          $firewallProfiles = @()

          try {

              $firewallProfiles = Get-NetFirewallProfile |

                  Select-Object Name, Enabled, DefaultInboundAction, DefaultOutboundAction

          } catch {}

          $rdpRegistry = Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server" -ErrorAction SilentlyContinue

          $rdpEnabled = if ($rdpRegistry.fDenyTSConnections -eq 0) { "Enabled" } else { "Disabled" }

          $winrmSvc = Get-Service -Name WinRM -ErrorAction SilentlyContinue

          $winrmStatus = if ($winrmSvc) { $winrmSvc.Status } else { "Not Found" }

          $pendingReboot = $false

          $pendingReasons = @()

          if (Test-Path 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Component Based Servicing\RebootPending') {

              $pendingReboot = $true

              $pendingReasons += 'CBS RebootPending'

          }

          if (Test-Path 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\WindowsUpdate\Auto Update\RebootRequired') {

              $pendingReboot = $true

              $pendingReasons += 'Windows Update RebootRequired'

          }

          $sessMgr = Get-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager' -ErrorAction SilentlyContinue

          if ($sessMgr.PendingFileRenameOperations) {

              $pendingReboot = $true

              $pendingReasons += 'PendingFileRenameOperations'

          }

          $scheduledTasks = @()

          try {

              $scheduledTasks = Get-ScheduledTask |

                  Select-Object -First 50 TaskName, TaskPath, State, Author

          } catch {}

          $pageFile = Get-CimInstance Win32_PageFileUsage |

              Select-Object Name,

              @{Name='AllocatedMB';Expression={$_.AllocatedBaseSize}},

              @{Name='CurrentUsageMB';Expression={$_.CurrentUsage}},

              @{Name='PeakUsageMB';Expression={$_.PeakUsage}}

          $securityServices = Get-Service |

              Where-Object { $_.DisplayName -match 'defender|antivirus|sentinel|crowdstrike|mcafee|symantec|trend|carbon black' } |

              Select-Object Name, DisplayName, Status, StartType

          $systemErrors = @()

          try {

              $systemErrors = Get-WinEvent -LogName System -MaxEvents 300 |

                  Where-Object { $_.LevelDisplayName -eq 'Error' } |

                  Select-Object -First 20 TimeCreated, Id, ProviderName, Message

          } catch {}

          $applicationErrors = @()

          try {

              $applicationErrors = Get-WinEvent -LogName Application -MaxEvents 300 |

                  Where-Object { $_.LevelDisplayName -eq 'Error' } |

                  Select-Object -First 20 TimeCreated, Id, ProviderName, Message

          } catch {}

          $baseline = [PSCustomObject]@{

              Status                = "Success"

              ServerName            = $serverName

              CollectedOn           = (Get-Date).ToString("yyyy-MM-dd HH:mm:ss")

              Domain                = $cs.Domain

              Manufacturer          = $cs.Manufacturer

              Model                 = $cs.Model

              OSName                = $os.Caption

              OSVersion             = $os.Version

              BuildNumber           = $os.BuildNumber

              OSArchitecture        = $os.OSArchitecture

              InstallDate           = $os.InstallDate

              LastBootUpTime        = $os.LastBootUpTime

              UptimeDays            = [math]::Round($uptime.TotalDays, 2)

              TimeZone              = (Get-TimeZone).DisplayName

              TotalRAMGB            = [math]::Round($cs.TotalPhysicalMemory / 1GB, 2)

              FreePhysicalMemoryGB  = [math]::Round(($os.FreePhysicalMemory * 1KB) / 1GB, 2)

              CPUName               = ($cpu | Select-Object -ExpandProperty Name) -join '; '

              CPUSockets            = ($cpu | Measure-Object).Count

              LogicalProcessors     = ($cpu | Measure-Object -Property NumberOfLogicalProcessors -Sum).Sum

              BIOSSerial            = $bios.SerialNumber

              WinRMStatus           = $winrmStatus

              RDPStatus             = $rdpEnabled

              PendingReboot         = $pendingReboot

              PendingRebootReasons  = ($pendingReasons -join '; ')

              RunningServices       = ($allServices | Where-Object Status -eq 'Running').Count

              StoppedServices       = ($allServices | Where-Object Status -eq 'Stopped').Count

              DiskInfo              = @($diskInfo)

              PartitionInfo         = @($partitionInfo)

              PhysicalDiskInfo      = @($physicalDiskInfo)

              NetworkInfo           = @($networkInfo)

              RouteInfo             = @($routeInfo)

              AutoServicesNotRunning= @($autoServicesNotRunning)

              StoppedServicesList   = @($stoppedServices)

              TopProcessesByCPU     = @($topProcessesByCPU)

              TopProcessesByMemory  = @($topProcessesByMemory)

              RecentHotFixes        = @($hotfixes)

              InstalledSoftware     = @($software)

              LocalUsers            = @($localUsers)

              LocalAdministrators   = @($localAdmins)

              Shares                = @($shares)

              FirewallProfiles      = @($firewallProfiles)

              ScheduledTasks        = @($scheduledTasks)

              PageFile              = @($pageFile)

              SecurityServices      = @($securityServices)

              SystemErrors          = @($systemErrors)

              ApplicationErrors     = @($applicationErrors)

          }

          $jsonPath = Join-Path $baselineRoot "$serverName`_Baseline.json"

          $baseline | ConvertTo-Json -Depth 10 | Out-File -FilePath $jsonPath -Encoding UTF8

          $result = [PSCustomObject]@{

              JsonPath = $jsonPath

              ServerName = $serverName

          }

          $result | ConvertTo-Json -Depth 3

      register: baseline_collection

    - name: Parse returned JSON metadata

      ansible.builtin.set_fact:

        baseline_result: "{{ baseline_collection.output | join('') | from_json }}"

    - name: Ensure local collection directory exists

      delegate_to: localhost

      ansible.builtin.file:

        path: "{{ local_collect_dir }}"

        state: directory

        mode: '0755'

    - name: Fetch baseline JSON from each target

      ansible.windows.win_fetch:

        src: "{{ baseline_result.JsonPath }}"

        dest: "{{ local_collect_dir }}/"

        flat: yes

- name: Build combined dashboard on control node

  hosts: localhost

  gather_facts: no

  vars:

    local_collect_dir: './collected_baselines'

    dashboard_output_dir: './dashboard_output'

  tasks:

    - name: Ensure dashboard output directory exists

      ansible.builtin.file:

        path: "{{ dashboard_output_dir }}"

        state: directory

        mode: '0755'

    - name: Find collected JSON files

      ansible.builtin.find:

        paths: "{{ local_collect_dir }}"

        patterns: "*_Baseline.json"

      register: baseline_files

    - name: Read collected baseline files

      ansible.builtin.slurp:

        src: "{{ item.path }}"

      loop: "{{ baseline_files.files }}"

      register: slurped_baselines

    - name: Build baseline list

      ansible.builtin.set_fact:

        baseline_data: >-

          {{

            slurped_baselines.results

            | map(attribute='content')

            | map('b64decode')

            | map('from_json')

            | list

          }}

    - name: Save combined JSON

      ansible.builtin.copy:

        dest: "{{ dashboard_output_dir }}/AllServerBaselines.json"

        content: "{{ baseline_data | to_nice_json }}"

        mode: '0644'

    - name: Create central HTML dashboard

      ansible.builtin.copy:

        dest: "{{ dashboard_output_dir }}/BaselineDashboard.html"

        mode: '0644'

        content: |
<html>
<head>
<title>Multi-Server Baseline Dashboard</title>
<style>

                  body { font-family: Arial; background:#f4f6f9; margin:20px; }

                  .card { background:white; padding:16px; margin-bottom:16px; border-radius:8px; box-shadow:0 1px 4px rgba(0,0,0,.15); }

                  table { border-collapse: collapse; width:100%; font-size:13px; margin-bottom:12px; }

                  th,td { border:1px solid #ccc; padding:8px; text-align:left; vertical-align:top; }

                  th { background:#ddebf7; }

                  .kv td:first-child { width:260px; font-weight:bold; }

                  input { padding:8px; width:340px; }
</style>
<script>

              function filterSummary() {

                  var input = document.getElementById('filterBox').value.toLowerCase();

                  var rows = document.getElementById('summaryTable').getElementsByTagName('tr');

                  for (var i = 1; i < rows.length; i++) {

                      var txt = rows[i].innerText.toLowerCase();

                      rows[i].style.display = txt.indexOf(input) > -1 ? '' : 'none';

                  }

              }
</script>
</head>
<body>
<h1>Multi-Server Baseline Dashboard</h1>
<div class='card'>
<b>Generated On:</b> {{ lookup('pipe', 'date "+%Y-%m-%d %H:%M:%S"') }}<br><br>
<input type='text' id='filterBox' placeholder='Filter servers...' onkeyup='filterSummary()'>
</div>
<div class='card'>
<h2>Summary</h2>
<table id='summaryTable'>
<tr>
<th>Server</th>
<th>Health</th>
<th>OS</th>
<th>Primary IP</th>
<th>Uptime Days</th>
<th>C: Free GB</th>
<th>Pending Reboot</th>
</tr>

                  {% for b in baseline_data %}

                  {% set cdrive = (b.DiskInfo | selectattr('Drive', 'equalto', 'C:') | list | first) %}

                  {% set primary_ip = (b.NetworkInfo[0].IPAddress if (b.NetworkInfo | length > 0) else '') %}

                  {% set health = 'Attention' if (b.PendingReboot or ((cdrive is defined and cdrive) and (cdrive.UsedPercent | float >= 90)) or ((b.AutoServicesNotRunning | default([]) | length) > 0)) else 'Healthy' %}
<tr>
<td><a href="#{{ b.ServerName }}">{{ b.ServerName }}</a></td>
<td>{{ health }}</td>
<td>{{ b.OSName }}</td>
<td>{{ primary_ip }}</td>
<td>{{ b.UptimeDays }}</td>
<td>{{ cdrive.FreeGB if cdrive is defined and cdrive else '' }}</td>
<td>{{ b.PendingReboot }}</td>
</tr>

                  {% endfor %}
</table>
</div>

          {% for b in baseline_data %}
<div class='card'>
<h2 id='{{ b.ServerName }}'>{{ b.ServerName }}</h2>
<table class='kv'>
<tr><td>Collected On</td><td>{{ b.CollectedOn }}</td></tr>
<tr><td>OS</td><td>{{ b.OSName }}</td></tr>
<tr><td>Version / Build</td><td>{{ b.OSVersion }} / {{ b.BuildNumber }}</td></tr>
<tr><td>Architecture</td><td>{{ b.OSArchitecture }}</td></tr>
<tr><td>Domain</td><td>{{ b.Domain }}</td></tr>
<tr><td>Manufacturer / Model</td><td>{{ b.Manufacturer }} / {{ b.Model }}</td></tr>
<tr><td>CPU</td><td>{{ b.CPUName }}</td></tr>
<tr><td>Total RAM / Free RAM (GB)</td><td>{{ b.TotalRAMGB }} / {{ b.FreePhysicalMemoryGB }}</td></tr>
<tr><td>Uptime Days</td><td>{{ b.UptimeDays }}</td></tr>
<tr><td>RDP</td><td>{{ b.RDPStatus }}</td></tr>
<tr><td>WinRM</td><td>{{ b.WinRMStatus }}</td></tr>
<tr><td>Pending Reboot</td><td>{{ b.PendingReboot }} - {{ b.PendingRebootReasons }}</td></tr>
</table>
<h3>Disk Information</h3>
<table>
<tr><th>Drive</th><th>Label</th><th>FS</th><th>SizeGB</th><th>FreeGB</th><th>UsedGB</th><th>Used%</th></tr>

                  {% for d in b.DiskInfo %}
<tr>
<td>{{ d.Drive }}</td>
<td>{{ d.Label }}</td>
<td>{{ d.FileSystem }}</td>
<td>{{ d.SizeGB }}</td>
<td>{{ d.FreeGB }}</td>
<td>{{ d.UsedGB }}</td>
<td>{{ d.UsedPercent }}</td>
</tr>

                  {% endfor %}
</table>
<h3>Network Information</h3>
<table>
<tr><th>Adapter</th><th>IP</th><th>Subnet</th><th>Gateway</th><th>DNS</th><th>MAC</th><th>DHCP</th></tr>

                  {% for n in b.NetworkInfo %}
<tr>
<td>{{ n.Description }}</td>
<td>{{ n.IPAddress }}</td>
<td>{{ n.SubnetMask }}</td>
<td>{{ n.Gateway }}</td>
<td>{{ n.DNSServers }}</td>
<td>{{ n.MACAddress }}</td>
<td>{{ n.DHCPEnabled }}</td>
</tr>

                  {% endfor %}
</table>
<h3>Automatic Services Not Running</h3>
<table>
<tr><th>Name</th><th>Display Name</th><th>Status</th><th>Start Type</th></tr>

                  {% for s in b.AutoServicesNotRunning | default([]) %}
<tr>
<td>{{ s.Name }}</td>
<td>{{ s.DisplayName }}</td>
<td>{{ s.Status }}</td>
<td>{{ s.StartType }}</td>
</tr>

                  {% endfor %}
</table>
<h3>Recent HotFixes</h3>
<table>
<tr><th>HotFixID</th><th>Description</th><th>Installed By</th><th>Installed On</th></tr>

                  {% for h in b.RecentHotFixes | default([]) %}
<tr>
<td>{{ h.HotFixID }}</td>
<td>{{ h.Description }}</td>
<td>{{ h.InstalledBy }}</td>
<td>{{ h.InstalledOn }}</td>
</tr>

                  {% endfor %}
</table>
<h3>Local Administrators</h3>
<table>
<tr><th>Name</th><th>Object Class</th><th>Principal Source</th></tr>

                  {% for a in b.LocalAdministrators | default([]) %}
<tr>
<td>{{ a.Name }}</td>
<td>{{ a.ObjectClass }}</td>
<td>{{ a.PrincipalSource }}</td>
</tr>

                  {% endfor %}
</table>
</div>

          {% endfor %}
</body>
</html>
 
