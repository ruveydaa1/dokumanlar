# Tarama Sonuçlarının İşlenmesi

SPM projesinde Semgrep ve Trivy tarafından gerçekleştirilen taramalar sonucunda elde edilen bulguların merkezi olarak yönetilebilmesi için ortak bir sonuç işleme yapısı oluşturulmuştur.

Her iki tarama aracı farklı türde güvenlik analizleri gerçekleştirse de sonuçlar uygulama içerisinde ortak `ScanResult` ve `ScanFinding` modelleri üzerinden taşınır. Böylece tarama aracından bağımsız olarak sonuçların aynı yapı içerisinde değerlendirilmesi sağlanır.

Tarama işlemleri tamamlandığında sonuçlar ScannerService tarafından HTTP aracılığıyla UserManagement servisine iletilir. UserManagement tarafında sonuçlar karşılanır, ilgili tarama kaydı güncellenir ve tespit edilen bulgular veritabanına kaydedilir.

### 1. Tarama Sonucunun Hazırlanması

SemgrepService veya TrivyService içerisinde gerçekleştirilen tarama işlemi tamamlandığında sonuçlar `ScanResult` nesnesi içerisinde hazırlanır.

Bu nesne içerisinde aşağıdaki bilgiler bulunur:

* `AssetId`: Taramanın gerçekleştirildiği Asset'in kimliği.
* `ScanRunId`: İlgili tarama kaydının kimliği.
* `Success`: Tarama işleminin başarılı olup olmadığını belirtir.
* `Scanner`: Kullanılan tarama aracını belirtir.
* `Source`: Taranan repository adresini içerir.
* `Output`: Tarama çıktısı için kullanılan alandır.
* `Error`: İşlem sırasında oluşan hata bilgisini içerir.
* `Findings`: Tarama sonucunda tespit edilen güvenlik bulgularını içerir.

Bu ortak yapı sayesinde Semgrep ve Trivy sonuçları aynı işleme sürecine dahil edilebilir.

### 2. ScannerService Üzerinden Sonucun İletilmesi

ScannerService, tarama sonucunu ilgili tarama servisinden aldıktan sonra UserManagement servisine iletir.

Bu işlemde `IScanResultSender` interface'i ve bu interface'i uygulayan `ScanResultSender` sınıfı kullanılır.

`ScanResultSender`, hazırlanan `ScanResult` nesnesini HTTP isteği içerisinde JSON formatında UserManagement servisine gönderir.

Sonuçların iletimi Kafka yerine HTTP üzerinden gerçekleştirildiği için tarama isteğinin ve tarama sonucunun aktarım yöntemleri birbirinden ayrılmıştır.

### 3. ScanRunController ile Sonucun Karşılanması

UserManagement içerisinde bulunan `ScanRunController`, ScannerService tarafından gönderilen tarama sonuçlarını karşılayan API kontrol sınıfıdır.

Sonuç aktarımı `POST api/scan-runs/result` uç noktası üzerinden gerçekleştirilir.

Bu uç nokta, tarama servislerinin UserManagement'a sonuç gönderebilmesi amacıyla `AllowAnonymous` olarak tanımlanmıştır. İsteklerin korunması ise servisler arasında kullanılan dahili API anahtarı mekanizmasıyla sağlanır.

Controller, gelen `ScanResult` nesnesini ilgili servis katmanına ileterek sonuçların işlenmesini başlatır.

### 4. Tarama Kaydının Güncellenmesi

Tarama sonuçlarının işlenmesinde `ScanRunService` ve ilgili repository yapısı kullanılır.

Sonuç içerisindeki `ScanRunId` bilgisi aracılığıyla ilgili tarama kaydı bulunur.

Tarama başarılıysa ilgili `ScanRun` kaydının durumu `Completed` olarak güncellenir. Tarama başarısızsa durum `Failed` olarak işaretlenir ve hata bilgisi kaydedilir.

Ayrıca taramanın tamamlanma zamanı `CompletedAt` alanına yazılır.

Bu sayede her tarama işleminin başlangıç ve tamamlanma bilgileri, kullanılan tarayıcı ve işlem sonucu veritabanında takip edilebilir.

### 5. Bulguların Veritabanına Kaydedilmesi

Tarama başarılı olduğunda `ScanResult` içerisindeki `Findings` listesi işlenerek veritabanına kaydedilir.

Semgrep ve Trivy tarafından üretilen bulgular önceden ortak `ScanFinding` modeline dönüştürüldüğü için UserManagement tarafında tarama aracına özel ayrı kayıt süreçlerine ihtiyaç duyulmaz.

Bulgular `Finding` entity yapısı üzerinden `findings` tablosuna kaydedilir.

Her bulgu, ilgili Asset ve ScanRun kayıtlarıyla ilişkilendirilir. Böylece bir güvenlik bulgusunun hangi Asset üzerinde ve hangi tarama işlemi sonucunda tespit edildiği takip edilebilir.

Bulgular içerisinde dosya yolu, satır ve sütun bilgileri, mesaj, kategori, önem derecesi, CWE, OWASP ve zafiyet sınıfı gibi bilgiler tutulur. Trivy kaynaklı bulgularda ayrıca paket adı, kurulu sürüm ve düzeltilebilecek sürüm gibi bağımlılık bilgileri de kullanılabilir.

### 6. Zamanlanmış Tarama Sonuçlarının Güncellenmesi

SPM içerisinde manuel taramaların yanında zamanlanmış taramalar da desteklenmektedir.

Zamanlanmış taramalar sonucunda oluşturulan `ScanRun` kaydı, ilgili `ScheduledScan` kaydıyla ilişkilendirilir.

Tarama sonucu UserManagement'a ulaştığında ilgili zamanlanmış tarama kaydı da güncellenir. İşlem başarılıysa tamamlanma durumu, başarısızsa hata bilgisi ilgili kayda aktarılır.

Bu ilişki sayesinde zamanlanmış taramanın hangi ScanRun işlemini oluşturduğu ve işlemin nasıl sonuçlandığı takip edilebilir.

### 7. Tarama Sonuçlarının Takip Edilmesi

Tarama sonuçları veritabanına kaydedildikten sonra uygulama içerisindeki ilgili ekranlarda görüntülenebilir.

`ScanRun` kayıtları üzerinden Asset bazında tarama geçmişi takip edilebilir. `Finding` kayıtları üzerinden ise taramalarda tespit edilen güvenlik bulguları listelenebilir.

Bu yapı sayesinde kullanıcılar geçmiş taramaları inceleyebilir, tarama durumlarını görebilir ve tespit edilen güvenlik sorunlarını değerlendirebilir.

### 8. Genel Değerlendirme

SPM projesinde tarama sonuçlarının işlenmesi, farklı tarama araçlarından elde edilen verilerin ortak bir yapıda toplanmasını sağlar.

ScannerService'in sonuçları HTTP üzerinden UserManagement'a iletmesi, `ScanRunController` aracılığıyla sonuçların karşılanması ve ilgili kayıtların veritabanında güncellenmesi sayesinde tarama süreçleri merkezi olarak yönetilir.

Bulguların Asset ve ScanRun kayıtlarıyla ilişkilendirilmesi, güvenlik sorunlarının hangi proje ve tarama işlemi sonucunda ortaya çıktığının takip edilmesini sağlar.
