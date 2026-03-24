---
title: Simpel Investigation CVE 2022-30190
description: CVE 2022-30190
author: abdi
date: 2022-06-10 16:30
categories: [Security Report]
tags: [Lets Defends, CVE, CVE-2022-30190, Folina]
pin: true
math: true
mermaid: true
---

### Latar Belakang
<p style="text-align:justify">
Pada tanggal 27 mei 2022, Tim teknikal Nao_Sec mencoba menaganalisa dan menemukan suatu dokumen dalam format .doc yang tampak malicious. Dimana Dokumen tersebut terindikasi terunggah dari alamat IP Belarus. Kecurigaan ini kemudian di telurusi lebih lanjut dan pada tanggal 30 mei 2022, tepatnya pada hari senin Microsoft mengumumkan 
adanya indikasi Zero day dengan nama CVE- 2022- 30190.
<br>

<img src="https://miro.medium.com/max/1100/0*zo85pbmHQ1uRGf0o.webp" alt="">
<br>

Diketahui file berbahaya tersebut terindikasi merujuk kepada kode area 0438 yang terdapat pada negara Italia yaitu desa Follina. Sehingga  Kevin Beamount sebagai peneliti yang termasuk kedalam orang pertama dalam melakukan analisis exploit ini menamakan “follina”. 
<br>
Menurut penelitian yang dilakukan oleh Kevin Beamount, Terdapat  bebera versi Microsoft office yang terindikasi rentan terhadap exploit  tersebut. Versi Microsoft office tersebut dimulai dari tahun 2013,  2016,2019, 2021 serta beberapa versi Microsoft office 365 dan edisi pro  plus. Dan untuk versi windows juga meliputi Windows 7, Windows 8.1,  Windows 10, Windows 11, Windows Server 2008, Windows Server 2012,  Windows Server 2016, Windows Server 2019, dan Windows Server 2022. 
<br>
Melalui exploit follina, attacker akan dengan mudah mendapatkan remote code execussion (RCE) sehingga dapat dengan mudah mengambil alih device korban.  Awalnya Attacker mencoba mengirimkan email phising ke korbannya. Dimana email tersebut melampirkan dokumen word yang telah dimodifikasi sebelumnya (Malicious). Saat korban membuka dokumen word tersebut, secara otomatis dokumen word tersebut akan memanggil fungsi ms-msdt untuk menjalankan command yang diinginkan oleh attacker. 
<br>
Target utama dari attacker tersebut menyasar ke Microsoft Windows  Support Diagnostic Tool (MSDT). Sehingga tools MSDT yang semula dikira  tidak berbahaya, ternyata dapat menjadi jalan masuk para attacker. Mellaui MSDT Attacker akan dengan mudah melakukan proses remote code execussion. Terdapat beberapa sampel dari contoh serangan yang 
memanfaatkan zero day follina ini.  Terdapat suatu dokumen word yang bertemakan undangan untuk melakukan proses wawancara pada sputnk radio. Dimana dokuemen tersebut ditujukan khusus untuk pengguna di negara Russia.
</p>

![Desktop View](/posts/picture4.png){: width="972" height="589" }
_Full screen width and center alignment_

![Desktop View](/posts/picture5.png){: width="972" height="589" }
_Full screen width and center alignment_

### Monitoring Alert  

![Desktop View](/posts/picture6.png){: width="972" height="589" }
_Full screen width and center alignment_

<p style="text-align:justify">
Pada tanggal 2 juni 2022 sekitar jam 3.22 siang, Tim Internal SOC (Security Operation Center) menemukan alert yang terindikasi sebagai CVE2022-30190. Dimana alert tersebut terindikasi sebagai zero day, dan memiliki nilai severity Medium. Sehingga tim SOC diharuskan untuk segera mungkin menganalisa alert tersebut apakah false positif atau true positif. Diketahui alert tersebut memiliki eventid yaitu 123, source address 172.16.17.39.
<br>
Setelah mengetahui detail dari alert tersebut, Tim SOC segera melakukan proses selanjutnya. Dimana proses selanjutnya yaitu adalah detection. Pada proses detection tim SOC diharuskan melakukan proses deteksi secara lebih mendalam untuk mengetahui apakah benar event tersebut termasuk ancaman atau hanya kesalahan deteksi.  
</p>

### Detection Event   
<p style="text-align:justify">
Tahap detection dimulai dengan memerika file dokumen word yang terlampir dalam alert tersebut. Dimana file tersebut memiliki nama 052022-0438.doc dan hash 52945af1def85b171870b31fa4782e52. File tersebut berukuran 10.01 Kb. Pengecekkan awal dilakukan dengan menggunakan tools virus totals.  
<br>
Melihat data dari virus total, membuat tim SOC yakin bahwa file tersebut benar terindikasi CVE 2022-30190. Dimana sebanyak 38 dari 50 threat intelligence menyatakan bahwa file tersebut sebagai malicious dan terindikasi melakukan komunikasi terhadap ip public. 
</p>

![Desktop View](/posts/picture7.png){: width="972" height="589" }
_Full screen width and center alignment_

### Analysis Event   

<p style="text-align:justify">
Setelah berhasil dideteksi bahwa benar file tersebut terindikasi sebagai exploit Follina, Tim SOC segera malakukan proses selanjutnya yaitu Analysis event. Dimana pada proses analysis event ini juga akan dilakukan proses therat hunting, guna mengetahui lebih detail siapa yang menjadi threat actor dan targetnya. Langkah pertama tim SOC melakukan pengecekkan pada log manajemen serta endpoint security, guna mengetahui apakah file tersebut dikarantina atau tidak.
</p>

![Desktop View](/posts/picture8.png){: width="972" height="589" }
_Full screen width and center alignment_

