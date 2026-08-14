# Kuis dan Jawaban — Modul 3.2 Modul & Pola Produksi

> **Target:** minimal 80%. Jangan memakai credential nyata untuk menjawab atau menguji command.

## Soal Pilihan Ganda

### 1. Apa peran utama child module?

A. Memilih production backend secara otomatis  
B. Mengemas capability/resource dengan input-output contract  
C. Menyimpan credential pada output  
D. Menghapus state caller

### 2. Root module biasanya ditentukan oleh…

A. Directory tempat command OpenTofu dijalankan  
B. Nama bucket saja  
C. Provider lock file global  
D. Workspace tanpa konfigurasi

### 3. Input module yang baik seharusnya…

A. Tidak memiliki type agar fleksibel  
B. Memiliki type, description, dan validation sesuai risiko  
C. Selalu default ke production  
D. Menyimpan password pada default

### 4. `sensitive = true` pada output…

A. Mengenkripsi state secara otomatis  
B. Menghapus value dari provider  
C. Menyamarkan presentasi, tetapi value dapat tetap berada di state  
D. Membolehkan secret di-commit

### 5. Mengapa provider sebaiknya dikonfigurasi root module?

A. Agar child module dapat memilih endpoint sembarang  
B. Agar environment/caller mengendalikan endpoint dan identity  
C. Agar state tidak diperlukan  
D. Agar provider tidak perlu di-init

### 6. `for_each` paling cocok ketika…

A. Instance memiliki key stabil dan metadata berbeda  
B. Semua instance harus menggunakan index yang dapat bergeser  
C. Key baru hanya diketahui setelah apply  
D. Resource tidak memiliki identity

### 7. Rename key `for_each` dari `admin` menjadi `backoffice` dapat…

A. Selalu menjadi no-op  
B. Menghasilkan delete/create karena address berubah  
C. Menghapus backend  
D. Mengubah workspace otomatis

### 8. `count` paling cocok untuk…

A. Instance homogen yang menerima identity index  
B. Object dengan role dan lifecycle unik  
C. Data source read-only  
D. Secret rotation

### 9. Menghapus elemen tengah list yang dipakai `count` berisiko…

A. Index instance berikutnya bergeser dan replacement terjadi  
B. Provider otomatis melakukan backup  
C. State menjadi remote  
D. Tidak ada perubahan address

### 10. Conditional `count = var.enabled ? 1 : 0` berarti…

A. false selalu hanya menyembunyikan output  
B. false dapat menghapus resource yang dikelola  
C. provider tidak dijalankan  
D. state dihapus tanpa plan

### 11. Data source umumnya digunakan untuk…

A. Membaca object yang ownership-nya berada di boundary lain  
B. Menghapus object shared  
C. Menghindari semua permission  
D. Menyimpan secret ke Git

### 12. Risiko backend key yang sama untuk dev dan staging adalah…

A. Hanya nama output berubah  
B. State collision/lost update dan salah scope resource  
C. Image otomatis multi-arch  
D. Lock menjadi lebih aman

### 13. Workspace bukan security boundary karena…

A. Workspace tidak memiliki nama  
B. Salah memilih workspace masih dapat mengarahkan command ke state berbahaya  
C. Workspace selalu memakai local state  
D. Workspace menghapus provider

### 14. Rekomendasi umum untuk environment on-prem dengan topology/permission berbeda adalah…

A. Satu directory dan satu state untuk semua  
B. Directory/root module terpisah dengan backend key terisolasi  
C. Selalu memakai workspace tanpa preflight  
D. Menyalin raw state antar environment

### 15. Plan artifact dev tidak boleh diterapkan ke staging karena…

A. Plan tidak pernah berisi resource  
B. Plan terikat pada state, provider, input, commit, dan context tertentu  
C. Staging tidak membutuhkan approval  
D. Backend key tidak relevan

### 16. Pilihan paling aman untuk credential CI adalah…

A. Nilai literal di `.tfvars` Git  
B. Default variable pada module  
C. Secret mechanism/credential helper dengan state exposure yang dipahami  
D. Menaruh password pada output biasa

### 17. `.gitignore` bukan secret manager karena…

A. Tidak dapat menghapus data yang sudah ter-commit atau masuk log/backup  
B. Selalu menampilkan semua file  
C. Mengubah provider endpoint  
D. Mengaktifkan locking

### 18. Tahap merge request yang tepat umumnya adalah…

A. `fmt → validate → plan → policy/review`  
B. `destroy → state rm → apply`  
C. `apply -auto-approve` tanpa plan  
D. `state mv` untuk semua rename

### 19. Handoff OpenTofu ke Ansible sebaiknya membawa…

A. Password SSH dan private key dalam output  
B. Metadata hostname/IP/role/environment non-secret  
C. Raw state ke Git  
D. Kubeconfig production ke artifact publik

### 20. Jika CI salah directory dan plan menyentuh production, tindakan pertama adalah…

A. Apply agar pipeline selesai  
B. Gunakan `-target` untuk menutupi diff  
C. Stop mutation, verifikasi context, dan buat plan baru setelah perbaikan  
D. Hapus state production

## Soal Esai Singkat

### 21. Jelaskan root module, child module, input contract, output contract, dan ownership.

### 22. Bandingkan workspace dan directory per environment untuk production on-prem.

### 23. Jelaskan `for_each` stable key versus `count` index serta contoh address churn.

### 24. Jelaskan data source dan mengapa data source bukan pengganti ownership resource.

### 25. Rancang CI `fmt/validate/plan` dengan approval, policy, artifact retention, dan apply terpisah.

### 26. Jelaskan secret boundary, `sensitive`, state exposure, Vault/SOPS, dan rotation.

### 27. Susun flow promotion commit/module version dari dev ke staging dan alasan plan baru harus dibuat.

### 28. Rancang output OpenTofu non-secret untuk handoff Ansible dan boundary menuju k3s.

### 29. Tulis runbook ketika plan artifact mengandung secret.

### 30. Jelaskan mengapa apply sukses belum membuktikan service sehat dan bagaimana evidence runtime dikumpulkan.

## Kunci Jawaban Pilihan Ganda

| No. | Jawaban | Penjelasan singkat |
|---:|:---:|---|
| 1 | B | Child module mengemas capability dengan contract. |
| 2 | A | Root module adalah konfigurasi pada directory command dijalankan. |
| 3 | B | Type/validation membuat input dapat direview dan dibatasi. |
| 4 | C | Sensitive memengaruhi presentasi, bukan enkripsi state. |
| 5 | B | Caller mengendalikan endpoint, identity, dan environment. |
| 6 | A | Map key stabil menjaga identity address. |
| 7 | B | Rename key mengubah address dan dapat create/delete. |
| 8 | A | Count cocok untuk instance homogen berbasis index. |
| 9 | A | Penghapusan list di tengah dapat menggeser index. |
| 10 | B | Count nol menghapus resource dari konfigurasi/state management. |
| 11 | A | Data source membaca object yang dikelola boundary lain. |
| 12 | B | Shared key dapat menyebabkan collision dan salah scope. |
| 13 | B | Workspace yang salah tetap dapat berbahaya. |
| 14 | B | Directory/key terpisah memperjelas boundary dan recovery. |
| 15 | B | Plan terikat context tempat ia dibuat. |
| 16 | C | Secret mechanism menjaga nilai di luar repository, dengan exposure tetap dikaji. |
| 17 | A | Ignore bukan penghapus jejak data atau perlindungan state. |
| 18 | A | Merge request memvalidasi dan merencanakan secara read-only. |
| 19 | B | Ansible memerlukan metadata, bukan credential OpenTofu. |
| 20 | C | Hentikan mutation dan pulihkan context sebelum plan baru. |

## Panduan Jawaban Esai

### 21. Module contract

Root module memilih backend, provider, input environment, composition, dan approval. Child module membungkus resource/capability dengan input typed/validated dan output minimum. Contract menjelaskan ownership: resource dibuat/dikelola module, sedangkan object shared dapat dibaca melalui data source. Perubahan contract diperlakukan seperti perubahan API.

### 22. Workspace versus directory

Workspace dapat mengisolasi state, tetapi salah workspace mudah terjadi dan topology/permission biasanya tetap berbagi root configuration. Directory per environment membuat path, backend, configuration, dan review lebih eksplisit serta lebih mudah dipisah dalam CI/identity. Keduanya bukan security boundary otomatis; backend permission, preflight, policy, locking, dan approval tetap diperlukan.

### 23. `for_each` versus `count`

`for_each` memakai key map/set sebagai identity, cocok untuk instance bernama/berbeda. `count` memakai index dan cocok untuk instance homogen. Rename key atau pergeseran index dapat menghasilkan create/delete/replace. Sebelum migration, backup state, lock writer, verifikasi address lama/baru, review plan, dan approval wajib dilakukan.

### 24. Data source

Data source membaca object yang dimiliki boundary lain, misalnya network shared. Ia tidak memberi permission update/delete atau menjadikan object bagian ownership state module. Data source berbahaya bila dipakai untuk menghindari pengelolaan resource yang sebenarnya harus dimiliki project; nama lookup yang berubah juga dapat menunjuk object salah.

### 25. CI plan

Checkout commit, verifikasi directory/environment, `tofu fmt -check`, init dengan backend terkontrol, validate, plan, policy/action check, lalu simpan artifact dengan permission/retention terbatas. Reviewer membaca plan dan approval terpisah diperlukan untuk apply. Apply memverifikasi commit/provider/backend/input/plan context, menggunakan locking dan identity minimum; production tidak auto-apply.

### 26. Secret boundary

Secret harus dibuat/disimpan pada Vault, SOPS/KMS, secret manager, atau credential helper sesuai policy, lalu diinjeksikan ke runner tanpa nilai literal di Git. `sensitive` menyamarkan output namun state/plan/provider log dapat tetap berisi value. Perlukan encryption, permission, audit, TTL/lease, rotation/revoke, cache/artifact control, dan incident procedure.

### 27. Promotion

Commit/module version dipin dan diuji di dev. Setelah approval dan health check, commit yang sama dipakai membuat plan baru pada staging backend/state/input. Plan staging direview dan di-apply sesuai approval; production mengikuti change process. Plan lama tidak dipindahkan karena terikat state/provider/input/context target.

### 28. Handoff

Output dapat berisi hostname, IP/address, role, environment, module version, dan metadata ownership. Ansible menggunakan metadata untuk inventory/bootstrap OS; credential SSH/token tetap dari secret mechanism. OpenTofu tidak menggantikan configuration management. Setelah host siap, k3s diinstal sesuai runbook; Helm/GitOps menangani aplikasi.

### 29. Artifact secret

Hentikan publication/apply, quarantine artifact, jangan menyalin value, hubungi owner security, revoke/rotate secret, audit log/cache/backup/runner, dan dokumentasikan exposure. Setelah containment, perbaiki masking, secret injection, artifact retention, state policy, dan test pipeline. Nilai yang pernah terekspos dianggap compromised sampai dibuktikan sebaliknya.

### 30. Apply versus health

Apply hanya menunjukkan provider menerima operasi dan state diperbarui. Service dapat tetap gagal karena process, port, dependency, network, readiness, data, metrics, atau SLO. Kumpulkan `state list/show` yang diredáksi, provider/runtime status, container/pod status, HTTP/TCP check, logs yang tidak berisi secret, metrics, dan acceptance check sesuai service.

## Skoring

- Pilihan ganda: 20 soal × 2 poin = 40.
- Esai: 10 soal × 6 poin = 60.
- Total: 100.
- Lulus minimal 80.
- Secret safety violation, credential commit, atau destructive operation pada scope salah dapat menggugurkan submission meskipun skor numerik mencapai 80.
