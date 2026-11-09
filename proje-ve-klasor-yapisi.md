# Proje Ve Klasör Yapısı

SPM (Software Project Management) uygulaması, sürdürülebilirlik, modülerlik ve geliştirilebilirlik esas alınarak birden fazla projeye ayrılmış bir yazılım mimarisiyle geliştirilmiştir. Uygulama; kullanıcı ve varlık yönetimi, güvenlik taramalarının yönetilmesi, tarama işlemlerinin yürütülmesi ve tarama sonuçlarının işlenmesi gibi farklı sorumluluklara sahip servislerden oluşmaktadır.

Uygulama içerisinde UserManagement, ScannerService, SemgrepService ve TrivyService olmak üzere dört temel backend projesi bulunmaktadır. Her proje kendi sorumluluk alanına göre katmanlara ve klasörlere ayrılmıştır.

Bu mimaride temel amaç, kullanıcı işlemleri ile güvenlik tarama işlemlerini birbirinden ayırmak, farklı güvenlik tarayıcılarının bağımsız servisler olarak çalışmasını sağlamak ve ileride sisteme yeni tarama araçlarının eklenmesini kolaylaştırmaktır.

### 6.1. UserManagement Projesi

UserManagement, uygulamanın temel yönetim servisidir. Kullanıcıların sisteme giriş yapması, kullanıcı ve rol işlemlerinin gerçekleştirilmesi, taranacak yazılım varlıklarının yönetilmesi, tarama işlemlerinin oluşturulması ve sonuçların veritabanına kaydedilmesi gibi işlemler bu proje üzerinden yürütülmektedir.

Proje, Domain, Application ve Infrastructure katmanlarından oluşmaktadır. Ayrıca API tarafındaki HTTP isteklerini karşılayan Controllers klasörü bulunmaktadır.

#### 6.1.1. Domain Katmanı

Domain katmanı, uygulamanın temel iş nesnelerini ve veri modellerini içermektedir. Uygulamada kullanılan temel entity sınıfları bu katmanda tanımlanmıştır.

**Entities klasörü içerisinde bulunan sınıflar:**

* **Asset.cs:** Sisteme eklenen ve güvenlik taramasına tabi tutulacak yazılım projelerini temsil eder. Proje adresi, adı ve ilgili varlık bilgilerini tutar.
* **Finding.cs:** Güvenlik taramaları sonucunda tespit edilen güvenlik bulgularını temsil eder. Bulgunun önem derecesi, açıklaması, dosya bilgileri ve diğer güvenlik detayları bu sınıfta tutulur.
* **Role.cs:** Sistemde tanımlanan kullanıcı rollerini temsil eder. Kullanıcıların yetkilendirilmesinde kullanılan rol bilgilerini içerir.
* **ScanRun.cs:** Gerçekleştirilen her bir tarama işlemini temsil eder. Taramanın hangi varlık üzerinde yapıldığı, hangi tarayıcının kullanıldığı, başlangıç ve tamamlanma zamanı gibi bilgiler tutulur.
* **ScheduledScan.cs:** İleri bir tarih ve saate planlanan tarama işlemlerini temsil eder. Planlanan taramanın zamanı, durumu ve ilişkili tarama kaydı gibi bilgiler bulunur.
* **User.cs:** Sistemdeki kullanıcıları temsil eder. Kullanıcıların kimlik, e-posta ve hesap bilgileri gibi verilerini içerir.
* **UserRole.cs:** Kullanıcılar ile roller arasındaki ilişkiyi temsil eder. Bir kullanıcının hangi role sahip olduğunu belirlemek için kullanılır.

#### 6.1.2. Application Katmanı

Application katmanı, uygulamanın iş akışlarının yönetildiği katmandır. Domain nesneleri üzerinde gerçekleştirilecek işlemler, servis arayüzleri ve veri aktarım modelleri bu katmanda tanımlanmıştır.

**DTOs klasörü**

DTO (Data Transfer Object) sınıfları, katmanlar ve API arasında veri aktarımını düzenlemek amacıyla kullanılmaktadır. İstek ve yanıt modellerinin birbirinden ayrılmasını sağlayarak doğrudan entity nesnelerinin dışarıya açılmasını önler.

Ana DTO klasöründe bulunan sınıflar:

* **CreateUserRequest.cs:** Yeni kullanıcı oluşturulurken gönderilen bilgileri tanımlar.
* **ForgotPasswordRequest.cs:** Şifresini unutan kullanıcının şifre yenileme sürecini başlatmak için gönderdiği bilgileri içerir.
* **LoginRequest.cs:** Kullanıcının giriş yaparken gönderdiği kimlik bilgilerini tanımlar.
* **LoginResponse.cs:** Başarılı giriş işlemi sonucunda döndürülen oturum ve kullanıcı bilgilerini içerir.
* **ResetPasseordRequest.cs:** Şifre yenileme işlemi sırasında gönderilen bilgileri tanımlar.
* **UpdateUserRequest.cs:** Mevcut kullanıcı bilgilerinin güncellenmesi için kullanılır.
* **UserResponse.cs:** Kullanıcı bilgilerini API yanıtlarında sunmak için kullanılır.

**Assets klasörü**

Varlık yönetimi işlemlerinde kullanılan DTO sınıflarını içerir.

* **AssetResponse.cs:** Varlık bilgilerini istemciye döndürmek için kullanılır.
* **CreateAssetRequest.cs:** Yeni bir varlık eklemek için gerekli bilgileri tanımlar.
* **UpdateAssetRequest.cs:** Mevcut varlık bilgilerinin güncellenmesi için kullanılır.

**Scan klasörü**

Tarama işlemleri ve tarama sonuçlarının aktarılmasında kullanılan DTO sınıflarını içerir.

* **ScanFindingDto.cs:** Tarama sonucunda elde edilen güvenlik bulgularının aktarılmasını sağlar.
* **ScanRequest.cs:** Yeni bir tarama işlemi başlatılırken gerekli bilgileri tanımlar.
* **ScanResponse.cs:** Tarama isteğine karşılık döndürülen yanıt bilgilerini içerir.
* **ScanResult.cs:** Tarama işlemi sonucunda oluşan verileri ve işlem durumunu temsil eder.
* **ScanRunDto.cs:** Tarama geçmişine ait bilgilerin aktarılmasını sağlar.
* **ScanRunResponse.cs:** Tarama işlemlerine ilişkin API yanıtlarını düzenler.
* **ScheduledScanRequest.cs:** Zamanlanmış tarama oluşturulurken kullanılacak istek bilgilerini tanımlar.

**Interfaces klasörü**

Application katmanındaki servislerin ve veri erişim işlemlerinin hangi metotları sunacağını belirleyen arayüzleri içerir. Bu sayede iş mantığı ile ilgili sınıfların somut uygulamalara doğrudan bağımlı olması önlenir.

* **IAssetRespository.cs:** Varlık verilerinin veritabanından alınması ve yönetilmesi için gerekli veri erişim sözleşmesini tanımlar.
* **IAssetService.cs:** Varlık yönetimi işlemlerini tanımlar.
* **IAuthService.cs:** Kullanıcı kimlik doğrulama işlemlerini tanımlar.
* **IEmailService.cs:** E-posta gönderme işlemlerini tanımlar.
* **IFindingRepository.cs:** Güvenlik bulgularına ilişkin veri erişim işlemlerini tanımlar.
* **IFindingService.cs:** Güvenlik bulgularının yönetilmesini sağlayan işlemleri tanımlar.
* **IJwtService.cs:** JWT token oluşturma ve token işlemlerine ilişkin sözleşmeyi tanımlar.
* **IScanRunRepository.cs:** Tarama kayıtlarına ilişkin veritabanı işlemlerini tanımlar.
* **IScanRunService.cs:** Tarama kayıtlarının yönetilmesini sağlayan işlemleri tanımlar.
* **IUserRepository.cs:** Kullanıcı verilerine erişmek için gerekli metotları tanımlar.
* **IUserService.cs:** Kullanıcı yönetimi işlemlerini tanımlar.

**Services klasörü**

Application katmanında tanımlanan servis arayüzlerinin iş mantığını gerçekleştiren sınıfları içerir.

* **AssetService.cs:** Varlık ekleme, listeleme, güncelleme ve yönetim işlemlerini gerçekleştirir.
* **AuthService.cs:** Kullanıcı giriş ve kimlik doğrulama süreçlerini yönetir.
* **FindingService.cs:** Güvenlik taramaları sonucunda oluşan bulguların yönetilmesini sağlar.
* **ScanRunService.cs:** Tarama kayıtlarının oluşturulması, sorgulanması ve yönetilmesinden sorumludur.
* **UserService.cs:** Kullanıcı oluşturma, listeleme ve güncelleme gibi kullanıcı yönetimi işlemlerini gerçekleştirir.

#### 6.1.3. Infrastructure Katmanı

Infrastructure katmanı, uygulamanın dış sistemlerle ve teknik altyapıyla iletişim kurmasını sağlayan bileşenleri içerir. Veritabanı bağlantısı, repository uygulamaları, e-posta gönderimi, JWT işlemleri ve Kafka mesajlaşma mekanizmaları bu katmanda yer almaktadır.

**Hubs klasörü**

* **ScanProgressHub.cs:** SignalR kullanılarak tarama işlemlerinin ilerleme bilgilerinin istemciye gerçek zamanlı olarak iletilmesini sağlar.

**Persistence klasörü**

* **AppDbContext.cs:** Entity Framework Core aracılığıyla uygulamanın veritabanı bağlantısını ve entity sınıflarının veritabanındaki karşılıklarını yönetir. Veritabanı tabloları arasındaki ilişkilerin tanımlanmasında da kullanılır.

**Repositories klasörü**

Veritabanına erişim işlemlerinin gerçekleştirildiği sınıfları içerir.

* **AssetRepository.cs:** Varlık kayıtlarıyla ilgili veritabanı işlemlerini gerçekleştirir.
* **FindingRepository.cs:** Güvenlik bulgularına ilişkin veritabanı işlemlerini gerçekleştirir.
* **ScanRunRepository.cs:** Tarama geçmişi ve tarama kayıtlarına ilişkin veritabanı işlemlerini gerçekleştirir.
* **UserRepository.cs:** Kullanıcı verilerine ilişkin veritabanı işlemlerini gerçekleştirir.

**Services klasörü**

Uygulamanın teknik servislerini ve arka plan görevlerini içerir.

* **EmailService.cs:** Kullanıcılara e-posta gönderilmesini sağlar. Özellikle şifre sıfırlama süreçlerinde kullanılır.
* **JwtService.cs:** Kullanıcıların kimlik doğrulama işlemleri sonucunda JWT token oluşturulmasını sağlar.
* **KafkaScanProducer.cs:** Tarama isteklerinin Kafka üzerinden ilgili tarama sistemine iletilmesini sağlar.
* **KafkaScanProgressConsumer.cs:** Kafka üzerinden gelen tarama ilerleme mesajlarını tüketerek uygulamanın ilgili bileşenlerine aktarır.
* **ScheduledScanJob.cs:** Hangfire tarafından çalıştırılan zamanlanmış tarama görevlerini yönetir. Planlanan tarih ve saat geldiğinde ilgili tarama işleminin başlatılmasını sağlar.

#### 6.1.4. Controllers

Controllers klasörü, istemcilerden gelen HTTP isteklerini karşılayan ve ilgili Application servislerine yönlendiren API kontrol sınıflarını içerir.

* **AssetController.cs:** Yazılım varlıklarının eklenmesi, listelenmesi, güncellenmesi ve tarama işlemlerinin başlatılmasıyla ilgili API uç noktalarını yönetir.
* **AuthController.cs:** Kullanıcı giriş, şifre unutma ve şifre sıfırlama gibi kimlik doğrulama işlemlerini yönetir.
* **FindingController.cs:** Güvenlik bulgularının sorgulanması ve istemciye sunulmasını sağlar.
* **HealthController.cs:** Uygulamanın sağlık durumunun kontrol edilmesini sağlayan endpoint'leri içerir.
* **ScanRunController.cs:** Tarama kayıtlarının ve tarama sonuçlarının yönetilmesini sağlar. Tarama servislerinden gelen sonuçların işlenmesinde de görev alır.
* **UsersController.cs:** Kullanıcı oluşturma, listeleme ve kullanıcı bilgilerinin güncellenmesi gibi işlemleri yönetir.

### 6.2. ScannerService Projesi

ScannerService, güvenlik tarama işlemlerinin yönetilmesi ve ilgili tarama servislerine yönlendirilmesinden sorumludur.

UserManagement tarafından oluşturulan tarama istekleri Kafka aracılığıyla ScannerService'e iletilir. ScannerService bu mesajları tüketerek isteğin türüne göre SemgrepService veya TrivyService ile iletişim kurar. Tarama işlemlerinin ilerleme ve sonuç bilgileri de ilgili mesajlaşma mekanizmaları üzerinden sisteme aktarılır.

#### 6.2.1. Application Katmanı

**Interfaces klasörü**

Servisin diğer bileşenlerle iletişim kurarken kullanacağı arayüzleri içerir.

* **IScanResultSender.cs:** Tarama sonuçlarının ilgili sisteme iletilmesini sağlayan arayüzdür.
* **ISemgrepClient.cs:** SemgrepService ile HTTP üzerinden iletişim kurulmasını sağlayan arayüzdür.
* **ITrivyClient.cs:** TrivyService ile HTTP üzerinden iletişim kurulmasını sağlayan arayüzdür.

#### 6.2.2. Controllers

* **ScanProgressController.cs:** Tarama işlemlerine ilişkin ilerleme bilgilerinin API üzerinden alınmasını sağlayan kontrol sınıfıdır.

#### 6.2.3. Domain Katmanı

**Models klasörü**

Tarama işlemleri sırasında kullanılan temel veri modellerini içerir.

* **ScanFinding.cs:** Tarama sonucunda tespit edilen güvenlik bulgularını temsil eder.
* **ScanProgress.cs:** Tarama işleminin ilerleme durumunu ve aşamalarını temsil eder.
* **ScanRequest.cs:** Tarama servislerine iletilen tarama isteğini temsil eder.
* **ScanResult.cs:** Tarama işleminin sonucunu, durumunu ve elde edilen bulguları temsil eder.

#### 6.2.4. Infrastructure Katmanı

ScannerService'in dış sistemlerle iletişim kurduğu ve tarama işlemlerini yönettiği teknik bileşenleri içerir.

**Http klasörü**

* **ScanResultSender.cs:** Tarama sonuçlarının HTTP üzerinden UserManagement API'sine iletilmesini sağlar.

**Kafka klasörü**

* **KafkaScanConsumer.cs:** UserManagement tarafından Kafka'ya gönderilen tarama isteklerini tüketerek ilgili tarama işleminin başlatılmasını sağlar.
* **KafkaScanProgressProducer.cs:** Tarama ilerleme bilgilerinin Kafka üzerinden diğer bileşenlere gönderilmesini sağlar.

**Semgrep klasörü**

* **SemgrepClient.cs:** ScannerService ile SemgrepService arasındaki HTTP iletişimini gerçekleştirir ve Semgrep tarama işlemlerini ilgili servise yönlendirir.

**Trivy klasörü**

* **TrivyClient.cs:** ScannerService ile TrivyService arasındaki HTTP iletişimini gerçekleştirir ve Trivy tarama işlemlerini ilgili servise yönlendirir.

### 6.3. SemgrepService Projesi

SemgrepService, uygulamada Semgrep güvenlik tarayıcısının çalıştırılmasından sorumlu bağımsız bir mikroservistir.

ScannerService tarafından gönderilen tarama isteklerini alır, Semgrep aracını kullanarak kaynak kod üzerinde güvenlik analizi gerçekleştirir ve elde edilen sonuçları ortak tarama veri modellerine uygun şekilde hazırlar.

#### 6.3.1. Application Katmanı

**Interfaces klasörü**

* **IScannerService.cs:** Tarama işlemlerinin yürütülmesi için gerekli genel tarayıcı arayüzünü tanımlar.
* **ISemgrepService.cs:** Semgrep tarama işlemlerinin gerçekleştirilmesini sağlayan servis arayüzünü tanımlar.

#### 6.3.2. Controllers

* **SemgrepController.cs:** ScannerService tarafından gönderilen HTTP tarama isteklerini karşılar ve ilgili Semgrep servis metodunu çalıştırır.

#### 6.3.3. Domain Katmanı

**Models klasörü**

* **ScanFinding.cs:** Semgrep tarafından tespit edilen güvenlik bulgularını temsil eder.
* **ScanProgress.cs:** Tarama işleminin ilerleme durumunu temsil eder.
* **ScanRequest.cs:** Semgrep tarama işlemi için gelen istek bilgilerini içerir.
* **ScanResult.cs:** Tarama sonucunda elde edilen bulguları ve işlem durumunu temsil eder.

#### 6.3.4. Infrastructure Katmanı

**Semgrep klasörü**

* **SemgrepScanService.cs:** Semgrep aracının çalıştırılmasını, tarama işleminin yürütülmesini ve sonuçların elde edilmesini sağlayan temel servis sınıfıdır.

### 6.4. TrivyService Projesi

TrivyService, uygulamada Trivy güvenlik tarayıcısının çalıştırılmasından sorumlu bağımsız mikroservistir.

ScannerService tarafından iletilen tarama isteklerini karşılar. Trivy aracını kullanarak ilgili yazılım deposunu analiz eder ve özellikle yazılım bağımlılıklarında bulunan güvenlik açıklarını tespit eder. Elde edilen sonuçları uygulamanın ortak tarama veri modellerine uygun biçimde hazırlar.

#### 6.4.1. Application Katmanı

**Interfaces klasörü**

* **ITrivyService.cs:** Trivy tarama işlemlerini gerçekleştirecek servisin uygulaması gereken metotları tanımlar.

#### 6.4.2. Controllers

* **TrivyController.cs:** ScannerService tarafından gönderilen Trivy tarama isteklerini karşılar ve ilgili servis metoduna yönlendirir.

#### 6.4.3. Domain Katmanı

**Models klasörü**

* **ScanFinding.cs:** Trivy tarafından tespit edilen güvenlik açıklarını temsil eder.
* **ScanProgress.cs:** Tarama işleminin ilerleme bilgilerini temsil eder.
* **ScanRequest.cs:** Trivy tarama işlemi için gerekli istek bilgilerini içerir.
* **ScanResult.cs:** Trivy taraması sonucunda elde edilen bulguları ve işlem durumunu temsil eder.

#### 6.4.4. Infrastructure Katmanı

**Trivy klasörü**

* **TrivyScanService.cs:** Trivy aracının çalıştırılmasını, yazılım deposunun analiz edilmesini ve güvenlik açıklarının sonuç olarak elde edilmesini sağlar.

### 6.5. Projeler Arasındaki Genel Mimari İlişki

SPM uygulamasında her servis belirli bir sorumluluğu üstlenmektedir. UserManagement kullanıcı, varlık ve tarama yönetimini gerçekleştirirken ScannerService tarama isteklerini ilgili tarayıcı servislerine yönlendirir. SemgrepService ve TrivyService ise kendi güvenlik araçlarını çalıştırarak analiz işlemlerini gerçekleştirir.

Bu yapı sayesinde tarama araçları ana yönetim uygulamasından ayrılmış ve bağımsız servisler olarak konumlandırılmıştır. Böylece yeni tarama araçlarının sisteme eklenmesi, mevcut yönetim mekanizmasının büyük ölçüde değiştirilmesine gerek kalmadan gerçekleştirilebilir.

<br>
