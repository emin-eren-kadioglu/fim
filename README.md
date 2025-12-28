# FIM – Windows Dosya Bütünlüğü / Etkinlik İzleyici

Bu proje, Windows üzerinde sistem sürücüsünü (örneğin `C:\`) gerçek zamanlı olarak izleyen ve dosya/dizin olaylarını bir SQLite veritabanına kaydeden basit bir **File Integrity Monitoring (FIM)** aracıdır.  

Ayrıca aktif Windows Explorer penceresinde açılan klasörleri takip eder, klasör erişimlerini ve o klasördeki ilk 50 öğeyi **ACCESS** olayı olarak loglar.

---

## Özellikler

- **Gerçek zamanlı dosya sistemi izleme**
  - Oluşturma (CREATED)
  - Silme (DELETED)
  - Değiştirme (MODIFIED)
  - Taşıma (MOVED)
  - Yeniden adlandırma (RENAMED)
- **Explorer erişim takibi**
  - Aktif Windows Explorer penceresinin yolu alınır.
  - Her klasör değişiminde:
    - Klasör için bir **ACCESS** kaydı,
    - İçindeki ilk 50 öğe için **ACCESS** kaydı oluşturulur.
- **Gelişmiş filtreleme**
  - Belirli alt klasörler, sistem klasörleri, cache/log/temp dizinleri hariç tutulur.
  - Sık değişen / önemsiz uzantılar (tmp, log, db, sys, vs.) loglanmaz.
  - Bazı özel dosyalar (pagefile.sys, hiberfil.sys, ntuser.dat, vb.) hariç tutulur.
- **SQLite veritabanına loglama**
  - Tüm olaylar `fim_logs.db` dosyasında `file_logs` tablosuna yazılır.
- **Metaveri toplama**
  - Dosya boyutu
  - Oluşturma, erişim, değiştirme zamanları
  - Dosya sahibi (Windows SID üzerinden)
  - Makine adı ve IP adresi
- **Hata loglama**
  - Çalışma sırasında oluşan hatalar `fim_errors.txt` dosyasına yazılır.
- **Çoklu thread kullanımı**
  - Dosya sistemi izleme ayrı bir thread’de,
  - Explorer takibi ana thread’de çalışır.

---

## Gereksinimler

- İşletim Sistemi: **Sadece Windows**
- Python: **3.8+** (tavsiye)
- Python paketleri:
  - Standart kütüphane: `os`, `time`, `sqlite3`, `socket`, `threading`, `traceback`, `datetime`, `random`, `urllib.parse`
  - Ek paketler:
    - `pywin32` (şunları içerir: `win32file`, `win32con`, `win32security`, `win32gui`, `win32com.client`, `pywintypes`)

---

## Kurulum

1. **Python kurulu olduğundan emin olun.**  
   [https://www.python.org/downloads/](https://www.python.org/downloads/)

2. **Gerekli paketi yükleyin:**

   ```bash
   pip install pywin32

3. **Betik dosyasını** (örneğin `fim.py`) uygun bir klasöre kaydedin.

4. **(Opsiyonel ama önerilir)**:  
   Komut istemcisini / PowerShell’i **Yönetici olarak** çalıştırın.  
   (Bazı sistem dizinlerine erişim için gerekebilir.)

---

## Çalıştırma

Komut satırında (CMD veya PowerShell) betiğin bulunduğu klasöre gidin:

```bash
python fim.py
```

Çıktı örneği:

```text
=== FIM ===
[*] Target Drive: C:\
[*] Logs will be saved to: C:\fim_logs.db
[*] Monitoring started for C:\
```

Bundan sonra:

- `C:\` sürücüsü gerçek zamanlı olarak izlenir.
- Explorer’da klasörler arasında dolaştıkça **ACCESS** logları oluşur.
- Hatalar `C:\fim_errors.txt` içine yazılır.

Programı durdurmak için:

- Konsol penceresinde `Ctrl + C` tuşlarına basabilirsiniz.

---

## Log Dosyaları

### 1. Veri Logu (SQLite) – `fim_logs.db`

- Konum:  
  Sistem sürücüsü kök dizini  
  Örnek: `C:\fim_logs.db` veya `D:\fim_logs.db`

- Tablo: `file_logs`

#### Tablo Şeması

```sql
CREATE TABLE IF NOT EXISTS file_logs (
    id       TEXT,
    date     TEXT,
    path     TEXT,
    type     TEXT,
    message  TEXT,
    event    TEXT,
    created  TEXT,
    accessed TEXT,
    modified TEXT,
    size     TEXT,
    machine  TEXT,
    ip       TEXT,
    owner    TEXT
);
```

Alanlar:

| Alan     | Açıklama                                                                 |
|----------|--------------------------------------------------------------------------|
| `id`     | Olay için 6 haneli rastgele ID                                          |
| `date`   | Olay zamanı (`dd.MM.yyyy HH:mm:ss`)                                     |
| `path`   | Dosya/dizin tam yolu                                                    |
| `type`   | `FILE`, `DIRECTORY` veya `UNKNOWN`                                      |
| `message`| İnsan okunabilir açıklama (örn. `Renamed: C:\a.txt -> C:\b.txt`)        |
| `event`  | Olay tipi: `CREATED`, `DELETED`, `MODIFIED`, `MOVED`, `RENAMED`, `ACCESS` |
| `created`| Dosyanın oluşturulma tarihi (varsa)                                     |
| `accessed`| Son erişim tarihi (varsa)                                              |
| `modified`| Son değiştirilme tarihi (varsa)                                        |
| `size`   | Dosya boyutu (byte cinsinden, string olarak)                            |
| `machine`| Bilgisayar adı (hostname)                                               |
| `ip`     | Makinenin IP adresi                                                     |
| `owner`  | Dosya sahip bilgisi (`DOMAIN\USER` formatında veya özel metin)          |

#### Örnek Sorgular

Tüm kayıtlar:
```sql
SELECT * FROM file_logs;
```

Son 100 olay:
```sql
SELECT * FROM file_logs
ORDER BY date DESC
LIMIT 100;
```

Sadece silinen dosyalar:
```sql
SELECT date, path, owner
FROM file_logs
WHERE event = 'DELETED'
ORDER BY date DESC;
```

Belirli bir klasör altındaki aktiviteler:
```sql
SELECT date, event, path, message
FROM file_logs
WHERE path LIKE 'C:\Users\KullaniciAdi\Documents%'
ORDER BY date DESC;
```

---

### 2. Sistem Logu (Metin) – `fim_errors.txt`

- Konum:
  - Sistem sürücüsü kök dizini  
    Örnek: `C:\fim_errors.txt`

- Format:

```text
[YYYY-MM-DD HH:MM:SS] [LEVEL] [Context] Mesaj
   >>> DETAILS: (opsiyonel detay metni)
```

Örnek:

```text
[2025-12-09 10:15:23] [WARNING] [DriveCheck] Access Denied to C:\ (Admin required?)
[2025-12-09 10:16:05] [ERROR] [EventLoop-C:\] Error processing events
   >>> DETAILS: Traceback (most recent call last): ...
```

`LEVEL` değerleri örneğin: `INFO`, `WARNING`, `ERROR`, `CRITICAL`

---

## Çalışma Mantığı (Kısa Özet)

### 1. Sistem Sürücüsünü İzleme

- `SYSTEM_DRIVE` ortam değişkeni ile sistem sürücüsü alınır (`C:` gibi).
- Hedef sürücü: `C:\` formatında `target_drive` değişkeniyle ayarlanır.
- `monitor_drive(drive_letter)` fonksiyonu:
  - `ReadDirectoryChangesW` API’sini kullanarak tüm alt dizinleriyle birlikte dinler.
  - Elde edilen olayları çözümler ve:
    - `CREATED`
    - `DELETED`
    - `MODIFIED`
    - `RENAMED` / `MOVED`  
    türlerini algılar.
  - Olayları `write_data_log` üzerinden veritabanına yazar.

**Not:**  
Silme ve yeniden adlandırma olaylarını daha doğru tespit edebilmek için kısa süreli bir `recent_deletes` önbelleği kullanılır.

### 2. Explorer Penceresi Takibi

- `get_active_explorer_path()` fonksiyonu:
  - `Shell.Application` COM nesnesini kullanarak açık Explorer pencerelerini alır.
  - Ön plandaki pencerenin (`GetForegroundWindow`) yolunu tespit eder.
- Ana döngü:
  - Her 0.5 saniyede bir aktif Explorer yolu değişmiş mi kontrol eder.
  - Değişmişse `scan_folder_access(folder_path)` çağrılır.
- `scan_folder_access`:
  - Klasör için bir `ACCESS` log kaydı yazar (`type=DIRECTORY`).
  - Sonra `os.listdir` ile ilk 50 öğe için `ACCESS` logları yazar (`type=FILE`).

---

## Filtreler ve Hariç Tutulanlar

Logların aşırı büyümesini engellemek ve gereksiz gürültüyü azaltmak için çeşitli filtreler uygulanır.

### 1. Hariç Tutulan Alt Yollar (`EXCLUDED_SUBPATHS`)

Örnekler (tam liste kodda):

- `.vscode\extensions`
- `.git`
- `node_modules`
- `appdata\local\temp`
- `windows\temp`
- `windows\softwaredistribution`
- `programdata\microsoft\windows defender`
- `windows\system32\winevt\logs`
- `windows\system32\tasks`
- `windows\system32\config`
- `system volume info`
- `$recycle.bin`
- vb.

Bu alt yolu içeren yollar **loglanmaz**.

### 2. Göz ardı edilen uzantılar (`IGNORED_EXTENSIONS`)

Örneğin:

- `.tmp`, `.log`, `.bak`, `.swp`
- `.db`, `.dat`, `.sqlite`
- `.sys`, `.bin`, `.pf`, `.etl`, `.evtx`
- `.pdb`, `.obj`, `.o`
- vb.

Bu uzantılara sahip dosyalar **loglanmaz**.

### 3. Hariç tutulan dosya adları (`EXCLUDED_FILES`)

Örneğin:

- `fim_logs.db`, `fim_logs.db-journal`
- `log.csv`
- `desktop.ini`
- `thumbs.db`
- `pagefile.sys`, `hiberfil.sys`, `swapfile.sys`
- `ntuser.dat`, `usrclass.dat`, `iconcache.db`
- vb.

Bu dosya adları **loglanmaz**.

---

## Özelleştirme

Kodu düzenleyerek bazı davranışları kolayca değiştirebilirsiniz:

### 1. Log dosyalarının yeri

Üst kısımdaki sabitler:

```python
SYSTEM_DRIVE = os.environ.get('SystemDrive', 'C:')

LOG_FILE_DATA   = f"{SYSTEM_DRIVE}\\fim_logs.db"
LOG_FILE_SYSTEM = f"{SYSTEM_DRIVE}\\fim_errors.txt"
```

Örneğin logları farklı bir klasöre almak için:

```python
LOG_FILE_DATA   = r"C:\Logs\fim_logs.db"
LOG_FILE_SYSTEM = r"C:\Logs\fim_errors.txt"
```

### 2. İzlenen sürücüyü değiştirme

Varsayılan:

```python
target_drive = SYSTEM_DRIVE + "\\"
```

Örneğin D sürücüsünü izlemek için:

```python
target_drive = "D:\\"
```

> Not: Kod sadece **bir** sürücüyü izliyor. Birden fazla sürücüyü izlemek için her sürücü için ayrı thread başlatmanız gerekir.

### 3. Filtreleri güncelleme

- `EXCLUDED_SUBPATHS` listesine yeni dizin parçaları ekleyebilirsiniz.
- `IGNORED_EXTENSIONS` kümesine yeni uzantılar ekleyebilirsiniz.
- `EXCLUDED_FILES` kümesine yeni dosya isimleri ekleyebilirsiniz.

---

## Yetkiler ve Sınırlamalar

- Bazı sistem dizinlerine erişim için **Yönetici yetkileri** gerekebilir.
  - `is_drive_usable` ve metadata toplamada `Access Denied` hataları alabilirsiniz.
- Bu araç:
  - Yalnızca **Windows** üzerinde çalışır.
  - NTFS/Windows izinleriyle uyumlu olarak çalışır.
  - Sadece betiği başlattıktan sonra gerçekleşen olayları yakalar (geçmişe dönük tarama yapmaz).
- Explorer takibi:
  - Sadece **Windows Explorer** pencerelerini takip eder.
  - Üçüncü parti dosya yöneticilerini algılamaz.

---

## Güvenlik ve Gizlilik

- Loglarda aşağıdaki bilgiler tutulur:
  - Tam dosya/dizin yolları
  - Dosya adları
  - Dosya sahibi (kullanıcı hesabı)
  - Makine adı ve IP adresi
- Bu nedenle:
  - Log dosyalarını sadece yetkili kullanıcılarla paylaşın.
  - `fim_logs.db` ve `fim_errors.txt` dosyalarının erişim izinlerini uygun şekilde ayarlayın.

---

## Geliştirme Önerileri

Projeyi genişletmek için bazı fikirler:

- Birden fazla sürücüyü aynı anda izlemek (C:\, D:\ vb.).
- Eski logları otomatik arşivleme / temizleme mekanizması.
- Basit bir web arayüzü veya GUI ile log görüntüleme.
- Belirli olaylar için e-posta / webhook / syslog entegrasyonu.
- Belirli klasörler için daha ayrıntılı politikalar (beyaz liste/siyah liste).

---
## EXE Oluşturma (PyInstaller)

Bu Python betiğini tek bir çalıştırılabilir `.exe` dosyasına dönüştürmek için PyInstaller kullanılır.

### PyInstaller Kurulumu

Komut istemcisini (CMD veya PowerShell) açın ve aşağıdaki komutu çalıştırın:

    pip install pyinstaller

### EXE Dosyası Oluşturma

`fim.py` dosyasının bulunduğu klasöre gidin:

    cd C:\path\to\your\script

Ardından aşağıdaki komutu çalıştırın:

    pyinstaller --onefile --noconsole fim.py

### Parametre Açıklamaları

- `--onefile`  
  Tüm bağımlılıkları tek bir `.exe` dosyası içine paketler.

- `--noconsole`  
  Konsol penceresi açılmadan arka planda çalışmasını sağlar.

### Oluşturulan Dosya

Derleme tamamlandığında çalıştırılabilir dosya aşağıdaki konumda oluşur:

    dist\fim.exe

Bu dosya tek başına çalıştırılabilir durumdadır.

### Yönetici Yetkisi (Önerilir)

Bu araç sistem dizinlerini izlediği için `fim.exe` dosyasının yönetici yetkisiyle çalıştırılması önerilir:

    Sağ tık → Yönetici olarak çalıştır

### Antivirus / Defender Uyarısı

Gerçek zamanlı dosya izleme ve arka planda çalışan (`--noconsole`) EXE dosyaları,
bazı antivirüs veya EDR çözümleri tarafından şüpheli davranış olarak algılanabilir.

Gerekirse:
- Test ortamında çalıştırın
- Windows Defender için exclusion ekleyin
- Kurumsal kullanımda Code Signing uygulayın

### Debug Amaçlı Konsollu EXE

Hata ayıklamak için konsollu bir EXE üretmek isterseniz:

    pyinstaller --onefile fim.py
