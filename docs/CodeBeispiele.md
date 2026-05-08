# Code-Beispiele

Produktionsreife PowerShell-Funktionen zum Copy-Paste.

## 1. WTSSendMessage Wrapper

Einfache Hilfsfunktion zum Senden von Nachrichten an Benutzersession.

```powershell
function Send-MessageToUser {
    <#
    .SYNOPSIS
        Sendet eine Nachricht an eine Benutzersession über WTS API
    .PARAMETER Title
        Dialog-Titel
    .PARAMETER Message
        Dialog-Nachricht
    .PARAMETER Style
        Button-Set: 0=OK, 1=OK/Cancel, 4=Yes/No, 5=Retry/Cancel, 6=Abort/Retry/Ignore
    .PARAMETER TimeoutSeconds
        Auto-Close nach N Sekunden (0 = kein Timeout)
    .EXAMPLE
        $response = Send-MessageToUser -Title "Frage" -Message "Weitermachen?" -Style 4
        if ($response -eq 6) { Write-Host "Benutzer: Ja" }
    #>
    [CmdletBinding()]
    param(
        [Parameter(Mandatory=$true)][string]$Title,
        [Parameter(Mandatory=$true)][string]$Message,
        [Parameter(Mandatory=$false)][int]$Style = 0,
        [Parameter(Mandatory=$false)][int]$TimeoutSeconds = 0,
        [Parameter(Mandatory=$false)][uint]$SessionId = 1
    )

    Add-Type -TypeDefinition @'
using System;
using System.Runtime.InteropServices;

public class WTSMessage {
    [DllImport("wtsapi32.dll", SetLastError=true)]
    public static extern bool WTSSendMessage(IntPtr hServer, uint SessionId, string pTitle,
        uint TitleLength, string pMessage, uint MessageLength, uint Style, uint TimeOut,
        out uint pResponse, bool bWait);

    [DllImport("wtsapi32.dll", SetLastError=true)]
    public static extern IntPtr WTSOpenServer(string ServerName);

    [DllImport("wtsapi32.dll", SetLastError=true)]
    public static extern void WTSCloseServer(IntPtr hServer);
}
'@ -ErrorAction SilentlyContinue

    try {
        $hServer = [WTSMessage]::WTSOpenServer("localhost")
        $response = 0

        $success = [WTSMessage]::WTSSendMessage($hServer, $SessionId, $Title,
            [uint]$Title.Length, $Message, [uint]$Message.Length, [uint]$Style,
            [uint]$TimeoutSeconds, [ref]$response, $true)

        [WTSMessage]::WTSCloseServer($hServer)

        if ($success) {
            return $response
        } else {
            Write-Error "WTSSendMessage fehlgeschlagen: $(Get-Error)"
            return -1
        }
    }
    catch {
        Write-Error "Fehler in Send-MessageToUser: $_"
        return -1
    }
}

# Verwendungsbeispiel
$answer = Send-MessageToUser -Title "Sicherheitsupdate" -Message "Neustarten?" -Style 4 -TimeoutSeconds 60
Write-Host "Antwort: $answer (6=Ja, 7=Nein)"
```

---

## 2. Enumerate All User Sessions

```powershell
function Get-UserSessions {
    <#
    .SYNOPSIS
        Listet alle Benutzersessions auf (RDS/Terminal Server + Console)
    .EXAMPLE
        Get-UserSessions | Where-Object { $_.State -eq 'Active' }
    #>

    Add-Type -TypeDefinition @'
using System;
using System.Runtime.InteropServices;

public class WTSSessionInfo {
    public uint SessionId { get; set; }
    public string StationName { get; set; }
    public string State { get; set; }
}

public class WTSEnumerator {
    [DllImport("wtsapi32.dll", SetLastError=true)]
    private static extern bool WTSEnumerateSessions(IntPtr hServer, uint Reserved, uint Version,
        out IntPtr ppSessionInfo, out uint pCount);

    [DllImport("wtsapi32.dll", SetLastError=true)]
    private static extern void WTSFreeMemory(IntPtr pMemory);

    [DllImport("wtsapi32.dll", SetLastError=true)]
    private static extern IntPtr WTSOpenServer(string ServerName);

    [DllImport("wtsapi32.dll", SetLastError=true)]
    private static extern void WTSCloseServer(IntPtr hServer);

    [StructLayout(LayoutKind.Sequential)]
    private struct WTS_SESSION_INFO {
        public uint SessionId;
        [MarshalAs(UnmanagedType.LPStr)]
        public string pWinStationName;
        public uint State;
    }

    public static WTSSessionInfo[] GetActiveSessions() {
        IntPtr hServer = WTSOpenServer("localhost");
        IntPtr pSessionInfo = IntPtr.Zero;
        uint sessionCount = 0;

        if (!WTSEnumerateSessions(hServer, 0, 1, out pSessionInfo, out sessionCount)) {
            WTSCloseServer(hServer);
            return new WTSSessionInfo[0];
        }

        WTSSessionInfo[] sessions = new WTSSessionInfo[sessionCount];
        int sessionSize = Marshal.SizeOf(typeof(WTS_SESSION_INFO));

        for (uint i = 0; i < sessionCount; i++) {
            WTS_SESSION_INFO info = (WTS_SESSION_INFO)Marshal.PtrToStructure(
                IntPtr.Add(pSessionInfo, (int)(i * sessionSize)),
                typeof(WTS_SESSION_INFO)
            );

            sessions[i] = new WTSSessionInfo {
                SessionId = info.SessionId,
                StationName = info.pWinStationName,
                State = GetStateName(info.State)
            };
        }

        WTSFreeMemory(pSessionInfo);
        WTSCloseServer(hServer);

        return sessions;
    }

    private static string GetStateName(uint state) {
        return state switch {
            0 => "Active",
            1 => "Connected",
            2 => "ConnectQuery",
            3 => "Shadow",
            4 => "Disconnected",
            5 => "Idle",
            6 => "Listen",
            7 => "Reset",
            8 => "Down",
            9 => "Init",
            _ => "Unknown"
        };
    }
}
'@ -ErrorAction SilentlyContinue

    [WTSEnumerator]::GetActiveSessions()
}

# Beispiel: Alle aktiven Sessions finden
Get-UserSessions | ForEach-Object {
    Write-Host "Session $($_.SessionId): $($_.StationName) ($($_.State))"
}
```

---

## 3. CreateProcessAsUser (Vereinfacht)

```powershell
function Invoke-ProcessAsUser {
    <#
    .SYNOPSIS
        Startet einen Prozess im Kontext des angemeldeten Benutzers
    .PARAMETER ProcessPath
        Pfad zum ausführbaren Programm
    .PARAMETER Arguments
        Kommandozeilen-Argumente
    .EXAMPLE
        Invoke-ProcessAsUser -ProcessPath "notepad.exe"
        Invoke-ProcessAsUser -ProcessPath "powershell.exe" -Arguments "-NoProfile -Command 'Write-Host Hello'"
    #>
    [CmdletBinding()]
    param(
        [Parameter(Mandatory=$true)][string]$ProcessPath,
        [Parameter(Mandatory=$false)][string]$Arguments = ''
    )

    Add-Type -TypeDefinition @'
using System;
using System.Runtime.InteropServices;

public class ProcessAsUser {
    [DllImport("kernel32.dll", SetLastError=true)]
    public static extern bool OpenProcessToken(IntPtr ProcessHandle, uint DesiredAccess, out IntPtr TokenHandle);

    [DllImport("advapi32.dll", SetLastError=true)]
    public static extern bool DuplicateTokenEx(IntPtr hExistingToken, uint dwDesiredAccess,
        IntPtr lpThreadAttributes, int ImpersonationLevel, int TokenType, out IntPtr phNewToken);

    [DllImport("kernel32.dll", SetLastError=true)]
    public static extern bool CreateProcessAsUser(IntPtr hToken, string lpApplicationName,
        string lpCommandLine, IntPtr lpProcessAttributes, IntPtr lpThreadAttributes,
        bool bInheritHandles, uint dwCreationFlags, IntPtr lpEnvironment, string lpCurrentDirectory,
        STARTUPINFO lpStartupInfo, out PROCESS_INFORMATION lpProcessInformation);

    [DllImport("kernel32.dll", SetLastError=true)]
    public static extern IntPtr OpenProcess(uint dwDesiredAccess, bool bInheritHandle, uint dwProcessId);

    [DllImport("kernel32.dll", SetLastError=true)]
    public static extern bool CloseHandle(IntPtr hObject);

    [StructLayout(LayoutKind.Sequential)]
    public struct STARTUPINFO {
        public uint cb;
        public IntPtr lpReserved;
        public IntPtr lpDesktop;
        public IntPtr lpTitle;
        public uint dwX;
        public uint dwY;
        public uint dwXSize;
        public uint dwYSize;
        public uint dwXCountChars;
        public uint dwYCountChars;
        public uint dwFillAttribute;
        public uint dwFlags;
        public ushort wShowWindow;
        public ushort cbReserved2;
        public IntPtr lpReserved2;
        public IntPtr hStdInput;
        public IntPtr hStdOutput;
        public IntPtr hStdError;
    }

    [StructLayout(LayoutKind.Sequential)]
    public struct PROCESS_INFORMATION {
        public IntPtr hProcess;
        public IntPtr hThread;
        public uint dwProcessId;
        public uint dwThreadId;
    }
}
'@ -ErrorAction SilentlyContinue

    $TOKEN_DUPLICATE = 0x0002
    $TOKEN_QUERY = 0x0008
    $TOKEN_ASSIGN_PRIMARY = 0x0001
    $DESIRED_ACCESS = $TOKEN_DUPLICATE -bor $TOKEN_QUERY -bor $TOKEN_ASSIGN_PRIMARY
    $SECURITY_IMPERSONATION = 2
    $TOKEN_TYPE_PRIMARY = 1
    $NORMAL_PRIORITY_CLASS = 0x20

    try {
        # Finde explorer.exe (angemeldeter User)
        $explorer = Get-Process -Name explorer -ErrorAction Stop | Select-Object -First 1
        if (-not $explorer) {
            Write-Error "explorer.exe nicht gefunden — Benutzer nicht angemeldet?"
            return $false
        }

        # Öffne Process und extrahiere Token
        $hProcess = [ProcessAsUser]::OpenProcess(0x0400, $false, $explorer.Id)
        if ($hProcess -eq 0) {
            Write-Error "Konnte explorer.exe-Handle nicht öffnen"
            return $false
        }

        $hToken = 0
        if (-not [ProcessAsUser]::OpenProcessToken($hProcess, $DESIRED_ACCESS, [ref]$hToken)) {
            Write-Error "Konnte Token nicht extrahieren"
            [ProcessAsUser]::CloseHandle($hProcess)
            return $false
        }

        # Dupliziere Token
        $hDupToken = 0
        if (-not [ProcessAsUser]::DuplicateTokenEx($hToken, $DESIRED_ACCESS, [IntPtr]::Zero,
            $SECURITY_IMPERSONATION, $TOKEN_TYPE_PRIMARY, [ref]$hDupToken)) {
            Write-Error "Konnte Token nicht duplizieren"
            [ProcessAsUser]::CloseHandle($hToken)
            [ProcessAsUser]::CloseHandle($hProcess)
            return $false
        }

        # Vorbereite StartupInfo
        $startupInfo = New-Object ProcessAsUser+STARTUPINFO
        $startupInfo.cb = [System.Runtime.InteropServices.Marshal]::SizeOf($startupInfo)
        $startupInfo.lpDesktop = [System.Runtime.InteropServices.Marshal]::StringToHGlobalAnsi("WinSta0\Default")
        $startupInfo.wShowWindow = 1  # SW_SHOWNORMAL

        # Process-Information
        $procInfo = New-Object ProcessAsUser+PROCESS_INFORMATION

        # Starte Prozess als User
        $cmdLine = if ($Arguments) { "`"$ProcessPath`" $Arguments" } else { $ProcessPath }
        $success = [ProcessAsUser]::CreateProcessAsUser($hDupToken, $ProcessPath, $cmdLine,
            [IntPtr]::Zero, [IntPtr]::Zero, $false, $NORMAL_PRIORITY_CLASS, [IntPtr]::Zero, $PWD,
            [ref]$startupInfo, [ref]$procInfo)

        # Cleanup
        [ProcessAsUser]::CloseHandle($hToken)
        [ProcessAsUser]::CloseHandle($hDupToken)
        [ProcessAsUser]::CloseHandle($hProcess)
        if ($procInfo.hProcess -ne 0) {
            [ProcessAsUser]::CloseHandle($procInfo.hProcess)
            [ProcessAsUser]::CloseHandle($procInfo.hThread)
        }

        return $success
    }
    catch {
        Write-Error "Fehler in Invoke-ProcessAsUser: $_"
        return $false
    }
}

# Beispiel
Invoke-ProcessAsUser -ProcessPath "notepad.exe"
```

---

## 4. Robuste Fehlerbehandlung

```powershell
function Send-MessageToUserSafe {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory=$true)][string]$Title,
        [Parameter(Mandatory=$true)][string]$Message,
        [Parameter(Mandatory=$false)][int]$MaxRetries = 3,
        [Parameter(Mandatory=$false)][int]$RetryDelayMs = 500
    )

    $logFile = "C:\ProgramData\SystemDialog.log"

    function Log-Message {
        param([string]$Msg, [ValidateSet('Info', 'Warning', 'Error')][string]$Level = 'Info')
        $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
        $logLine = "[$timestamp] [$Level] $Msg"
        Write-Host $logLine
        try { Add-Content -Path $logFile -Value $logLine } catch { }
    }

    Log-Message "Sende Nachricht: $Title"

    for ($attempt = 1; $attempt -le $MaxRetries; $attempt++) {
        try {
            Log-Message "Versuch $attempt/$MaxRetries"

            # Check: explorer.exe läuft?
            $explorer = Get-Process -Name explorer -ErrorAction SilentlyContinue
            if (-not $explorer) {
                Log-Message "explorer.exe nicht gefunden, warte..." -Level Warning
                Start-Sleep -Milliseconds $RetryDelayMs
                continue
            }

            # Sende Nachricht
            $response = Send-MessageToUser -Title $Title -Message $Message

            Log-Message "Nachricht erfolgreich, Response: $response"
            return $response
        }
        catch {
            Log-Message "Fehler: $_" -Level Error
            if ($attempt -lt $MaxRetries) {
                Start-Sleep -Milliseconds $RetryDelayMs
            }
        }
    }

    Log-Message "Nach $MaxRetries Versuchen fehlgeschlagen" -Level Error
    return -1
}

# Verwendung
Send-MessageToUserSafe -Title "Test" -Message "OK?" -MaxRetries 5
```

---

Siehe auch: [Lösungen](Loesungen.md) | [Szenarien](Szenarien.md) | [Troubleshooting](Troubleshooting.md)
