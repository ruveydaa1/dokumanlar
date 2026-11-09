# Zamanlanmış Tarama Sistemi (Hangfire)

SPM uygulamasında kullanıcıların taramaları istedikleri tarih ve saatte otomatik olarak başlatabilmeleri için Hangfire kullanılmıştır. Bu yapı sayesinde kullanıcı, taramanın gerçekleşeceği zamanı önceden belirleyebilir. Belirlenen zaman geldiğinde tarama işlemi arka planda başlatılır. Kullanıcının uygulama sayfasında açık kalmasına gerek yoktur.

### 1. Hangfire Nedir?

Hangfire, .NET uygulamalarında arka plan görevlerini yönetmek ve belirli zamanlarda çalıştırmak için kullanılan bir kütüphanedir. Arka plan görevlerinin oluşturulmasını, saklanmasını ve zamanı geldiğinde çalıştırılmasını sağlar.

SPM projesinde Hangfire, zamanlanmış taramaların yönetilmesi amacıyla kullanılmıştır. Kullanıcının belirlediği tarih ve saat bilgisine göre bir arka plan görevi oluşturulur. Bu görev, belirlenen zaman geldiğinde ilgili asset'in tarama sürecini başlatır.

Projede Hangfire'ın PostgreSQL entegrasyonu kullanılmıştır. Böylece oluşturulan görevler veritabanında saklanır ve Hangfire tarafından yönetilir.

### 2. Zamanlanmış Tarama Kaydının Oluşturulması

Kullanıcı, Asset detay sayfasında bulunan zamanlanmış tarama alanından tarama yapılacak tarih ve saati seçer. Seçilen bilgiler `/api/assets/{id}/schedule-scan` endpoint'i üzerinden `AssetsController` sınıfına gönderilir.

İstek alındığında öncelikle aşağıdaki kontroller gerçekleştirilir:

* İlgili asset'in sistemde bulunup bulunmadığı kontrol edilir.
* Asset'in aktif durumda olması gerekir.
* Asset'e ait kaynak adresinin boş olmaması kontrol edilir.
* Tarama aracı olarak yalnızca Semgrep veya Trivy seçilebilir.
* Belirlenen tarama zamanının geçmiş bir tarih olmaması gerekir.

Kullanıcının seçtiği tarih ve saat Türkiye saat dilimine göre değerlendirilir ve UTC formatına dönüştürülür. Bu işlem, zaman bilgisinin arka plan görevleri tarafından doğru şekilde değerlendirilmesini sağlar.

Kontroller başarılı olduğunda `ScheduledScan` tablosuna yeni bir kayıt oluşturulur. Bu kayıtta asset bilgisi, taramayı planlayan kullanıcı, kullanılacak tarama aracı, planlanan zaman ve görev durumu tutulur.

Yeni oluşturulan kaydın başlangıç durumu `Scheduled` olarak belirlenir.

### 3. ScheduledScan ve ScanRun Arasındaki Fark

Zamanlanmış tarama sisteminde `ScheduledScan` ve `ScanRun` birbirinden farklı amaçlara hizmet eder.

**ScheduledScan**, taramanın ne zaman yapılacağını ve planlama sürecine ait bilgileri tutar. Kullanıcının oluşturduğu zamanlanmış tarama kaydını temsil eder.

**ScanRun** ise taramanın gerçekten başlatılmasıyla oluşturulan çalıştırma kaydıdır. Taramanın başlangıç ve tamamlanma zamanları, durumu ve sonuçları bu kayıt üzerinden takip edilir.

Bu iki yapının ayrı tutulması sayesinde bir taramanın planlanması ile gerçekten çalıştırılması birbirinden ayrılır. Tarama başlatıldığında ilgili `ScheduledScan` kaydı, oluşturulan `ScanRun` kaydıyla ilişkilendirilir.

### 4. Hangfire Üzerinden Görevin Planlanması

`ScheduledScan` kaydı veritabanına kaydedildikten sonra Hangfire üzerinden arka plan görevi oluşturulur.

Bu işlemde `BackgroundJob.Schedule` metodu kullanılır. İlgili görev, `ScheduledScanJob` sınıfının `ExecuteAsync` metodu üzerinden çalıştırılmak üzere planlanır.

Göreve aşağıdaki bilgiler aktarılır:

* Asset ID
* Taramayı planlayan kullanıcı ID'si
* Kullanılacak tarama aracı
* ScheduledScan ID

Görevin çalıştırılacağı zaman, kullanıcının belirlediği UTC tarih ve saatine göre ayarlanır.

Hangfire, oluşturulan görevi PostgreSQL üzerinde saklar ve zamanı geldiğinde çalıştırılmasını sağlar. Böylece tarama işleminin başlatılması için kullanıcının tekrar bir işlem yapmasına gerek kalmaz.

### 5. ScheduledScanJob ile Taramanın Başlatılması

`ScheduledScanJob`, zamanı gelen zamanlanmış taramanın çalıştırılmasını sağlayan arka plan görevidir.

Görev çalıştığında ilgili `ScheduledScan` kaydı üzerinden planlanan tarama bilgileri alınır. Taramanın başlatılmasına yönelik işlemler gerçekleştirilir ve ilgili kayıtların durumları güncellenir.

Bu süreçte:

1. İlgili zamanlanmış tarama kaydı bulunur ve çalıştırma süreci başlatılır.
2. Taramanın gerçekten çalıştırıldığını takip edebilmek için yeni bir `ScanRun` kaydı oluşturulur.
3. Oluşturulan `ScanRun` kaydı, ilgili `ScheduledScan` kaydıyla ilişkilendirilir.
4. Tarama isteği Kafka üzerinden ScannerService'e gönderilir.

Bu noktadan sonra tarama, normal tarama sürecinde olduğu gibi ScannerService tarafından işlenir. Seçilen tarama aracına göre Semgrep veya Trivy entegrasyonu devreye girer.

### 6. Tarama Sonucunun Zamanlanmış Kayda Yansıtılması

Zamanlanmış tarama üzerinden başlatılan işlemler de normal taramalarla aynı sonuç işleme mekanizmasını kullanır.

ScannerService taramayı tamamladıktan sonra sonucu UserManagement uygulamasına iletir. `ScanRunController` üzerinden işlenen sonuçla birlikte ilgili `ScanRun` kaydının durumu güncellenir.

Tarama bir zamanlanmış görev üzerinden başlatılmışsa ilişkili `ScheduledScan` kaydı da güncellenir. Böylece planlanan görevin çalışıp çalışmadığı ve tarama sonucunun durumu takip edilebilir.

Zamanlanmış tarama sürecinde kullanılan temel durumlar şunlardır:

* **Scheduled:** Tarama planlanmış, ancak henüz çalıştırılmamıştır.
* **Running:** Zamanlanmış tarama çalıştırılmaktadır.
* **Completed:** Tarama başarıyla tamamlanmıştır.
* **Failed:** Tarama işlemi sırasında hata oluşmuştur.

### 7. Zamanlanmış Tarama Kayıtlarının Görüntülenmesi

Asset detay sayfasında, ilgili asset için oluşturulan zamanlanmış taramalar görüntülenebilir.

Bu bilgiler `/api/assets/{id}/scheduled-scans` endpoint'i üzerinden alınır. İlgili asset'e ait zamanlanmış taramalar planlanan tarihe göre sıralanır.

Listede tarama aracı, planlanan tarih, görev durumu ve taramayı oluşturan kullanıcı gibi bilgiler gösterilir. Böylece kullanıcılar daha önce oluşturdukları zamanlanmış taramaları takip edebilir.

### 8. Hangfire Dashboard

Hangfire tarafından yönetilen arka plan görevlerini incelemek için Hangfire Dashboard kullanılmıştır.

Dashboard üzerinden oluşturulan görevler, bekleyen işler ve çalıştırma geçmişi takip edilebilir. Projede dashboard adresi aşağıdaki şekildedir:

`http://localhost:5245/hangfire`

Hangfire'ın PostgreSQL entegrasyonu sayesinde görev bilgileri veritabanında saklanır. Böylece zamanlanmış görevlerin yönetimi uygulamanın kendi tarama kayıtlarından bağımsız olarak Hangfire tarafından da takip edilebilir.

### 9. Genel İş Akışı

SPM uygulamasındaki zamanlanmış tarama süreci genel olarak aşağıdaki sırayla gerçekleşir:

1. Kullanıcı, Asset detay sayfasından tarama tarihini ve saatini belirler.
2. `AssetsController`, gelen isteği ve asset bilgilerini doğrular.
3. `ScheduledScan` tablosuna planlama kaydı oluşturulur.
4. Hangfire üzerinden `ScheduledScanJob` görevi planlanır.
5. Belirlenen zaman geldiğinde `ScheduledScanJob` çalışır.
6. Yeni bir `ScanRun` kaydı oluşturulur ve `ScheduledScan` kaydıyla ilişkilendirilir.
7. Tarama isteği Kafka üzerinden ScannerService'e gönderilir.
8. ScannerService, seçilen tarama aracına göre taramayı başlatır.
9. Tarama sonucu UserManagement uygulamasına iletilir.
10. `ScanRun` ve ilişkili `ScheduledScan` kayıtlarının durumları güncellenir.

Bu yapı sayesinde SPM uygulamasında taramaların yalnızca manuel olarak başlatılması değil, önceden belirlenen zamanlarda otomatik olarak gerçekleştirilmesi de sağlanmıştır.
