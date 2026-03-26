---
title: SigmaPredator Lab - CyberDefender
description: Detection Engineering
author: abdi
date: 2026-03-25 21:30
categories: [Write Up]
tags: [Detection Engineering, Sigma Rule, Windows Event Log, Sysmon, Powershell]
pin: true
math: true
mermaid: true
---

<p style="text-align: justify;">
Kalau kamu pernah kepikiran gimana rasanya jadi defender yang harus “ngejar” jejak attacker dari berbagai artefak log, CTF yang satu ini bener-bener menggambarkan itu. Beberapa waktu lalu aku nyobain challenge dari CyberDefenders, yaitu SigmaPredator Lab. Di sini kita nggak cuma disuruh baca log, tapi juga ditantang untuk bikin dan validasi Sigma rules sendiri—khususnya buat mendeteksi teknik event log clearing yang sering dipakai attacker buat nutup jejak.
<br>
Yang menarik, scenarionya nggak cuma dari satu sudut aja. Kita harus melihat berbagai jalur eksekusi—mulai dari command line (CLI), WMI, sampai PowerShell. Jadi kerasa banget kayak lagi investigasi beneran, di mana attacker bisa pakai banyak cara buat achieve tujuan yang sama: defense evasion.
<br>
Di tulisan ini aku mau sharing gimana proses aku ngerjain lab ini—mulai dari eksplorasi log, trial and error bikin Sigma rules, sampai validasi pakai Chainsaw. Santai aja, anggap kita lagi diskusi ringan tapi tetap ngopi ☕.
</p>

---

### Sedikit Cerita mengenai CTF kali ini yaitu : 

<p style="text-align: justify;">
Within the CyberPredator Enterprise, you operate as a Detection & Hunting Engineer, translating adversary TTP research into operational detection capabilities. This workflow focuses on an in-depth analysis of Indicator Removal—specifically, Windows Event Log Clearing (T1070.001) as observed in campaigns attributed to APT28, APT41, and Aquatic Panda, aimed at degrading investigative visibility.
<br>
Your assignment is to deconstruct the Windows Event Log Clearing tradecraft—mapping associated tools, impacted event channels, and residual forensic traces. Following this, you will identify high-fidelity telemetry sources, craft resilient Sigma detection rules, and validate their efficacy against historical datasets using chainsaw, iteratively tuning detection logic before staging and production deployment.
<br>
nahh dari cerita itu terbagi menjadi 2 section yaitu Tecnique Research dan Rule Deployment.
</p>

### Tecnique Research 
#### 1. Soal Pertama,

> Process Creation Detection: Which built-in Windows command-line utility provides native support for managing—and in particular clearing—event logs?
{: .prompt-info }

<p style="text-align: justify;">
Dalam soal pertama ini, kita sebagai player diminta untuk mencari tahu mengenai bagaimana Mendeteksi dari process creation melalui built-in Windows commandline utility yang bertujuan untuk mengelola dan menghapus log event yang tersedia secara native. 
<br>
hmm, dari pertanyaan ini cukup memancing rasa penasaran hehe. Disini kita diharuskan untuk melakukan research baik melalui bantuan dari internet (Namun jangan chat GPT ya) 😄. 
<br>
Ok berdasarkan informasi dari hasil research, terdapat beberapa built-in windows commandline seperti yang dapat dilihat pada gambar dibawah ini : 
<br><br>
<img src="/posts/picture9.png">
<br><br>
Namun yang mendukung aktivitas mengelola dan menghapus log event tersebut adalah wevtutil. Dimana terdapat beberapa syntax yang umum digunakan seperti Clear-log, Export-log serta export-log.
</p>

#### 2. Soal kedua,

> Process Creation Detection: Which WMI class typically appears in logs when attackers use WMIC commands to clear critical Windows event logs via the command line?
{: .prompt-info }

<p style="text-align: justify;">
Selanjutnya, kita diminta untuk mengetahui Class dari WMI (Windows Management Instrumentation) yang umum digunakan oleh attacker dalam melakukan malicious activity dan muncul disi log monitoring via Windows agent. Pertanyaan ini sangat bagus untuk membantu kita dalam mengetahui, Parameter commandline apa yang sering digunakan oleh threat actor terlebih spesifik menggunakan WMI. 
<br><br>
<img src="/posts/picture10.png">
<br><br>
Kami mengulangi langkah sebelumnya, dimana kembali melakukan research melalui google sampai kami mendapati informasi yang relevan. Umumnya threat actor akan menjalankan commandline sperti 
</p>

```bash
wmic nteventlog where LogFileName='Security' call ClearEventLog
```

<p style="text-align: justify;">
Dimana commmandline tersebut berisikan parameter methode cleareventlog() dan class yang diakses adalah Win32_NTEventLogFile. 
</p>

#### 3. Soal ketiga,

> PowerShell Detection: Which PowerShell logging channel must be enabled to capture this technique’s execution, and which Event ID should be monitored as recommended in the MITRE ATT&CK technique page?
{: .prompt-info }

<p style="text-align: justify;">
Selanjutnya pertanyaan mengenai Powershel logging channel mana yang harus diaktifkan dari sisi Defender untuk dapat memonitoring terkait dengan process execution dan sebutkan juga nama eventid terkait. Dari sini sudah mulai membahas mengenai logging log serta event id. Jujur Pengetahuan eventid bagi sisi defender akan sangat menguntungkan karena akan membuat efisiensi dalam process detection.
<br>
Berdasarkan informasi yang dimuat pada gambar gambar diras, terlihat bahwa untuk memonitoring berakitan dengan process execution, defender dapat memanfaatkan logging script block di powershell. Dimana melalui logging tersebut memungkinkan defender mengetahui adanya malicious process berdasarkan eventID 4104.
<br><br>
<img src="/posts/picture11.png">
<br><br>
</p>

#### 4. Soal keempat,

> PowerShell detection: Which built-in PowerShell cmdlets are commonly leveraged by attackers to clear Windows Event Logs?
{: .prompt-info }

<p style="text-align: justify;">
Disini lebih mendalam untuk membahas mengenai buil-tin Cmdlet powershell mana yang umum digunakan oleh attacker dalam melakukan penghapusan log event disisi windows.
<br><br>
<img src="/posts/picture12.png">
<br><br>
<img src="/posts/picture13.png">
<br><br>
Berdasarkan hasil informasi yang kami baca seperti pada gambar diatas, terdapat 2 cmdlet yang dapat disalahgunakan oleh attacker dalam melakukan penghapusan log windows event. dimana 2 cmdlet tersebut adala clear event log dan remove event log.
</p>

#### 5. Soal kelima,

> PowerShell detection: Which native .NET API methods in the System.Diagnostics.Eventing.Reader and System.Diagnostics namespaces can an attacker invoke from PowerShell to clear Windows Event Logs?
{: .prompt-info }

<p style="text-align: justify;">
Soal ini memberitahukan defender bahwa, attacker tidak selalu bergantung terhadap penggunaan cmdlet seperti Clear-EventLog. Dimana attacker juga dapat secara langsung memanfaatkan .NET API untuk melakukan penghapusan Event log.
<br>
Berdasarkan informasi yang kami daptkan terdapat 2 metode yang dapat dimanfaatkan oleh attacker yaitu :
</p>

##### NameSpace: System.Diagnostics.Evening.Reader

```bash
EventLogSession.ClearLog()
```
Contoh untuk di Powershell 

```bash
([System.Diagnostics.Eventing.Reader.EventLogSession]::GlobalSession).ClearLog("Security")
```

##### Namespace: System.Diagnostics

```bash
EventLog.Clear()
```
Contoh untuk di powershell 

```bash
$log = New-Object System.Diagnostics.EventLog("Security")
$log.Clear()
``` 
<p style="text-align: justify;">
<img src="/posts/picture14.png">
<br><br>
<img src="/posts/picture15.png">
<br><br>

Sehingga dari informasi yang kami dapatkan tersebut, maka jawaban untuk sola ke lima ini adalah EventLogSession.ClearLog & EventLog.Clear .
</p>

#### 6. Soal keenam,

> File Deletion Detection:  Which Windows Event IDs should be monitored to detect the clearing of System or Security event logs as an indication of potential log removal?
{: .prompt-info }

<p style="text-align: justify;">
Untuk soal ini, kita sebagai defender diminta untuk memahami Eventid berapa yang dapat dimanfaatkan dalam memonitoring berkaitan dengan bukti penghapusan event log berhasil di windows. Disini kami mendapati bahwa eventID yang dapat dimanfaatkan adalah 1102 seeperti pada gambar dibawah ini. 
<br>
<img src="/posts/picture16.png">
</p>

```bash
- <Event xmlns="http://schemas.microsoft.com/win/2004/08/events/event">
- <System>
 <Provider Name="Microsoft-Windows-Eventlog" Guid="{fc65ddd8-d6ef-4962-83d5-6e5cfe9ce148}" /> 
 <EventID>1102</EventID> 
 <Version>0</Version> 
 <Level>4</Level> 
 <Task>104</Task> 
 <Opcode>0</Opcode> 
 <Keywords>0x4020000000000000</Keywords> 
 <TimeCreated SystemTime="2015-10-16T00:39:58.656871200Z" /> 
 <EventRecordID>1087729</EventRecordID> 
 <Correlation /> 
 <Execution ProcessID="820" ThreadID="2644" /> 
 <Channel>Security</Channel> 
 <Computer>DC01.contoso.local</Computer> 
 <Security /> 
 </System>
- <UserData>
- <LogFileCleared xmlns="http://manifests.microsoft.com/win/2004/08/windows/eventlog">
 <SubjectUserSid>S-1-5-21-3457937927-2839227994-823803824-1104</SubjectUserSid> 
 <SubjectUserName>dadmin</SubjectUserName> 
 <SubjectDomainName>CONTOSO</SubjectDomainName> 
 <SubjectLogonId>0x55cd1d</SubjectLogonId> 
 </LogFileCleared>
 </UserData>
 </Event>
```

### Rule Development (Section 2)

#### 1. Soal Pertama,

> PowerShell Detection: Create a Sigma rule to detect event log clearing through ScriptBlock Logging. Validate it using Chainsaw with your historical logs. What is the timestamp of the earliest detection? <br><br>
Note: PowerShell is operating in Constrained Language Mode, so your Sigma rule should disregard any .NET binaries.
{: .prompt-info }

<p style="text-align: justify;">
Saatnya kita memasukin bagian practical. Dimana pada bagian ini, kita sebagai defender diminta untuk membuat sigma Rule untuk mendeteksi adanya aktivitas penghapusan log menggunakan scriptblock logging. untuk mendeteksinya, kita diberikan tools chainsaw serta historical data event log yang dapat diaksess.
<br><br>
<img src="/posts/picture17.png">
<br><br>
Disini kami membuat sigma rule yang dapat dilihat seperti dibawah ini : 
</p>

```yaml
title: PowerShell Event Log Clearing via ScriptBlock
id: powershell-eventlog-clearing
status: experimental
description: Detects clearing of Windows Event Logs via PowerShell ScriptBlock Logging
logsource:
  product: windows
  service: powershell
detection:
  selection:
    EventID: 4104
    ScriptBlockText|contains:
      - 'Clear-EventLog'
      - 'Remove-EventLog'
      - 'wevtutil cl'
  condition: selection
fields:
  - ScriptBlockText
  - Computer
  - UserID
falsepositives:
  - Administrative activity
level: high
``` 
<p style="text-align: justify;">
	Terlihat bahwa dari sigma rule tersebut berasal dari windows powershell log dan spesifik berhubungan dengan event id 4104. Disini pencarian berdasarkan beberapa strings yang terdapat dalam scriptblocktext tersebut seperti 
	<br>
	<li>Clear-EventLog</li>
	<li>Remove-EventLog</li>
	<li>Wevtutil cl</li>
	<br>
Dari hasil pencarian menggunakan tools dan sigma rule yang sudah dibuat seperti diatas tadi. kami mendapati jawabanya seperti pada gambar dibawah ini.

<img src="/posts/picture18.png">
	<br>
</p>


#### 2. Soal Kedua,

> Process Creation Detection: Create a Sigma rule to detect event log clearing via native CLI utilities (e.g., wevtutil, excluding WMI). Validate it using Chainsaw with the historical logs you have. What’s the timestamp of the latest match?
<br><br>
Note: You can use Sysmon as a data source in your rule.
{: .prompt-info }

<p style="text-align: justify;">
Selanjutnya kita diminta untuk mendeteksi adanya aktivitas penghapusan log via CLI dan untuk log sourcenya bisa memanfaatkan sysmon melalui process creation yaitu dengan event id 1. Perlu di perhatikan untuk pertanyaaa ini, kita diminta untuk menemukan timestamp paling akhir.
<br><br>
Oh iyaa, sebelum terlalu jauh. Kembali kita ingat bahwa process penghapusann log event melalui cli bisa menggunakan built-in powershell windows utility yaitu wevtutil
</p>

```bash
wevtutil cl System
wevtutil cl Security
````
<p style="text-align: justify;">
Sehingga dari hipotesa yang sudah kita dapatkan diawal tadi, menjadikan landasan sigma rule yang akan kita buat seperti ini
</p>

```yaml
title: Wevtutil Log Clearing Sysmon
id: 123
status: experimental
description: Detect clearing event logs using wevtutil
logsource:
  product: windows
  service: sysmon
detection:
  selection:
    EventID: 1
    CommandLine|contains: wevtutil
  condition: selection
fields:
  - CommandLine
  - Image
falsepositives:
  - Administrative activity
level: medium
```

<p style="text-align: justify;">
	Dimana dari hasil penggunaan sigma rule berdasarkan log sysmon dan spesifik eventid 1, berhasil membantu kita untuk mengetahui adanya malicious commandline yang berhubungan dengan penghapusan log.
<img src="/posts/picture19.png">
</p>

#### 3. Soal Ketiga,

> File Deletion Detection:  Create a Sigma rule to detect event-log clearing. Validate it using Chainsaw with the historical logs you have. What is the timestamp of the earliest match?
{: .prompt-info }

<p style="text-align: justify;">
Sebelumnya kita diminta untuk mendeteksi adanya percobaan penghapusan log melalui cli, sedangkan pada soal terkahir ini kita sebagai defender diminta untuk mendeteksi adanya bukti valid bahwa log windows event sudah benar benar terhapus.
<br>
Berdasarkan hasil research kami sebelumnya untuk membuktikan adanya penghapusan log windows event tersebut sudah berhasil dilakukan dapat memanfaatkan eventID 1102 (Security log cleared) dan 104 (Event log service cleared log), sehingg sigma rule yang kami bikin menjadi seperti ini 
</p>

```yaml
title: Windows Event Log Cleared
id: log-cleared-104-1102
status: experimental
description: Detects clearing of Windows event logs
logsource:
  product: windows
detection:
  selection1:
    EventID: 1102
  selection2:
    EventID: 104
  condition: selection1 or selection2
fields:
  - EventID
  - Computer
falsepositives:
  - Legitimate admin activity
level: medium
```
<img src="/posts/picture20.png">

<p style="text-align: justify;"> 
Akhirnya, melalui dari lab ini kita sebagai defender dapat mengetahui dan mendeteksi aktivitas seperti penghapusan event log  bukan cuma soal tahu satu teknik, tapi bagaimana kita memahami berbagai cara attacker bekerja dan menerjemahkannya ke dalam detection yang efektif. Semoga tulisan ini bisa jadi gambaran sekaligus referensi buat kita semua yang lagi belajar ataupun mendalami detection engineering. Kalau ada pendekatan lain ataupun insight tambahan, feel free buat sharing juga ya karena di dunia blue team, belajar itu nggak pernah berhenti 🚀. 
</p>

### Reference

- https://learn.microsoft.com/en-us/answers/questions/342398/windows-event-logs-clea
- https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-1102
- https://learn.microsoft.com/en-us/dotnet/api/system.diagnostics.eventlog.clear?view=windowsdesktop-10.0
- https://learn.microsoft.com/en-us/dotnet/api/system.diagnostics.eventing.reader.eventlogsession.clearlog?view=windowsdesktop-10.0
- https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_logging?view=powershell-5.1
- https://learn.microsoft.com/en-us/previous-versions/windows/desktop/eventlogprov/cleareventlog-method-in-class-win32-nteventlogfile
- https://github.com/WithSecureLabs/chainsaw