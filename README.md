# SPM Projesine Genel Bakış

### 1. Proje Hakkında

SPM (Software Security Posture Management), yazılım projelerinin güvenlik durumunu değerlendirmek, kaynak kod depolarındaki güvenlik açıklarını tespit etmek ve bu açıkları merkezi bir sistem üzerinden takip etmek amacıyla geliştirilen bir uygulamadır.

Uygulama, kullanıcıların GitHub üzerinde bulunan yazılım depolarını sisteme eklemesine ve bu depolar üzerinde farklı güvenlik tarayıcıları aracılığıyla analiz gerçekleştirmesine olanak tanır.

Sistem kapsamında Semgrep ve Trivy olmak üzere iki farklı güvenlik tarayıcısı entegre edilmiştir. Bu tarayıcılar farklı analiz yöntemleri kullanarak yazılım projelerindeki güvenlik problemlerini tespit eder. Elde edilen sonuçlar ortak bir veri yapısına dönüştürülerek uygulama üzerinden görüntülenebilir hale getirilir.

Uygulama yalnızca manuel tarama işlemlerini değil, belirlenen tarih ve saatlerde otomatik olarak gerçekleştirilecek zamanlanmış taramaları da desteklemektedir. Böylece yazılım depolarının güvenlik durumunun belirli aralıklarla kontrol edilebilmesi amaçlanmıştır.

SPM, kullanıcı yönetimi, yetkilendirme, varlık yönetimi, güvenlik taraması, tarama geçmişi ve güvenlik bulgularının takibi gibi işlevleri tek bir platform altında birleştirmektedir.

### 2. Projenin Amacı Ve Kapsamı

Projenin temel amacı, yazılım depolarındaki güvenlik açıklarının farklı tarama araçları kullanılarak tespit edilmesini ve bu sonuçların merkezi bir platform üzerinden yönetilmesini sağlamaktır.

Yazılım projelerinde kullanılan bağımlılıklar, kaynak kod içerisindeki hatalı uygulamalar ve bilinen güvenlik zafiyetleri uygulamaların güvenliğini olumsuz etkileyebilmektedir. Bu problemlerin tespit edilmesi ve takip edilmesi, yazılım güvenliği açısından önem taşımaktadır.

Bu doğrultuda geliştirilen SPM uygulaması aşağıdaki işlevleri kapsamaktadır:

* **Kullanıcı Yönetimi:** Sisteme kullanıcı eklenmesi, kullanıcı bilgilerinin yönetilmesi ve kullanıcıların rollerine göre yetkilendirilmesi.
* **Asset Yönetimi:** Güvenlik taraması gerçekleştirilecek GitHub depolarının sisteme eklenmesi ve yönetilmesi.
* **Güvenlik Taramaları:** Sisteme entegre edilen Semgrep ve Trivy araçlarıyla yazılım depolarının analiz edilmesi.
* **Güvenlik Bulgularının Yönetimi:** Tarama sonucunda tespit edilen güvenlik açıklarının ortak bir veri yapısında saklanması ve görüntülenmesi.
* **Tarama Geçmişi:** Gerçekleştirilen taramaların durumlarının, başlangıç ve tamamlanma bilgilerinin takip edilmesi.
* **Zamanlanmış Taramalar:** Hangfire altyapısı kullanılarak belirlenen tarih ve saatlerde otomatik tarama işlemlerinin başlatılması.
* **Asenkron İşlem Yönetimi:** Kafka aracılığıyla tarama isteklerinin ilgili servisler arasında iletilmesi ve tarama işlemlerinin ana uygulamadan bağımsız servisler üzerinden yürütülmesi.

Proje kapsamında Semgrep ve Trivy farklı analiz ihtiyaçlarına yönelik kullanılmaktadır. Semgrep kaynak kod içerisindeki güvenlik problemlerinin tespit edilmesine odaklanırken Trivy, yazılım projelerinin bağımlılıklarını inceleyerek bilinen güvenlik açıklarını belirlemektedir.

Bu iki aracın sonuçlarının ortak bir yapı altında toplanması sayesinde farklı tarama araçlarından elde edilen güvenlik bulgularının aynı platform üzerinden takip edilmesi sağlanmaktadır.



