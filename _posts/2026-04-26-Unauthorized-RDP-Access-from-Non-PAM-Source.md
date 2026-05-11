---
title: Unauthorized RDP Access from Non-PAM Source
description: Potentially RDP with Logon type 10 or 3 from Non-PAM IP 
author: abdi
date: 2026-05-12 00:05
categories: [Detection Engineering]
tags: [Windows Event Log, RDP, Logon Type 10, Logon Type 3, SIEM]
pin: true
math: true
mermaid: true
---
### Sedikit Cerita dari saya 😆😆 : 

<p style="text-align: justify;">
	Sebagai security analyst, tentunya melihat beragam macam log menjadi konsumsi sehari hari. Sehingga analisa yang tepat menjadi kunci dalam efisiensi monitoring. Secara umum, monitoring tersebut dilihat melalui 1 dashboard utama yang dikenal dengan SIEM (sistem integrasi event monitoring). Iyaa, beragam log terkumpul menjadi satu dalam perangkat dashboard tersebut. Salah satu penyumbang terbanyak log di SIEM adalah log enpoint.Hal ini senada dengan jumlah endpoint yang banyak di integrasikan maka akan banyak pula log yang ditimbulkan.
	<br>
	Salah satu yang menarik dibahas adalah log authentication di environment endpoint Windows. Dimana saat melakukan review terhadap aktivitas remote access, umumnya login RDP akan tercatat sebagai Logon Type 10 atau RemoteInteractive. Karena itu, ketika ditemukan aktivitas yang terlihat seperti RDP namun muncul sebagai Logon Type 3, situasi tersebut langsung terasa tidak biasa dan memunculkan pertanyaan mengenai apa yang sebenarnya terjadi di balik aktivitas tersebut.
	<br>
	Umumnya, Logon Type 3 identik dengan network logon yang sangat sering muncul dalam aktivitas normal Windows, seperti SMB access, komunikasi antar host, maupun autentikasi service account. Hal ini membuat event dengan Logon Type 3 cenderung terlihat legitimate di antara banyaknya authentication event harian. Namun dalam beberapa kondisi tertentu, aktivitas yang berkaitan dengan remote access ternyata dapat menghasilkan pola log yang berbeda dari perilaku RDP pada umumnya, sehingga berpotensi menyulitkan proses identifikasi apabila hanya bergantung pada pendekatan detection standar. 
	<br>
	Dari perspektif detection engineering, kondisi ini menjadi menarik untuk dibahas karena membuka kemungkinan adanya aktivitas remote access yang “bersembunyi” di balik network logon biasa. Melalui tulisan ini, saya ingin membahas lebih lanjut mengenai fenomena RDP login dengan Logon Type 3, mulai dari perbedaan karakteristiknya dengan Logon Type 10, kemungkinan skenario yang menyebabkan pola tersebut muncul, hingga pendekatan dalam membangun use case detection untuk membantu mengidentifikasi aktivitas serupa secara lebih efektif. 
</p>
---
<p style="text-align: justify;">
	Bagian ini dimulai dengan apa sih itu RDP. RDP sendiri merupakan singkatan dari Remote Desktop Protocol, yaitu sebuah protokol milik Microsoft yang memungkinkan pengguna untuk melakukan akses dan mengontrol sebuah perangkat Windows dari jarak jauh melalui jaringan. Dengan menggunakan RDP, seorang user dapat melihat tampilan desktop, menjalankan aplikasi, hingga melakukan aktivitas administratif seolah-olah sedang berada langsung di depan perangkat tersebut.
	<br><br>
	<img src="posts/Picture21.png">
	<br><br>
	Dalam environment enterprise, penggunaan RDP sudah menjadi hal yang sangat umum, terutama untuk kebutuhan administrasi server, remote troubleshooting, maupun operasional harian tim IT. Karena sifatnya yang memberikan akses interaktif secara langsung ke sebuah sistem, aktivitas login melalui RDP biasanya akan tercatat di Windows Event Log sebagai Logon Type 10 atau RemoteInteractive. Tipe logon ini menjadi salah satu indikator penting yang sering digunakan analyst untuk mengidentifikasi aktivitas remote login pada endpoint maupun server Windows.
	<br>
	Namun dalam beberapa kasus, aktivitas yang berkaitan dengan remote access ternyata tidak selalu muncul menggunakan Logon Type 10. Pada kondisi tertentu, event authentication dapat tercatat sebagai Logon Type 3 yang umumnya diasosiasikan sebagai network logon. Perbedaan inilah yang kemudian menjadi menarik untuk dianalisis lebih lanjut, terutama dalam konteks monitoring dan pengembangan use case detection.
	<br>
	Disini kami mensimulasikan dimana, adanya aktivitas RDP dari komputer Kami ke arah komputer lainnya. Disini terlihat bawah kami memasukkan Kredential yang sesuai dan berhasil login pada komputer target. Pararel kami juga melakukan validasi menggunakan windows event viewer untuk mengetahui apakah dari aktivitas RDP login tersebut tercatat sebagai log.
	<br><br>
	<img src="posts/Picture22.png">
	<br><br>	
	Namun hasil dari validasi yang kami lakukan, mendapatkan bahwa aktivitas RDP tersebut tidak dinyatakan sebagai logon type 10 melainkan logon type 3. Temuan ini terlihat berbeda bila berdasarkan referensi awal yang menyebukan bhawa Aktivitas RDP akan selalu memunculkan logon type 10. Berdasarkan informasi yang dimuat pada <a href = 'https://www.cyberengage.org/post/log-analysis-it-s-not-about-knowing-it-s-about-correlating'>cyberengage.org</a> mengkonfirmasi bahwa temuan tersebut valid adanya. 
	<br><br>
	<img src="posts/Picture23.png">
	<br><br>
	Dimana untuk RDP tidak selalu muncul sebagai Logon Type 10 (Remote Interactive) karena pada sistem modern umumnya sudah menggunakan Network Level Authentication (NLA), di mana proses autentikasi dilakukan terlebih dahulu melalui mekanisme jaringan (Kerberos atau NTLM) sebelum sesi RDP benar-benar dibuat. Proses awal ini tercatat sebagai Logon Type 3 (Network Logon), sehingga sering kali yang terlihat di log hanyalah Type 3 tanpa diikuti Type 10. Oleh karena itu, mengasumsikan bahwa RDP selalu identik dengan Logon Type 10 adalah keliru, karena aktivitas RDP juga bisa muncul sebagai Type 3 (autentikasi awal).
	<br>
	Sehingga untuk mendeteksi berkaitan aktivitas RDP, kita sebagai security analys tidak bisa hanya berdasarkan eventid 4624/4625 spesifik logontype 10. Sehingga diperlukan beberapa eventid pendukung seperti eventid 4778/4779 (session RDP) ataupun core RDP log activity yaitu eventid 1149 (RDP authentication berhasil) 
	<br><br>
	<img src="posts/Picture24.png">
	<br><br>

	Sehingga dari validasi dan testing yang kami lakukan, mendapatkan hasil bahwa untuk mendeteksi aktivitas RDP session tidak bisa hanya mengandalkan logontype 10. perlu adanya korelasi logic yang diterapkan pada perimeter detection untuk membantu security analyst dalam memonitoring malicious activity. Berikut adalasah final usecase logic untuk mendeteksi potential RDP connection tersebut :
</p>

```yaml
title: Potential RDP Connection Activity Detection
id: potential-rdp-connection-activity
status: experimental
description: >
  Detects potential Remote Desktop Protocol (RDP) activity using multiple indicators including 
  direct RDP logon (Logon Type 10), Network Logon (Type 3) followed by session reconnect, 
  and Terminal Services session creation events. This helps identify RDP usage including 
  scenarios where Network Level Authentication (NLA) is enabled.
logsource:
  product: windows
detection:
  selection_logon_type_10:
    EventID:
      - 4624
      - 4625
    LogonType: 10

  selection_logon_type_3:
    EventID:
      - 4624
      - 4625
    LogonType: 3

  selection_reconnect:
    EventID: 4778

  selection_terminal_session:
    EventID: 21

  timeframe: 5m

  condition: selection_logon_type_10 
    or selection_terminal_session 
    or (selection_logon_type_3 and selection_reconnect)

fields:
  - EventID
  - LogonType
  - TargetUserName
  - IpAddress
  - WorkstationName
  - Computer

falsepositives:
  - Legitimate administrative RDP access
  - IT support remote sessions
  - Jump server or bastion host activity

level: medium
``` 
<p style="text-align: justify;"> 
Pada akhirnya, apa yang terlihat “normal” di dalam log belum tentu benar-benar normal. Aktivitas seperti RDP yang selama ini dianggap mudah dikenali melalui Logon Type 10, ternyata dapat bertransformasi menjadi sesuatu yang jauh lebih subtle dan sulit dideteksi ketika mekanisme seperti Network Level Authentication (NLA) diterapkan. Inilah yang menjadikan log bukan sekadar kumpulan event, melainkan sebuah cerita yang membutuhkan konteks dan korelasi untuk dapat dipahami secara utuh. 
<br>
Bagi seorang security analyst, tantangan utamanya bukan lagi sekadar melihat alert, tetapi memahami apa yang tidak terlihat secara eksplisit. Detection yang efektif bukan dibangun dari satu indikator tunggal, melainkan dari kemampuan untuk menghubungkan potongan-potongan kecil yang tampak tidak signifikan menjadi sebuah gambaran besar yang utuh. 
<br>
Karena di dunia security, seringkali yang berbahaya bukanlah aktivitas yang terlihat mencurigakan, tetapi aktivitas yang berhasil terlihat biasa saja. </p>