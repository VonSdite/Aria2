<# :
@echo off
copy/b "%~f0" "%temp%\%~n0.ps1" >nul
powershell -v 2 -ep bypass -noprofile "%temp%\%~n0.ps1" "'%cd% '" "'%~1'"
del "%temp%\%~n0.ps1"
echo: & pause
exit /b
#>
param([string]$cwd='.', [string]$dll)

function main {
    write-host -f white -b black `
        "Chrome 'developer mode extensions' warning disabler 20170328"
    $pathsDone = @{}
    if ($dll -and (gi -literal $dll)) {
        doPatch "DRAG'n'DROPPED" ((gi -literal $dll).directoryName + '\')
        exit
    }
    doPatch 'CURRENT DIRECTORY' ((gi -literal $cwd).fullName + '\')
    ('HKLM', 'HKCU') | %{ $hive = $_
        ('', '\Wow6432Node') | %{
            $key = "${hive}:\SOFTWARE$_\Google\Update\Clients"
            gci -ea silentlycontinue $key -r | gp | ?{ $_.CommandLine } | %{
                $path = $_.CommandLine -replace '"(.+?\\\d+\.\d+\.\d+\.\d+\\).+', '$1'
                if (!$pathsDone[$path.toLower()]) {
                    doPatch REGISTRY $path
                    $pathsDone[$path.toLower()] = $true
                }
            }
        }
    }
}

function doPatch([string]$pathLabel, [string]$path) {
    $dll = Join-Path $path chrome.dll
    if (!(Test-Path -literal $dll)) {
        return
    }
    ''
    $localAppData = [Environment]::GetFolderPath('LocalApplicationData')
    "$pathLabel $((split-path $dll).Replace($localAppData, '%LocalAppData%'))"

    "`tREADING Chrome.dll..."
    $bin = [IO.BinaryReader][IO.File]::OpenRead($dll)
    $bytes = $bin.ReadBytes(1MB)

    # process PE headers
    $BC = [BitConverter]
    $coff = $BC::ToUInt32($bytes,0x3C) + 4
    $is64 = $BC::ToUInt16($bytes,$coff) -eq 0x8664
    $opthdr = $coff+20
    $codesize = $BC::ToUInt32($bytes,$opthdr+4)
    $imagebase32 = $BC::ToUInt32($bytes,$opthdr+28)

    # patch the flag in data section
    $bin.BaseStream.Position = $codesize
    $data = $BC::ToString($bin.ReadBytes($bin.BaseStream.Length - $codesize))
    $bin.Close()
    $flag = 'ExtensionDeveloperModeWarning'
    $stroffs = $data.IndexOf($BC::ToString($flag[1..99]))
    if ($stroffs -lt 0) {
        write-host -f red "`t$flag not found"
        return
    }
    if ($data.substring($stroffs-3, 2) -eq '00') {
        write-host -f darkgreen "`tALREADY PATCHED"
        return
    }
    $stroffs = $stroffs/3 - 1 + $codesize

    $centbrowser = $data.indexOf($BC::ToString('CentBrowser'[0..99])) -gt 0

    $EA = $ErrorActionPreference
    $ErrorActionPreference = 'silentlyContinue'
    $exe = join-path (split-path $path) chrome.exe
    while ((get-process chrome -module | ?{ $_.FileName -eq $exe })) {
        forEach ($timeout in 15..0) {
            write-host -n -b yellow -f black `
                "`rChrome is running and will be terminated in $timeout sec. "
            write-host -n -b yellow -f darkyellow "Press ENTER to do it now. "
            if ([console]::KeyAvailable) {
                $key = $Host.UI.RawUI.ReadKey("AllowCtrlC,IncludeKeyDown,NoEcho")
                if ($key.virtualKeyCode -eq 13) { break }
                if ($key.virtualKeyCode -eq 27) { write-host; exit }
            }
            sleep 1
        }
        write-host
        get-process chrome | ?{
            $_.MainWindowHandle.toInt64() -and ($_ | gps -file).FileName -eq $exe
        } | %{
            "`tTrying to exit gracefully..."
            if ($_.CloseMainWindow()) {
                sleep 1
            }
        }
        $killLabelShown = 0
        get-process chrome | ?{
            ($_ | gps -file | select -expand FileName) -eq $exe
        } | %{
            if (!$killLabelShown++) {
                "`tTerminating background chrome processes..."
            }
            stop-process $_ -force
        }
        sleep -milliseconds 200
    }
    $ErrorActionPreference = $EA

    $bytes = [IO.File]::ReadAllBytes($dll)
    $bytes[$stroffs] = 0
    "`tPATCHED $flag flag"

    # patch the channel restriction code for stable/beta
    $rxChannel = '83-F8-(?:03-7D|02-7F|02-0F-8F)'
    # old code: cmp eax,3; jge ...
    # new code: cmp eax,2; jg ... (jg can be 2-byte)
    function patch64 {
        $pos = 0
        $rx = [regex]"$rxChannel-.{1,100}-48-8D"
        do {
            $m = $rx.match($code,$pos)
            if (!$m.success) { break }
            $chanpos = $searchBase + $m.index/3 + 2
            $pos = $m.index + $m.length + 1
            $offs = $BC::ToUInt32($bytes, $searchBase + $pos/3+1)
            $diff = $searchBase + $pos/3+5+$offs - $stroffs
        } until ($diff -ge 0 -and $diff -le 4096 -and $diff % 256 -eq 0)
        if (!$m.success) {
            $rx = [regex]"84-C0.{18,48}($rxChannel)-.{30,60}84-C0"
            $m = $rx.matches($code)
            if ($m.count -ne 1) { return }
            $chanpos = $searchBase + $m[0].groups[1].index/3 + 2
        }
        $chanpos
    }
    function patch86 {
        $flagOffs = [uint32]$stroffs + [uint32]$imagebase32
        $flagOffsStr = $BC::ToString($BC::GetBytes($flagOffs))
        $variants = "(?<channel>$rxChannel-.{1,100})-68-(?<flag>`$1-.{6}`$2)",
                "68-(?<flag>`$1-.{6}`$2).{300,500}E8.{12,32}(?<channel>$rxChannel)",
                "E8.{12,32}(?<channel>$rxChannel).{300,500}68-(?<flag>`$1-.{6}`$2)"
        forEach ($variant in $variants) {
            $pattern = $flagOffsStr -replace '^(..)-.{6}(..)', $variant
            $patternDisplay = $pattern -replace '^(.{40}).+', '$1'
            write-host -f darkgray "`tLooking for $patternDisplay..."
            $minDiff = 65536
            foreach ($m in [regex]::matches($code, $pattern)) {
                $maybeFlagOffs = $BC::toUInt32($bytes, $searchBase +
                                                       $m.groups['flag'].index/3)
                $diff = [Math]::abs($maybeFlagOffs - $flagOffs)
                if ($diff % 256 -eq 0 -and $diff -lt $minDiff) {
                    $minDiff = $diff
                    $chanpos = $searchBase + $m.groups['channel'].index/3 + 2
                }
            }
        }
        $chanpos
    }
    $searchBase = [int]($codesize/2)
    foreach ($pass in 1..2) {
        if ($centbrowser) { break }
        $code = $BC::ToString($bytes, $searchBase, $codesize - $searchBase)
        $chanpos = if ($is64) { patch64 } else { patch86 }
        if ($chanpos) { break }
        $codesize = $searchBase
        $searchBase = 0
    }
    if ($chanpos) {
        $bytes[$chanpos] = 9
        "`tPATCHED Chrome release channel restriction"
    } elseif (!$centbrowser) {
        write-host -f red "`tUnable to find the channel code, try updating me"
        write-host -f red "`thttp://stackoverflow.com/a/30361260"
        return
    }

    "`tWriting to a temporary dll..."
    [IO.File]::WriteAllBytes("$dll.new",$bytes)

    "`tBacking up the original dll..."
    move -literal $dll "$dll.bak" -force

    "`tRenaming the temporary dll as the original dll..."
    move -literal "$dll.new" $dll -force

    write-host -f green "`tDONE.`n"
    [GC]::Collect()
}

main