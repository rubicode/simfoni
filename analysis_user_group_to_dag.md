# Implementasi Struktur DAG untuk Grup User di Simfoni v3 Primero

Untuk mengubah model user group saat ini menjadi struktur Directed Acyclic Graph (DAG) dengan hubungan atasan-bawahan untuk penugasan dan pelaporan, saya akan menganalisis perubahan yang diperlukan dan dampaknya.

## Struktur Saat Ini vs Struktur DAG

**Struktur Saat Ini:**
- User group adalah entitas datar dengan hubungan many-to-many ke pengguna dan lembaga
- Tidak ada hubungan hierarkis antar user group
- Penugasan berdasarkan keanggotaan grup, bukan hierarki

**Struktur DAG yang Diusulkan:**
- User group akan memiliki hubungan atasan-bawahan (banyak atasan dapat memiliki banyak bawahan)
- Satu bawahan dapat memiliki lebih dari satu atasan (multi-parent DAG)
- Alur penugasan mengalir ke bawah (atasan ke bawahan)
- Alur pelaporan mengalir ke atas (bawahan ke atasan)
- Setiap tugas menyimpan referensi ke atasan yang memberikan tugas
- Mencegah ketergantungan melingkar (sifat asiklik)

## Perubahan Database yang Dibutuhkan

1. **Tabel Join Baru**: Buat `user_group_hierarchies` untuk menjalin hubungan atasan-bawahan (memungkinkan banyak atasan untuk satu bawahan):
   ```ruby
   create_table :user_group_hierarchies do |t|
     t.belongs_to :atasan_group, foreign_key: { to_table: :user_groups }
     t.belongs_to :bawahan_group, foreign_key: { to_table: :user_groups }
     t.index [:atasan_group_id, :bawahan_group_id], unique: true
   end
   ```
   
2. **Perubahan pada Tabel Tasks**: Tambahkan kolom untuk melacak user group yang memberikan tugas:
   ```ruby
   add_column :tasks, :assigned_by_group_id, :integer, foreign_key: { to_table: :user_groups }
   add_index :tasks, :assigned_by_group_id
   ```

2. **Perubahan Model**: Perbarui model `UserGroup` untuk mendukung hubungan DAG:
   ```ruby
   # Di model UserGroup
   has_many :atasan_relationships, foreign_key: :bawahan_group_id, class_name: 'UserGroupHierarchy'
   has_many :bawahan_relationships, foreign_key: :atasan_group_id, class_name: 'UserGroupHierarchy'
   has_many :atasan_groups, through: :atasan_relationships, source: :atasan_group
   has_many :bawahan_groups, through: :bawahan_relationships, source: :bawahan_group
   
   # Metode untuk mencegah siklus
   validate :no_circular_references
   ```

## File yang Perlu Diubah

1. **Perubahan Model (≈4 file):**
   - `app/models/user_group.rb` - Menambahkan asosiasi hierarki dan validasi
   - `app/models/user_group_hierarchy.rb` (baru) - Membuat model untuk hierarki
   - `app/models/user.rb` - Memperbarui untuk menangani izin hierarkis
   - `app/models/permission.rb` - Menambahkan izin baru untuk operasi hierarkis

2. **Perubahan Controller (≈3 file):**
   - `app/controllers/api/v2/user_groups_controller.rb` - Memperbarui untuk mendukung operasi hierarki
   - `app/controllers/api/v2/tasks_controller.rb` - Menambahkan logika untuk penugasan hierarkis
   - `app/controllers/api/v2/users_controller.rb` - Memperbarui filter berdasarkan hierarki

3. **Perubahan Database (≈2 file):**
   - Migrasi baru untuk tabel `user_group_hierarchies`
   - Migrasi untuk memperbarui izin dan indeks

4. **Perubahan Frontend (≈6-8 file):**
   - Komponen untuk visualisasi dan pengelolaan hierarki
   - Formulir untuk membuat hubungan atasan-bawahan
   - Antarmuka penugasan yang menghormati hierarki
   - Tampilan pelaporan yang menunjukkan data hierarkis

5. **Objek Layanan (≈3-4 file):**
   - Layanan validasi DAG untuk mencegah siklus
   - Layanan penugasan untuk menegakkan aliran atasan-ke-bawahan
   - Layanan pelaporan untuk menegakkan aliran bawahan-ke-atasan

## Logika Baru yang Diperlukan

1. **Logika Validasi DAG:**
   ```ruby
   def no_circular_references
     # Periksa apakah menambahkan hubungan ini akan menciptakan siklus
     visited = Set.new
     to_visit = bawahan_groups.to_a
     
     while node = to_visit.pop
       return errors.add(:base, 'Referensi melingkar terdeteksi') if node.id == id
       next if visited.include?(node.id)
       
       visited.add(node.id)
       to_visit.concat(node.bawahan_groups.to_a)
     end
   end
   ```

2. **Logika Penugasan Atasan-ke-Bawahan:**
   ```ruby
   def can_assign_to_group?(user, target_group)
     # User dapat menugaskan jika grup mereka adalah atasan dari grup target
     user.user_groups.any? do |user_group|
       user_group.bawahan_groups.include?(target_group) ||
       user_group == target_group # Dapat menugaskan dalam grup yang sama
     end
   end
   ```

3. **Logika Pelaporan Bawahan-ke-Atasan:**
   ```ruby
   def can_report_to_group?(user, target_group)
     # User dapat melaporkan jika grup mereka adalah bawahan dari grup target
     user.user_groups.any? do |user_group|
       user_group.atasan_groups.include?(target_group) ||
       user_group == target_group # Dapat melaporkan dalam grup yang sama
     end
   end
   
   # Mekanisme pelacakan atasan yang memberikan tugas
   def report_task_to_assigning_superior(task)
     # Mendapatkan atasan yang memberikan tugas ini
     assigning_superior = task.assigned_by_group
     
     # Memastikan pelaporan hanya ke atasan yang memberikan tugas
     return false unless user.user_groups.any? do |user_group|
       user_group.atasan_groups.include?(assigning_superior)
     end
     
     # Lanjutkan dengan pelaporan
     submit_report_to_superior(task, assigning_superior)
   end
   ```

## Fitur yang Terpengaruh

1. **Manajemen User (Dampak Tinggi)**
   - Logika penugasan grup harus menghormati hierarki
   - Perhitungan izin akan menjadi lebih kompleks

2. **Penugasan Catatan (Dampak Tinggi)**
   - Alur penugasan perlu menghormati hierarki
   - UI baru diperlukan untuk menampilkan/memfilter berdasarkan hierarki

3. **Pelaporan & Analitik (Dampak Menengah)**
   - Laporan perlu memperhitungkan struktur hierarkis
   - Opsi agregasi baru untuk menyusun data berdasarkan hierarki

4. **Pencarian & Filter (Dampak Menengah)**
   - Pencarian akan membutuhkan filter yang sadar hierarki
   - Izin hasil akan bergantung pada posisi hierarki

5. **Otorisasi (Dampak Tinggi)**
   - Sistem izin akan membutuhkan kesadaran hierarki
   - Izin dapat mengalir turun dalam hierarki

6. **Endpoint API (Dampak Menengah)**
   - Endpoint baru untuk manajemen hierarki
   - Endpoint yang dimodifikasi untuk mendukung operasi hierarkis

## Total Estimasi

- **File yang Diubah:** Sekitar 18-22 file
- **File Baru yang Dibuat:** Sekitar 3-5 file
- **Fitur yang Terpengaruh:** 6 fitur inti dengan tingkat dampak yang bervariasi
- **Perkiraan Usaha:** Menengah-Tinggi (memerlukan perencanaan dan pengujian yang cermat)

## Tantangan Implementasi

1. **Mempertahankan Properti DAG**: Memastikan tidak ada siklus yang diperkenalkan saat membangun hubungan
2. **Masalah Kinerja**: Kueri hierarkis dapat mahal; mungkin perlu optimasi
3. **Kompleksitas Izin**: Model izin menjadi jauh lebih kompleks
4. **Strategi Migrasi**: Data yang ada akan membutuhkan strategi untuk membangun hierarki awal
5. **Pelacakan Atasan Pemberi Tugas**: Mengembangkan mekanisme yang melacak atasan mana yang memberikan tugas tertentu, terutama ketika bawahan memiliki beberapa atasan
6. **Manajemen Konflik**: Membuat mekanisme untuk menangani konflik ketika beberapa atasan memberikan tugas yang bertentangan kepada bawahan yang sama
