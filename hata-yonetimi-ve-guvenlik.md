# Hata Yönetimi ve Güvenlik

SPM uygulamasında tarama işlemlerinin güvenilir şekilde yürütülmesi ve yetkisiz erişimlerin önlenmesi için çeşitli hata yönetimi ve güvenlik mekanizmaları kullanılmıştır. Bu mekanizmalar; kullanıcı kimlik doğrulaması, rol tabanlı yetkilendirme, servisler arası iletişim güvenliği ve tarama sırasında oluşabilecek hataların yönetilmesini kapsamaktadır.

### 1. JWT ile Kimlik Doğrulama

SPM uygulamasında kullanıcı kimlik doğrulaması JWT (JSON Web Token) üzerinden gerçekleştirilmektedir.

Kullanıcı giriş yaptığında kimlik bilgileri doğrulanır ve başarılı giriş sonucunda bir JWT oluşturulur. Bu token, kullanıcının kimliğini temsil eder ve sonraki API isteklerinde kullanılır.

Frontend tarafında token saklanır ve korumalı API isteklerinde `Authorization` başlığı içerisinde `Bearer` formatında gönderilir.

Backend, gelen token'ın geçerliliğini kontrol ederek kullanıcının kimliğini doğrular. Geçersiz veya süresi dolmuş token ile yapılan istekler reddedilir.

### 2. Rol Tabanlı Yetkilendirme

Uygulamada kullanıcıların gerçekleştirebileceği işlemler roller üzerinden sınırlandırılmıştır.

Sistemde aşağıdaki roller bulunmaktadır:

* **SuperAdmin:** Sistem genelinde yönetim yetkilerine sahip kullanıcı rolüdür.
* **Admin:** Yönetim işlemlerini gerçekleştirebilen kullanıcı rolüdür.
* **Developer:** Kendisine tanımlanan geliştirme ve tarama işlemlerini gerçekleştirebilen kullanıcı rolüdür.
* **User:** Daha sınırlı erişim yetkilerine sahip kullanıcı rolüdür.

Backend tarafında `[Authorize]` ve `[Authorize(Roles = "...")]` gibi yetkilendirme mekanizmaları kullanılarak endpoint'lere erişim kontrol edilir.

Bu sayede yalnızca giriş yapmış olmak yeterli değildir; kullanıcının gerçekleştirmek istediği işlem için gerekli role de sahip olması gerekir.

### 3. Servisler Arası API Key Güvenliği

SPM mimarisinde UserManagement, ScannerService, SemgrepService ve TrivyService gibi birbirleriyle iletişim kuran servisler bulunmaktadır.

Bu servislerin birbirlerine gönderdiği isteklerin yetkisiz kişiler tarafından kullanılmasını önlemek amacıyla internal API Key kontrolü uygulanmıştır.

ScannerService tarafından SemgrepService ve TrivyService'e gönderilen HTTP isteklerinde `X-Internal-Api-Key` başlığı kullanılır. İlgili servis, gelen istekteki anahtarı kendi yapılandırmasında bulunan değerle karşılaştırır.

Anahtar geçerli değilse istek işleme alınmaz. Böylece tarama servislerinin ilgili API endpoint'lerine yalnızca gerekli erişim bilgisine sahip servislerin ulaşması hedeflenmiştir.

### 4. API Erişim Kontrolleri

Backend tarafında controller ve endpoint seviyesinde yetkilendirme kontrolleri uygulanmıştır.

Örneğin Asset yönetimi gibi işlemlerde yalnızca belirli rollere sahip kullanıcıların erişimine izin verilmektedir. Böylece kullanıcıların yetkileri dışında asset oluşturması, güncellemesi, silmesi veya tarama başlatması engellenir.

Bunun yanında endpoint'lere gönderilen veriler üzerinde de doğrulamalar yapılır. Örneğin tarama başlatma işlemlerinde desteklenen tarama aracının seçilmesi ve kaynak adresinin geçerli olması kontrol edilir.

### 5. Tarama İşlemlerinde Hata Yönetimi

Tarama işlemleri birden fazla servis ve teknoloji üzerinden yürütüldüğü için farklı aşamalarda hatalar meydana gelebilir.

SPM uygulamasında bu hataların takip edilebilmesi için tarama kayıtlarında durum ve hata bilgileri tutulmaktadır.

Tarama işlemi sırasında hata oluştuğunda ilgili `ScanRun` kaydının durumu `Failed` olarak güncellenir. Hatanın açıklaması `Error` alanında saklanır ve tamamlanma zamanı kaydedilir.

SemgrepService ve TrivyService tarafında da tarama sırasında oluşabilecek hatalar ele alınarak sonuç nesnesi içerisinde hata bilgisi döndürülür. Böylece tarama başarısız olsa bile sonuç UserManagement uygulamasına iletilebilir ve ilgili kayıt güncellenebilir.

### 6. Kafka İletişiminde Hata Yönetimi

Tarama istekleri Kafka üzerinden ScannerService'e iletildiği için mesaj gönderimi ve tüketimi sırasında oluşabilecek hataların da yönetilmesi gerekir.

UserManagement tarafında Kafka'ya tarama isteği gönderilemediğinde ilgili `ScanRun` kaydı `Failed` durumuna getirilir. Hata bilgisi kaydedilerek taramanın neden başlatılamadığı takip edilebilir.

Bu yaklaşım, tarama isteğinin Kafka'ya gönderilememesi durumunda sistemde yalnızca `Started` durumunda kalan bir kayıt oluşmasının önüne geçmeyi amaçlar.

### 7. Zamanlanmış Tarama Hatalarının Yönetimi

Hangfire ile oluşturulan zamanlanmış taramalarda da hata durumları takip edilmektedir.

Zamanlanmış tarama kaydı oluşturulduktan sonra Hangfire görevinin planlanması sırasında hata oluşursa ilgili `ScheduledScan` kaydı `Failed` durumuna getirilir. Hata açıklaması ve tamamlanma zamanı kaydedilir.

Zamanı gelen görev çalışırken veya tarama başlatılırken hata meydana gelmesi durumunda da ilgili kayıtlar üzerinden süreç takip edilir.

Böylece zamanlanmış taramalarda planlama aşamasındaki hatalar ile taramanın çalıştırılması sırasında oluşan hatalar birbirinden ayırt edilebilir.

### 8. Geçici Dosyaların Temizlenmesi

Semgrep taraması sırasında kaynak kod deposu geçici bir dizine klonlanır. Tarama tamamlandığında veya işlem sırasında hata oluştuğunda geçici dizinin temizlenmesi sağlanır.

Bu işlem, tarama sonrasında kaynak kod dosyalarının gereksiz yere sistemde tutulmasını önler ve geçici dosyaların birikmesinden kaynaklanabilecek disk kullanımını azaltır.

### 9. Genel Değerlendirme

SPM uygulamasında güvenlik ve hata yönetimi, sistemin farklı katmanlarında ele alınmıştır. JWT ile kullanıcı kimlik doğrulaması, rol tabanlı yetkilendirme ile erişim kontrolü ve servisler arası API Key doğrulaması uygulanmıştır.

Tarama, mesajlaşma ve zamanlanmış görev süreçlerinde oluşabilecek hatalar ise ilgili kayıtların durum ve hata alanları üzerinden takip edilmektedir.

Bu mekanizmalar sayesinde kullanıcı işlemlerinin kontrol altında tutulması, servisler arası iletişimin korunması ve tarama süreçlerinde meydana gelen sorunların izlenebilmesi amaçlanmıştır.
