# Asset Yönetimi

SPM projesinde Asset Yönetimi, güvenlik taraması gerçekleştirilecek projelerin ve kaynakların sistem üzerinden kaydedilmesini ve yönetilmesini sağlar. Her asset; adı, türü, kaynak adresi, açıklaması ve aktiflik durumu gibi bilgilerle veritabanında saklanır.

Asset işlemleri backend tarafında `AssetsController` sınıfı üzerinden yönetilmektedir. Bu controller, asset kayıtlarının oluşturulması, listelenmesi, güncellenmesi ve silinmesinin yanı sıra asset'lere ait tarama geçmişinin görüntülenmesini, yeni tarama başlatılmasını ve taramaların zamanlanmasını sağlar.

Asset yönetimi işlemlerine yalnızca `SuperAdmin`, `Admin` ve `Developer` rollerine sahip kullanıcılar erişebilir.

### 1. Asset Oluşturma

Sisteme yeni bir asset eklemek için frontend üzerinden gerekli bilgiler girilir ve backend'e HTTP POST isteği gönderilir.

Bu işlemde `CreateAssetRequest` DTO'su kullanılır. Backend, isteği `IAssetService` aracılığıyla ilgili servis katmanına iletir. İşlem başarılı olduğunda oluşturulan asset'in kimliğiyle birlikte başarı mesajı döndürülür.

Kullanılan API uç noktası:

```
POST /api/assets
```

### 2. Asset Listeleme ve Detay Görüntüleme

Sisteme kayıtlı asset'lerin listelenmesi ve belirli bir asset'in detaylarının görüntülenmesi mümkündür.

Tüm asset kayıtlarını getirmek için aşağıdaki uç nokta kullanılır:

```
GET /api/assets
```

Belirli bir asset'in bilgilerini görüntülemek için ise asset kimliği kullanılır:

```
GET /api/assets/{id}
```

İstenen asset bulunamazsa backend `404 Not Found` yanıtı döndürür.

### 3. Asset Güncelleme ve Silme

Mevcut bir asset'in bilgilerinin değiştirilmesi için HTTP PUT isteği kullanılır. Güncelleme işleminde `UpdateAssetRequest` DTO'su aracılığıyla gönderilen bilgiler `IAssetService` üzerinden işlenir.

```
PUT /api/assets/{id}
```

Bir asset'in sistemden silinmesi için HTTP DELETE isteği kullanılır:

```
DELETE /api/assets/{id}
```

Güncelleme veya silme işlemlerinde ilgili asset bulunamazsa backend `404 Not Found` yanıtı döndürür.

### 4. Asset Üzerinden Tarama Başlatılması

SPM projesinde kayıtlı asset'ler üzerinden güvenlik taraması başlatılabilir. Tarama işlemlerinde Semgrep ve Trivy olmak üzere iki farklı tarayıcı desteklenmektedir.

Frontend üzerinden tarayıcı seçilerek backend'e HTTP POST isteği gönderilir. Backend öncelikle ilgili asset'in varlığını ve repository URL bilgisinin tanımlı olup olmadığını kontrol eder. Ayrıca gönderilen tarayıcı bilgisinin Semgrep veya Trivy olması gerekir.

Tarama başlatılmadan önce `ScanRun` kaydı oluşturulur. Bu kayıtta asset bilgisi, tarayıcı türü, kaynak adresi, taramayı başlatan kullanıcı, başlangıç zamanı ve işlem durumu tutulur.

Tarama kaydı veritabanına kaydedildikten sonra `KafkaScanProducer` aracılığıyla tarama isteği Kafka'ya gönderilir. Böylece tarama işlemi backend API isteğinden ayrılarak ScannerService tarafından işlenebilecek bir mesaj hâline getirilir.

Kullanılan API uç noktası:

```
POST /api/assets/{id}/scan
```

Tarama isteği Kafka'ya başarıyla gönderildiğinde backend, taramanın başlatıldığını ve oluşturulan `ScanRun` kaydının kimliğini içeren bir yanıt döndürür.

Kafka'ya mesaj gönderilirken hata oluşursa ilgili `ScanRun` kaydının durumu `Failed` olarak güncellenir. Hata bilgisi ve tamamlanma zamanı da kayda eklenir.

### 5. Tarama Geçmişinin Görüntülenmesi

Her asset için daha önce gerçekleştirilen tarama işlemlerinin geçmişi görüntülenebilir.

Bu işlemde `IScanRunService` kullanılarak ilgili asset'e ait tarama kayıtları getirilir.

Kullanılan API uç noktası:

```
GET /api/assets/{id}/scan-runs
```

Bu sayede bir asset üzerinde hangi taramaların gerçekleştirildiği ve bu taramaların durumları takip edilebilir.

### 6. Zamanlanmış Tarama Oluşturulması

SPM projesinde güvenlik taramaları yalnızca anlık olarak başlatılmamakta, ileri bir tarih ve saate de planlanabilmektedir.

Zamanlanmış tarama oluşturmak için frontend tarafından tarayıcı bilgisi ve planlanan tarih-saat bilgisi gönderilir. Bu işlemde `ScheduleScanRequest` DTO'su kullanılır.

Backend öncelikle aşağıdaki kontrolleri gerçekleştirir:

* İlgili asset'in sistemde bulunup bulunmadığı kontrol edilir.
* Asset'in aktif durumda olması gerekir.
* Asset için repository URL tanımlanmış olmalıdır.
* Tarayıcı olarak yalnızca Semgrep veya Trivy seçilebilir.
* Planlanan tarama zamanı mevcut zamandan ileri bir tarih ve saat olmalıdır.

Tarih ve saat bilgisi Türkiye saat dilimi dikkate alınarak UTC zamanına dönüştürülür.

Kontroller başarılı olduğunda `ScheduledScan` nesnesi oluşturulur ve `scheduled_scans` tablosuna kaydedilir. Bu kayıtta asset kimliği, taramayı planlayan kullanıcı, tarayıcı, planlanan zaman ve başlangıç durumu tutulur.

Ardından Hangfire'ın `BackgroundJob.Schedule` metodu kullanılarak `ScheduledScanJob` sınıfının `ExecuteAsync` metodu belirlenen zamanda çalıştırılmak üzere planlanır.

Kullanılan API uç noktası:

```
POST /api/assets/{id}/schedule-scan
```

Hangfire zamanlama işlemi başarılı olduğunda backend, zamanlanmış taramanın kimliğini, Hangfire job kimliğini, tarayıcı bilgisini ve planlanan zamanı içeren bir yanıt döndürür.

Zamanlama sırasında hata oluşursa ilgili `ScheduledScan` kaydının durumu `Failed` olarak güncellenir. Hata mesajı ve tamamlanma zamanı da veritabanına kaydedilir.

### 7. Zamanlanmış Tarama Kayıtlarının Görüntülenmesi

Bir asset için oluşturulmuş zamanlanmış taramaların listesi görüntülenebilir.

Bu işlemde ilgili asset'e ait `ScheduledScans` kayıtları veritabanından alınır ve planlanan zamana göre yeniden eskiye doğru sıralanır.

Listede taramanın kimliği, asset bilgisi, tarayıcı türü, durumu, planlanan zamanı, başlangıç ve tamamlanma zamanları, ilişkili `ScanRun` kimliği ve varsa hata bilgisi gösterilir. Ayrıca zamanlanmış taramayı oluşturan kullanıcının adı ve soyadı da görüntülenebilir.

Kullanılan API uç noktası:

```
GET /api/assets/{id}/scheduled-scans
```

### 8. Sonuç

SPM projesindeki Asset Yönetimi, güvenlik taramalarının gerçekleştirileceği kaynakların merkezi olarak yönetilmesini sağlar.

Asset kayıtları üzerinden yeni taramalar başlatılabilir, geçmiş taramalar görüntülenebilir ve ileri bir tarih için tarama planlanabilir. Normal tarama işlemlerinde Kafka, zamanlanmış tarama işlemlerinde ise Hangfire kullanılmaktadır.

Bu yapı sayesinde asset yönetimi, tarama işlemleri ve tarama geçmişi aynı sistem içerisinde birbiriyle ilişkili şekilde yürütülmektedir.
