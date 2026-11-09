# Trivy Entegrasyonu

Trivy, yazılım projelerinde kullanılan açık kaynaklı bileşenleri, kütüphaneleri ve paketleri güvenlik açıkları açısından taramak amacıyla kullanılan bir güvenlik tarama aracıdır. SPM projesinde Trivy, özellikle yazılım bağımlılıklarından kaynaklanabilecek güvenlik açıklarının tespit edilmesi amacıyla entegre edilmiştir.

Semgrep kaynak kod içerisindeki güvenlik sorunlarına odaklanırken Trivy, repository içerisinde kullanılan paket ve bağımlılıkları analiz ederek bilinen güvenlik açıklarını tespit eder. Bu iki aracın birlikte kullanılmasıyla uygulamada farklı güvenlik analizlerinin gerçekleştirilmesi sağlanır.

Trivy entegrasyonu da Semgrep gibi ayrı bir servis yapısı üzerinden tasarlanmıştır. Trivy işlemleri TrivyService içerisinde gerçekleştirilirken ScannerService, tarama isteğinin yönlendirilmesi ve sonucun ana uygulamaya iletilmesi görevlerini üstlenir.

### 1. Tarama İsteğinin Oluşturulması ve İletilmesi

Kullanıcı, uygulama arayüzünden bir Asset için Trivy taraması başlattığında istek UserManagement içerisindeki `AssetsController` sınıfına ulaşır.

Controller, tarama isteğini kontrol ettikten sonra ilgili Asset için bir `ScanRun` kaydı oluşturur. Bu kayıtta Asset kimliği, tarayıcı bilgisi, repository adresi, başlangıç zamanı ve tarama durumu tutulur.

Ardından `KafkaScanProducer` aracılığıyla tarama isteği Kafka üzerindeki `asset-scan` konusuna gönderilir.

### 2. ScannerService Tarafından İsteğin Karşılanması

ScannerService içerisindeki `KafkaScanConsumer`, Kafka üzerinden gelen tarama isteğini karşılar ve mesajı `ScanRequest` nesnesine dönüştürür.

İstek içerisindeki tarayıcı bilgisi kontrol edilir. Tarayıcı olarak Trivy seçilmişse ScannerService, `ITrivyClient` üzerinden `TrivyClient` sınıfını kullanarak TrivyService'e isteği iletir.

Bu aşamada taramanın başlangıç ve devam durumları Kafka üzerinden ilerleme mesajları olarak gönderilir.

### 3. TrivyService ile İletişim

ScannerService içerisinde bulunan `ITrivyClient`, Trivy tarama isteğinin nasıl gerçekleştirileceğini tanımlayan interface'tir. Bu interface'i uygulayan `TrivyClient`, TrivyService ile HTTP üzerinden iletişim kurar.

TrivyService tarafında gelen HTTP isteği `TrivyController` tarafından karşılanır. Controller, tarama isteğini `ITrivyService` üzerinden tarama işlemini gerçekleştiren servis sınıfına iletir.

TrivyService, kendi API'si üzerinden gelen dahili isteklere yönelik API anahtarı kontrolü uygular. Böylece servisler arasındaki iletişimde yetkisiz isteklerin engellenmesi amaçlanır.

### 4. Repository'nin Trivy ile Taranması

TrivyService içerisinde `ITrivyService` interface'ini uygulayan `TrivyScanService`, tarama işleminin gerçekleştirilmesinden sorumludur.

Tarama işleminde Docker üzerinden Trivy aracı çalıştırılır. `aquasec/trivy:latest` imajı kullanılarak repository taraması gerçekleştirilir.

Tarama komutunda kullanılan `repo` parametresi, Trivy'nin bir repository üzerinde analiz yapmasını sağlar. `--format json` parametresiyle tarama sonuçları JSON formatında üretilir.

Bu işlem sonucunda repository içerisinde kullanılan bağımlılıklar ve bu bağımlılıklarla ilişkili bilinen güvenlik açıkları tespit edilir.

### 5. Tarama Sonuçlarının Ortak Formata Dönüştürülmesi

Trivy tarafından üretilen JSON çıktısı işlenerek uygulamanın ortak bulgu modeli olan `ScanFinding` nesnelerine dönüştürülür.

Trivy çıktısının `Results` alanı içerisindeki `Vulnerabilities` kayıtları incelenir. Her bir güvenlik açığı için ilgili bilgiler alınarak ortak bulgu yapısına aktarılır.

Bu kapsamda bulgunun kimliği, başlığı, etkilenen paket, kurulu sürüm, düzeltilmiş sürüm, önem derecesi ve referans adresi gibi bilgiler değerlendirilir.

Trivy bulguları `SCA` kategorisiyle işaretlenir. Kimlik türü ise bulgunun tanımlayıcısına göre CVE, GHSA veya diğer türler şeklinde belirlenebilir.

Dönüştürülen bulgular `ScanResult` nesnesinin `Findings` alanına eklenir. Böylece Trivy sonuçları da Semgrep sonuçlarıyla aynı genel veri yapısı üzerinden taşınabilir.

### 6. Sonuçların UserManagement Servisine İletilmesi

TrivyService tarafından hazırlanan tarama sonucu HTTP yanıtı olarak ScannerService'e döndürülür.

ScannerService içerisindeki `TrivyClient`, dönen yanıtı `ScanResult` nesnesi olarak alır. Ardından `IScanResultSender` ve `ScanResultSender` aracılığıyla sonuç UserManagement servisine HTTP üzerinden iletilir.

UserManagement tarafında `ScanRunController` gelen sonucu karşılar. İlgili `ScanRun` kaydı güncellenir ve tarama başarılıysa elde edilen bulgular veritabanına kaydedilir. Tarama başarısızsa hata bilgisi ilgili kayda aktarılır.

### 7. Semgrep ve Trivy Sonuçlarının Ortak Yapıda Yönetilmesi

SPM içerisinde Semgrep ve Trivy farklı güvenlik analizleri gerçekleştirse de sonuçları ortak veri modelleri üzerinden taşınır.

Semgrep kaynak kod içerisindeki güvenlik sorunlarını tespit ederken Trivy yazılım bağımlılıklarını ve bu bağımlılıklarla ilişkili bilinen güvenlik açıklarını inceler.

Her iki tarayıcıdan elde edilen sonuçların `ScanFinding` ve `ScanResult` modellerine dönüştürülmesi sayesinde bulguların UserManagement tarafında ortak bir yapı üzerinden işlenmesi ve görüntülenmesi sağlanır.

Bu yaklaşım, uygulamaya farklı tarama araçlarının eklenmesini kolaylaştırırken tarama sonuçlarının yönetiminde de tutarlılık sağlar.

### 8. Genel Değerlendirme

Trivy entegrasyonu sayesinde SPM, repository içerisinde kullanılan yazılım bağımlılıklarını güvenlik açıkları açısından analiz edebilir. Tarama işlemlerinin ayrı bir TrivyService üzerinden gerçekleştirilmesi, ana uygulamanın tarama aracından bağımsız çalışmasını sağlar.

Kafka ile tarama isteği ve ilerleme bilgilerinin aktarılması, HTTP ile servisler arasında istek ve sonuç iletişiminin sağlanması ve ortak bulgu modellerinin kullanılması sayesinde Trivy, mevcut tarama mimarisine entegre edilmiştir.
