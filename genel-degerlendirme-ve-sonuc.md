# Genel Değerlendirme ve Sonuç

SPM (Software Project Management) projesi kapsamında yazılım projelerinin güvenlik açıkları açısından taranabilmesini, tarama süreçlerinin yönetilmesini ve elde edilen bulguların merkezi bir yapı üzerinden takip edilebilmesini sağlayan bir sistem geliştirilmiştir.

Proje sürecinde öncelikle kullanıcı yönetimi ve kimlik doğrulama altyapısı oluşturulmuş, ardından asset yönetimi ve tarama işlemleri sisteme dahil edilmiştir. Kullanıcıların sisteme giriş yapabilmesi, rollerine göre yetkilendirilmesi ve tarama işlemlerini gerçekleştirebilmesi sağlanmıştır.

Güvenlik taramaları için Semgrep ve Trivy olmak üzere iki farklı araç entegre edilmiştir. Semgrep ile kaynak kod içerisindeki güvenlik problemlerinin tespit edilmesi, Trivy ile ise projelerin bağımlılıklarında bulunan bilinen güvenlik açıklarının incelenmesi hedeflenmiştir. Her iki tarama aracından elde edilen sonuçlar ortak bir veri modeli üzerinden işlenerek aynı sistem içerisinde görüntülenebilir hale getirilmiştir.

Sistemin ölçeklenebilir ve birbirinden bağımsız servislerle çalışabilen bir yapıya sahip olması amacıyla UserManagement, ScannerService, SemgrepService ve TrivyService bileşenleri ayrı projeler halinde yapılandırılmıştır. Kafka kullanılarak tarama isteklerinin servisler arasında asenkron şekilde iletilmesi sağlanmıştır. Tarama ilerlemesinin takip edilebilmesi için SignalR altyapısından yararlanılmış, tarama sonuçları ise HTTP üzerinden merkezi uygulamaya aktarılmıştır.

Bunların yanında Hangfire entegrasyonu ile zamanlanmış tarama özelliği geliştirilmiştir. Kullanıcıların belirledikleri tarih ve saatlerde taramaların otomatik olarak başlatılması sağlanmış, bu işlemlerin planlama ve çalıştırma bilgileri ayrı kayıtlar üzerinden takip edilebilir hale getirilmiştir.

Proje boyunca .NET, PostgreSQL, Entity Framework Core, Next.js, Kafka, Docker, SignalR ve Hangfire gibi teknolojiler birlikte kullanılmıştır. Bu süreçte katmanlı mimari, servisler arası iletişim, veritabanı ilişkileri, API tasarımı, kimlik doğrulama, yetkilendirme ve arka plan görev yönetimi gibi konularda uygulamalı deneyim kazanılmıştır.

Sonuç olarak SPM projesi ile farklı güvenlik tarama araçlarını tek bir platform altında birleştiren, tarama süreçlerini merkezi olarak yöneten ve sonuçların takip edilmesini sağlayan bir yapı ortaya çıkarılmıştır. Proje süreci, yalnızca uygulama geliştirme açısından değil, birden fazla servisin birlikte çalıştığı bir yazılım sisteminin tasarlanması, geliştirilmesi ve yönetilmesi açısından da önemli bir deneyim sağlamıştır.
