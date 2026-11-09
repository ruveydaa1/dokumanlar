# Semgrep Entegrasyonu

Semgrep, uygulamada kaynak kod içerisindeki güvenlik açıklarını ve yazılım güvenliği açısından risk oluşturabilecek kod yapılarını tespit etmek amacıyla kullanılan statik kod analiz aracıdır (SAST).

SPM projesinde Semgrep entegrasyonu ayrı bir servis olarak tasarlanmıştır. Tarama işlemleri doğrudan ana uygulama içerisinde gerçekleştirilmez. Bunun yerine UserManagement, ScannerService ve SemgrepService servisleri arasında görev paylaşımı yapılır. Bu yapı sayesinde tarama işlemleri ana uygulamanın işleyişinden ayrılarak bağımsız şekilde yürütülür.

### 1. Tarama İsteğinin Oluşturulması ve İletilmesi

Kullanıcı, uygulamanın arayüzünden bir Asset için Semgrep taraması başlattığında istek öncelikle UserManagement içerisindeki `AssetsController` sınıfına ulaşır.

Controller, taranacak Asset'in bilgilerini kontrol eder ve tarama işlemini temsil etmek üzere `ScanRun` kaydı oluşturur. Bu kayıtta Asset bilgisi, kullanılacak tarayıcı, repository adresi, taramayı başlatan kullanıcı, başlangıç zamanı ve tarama durumu tutulur.

Ardından `KafkaScanProducer` aracılığıyla tarama isteği Kafka üzerindeki `asset-scan` konusuna gönderilir. Böylece tarama işlemi ana uygulamadan ayrılarak arka planda yürütülmek üzere ScannerService'e iletilir.

### 2. ScannerService Tarafından İsteğin Karşılanması

ScannerService içerisinde bulunan `KafkaScanConsumer`, Kafka üzerindeki `asset-scan` konusunu dinler ve gelen tarama mesajlarını karşılar.

Mesaj, `ScanRequest` nesnesine dönüştürülür. Bu nesne içerisinde taranacak Asset'in kimliği, repository adresi, tarama numarası ve kullanılacak tarayıcı bilgisi bulunur.

Tarama işleminin başlangıcında ve devamında ilgili durum bilgileri Kafka üzerinden ilerleme mesajları olarak gönderilir. ScannerService, istek içerisinde belirtilen tarayıcıya göre ilgili istemciyi kullanır. Semgrep seçilmişse `ISemgrepClient` üzerinden `SemgrepClient` sınıfı çağrılır.

### 3. SemgrepService ile İletişim

`ISemgrepClient`, ScannerService içerisinde Semgrep tarama isteğinin nasıl gerçekleştirileceğini tanımlayan interface'tir. Bu interface'i uygulayan `SemgrepClient`, SemgrepService ile HTTP üzerinden iletişim kurar.

İstek gönderilmeden önce `SemgrepService:InternalApiKey` yapılandırma değerinden dahili API anahtarı alınır. Bu anahtar, HTTP isteğinin `X-Internal-Api-Key` başlığına eklenir.

Tarama isteği `POST /api/semgrep/scan` adresine JSON formatında gönderilir.

SemgrepService tarafında bu isteği `SemgrepController` karşılar. Controller, gelen `ScanRequest` nesnesini `ISemgrepService` üzerinden tarama işlemini gerçekleştiren servise iletir.

### 4. Repository'nin İndirilmesi ve Taranması

SemgrepService içerisinde `ISemgrepService` interface'ini uygulayan `SemgrepScanService`, tarama işleminin temel sorumluluğunu üstlenir.

İlk olarak repository adresinin geçerli olup olmadığı kontrol edilir. Ardından işletim sisteminin geçici dosya dizininde her tarama için benzersiz bir klasör oluşturulur.

Git komutu kullanılarak repository bu klasöre indirilir. Repository indirme işleminde `--depth 1` parametresi kullanılarak yalnızca en son commit geçmişini içeren kopya alınır.

Repository başarıyla indirildikten sonra Semgrep aracı çalıştırılır. Tarama işleminde:

* `--config=auto` parametresiyle uygun tarama kuralları belirlenir.
* `--json` parametresiyle tarama sonuçları JSON formatında üretilir.
* `--exclude=.git` parametresiyle `.git` klasörü tarama dışında bırakılır.

Bu işlem sonucunda repository içerisindeki güvenlik bulguları Semgrep tarafından tespit edilir ve JSON formatında çıktı oluşturulur.

### 5. Tarama Sonuçlarının Ortak Formata Dönüştürülmesi

Semgrep tarafından oluşturulan JSON çıktısı, `SemgrepScanService` içerisinde bulunan `ParseFindings` metodu tarafından işlenir.

Bu metot, JSON içerisindeki `results` alanını inceleyerek tespit edilen her bulguyu uygulamanın ortak veri modeli olan `ScanFinding` nesnesine dönüştürür.

Bulguların kimliği, başlığı, mesajı, dosya yolu, önem derecesi, güven seviyesi, etki bilgisi, CWE, OWASP, zafiyet sınıfı ve referans bilgileri bu aşamada alınır. Ayrıca bulgunun kaynak kod içerisindeki başlangıç ve bitiş satır-sütun bilgileri de kaydedilir.

Semgrep'e ait bulgular `SAST` kategorisiyle işaretlenir. Böylece farklı tarama araçlarından elde edilen sonuçların uygulama içerisinde ortak bir yapıda değerlendirilmesi sağlanır.

Dönüştürülen bulgular `ScanResult` nesnesinin `Findings` alanına eklenir. Tarama sonucu; Asset kimliği, ScanRun kimliği, kullanılan tarayıcı, repository adresi, işlem durumu ve bulgularla birlikte hazırlanır.

### 6. Sonuçların UserManagement Servisine İletilmesi

SemgrepService tarafından hazırlanan tarama sonucu HTTP yanıtı olarak ScannerService'e döndürülür.

ScannerService içerisindeki `SemgrepClient`, bu yanıtı `ScanResult` nesnesi olarak alır. Ardından `IScanResultSender` üzerinden `ScanResultSender` sınıfı kullanılarak sonuç UserManagement servisine HTTP aracılığıyla iletilir.

UserManagement tarafında bu isteği `ScanRunController` karşılar. Gelen sonuca göre ilgili `ScanRun` kaydının durumu güncellenir. Tarama başarılıysa bulgular veritabanına kaydedilir ve tarama tamamlanmış olarak işaretlenir. İşlem başarısızsa hata bilgisi kaydedilerek tarama başarısız olarak işaretlenir.

### 7. Hata Yönetimi ve Geçici Dosyaların Temizlenmesi

Semgrep entegrasyonunda repository adresinin geçersiz olması, Git ile repository'nin indirilememesi, Semgrep işleminin başarısız olması veya beklenmeyen bir hata oluşması gibi durumlar kontrol edilir.

Tarama sırasında oluşan hatalar `ScanResult` nesnesi üzerinden ilgili servise iletilir. Böylece tarama işleminin başarısız olma nedeni ana uygulamaya aktarılabilir.

Ayrıca `SemgrepScanService` içerisindeki `finally` bloğunda tarama için oluşturulan geçici repository klasörü silinir. Bu sayede tarama sonrasında geçici dosyaların sistemde gereksiz yere birikmesi önlenir.

### 8. Genel Değerlendirme

Semgrep entegrasyonu sayesinde SPM, Git repository adresleri üzerinden kaynak kod güvenlik taraması gerçekleştirebilir. Tarama işlemlerinin ayrı servisler üzerinden yürütülmesi, ana uygulama ile tarama süreçlerinin birbirinden ayrılmasını sağlar.

Kafka, tarama isteklerinin ScannerService'e iletilmesinde ve tarama ilerleme bilgilerinin aktarılmasında kullanılırken servisler arasındaki tarama isteği ve sonuç iletişiminde HTTP kullanılmaktadır.

Semgrep tarafından elde edilen bulgular ortak `ScanFinding` ve `ScanResult` modellerine dönüştürülerek UserManagement tarafına iletilir. Böylece tarama sonuçları uygulamanın diğer tarama ve bulgu yönetimi bileşenleriyle aynı yapı içerisinde değerlendirilebilir.
