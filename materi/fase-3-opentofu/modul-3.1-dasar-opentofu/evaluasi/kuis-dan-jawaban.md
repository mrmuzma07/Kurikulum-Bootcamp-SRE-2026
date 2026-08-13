# Kuis dan Jawaban — Modul 3.1 Dasar OpenTofu

> **Target:** minimal 80%. Jangan memakai credential nyata untuk menjawab atau menguji command.

## Soal Pilihan Ganda

### 1. Apa fokus utama pendekatan declarative?

A. Menentukan setiap urutan command secara manual  
B. Menjelaskan keadaan akhir yang diinginkan  
C. Menghapus state sebelum setiap apply  
D. Menjalankan command dengan `-auto-approve`

### 2. Mana yang paling tepat menggambarkan state?

A. Backup database aplikasi  
B. Satu-satunya sumber kebenaran provider  
C. Mapping address resource ke object dan atribut yang diketahui OpenTofu  
D. File konfigurasi yang selalu aman dibagikan

### 3. Idempotency berarti…

A. Apply selalu membuat resource baru  
B. Input dan kondisi sama menghasilkan keadaan akhir yang sama saat diulang  
C. State boleh dihapus setiap kali plan  
D. Provider tidak pernah mengalami drift

### 4. Mengapa image `latest` kurang cocok untuk reproduksi incident?

A. Selalu hanya tersedia pada amd64  
B. Tag dapat menunjuk content berbeda dari waktu ke waktu  
C. OpenTofu tidak dapat membaca string  
D. Image tidak dapat disimpan dalam state

### 5. Pernyataan OpenTofu vs Terraform yang paling tepat adalah…

A. Semua versi, provider, dan backend pasti kompatibel  
B. Keduanya memiliki konsep mirip, tetapi kompatibilitas harus diverifikasi per versi/provider/backend  
C. Terraform tidak memakai HCL  
D. OpenTofu menghilangkan kebutuhan state

### 6. Fungsi provider adalah…

A. Menyimpan password pada README  
B. Menerjemahkan resource HCL ke API eksternal  
C. Menggantikan monitoring service  
D. Menghapus semua resource di luar state

### 7. Apa fungsi `docker_image.web.image_id` pada container?

A. Membuat dependency implicit terhadap image  
B. Mengaktifkan backend remote  
C. Menyembunyikan port  
D. Menghapus image saat plan

### 8. Variable validation berguna untuk…

A. Mengenkripsi state  
B. Menolak input yang melanggar kontrak module  
C. Membuat provider tidak perlu di-init  
D. Menghindari semua drift

### 9. `sensitive = true` pada output…

A. Mengenkripsi state secara otomatis  
B. Menghapus value dari provider  
C. Menyamarkan presentasi, tetapi value dapat tetap berada dalam state  
D. Membuat credential aman untuk di-commit

### 10. Urutan workflow yang paling tepat adalah…

A. apply → init → plan → validate  
B. init → fmt → validate → plan → review → apply  
C. destroy → state rm → apply  
D. import → auto-approve → delete state

### 11. Apa yang harus dilakukan sebelum apply?

A. Mengabaikan semua `destroy` pada plan  
B. Membaca resource address, action, scope, dan blast radius  
C. Menjalankan `state rm` agar plan kosong  
D. Menghapus lock

### 12. Remote state memberikan manfaat utama berupa…

A. Tidak membutuhkan permission  
B. Collaboration, locking, dan backup terpusat bila dikonfigurasi dengan benar  
C. Enkripsi otomatis semua output pada terminal  
D. Penghapusan kebutuhan provider API

### 13. Apa risiko dua apply terhadap key state yang sama tanpa locking?

A. Hanya output yang berubah  
B. Lost update dan state/resource tidak konsisten  
C. Image otomatis menjadi immutable  
D. Tidak ada risiko karena plan selalu identik

### 14. `tofu import`…

A. Menghapus object manual  
B. Mengadopsi object provider ke address state, tetapi tidak otomatis menulis HCL lengkap  
C. Memindahkan resource ke module tanpa plan  
D. Mengganti semua resource dalam backend

### 15. `tofu state mv` terutama digunakan untuk…

A. Mengubah address mapping tanpa memindahkan object provider  
B. Menghapus object provider  
C. Mengubah image tag  
D. Membuat bucket S3

### 16. `tofu state rm` biasanya…

A. Menghapus mapping state dan dapat meninggalkan object provider unmanaged  
B. Selalu menghapus object provider dengan aman  
C. Mem-backup state otomatis ke Git  
D. Menyelesaikan semua drift

### 17. Jika state lock error muncul, tindakan pertama yang tepat adalah…

A. Menghapus lock object manual  
B. Force unlock dengan ID acak  
C. Memeriksa apakah ada writer aktif dan mengikuti prosedur backend  
D. Menjalankan destroy

### 18. Drift adalah…

A. Semua perubahan pada HCL  
B. Perbedaan actual infrastructure dengan desired/state yang diharapkan  
C. Provider yang berhasil di-init  
D. Output yang ditandai sensitive

### 19. Mengapa `.gitignore` tidak cukup untuk secret safety?

A. Git tidak mengenal file tfvars  
B. File dapat sudah ter-commit, tercopy, masuk backup/log, atau state memang berada di luar Git  
C. `.gitignore` menghapus credential dari process list  
D. State selalu kosong

### 20. Handoff OpenTofu → Ansible → k3s yang tepat adalah…

A. OpenTofu menyimpan password SSH dalam output lalu Ansible menyalinnya ke Git  
B. OpenTofu menyediakan metadata provisioning; Ansible mengonfigurasi OS; k3s dipasang setelah prasyarat host siap  
C. Ansible menggantikan provider dan state OpenTofu  
D. k3s harus dipasang sebelum host diprovision

## Soal Esai Singkat

### 21. Jelaskan tiga kondisi `desired configuration`, `current infrastructure`, dan `state file`, serta cara plan membantu menemukan drift.

### 22. Tulis checklist aman untuk migrasi local state ke S3-compatible/MinIO.

### 23. Bedakan remediation drift dengan mengubah HCL, mengembalikan object provider, dan memakai `state rm`.

### 24. Jelaskan mengapa plan sukses dan apply sukses belum membuktikan service sehat.

### 25. Jelaskan guardrail untuk menjalankan `tofu destroy` pada lab.

## Kunci Jawaban Pilihan Ganda

| No. | Jawaban | Penjelasan singkat |
|---:|:---:|---|
| 1 | B | Declarative menyatakan hasil akhir yang diinginkan. |
| 2 | C | State menyimpan mapping dan atribut yang diketahui OpenTofu. |
| 3 | B | Pengulangan dengan input/kondisi sama menuju keadaan akhir sama. |
| 4 | B | Mutable tag dapat menunjuk content berbeda. |
| 5 | B | Konsep mirip, kompatibilitas tidak boleh diasumsikan. |
| 6 | B | Provider adalah boundary ke API eksternal. |
| 7 | A | Referensi resource membentuk dependency implicit. |
| 8 | B | Validation membatasi input invalid. |
| 9 | C | Sensitive menyamarkan tampilan, bukan enkripsi state. |
| 10 | B | Init, format/validasi, plan, review, lalu apply. |
| 11 | B | Action dan blast radius harus dipahami sebelum apply. |
| 12 | B | Remote backend mendukung kolaborasi/locking/backup bila dikonfigurasi. |
| 13 | B | Concurrent writer dapat menimbulkan lost update/inconsistency. |
| 14 | B | Import mengadopsi state; konfigurasi tetap harus ditulis. |
| 15 | A | `state mv` mengubah address mapping. |
| 16 | A | Mapping dihapus; object dapat menjadi unmanaged. |
| 17 | C | Verifikasi writer dan ikuti prosedur lock. |
| 18 | B | Drift adalah perbedaan current dengan desired/state expectation. |
| 19 | B | Ignore bukan encryption atau penghapus jejak data. |
| 20 | B | Boundary provisioning/configuration/cluster harus jelas. |

## Panduan Jawaban Esai

### 21. Tiga kondisi dan drift

`desired configuration` adalah HCL dan input yang menyatakan target. `current infrastructure` adalah object nyata pada provider. `state file` adalah mapping dan atribut terakhir yang diketahui OpenTofu. Plan membaca configuration/state lalu refresh melalui provider untuk menampilkan perubahan yang diperlukan. Jika perubahan manual tidak berada pada schema/boundary yang dikelola, plan dapat tetap kosong; hal itu harus dijelaskan, bukan dianggap bukti global.

### 22. Migrasi state

Jawaban minimal memuat: verifikasi directory/backend/workspace; hentikan writer dan pahami lock; backup local state serta catat checksum/metadata; siapkan bucket/key/permission remote; pastikan credential melalui environment/credential helper; jalankan `tofu init` dan review migration prompt; inspeksi remote state; jalankan plan; uji recovery; simpan backup sesuai retention. Tidak boleh copy/merge raw state manual tanpa prosedur.

### 23. Pilihan remediation

Ubah HCL bila perubahan manual sah dan ingin menjadi desired state. Kembalikan object provider bila perubahan tidak disetujui dan plan menunjukkan scope aman. `state rm` hanya melepas ownership mapping, bukan remediasi default; object dapat menjadi unmanaged sehingga harus ada owner/cleanup plan, backup, lock, dan approval.

### 24. Plan/apply vs health

Plan hanya menghitung perubahan berdasarkan state/config/provider. Apply hanya menunjukkan provider menerima operasi. Network, process, HTTP response, dependency, metrics, data integrity, dan SLO masih harus divalidasi dengan runtime checks/observability.

### 25. Destroy lab

Verifikasi directory dan backend/workspace; pastikan state hanya resource disposable; backup data yang dibutuhkan; buat `tofu plan -destroy -out=destroy.tfplan`; baca setiap address/action; minta approval; apply plan yang sama; validasi state dan provider; cleanup sisa hanya bila owner/tag terverifikasi. Jangan memakai `-auto-approve` sebagai default dan jangan menjalankan destroy pada scope yang tidak jelas.

## Skoring

- Pilihan ganda: 20 soal × 3 poin = 60.
- Esai: 5 soal × 8 poin = 40.
- Total: 100.
- Lulus minimal 80.
- Pelanggaran secret safety atau operasi destructive tanpa scope/approval dapat menggugurkan submission meskipun skor angka mencapai 80.
