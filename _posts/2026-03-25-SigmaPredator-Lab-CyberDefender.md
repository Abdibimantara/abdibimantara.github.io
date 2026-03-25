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

<img src="https://github.com/user-attachments/assets/f3ad6baa-113f-4eb3-8a4e-336964a9fe7f">


Namun yang mendukung aktivitas mengelola dan menghapus log event tersebut adalah wevtutil. Dimana terdapat beberapa syntax yang umum digunakan seperti Clear-log, Export-log serta export-log.

2. Selanjutnya, kita diminta untuk mengetahui Class dari WMI (Windows Management Instrumentation) yang umum digunakan oleh attacker dalam melakukan malicious activity dan muncul disi log monitoring via Windows agent.

Pertanyaan ini sangat bagus untuk membantu kita dalam mengetahui, Parameter commandline apa yang sering digunakan oleh threat actor terlebih spesifik menggunakan WMI. Kami mengulangi langkah sebelumnya, dimana kembali melakukan research melalui google sampai kami mendapati informasi yang relevan.

Umumnya threat actor akan menjalankan commandline sperti 
wmic nteventlog where LogFileName='Security' call ClearEventLog
Dimana commmandlin tersebut berisikan parameter methode cleareventlog() dan class yang diakses adalah Win32_NTEventLogFile. 

https://learn.microsoft.com/en-us/previous-versions/windows/desktop/eventlogprov/cleareventlog-method-in-class-win32-nteventlogfile

3. Selanjutnya pertanyaan mengenai Powershel logging channel mana yang harus diaktifkan dari sisi Defender untuk dapat memonitoring terkait dengan process execution dan sebutkan juga nama eventid terkait.

Dari sini sudah mulai membahas mengenai logging log serta event id. Jujur Pengetahuan eventid bagi sisi defender akan sangat menguntungkan karena akan membuat efisiensi dalam process detection.




4. Disin lebih mendalam untuk membahas mengenai buil-tin Cmdlet powershell mana yang umum digunakan oleh attacker dalam melakukan penghapusan log event disisi windows.

https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/clear-eventlog?view=powershell-5.1

Berdasrkan hasil informasi yang kami baca melalui url diatas, terdapat 2 cmdlet yang dapat disalahgunakan oleh attacker dalam melakukan penghapusan log windows event. dimana 2 cmdlet tersebut adala clear event log dan remove event log.

