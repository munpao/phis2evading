# AV/EDR Evasion & AMSI Bypass — Study Notes

> ⚠️ **Educational Purpose Only** — เอกสารนี้สร้างขึ้นเพื่อการเรียนรู้ด้าน offensive security ในสภาพแวดล้อมที่ได้รับอนุญาตเท่านั้น (authorized lab, CTF, certification study) ห้ามใช้กับระบบที่ไม่ได้รับอนุญาต

## 📌 เกี่ยวกับ Repo นี้

Repo นี้รวบรวม study notes และ reference materials ด้าน **AV/EDR evasion** และ **AMSI bypass** ที่ผมเขียนขึ้นระหว่างเตรียมสอบ OSCP/OSEP — เป็น portfolio สำหรับอ้างอิงและแสดงความเข้าใจด้าน offensive security

**Author:** [atipong](https://github.com/munpao)

---

## 📂 ไฟล์ในนี้

| ไฟล์ | เนื้อหา |
|---|---|
| [`rasta-hta-amsi-writeup.md`](rasta-hta-amsi-writeup.md) | **Lab Writeup** — วิเคราะห์ HTA dropper → AMSI bypass (Reflection) → hoaxshell kill chain จาก RastaLabs, decode ทุก obfuscation layer ทีละบรรทัด, MITRE ATT&CK mapping |

---

## 🎯 หัวข้อที่ครอบคลุม

### AMSI Bypass
- **AmsiScanBuffer Patching** — memory patching ให้ return clean เสมอ
- **Reflection-Based (amsiInitFailed)** — ใช้ .NET Reflection set internal flag
- **String Obfuscation** — concatenation, char array, base64, XOR encoding
- **PowerShell Downgrade** — ใช้ v2 ที่ไม่มี AMSI
- **CLM Bypass** — custom Runspace, InstallUtil, MSBuild

### EDR Evasion
- **User-Mode Hooking** — ทำไม EDR hook ntdll.dll
- **Direct Syscalls** — เรียก syscall โดยตรง ข้าม hooks
- **Hell's Gate / Halo's Gate / Tartarus' Gate** — dynamic SSN resolution
- **Full ntdll Unhooking** — overwrite hooked .text section ด้วย clean copy
- **PPID Spoofing** — fake parent process relationship
- **Process Injection** — Classic, Hollowing, APC, Module Stomping, Phantom DLL

### Lab Writeup (RastaLabs)
- **HTA Dropper** — VBScript + mshta.exe (LOLBIN) analysis
- **AMSI Bypass Stage** — Reflection + char array + string concatenation (de-obfuscate ทุกบรรทัด)
- **hoaxshell** — HTTP-based reverse shell mechanism
- **MITRE ATT&CK Mapping** — จับคู่ทุก step กับ technique ID
- **Blue Team Detection** — Event logs, Sysmon, Script Block Logging

---

## 🛠️ Tools ที่อ้างถึง

| Category | Tools |
|---|---|
| AMSI Testing | AmsiTrigger, PSAmsi, Invoke-Obfuscation, Chimera |
| Payload Generation | ScareCrow, Nimcrypt2, Freeze, Donut |
| C2 Frameworks | hoaxshell, Sliver, Havoc |
| EDR Evasion | SharpBlock, BokuLoader, direct syscall implementations |

---

## 📖 วิธีใช้

เอกสารเหล่านี้เป็น **study reference** — ไม่ใช่ ready-to-use exploit scripts
1. ทดลองใน **authorized lab environment** เท่านั้น (HTB,local VM)

---

## ⚖️ Disclaimer

- เนื้อหาทั้งหมดเป็นไปเพื่อ **การศึกษาและป้องกัน** (educational & defensive purpose)
- ห้ามนำไปใช้กับระบบที่ไม่ได้รับอนุญาต
- ผู้เขียนไม่รับผิดชอบต่อการนำไปใช้ในทางที่ผิด
- Code ตัวอย่างเป็น pseudocode / concept — ไม่ใช่ working exploit สำเร็จรูป
- ดูรายละเอียดใน [LICENSE](LICENSE)

---

## 📚 References

- [MITRE ATT&CK — Defense Evasion (TA0005)](https://attack.mitre.org/tactics/TA0005/)
- [Microsoft AMSI Documentation](https://learn.microsoft.com/en-us/windows/win32/amsi/)
- [Elastic Security Labs — Blinding EDR On Windows](https://www.elastic.co/security-labs)
- [ired.team — Red Team Notes](https://www.ired.team/)

---

## 📝 License

This project is licensed under the MIT License — see [LICENSE](LICENSE) for details.

---

*สร้างขึ้นระหว่างเตรียมสอบ OSCP/OSEP — [munpao](https://github.com/munpao)*
