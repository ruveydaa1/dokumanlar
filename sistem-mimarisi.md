# Sistem Mimarisi

SPM uygulaması, farklı sorumluluklara sahip servislerin bir arada çalıştığı modüler bir mimariyle geliştirilmiştir. Uygulama; ana backend, tarama işlemlerini yöneten servis ve güvenlik taramalarını gerçekleştiren bağımsız servislerden oluşmaktadır.

Bu mimaride temel amaç, kullanıcı işlemlerinin ve veritabanı yönetiminin gerçekleştirildiği ana uygulama ile yoğun işlem gerektiren güvenlik taramalarını birbirinden ayırmaktır. Böylece tarama işlemleri ana uygulamanın işleyişinden bağımsız servisler üzerinden yürütülebilmektedir.

### 1. Sistem Bileşenleri

Uygulama temel olarak dört ana bileşenden oluşmaktadır.

#### **1.1. UserManagement**

UserManagement, uygulamanın ana backend servisidir. Sistemin merkezi yönetiminden ve veritabanı işlemlerinden sorumludur.

Başlıca görevleri şunlardır:

* Kullanıcı kayıtlarının ve kullanıcı rollerinin yönetilmesi.
* JWT tabanlı kimlik doğrulama ve yetkilendirme işlemlerinin gerçekleştirilmesi.
* GitHub depolarının sisteme eklenmesi ve yönetilmesi.
* Tarama isteklerinin oluşturulması ve Kafka'ya gönderilmesi.
* ScanRun kayıtlarının oluşturulması ve tarama geçmişinin tutulması.
* Tarama sonuçlarının karşılanması ve Finding kayıtlarının veritabanına kaydedilmesi.
* Hangfire aracılığıyla zamanlanmış tarama görevlerinin yönetilmesi.

UserManagement, sistemin merkezi yönetim noktasıdır. Tarama işlemlerini doğrudan gerçekleştirmek yerine ilgili tarama servislerinin çalışmasını koordine eder.

#### **1.2. ScannerService**

ScannerService, tarama isteklerinin alınması ve uygun tarama servisine yönlendirilmesinden sorumludur.

UserManagement tarafından Kafka'ya gönderilen tarama mesajları ScannerService içerisindeki KafkaScanConsumer tarafından tüketilir.

Gelen istekte belirtilen tarayıcı bilgisine göre ilgili servisle HTTP iletişimi kurulur. Semgrep seçilmişse SemgrepService, Trivy seçilmişse TrivyService kullanılır.

ScannerService'in temel görevleri şunlardır:

* Kafka üzerinden tarama isteklerini tüketmek.
* Tarama isteğindeki scanner bilgisine göre ilgili servisi belirlemek.
* SemgrepService ve TrivyService ile HTTP üzerinden iletişim kurmak.
* Tarama ilerleme bilgilerinin ilgili mekanizma üzerinden iletilmesini sağlamak.
* Tarama sonuçlarını HTTP tabanlı ScanResultSender aracılığıyla UserManagement Backend'e göndermek.

ScannerService, tarama işlemlerinin koordinasyonunu sağlayan ara servis görevini üstlenmektedir.

#### **1.3. SemgrepService**

SemgrepService, Semgrep güvenlik tarayıcısının çalıştırılmasından sorumlu bağımsız servistir.

ScannerService tarafından HTTP isteğiyle çağrılır. Gelen repository bilgisi üzerinden Semgrep taraması gerçekleştirilir ve elde edilen güvenlik bulguları sonuç formatına dönüştürülerek ScannerService'e iletilir.

#### **1.4. TrivyService**

TrivyService, Trivy güvenlik tarayıcısının çalıştırılmasından sorumlu bağımsız servistir.

ScannerService tarafından HTTP isteğiyle çağrılır. Trivy taraması Docker üzerinden çalıştırılır ve tarama sonucunda elde edilen JSON verisi işlenerek ScannerService'e iletilir.

SemgrepService ve TrivyService birbirinden bağımsız çalışmaktadır. Böylece yeni tarama araçlarının sisteme eklenebilmesi için modüler bir yapı oluşturulmuştur.

### 2. Katmanlı Mimari

UserManagement backend projesinde katmanlı mimari yaklaşımı kullanılmıştır. Bu yapı, uygulamanın farklı sorumluluklarının birbirinden ayrılmasını ve kodun daha düzenli yönetilmesini sağlar.

**Domain Katmanı**

Uygulamanın temel iş modellerini ve kurallarını içerir. Kullanıcı, rol, asset, scan run, scheduled scan ve finding gibi temel Entity sınıfları bu katmanda bulunur.

**Application Katmanı**

Uygulamanın iş süreçlerinin yürütülmesi için gerekli DTO, arayüz ve uygulama mantığını içerir. Katmanlar arasındaki iletişimde kullanılan sözleşmeler ve veri aktarım modelleri burada tanımlanır.

**Infrastructure Katmanı**

Veritabanı erişimi, Entity Framework Core yapılandırmaları, dış servislerle iletişim ve diğer teknik altyapı işlemleri bu katmanda gerçekleştirilir.

Bu ayrım sayesinde uygulamanın iş modelleri, iş süreçleri ve teknik altyapısı birbirinden ayrılmıştır.

### 3. Servisler Arası İletişim

SPM uygulamasında servisler arasındaki iletişim ihtiyaca göre Kafka ve HTTP kullanılarak gerçekleştirilmektedir.

**Kafka ile iletişim:** UserManagement tarafından oluşturulan tarama istekleri Kafka üzerinden ScannerService'e iletilir. ScannerService, ilgili mesajı consumer aracılığıyla tüketerek tarama sürecini başlatır.

**HTTP ile iletişim:** ScannerService, tarama işlemini gerçekleştirmek için SemgrepService veya TrivyService'e HTTP isteği gönderir. Tarama tamamlandığında sonuçlar ScannerService'e döner. ScannerService ise ScanResultSender aracılığıyla sonucu UserManagement Backend'e HTTP üzerinden iletir.

Bu yaklaşımda Kafka, tarama isteklerinin asenkron iletiminde; HTTP ise tarama servislerinin çağrılmasında ve sonuçların ana backend'e aktarılmasında kullanılmaktadır.

### 4. Genel Mimari Yaklaşımın Avantajları

* **Modülerlik:** Tarama araçları bağımsız servisler olarak yapılandırılmıştır.
* **Sorumlulukların ayrılması:** Ana backend, tarama koordinasyonu ve tarama işlemleri farklı bileşenler tarafından yürütülmektedir.
* **Genişletilebilirlik:** Sisteme yeni güvenlik tarayıcılarının eklenebilmesi için uygun bir yapı oluşturulmuştur.
* **Asenkron işlem yönetimi:** Kafka sayesinde tarama istekleri ana uygulamadan ayrılarak işlenebilmektedir.
* **Merkezi sonuç yönetimi:** Farklı tarayıcılardan gelen güvenlik bulguları UserManagement üzerinden ortak bir veritabanında saklanmaktadır.
