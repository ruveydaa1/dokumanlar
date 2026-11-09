# Kafka ile Mesajlaşma Mimarisi

SPM projesinde Kafka, servisler arasında asenkron mesajlaşma sağlamak ve tarama işlemlerinin ana uygulamadan bağımsız şekilde yürütülmesini desteklemek amacıyla kullanılmaktadır.

Tarama işlemlerinin uzun sürebilmesi nedeniyle kullanıcıdan gelen istekler doğrudan tarama servislerine senkron olarak aktarılmak yerine Kafka üzerinden iletilir. Böylece UserManagement servisi tarama işleminin tamamlanmasını beklemeden isteği kuyruğa gönderebilir ve tarama süreci arka planda devam edebilir.

Projede Kafka; tarama isteklerinin ScannerService'e iletilmesi ve tarama ilerleme bilgilerinin UserManagement tarafına aktarılması amacıyla kullanılmaktadır. Tarama sonuçları ise Kafka yerine HTTP üzerinden iletilmektedir.

### 1. Kafka Altyapısı

Projede Kafka altyapısı Docker üzerinden çalıştırılmaktadır. Kafka servisi `apache/kafka:4.3.1` imajı kullanılarak oluşturulmuştur.

Kafka ile uygulama arasındaki bağlantı `9092` portu üzerinden sağlanır. Servislerin Kafka bağlantı adresi ve kullanılacak topic bilgileri yapılandırma dosyaları üzerinden belirlenir.

Kafka, mesajların ilgili topic'ler üzerinde tutulmasını ve bu mesajların consumer grupları tarafından alınmasını sağlar.

### 2. Kullanılan Kafka Topic'leri

Projede üç temel Kafka topic'i tanımlanmıştır:

| Topic                 | Görevi                                                                               |
| --------------------- | ------------------------------------------------------------------------------------ |
| `asset-scan`          | Tarama isteklerinin UserManagement tarafından ScannerService'e iletilmesini sağlar.  |
| `asset-scan-progress` | Tarama işlemlerinin ilerleme ve durum bilgilerinin aktarılmasını sağlar.             |
| `asset-scan-result`   | Önceki sonuç iletim mimarisinde tarama sonuçlarının aktarılması için kullanılmıştır. |

Mevcut mimaride tarama sonuçlarının iletimi HTTP üzerinden gerçekleştirildiğinden `asset-scan-result` topic'i aktif sonuç aktarım akışında kullanılmamaktadır.

### 3. Tarama İsteklerinin Kafka Üzerinden İletilmesi

Kullanıcı bir Asset için tarama başlattığında UserManagement içerisindeki `AssetsController`, öncelikle ilgili tarama kaydını oluşturur.

Ardından `KafkaScanProducer` sınıfı aracılığıyla tarama isteği `asset-scan` topic'ine gönderilir.

Mesaj içerisinde tarama işleminin gerçekleştirilebilmesi için gerekli bilgiler bulunur:

* Asset kimliği (`AssetId`)
* Tarama kaydı kimliği (`ScanRunId`)
* Repository adresi (`Source`)
* Kullanılacak tarayıcı (`Scanner`)

ScannerService içerisinde bulunan `KafkaScanConsumer`, `asset-scan` topic'ini dinler ve gelen mesajları karşılar.

Consumer tarafından alınan mesaj `ScanRequest` nesnesine dönüştürülür. İstek içerisindeki tarayıcı bilgisine göre Semgrep veya Trivy istemcisi çağrılır.

Bu yapı sayesinde tarama istekleri ana uygulamadan ayrılarak ScannerService tarafından arka planda işlenir.

### 4. Consumer Group Yapısı

Kafka mesajlarının servisler tarafından kontrollü şekilde tüketilmesi için consumer group yapısı kullanılmaktadır.

ScannerService, tarama isteklerini `scanner-service-workers` consumer group'u üzerinden tüketir.

Tarama ilerleme mesajlarının UserManagement tarafında tüketilmesi için ise `KafkaScanProgressConsumer` kullanılır.

Consumer group yapısı, Kafka mesajlarının ilgili tüketici grupları tarafından yönetilmesini sağlar. Böylece mesajların üretildiği ve tüketildiği servisler birbirinden bağımsız çalışabilir.

### 5. Tarama İlerleme Bilgilerinin Aktarılması

Tarama işleminin yalnızca başlangıç ve bitiş sonuçlarının değil, işlem sırasında oluşan durum bilgilerinin de takip edilebilmesi amacıyla `asset-scan-progress` topic'i kullanılmaktadır.

ScannerService içerisindeki `KafkaScanProgressProducer`, tarama sürecindeki ilerleme bilgilerini bu topic'e gönderir.

Tarama sürecinde kullanılan durum bilgileri şunlardır:

* `Started`: Tarama işleminin başlatıldığını belirtir.
* `Scanning`: Tarama işleminin devam ettiğini belirtir.
* `Completed`: Tarama işleminin başarıyla tamamlandığını belirtir.
* `Failed`: Tarama işleminin başarısız olduğunu belirtir.
* `Finished`: Tarama sürecinin sona erdiğini belirtir.

UserManagement içerisindeki `KafkaScanProgressConsumer`, bu mesajları tüketir ve ilerleme bilgilerini uygulama tarafına aktarır.

Bu yapı sayesinde kullanıcı, arayüz üzerinden tarama işleminin güncel durumunu takip edebilir.

### 6. SignalR ile İlerleme Bilgilerinin Arayüze Aktarılması

Kafka üzerinden alınan ilerleme mesajlarının kullanıcı arayüzüne gerçek zamanlı aktarılmasında SignalR teknolojisi kullanılmaktadır.

UserManagement içerisinde bulunan `KafkaScanProgressConsumer`, Kafka'dan aldığı ilerleme bilgilerini `ScanProgressHub` üzerinden ilgili istemcilere iletir.

Frontend tarafında bu bilgiler dinlenerek tarama durumunun arayüzde güncellenmesi sağlanır.

Böylece kullanıcı, tarama işlemi devam ederken sayfayı sürekli yenilemek zorunda kalmadan işlem durumunu takip edebilir.

### 7. Tarama Sonuçlarının İletimi

Projede ilk tasarlanan mimaride tarama sonuçlarının Kafka üzerinden UserManagement servisine aktarılması planlanmıştır. Bu amaçla `KafkaScanResultProducer` ve `KafkaScanResultConsumer` sınıfları oluşturulmuştur.

Ancak mevcut mimaride sonuç iletimi HTTP tabanlı yapıya dönüştürülmüştür.

ScannerService içerisinde bulunan `ScanResultSender`, tarama sonucunu HTTP isteğiyle UserManagement servisine gönderir. UserManagement tarafındaki `ScanRunController` ise bu isteği karşılayarak ilgili tarama kaydını günceller ve bulguların işlenmesini sağlar.

Bu nedenle `asset-scan-result` topic'i mevcut tarama sonuçlarının aktarımında kullanılmamaktadır.

### 8. Genel Mesajlaşma Akışı

SPM projesindeki Kafka mesajlaşma akışı genel olarak şu şekildedir:

1. UserManagement, tarama isteğini `KafkaScanProducer` ile `asset-scan` topic'ine gönderir.
2. ScannerService, `KafkaScanConsumer` ile tarama isteğini tüketir.
3. ScannerService, seçilen tarayıcıya göre SemgrepService veya TrivyService ile iletişim kurar.
4. Tarama sürecindeki durum bilgileri `KafkaScanProgressProducer` aracılığıyla `asset-scan-progress` topic'ine gönderilir.
5. UserManagement, `KafkaScanProgressConsumer` ile ilerleme mesajlarını tüketir.
6. İlerleme bilgileri `ScanProgressHub` üzerinden frontend'e aktarılır.
7. Tarama tamamlandığında ScannerService, sonucu HTTP üzerinden UserManagement servisine iletir.

### 9. Genel Değerlendirme

Kafka kullanımı sayesinde SPM içerisinde tarama istekleri ve ilerleme mesajları servisler arasında asenkron olarak taşınmaktadır.

Bu mimari, tarama işlemlerinin ana uygulamanın çalışma akışından ayrılmasını ve servislerin birbirinden bağımsız şekilde görev yapmasını sağlar.

Tarama istekleri ve ilerleme bilgilerinde Kafka, tarama sonuçlarının UserManagement servisine aktarılmasında ise HTTP kullanılmasıyla sistem içerisinde görevine uygun farklı iletişim yöntemleri birlikte değerlendirilmiştir.
