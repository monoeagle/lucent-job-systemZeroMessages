# Troubleshooting & FAQ

Häufige Probleme und deren Lösungen.

## Dialog erscheint nicht / Fenster ist unsichtbar

### Symptom
Scheduled Task läuft, aber kein Dialog zu sehen. Skript scheint zu "hängen".

### Ursachen & Lösungen

| Ursache | Lösung |
|--------|--------|
| Task läuft VOR Benutzer-Anmeldung | Trigger auf "Bei Benutzer-Anmeldung" oder Startup + 2-3 Min Verzögerung |
| Task läuft im falschen Kontext | "Höchste Berechtigungen" aktivieren, Account = SYSTEM |
| Explorer.exe läuft nicht | Benutzer angemeldet? `Get-Process explorer` testen |
| Falsche Session-ID | Standardmäßig `SessionId=1` für Console, RDS = 2+ |
| Firewall/Windows Defender | API-Aufrufe blockiert (selten) |

### Diagnostic-Befehle

```powershell
# Prüfe: Benutzer angemeldet?
Get-Process explorer

# Prüfe: Welche Sessions aktiv?
# (siehe Get-UserSessions in CodeBeispiele.md)

# Prüfe: Gibt es Fehler beim API-Aufruf?
# In Log-Datei schreiben:
Add-Content -Path "C:\Temp\debug.log" -Value "Dialog-Versuch um $(Get-Date)"
```

---

## ServiceUI.exe nicht gefunden

### Fehlermeldung
```
ServiceUI.exe: Datei nicht gefunden
Konnte C:\MDT\ServiceUI.exe nicht öffnen
```

### Lösungen

**Option 1: MDT installieren** (kostenlos)
```powershell
# Download von Microsoft
# Installer: MicrosoftDeploymentToolkit_x64.msi
# Danach: ServiceUI.exe liegt unter
# C:\Program Files\Microsoft Deployment Toolkit\Templates\ServiceUI\
```

**Option 2: ServiceUI.exe kopieren**
```powershell
# Von anderem System mit MDT kopieren
Copy-Item "\\AdminPC\C$\Program Files\Microsoft Deployment Toolkit\Templates\ServiceUI\ServiceUI.exe" `
    -Destination "C:\Tools\ServiceUI.exe"

# Im Task dann:
C:\Tools\ServiceUI.exe powershell.exe ...
```

**Option 3: PATH aktualisieren**
```powershell
$serviceUIPath = "C:\Program Files\Microsoft Deployment Toolkit\Templates\ServiceUI"
$env:Path += ";$serviceUIPath"

ServiceUI.exe powershell.exe ...
```

---

## CreateProcessAsUser: "Access Denied"

### Fehlermeldung
```
Exception: Access Denied beim Token-Handling
OpenProcessToken fehlgeschlagen
DuplicateTokenEx fehlgeschlagen
```

### Ursachen & Lösungen

1. **Nicht als Administrator**
   ```powershell
   # Task muss sein:
   # - User: SYSTEM
   # - Run Level: Highest
   ```

2. **Explorer.exe läuft unter anderem User**
   ```powershell
   # Prüfe Prozess-Owner
   Get-Process explorer | Select-Object -Property ProcessName, @{
       Name='Owner'
       Expression={$_.Owner}
   }
   ```

3. **Token-Leak (Handles nicht freigegeben)**
   ```powershell
   # Stelle sicher, dass CloseHandle() aufgerufen wird
   # (siehe CodeBeispiele.md)
   ```

---

## WTSSendMessage zeigt nichts

### Symptom
API wird aufgerufen, aber kein Dialog sichtbar.

### Ursachen

| Ursache | Check | Fix |
|--------|-------|-----|
| Keine aktive Session | `Get-UserSessions` | Warte auf Benutzer-Anmeldung |
| Falsche Session-ID | SessionId in WTSSendMessage | 1 = Console, 2+ = RDS |
| Session ist "Disconnected" | `Get-UserSessions \| State` | RDP-User hat sich abgemeldet |
| Timeout zu kurz | Setze `TimeoutSeconds=0` (unbegrenzt) | Erhöhe Timeout-Wert |

### Diagnostic-Skript

```powershell
function Test-WTSConnection {
    Add-Type -AssemblyName "System.Runtime.InteropServices" -ErrorAction SilentlyContinue

    $code = @'
using System;
using System.Runtime.InteropServices;

public class WTSTest {
    [DllImport("wtsapi32.dll", SetLastError=true)]
    public static extern IntPtr WTSOpenServer(string ServerName);
    [DllImport("wtsapi32.dll", SetLastError=true)]
    public static extern void WTSCloseServer(IntPtr hServer);
}
'@
    
    Add-Type -TypeDefinition $code -ErrorAction SilentlyContinue
    
    try {
        $hServer = [WTSTest]::WTSOpenServer("localhost")
        if ($hServer -eq [IntPtr]::Zero) {
            Write-Host "❌ WTSOpenServer fehlgeschlagen"
            return $false
        }
        
        [WTSTest]::WTSCloseServer($hServer)
        Write-Host "✅ WTS-Verbindung OK"
        return $true
    }
    catch {
        Write-Host "❌ Fehler: $_"
        return $false
    }
}

Test-WTSConnection
```

---

## Task "hängt" oder wartet auf Input

### Symptom
Task läuft, aber wird nicht fertig. Unter "Task Scheduler" wird Status "Running" angezeigt.

### Ursachen

1. **Skript wartet auf Benutzereingabe**
   ```powershell
   # Schlecht: Wartet auf Input, die nie kommt
   $input = Read-Host "Eingabe"
   
   # Besser: Mit Timeout
   # Nutze Send-MessageToUser statt Read-Host
   ```

2. **Dialog hat zu langen Timeout**
   ```powershell
   # Schlecht: Wartet 3600 Sekunden
   Send-MessageToUser -Title "Test" -TimeoutSeconds 3600
   
   # Besser: Vernünftiger Timeout
   Send-MessageToUser -Title "Test" -TimeoutSeconds 30
   ```

3. **Prozess terminiert nicht**
   ```powershell
   # Am Ende explizit beenden:
   exit 0
   ```

### Lösung: Task-Timeout setzen

```powershell
# Im Scheduled Task:
# Registrierungseditor → Task Properties → Settings
# "Stop the task if it runs longer than: 1 hour"
```

---

## Toast Notifications funktionieren nicht

### Fehlermeldung
```
WinRT API nicht verfügbar
User-Context erforderlich
```

### Problem
Toast Notifications können **nicht direkt** aus Session 0 (SYSTEM) aufgerufen werden.

### Lösung: Wrapper verwenden

```powershell
# Methode 1: ServiceUI
ServiceUI.exe powershell.exe -NoProfile -File C:\Scripts\ShowToast.ps1

# Methode 2: CreateProcessAsUser
Invoke-ProcessAsUser -ProcessPath "powershell.exe" -Arguments "-NoProfile -File C:\Scripts\ShowToast.ps1"

# ShowToast.ps1:
[Windows.UI.Notifications.ToastNotificationManager, Windows.UI.Notifications, ContentType = WindowsRuntime] > $null
# ... (Toast-Code)
```

---

## RDP: Welche Session wird angesprochen?

### Problem
Auf Terminal Server mehrere Benutzer angemeldet, aber Nachricht geht an falschen Benutzer.

### Lösung

**Console (lokaler Benutzer):** SessionId = 1
```powershell
Send-MessageToUser -SessionId 1 -Title "Für lokal angemeldeten Benutzer"
```

**Erste RDP-Session:** SessionId = 2
```powershell
Send-MessageToUser -SessionId 2 -Title "Für ersten RDP-Benutzer"
```

**Alle Sessions durchloopen:**
```powershell
$sessions = Get-UserSessions  # Siehe CodeBeispiele.md
foreach ($session in $sessions) {
    if ($session.State -eq 'Active') {
        Write-Host "Sende an Session $($session.SessionId)"
        Send-MessageToUser -SessionId $session.SessionId -Title "Test" -Message "Hallo"
    }
}
```

---

## Scheduled Task wird nicht ausgeführt

### Symptom
Task ist erstellt, aber "Last Run Result" = 0x0 (nicht gelaufen).

### Ursachen

1. **Trigger stimmt nicht**
   ```
   ❌ Trigger: "At 3 AM" (nachts, Benutzer nicht angemeldet)
   ✅ Trigger: "At log on" oder "At startup + 2 minutes"
   ```

2. **Account hat keine Rechte**
   ```
   ❌ Account: "Restricted User"
   ✅ Account: "SYSTEM" oder lokaler Administrator
   ```

3. **PowerShell Execution Policy**
   ```powershell
   # Im Task Argument hinzufügen:
   -ExecutionPolicy Bypass
   ```

4. **Skript-Pfad ist UNC-Share**
   ```
   ❌ \\fileserver\scripts\task.ps1 (nicht erreichbar im SYSTEM-Kontext)
   ✅ C:\Scripts\task.ps1 (lokal)
   ```

### Diagnostic

```powershell
# Logs prüfen
Get-WinEvent -LogName "Microsoft-Windows-TaskScheduler/Operational" `
    -FilterXPath "*[EventData[Data[@Name='TaskName'] and (Data='MyTask')]]" |
    Select-Object TimeCreated, Message
```

---

## XLSX-Export zeigt nur Raw-Code in GitHub

### Problem
Markdown-Datei wird auf GitHub aber raw angezeigt statt gerendert.

### Lösung: Richtige Datei-Extension
```
❌ Datei.txt (wird als Text angezeigt)
✅ Datei.md  (wird als Markdown gerendert)
```

---

## Performance: Script ist zu langsam

### Tipps

1. **Token-Operationen sind teuer**
   ```powershell
   # Schlecht: Einmal pro Nachricht
   for ($i=1; $i -le 100; $i++) {
       Invoke-ProcessAsUser -ProcessPath ...
   }
   
   # Besser: Batch-Operationen
   # Oder nutze WTSSendMessage (schneller)
   ```

2. **API-Aufrufe minimieren**
   ```powershell
   # Schlecht: Jedes Mal neues Handle öffnen
   $hServer = WTSOpenServer(...)
   WTSSendMessage(...)
   WTSCloseServer(...)
   
   # Besser: Handle wiederverwenden
   $hServer = WTSOpenServer(...)
   WTSSendMessage(..., $hServer, ...)
   WTSSendMessage(..., $hServer, ...)
   WTSCloseServer($hServer)
   ```

3. **Parallel-Processing**
   ```powershell
   # Multi-User Nachricht schneller
   $sessions | ForEach-Object -Parallel {
       Send-MessageToUser -SessionId $_.SessionId
   }
   ```

---

## Weitere Ressourcen

- [Microsoft WTS API Docs](https://learn.microsoft.com/en-us/windows/win32/api/wtsapi32/)
- [CreateProcessAsUser Reference](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createprocessasusera)
- [Session 0 Isolation Blog](https://www.alex-ionescu.com/)

---

Siehe auch: [Lösungen](Loesungen.md) | [Szenarien](Szenarien.md) | [Code-Beispiele](CodeBeispiele.md)
