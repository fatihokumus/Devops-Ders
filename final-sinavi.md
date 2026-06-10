### 2025-2026 Bahar Dönemi Final Sınavı



### Soru 1

Bir ekipte geliştiriciler doğrudan `main` dalına push yapmak yerine özellik dalı üzerinden çalışıyor, pull request açıyor ve Jenkins kontrollerinin geçmesini bekliyor. Ekipten biri "Zaten Jenkins test çalıştırıyor, code review'a gerek yok" diyor.

Bu yoruma karşı en doğru değerlendirme hangisidir?

A. Code review, Jenkins kontrolleri başarılıysa gereksizdir.  
B. Code review, Jenkins kontrollerini tamamlayıcı bir unsur olarak gereklidir.  
C. Pull request, yalnızca Jenkins'i tetiklemek için kullanılan teknik bir adımdır.  
D. Jenkins, pull request ve code review sürecinden bağımsız tutulmalıdır.

**Seçiminizi açıklayınız:**

---

### Soru 2

İki geliştirici aynı dosyada farklı özellikler geliştirmiştir. Birinci geliştiricinin pull request'i `main` dalına alınmıştır. İkinci geliştirici kendi özellik dalını güncellemek istemektedir. Bu dal henüz yalnızca kendisi tarafından kullanılmaktadır.

Bu durumda aşağıdaki yaklaşımlardan hangisi en uygun tercih olabilir?

A. Ortak `main` dalını kendi özellik dalı üzerine rebase etmek.  
B. Çakışma ihtimalini azaltmak için kendi değişikliklerini silip yeniden başlamak.  
C. Kendi özellik dalını güncel `main` dalı üzerine rebase etmek.  
D. Pull request açmadan doğrudan `main` dalına force push yapmak.

**Seçiminizi açıklayınız:**

---

### Soru 3

Bir Spring Boot uygulaması için Docker imajı üretilmektedir. Pipeline sonunda imaj Docker Hub'a `latest` etiketiyle gönderiliyor ve staging ile production bu etiketi kullanarak güncelleniyor. Production'da hata çıkınca hangi commit'in çalıştığı anlaşılamıyor.

Bu problemin temel nedeni ve en doğru çözüm yaklaşımı hangisidir?

A. Docker imajı yerine JAR dosyası elle sunucuya taşınmalıdır.  
B. Registry kullanımı bırakılıp imajlar yalnızca Jenkins ajanında tutulmalıdır.  
C. Staging ortamı kaldırılıp doğrudan production'a geçilmelidir.  
D. `latest` yerine sabit ve izlenebilir imaj etiketi kullanılmalıdır.

**Seçiminizi açıklayınız:**

---

### Soru 4

Bir kurumda her başarılı build sonrasında imaj üretiliyor, staging ortamında otomatik deploy yapılıyor ve endpoint kontrolleri çalışıyor. Ancak production'a geçiş için manuel onay bekleniyor.

Bu süreç aşağıdaki kavramlardan hangisine daha yakındır?

A. Continuous Integration  
B. Continuous Deployment  
C. Continuous Delivery  
D. Manual Deployment

**Seçiminizi açıklayınız:**

---

### Soru 5

Bir Jenkins kurulumu GitHub depolarını her 5 dakikada bir polling ile kontrol etmektedir. Organizasyonda 30 depo vardır ve çoğunlukla sorgular "değişiklik yok" sonucu dönmektedir. Geliştiriciler ayrıca build'in geç başlamasından şikâyet etmektedir.

Bu durumda webhook kullanımı için en doğru değerlendirme hangisidir?

A. Polling yerine webhook tabanlı tetikleme tercih edilmelidir.  
B. Webhook kullanıldığında Jenkinsfile ihtiyacı ortadan kalkar.  
C. Webhook, pipeline içindeki testlerin başarılı geçmesini garanti eder.  
D. Webhook yalnızca production deploy aşamasında kullanılmalıdır.

**Seçiminizi açıklayınız:**

---

### Soru 6

Bir monorepo içinde `backend/` ve `frontend/` dizinleri bulunuyor. Backend Maven ile test ediliyor, iki katman için ayrı Docker imajları üretiliyor ve aynı sürüm bağlamında staging'e alınması isteniyor.

Bu pipeline için en doğru tasarım kararı hangisidir?

A. Backend test edildiği için yalnızca frontend imajı üretmek.  
B. Registry kullanmadan hedef sunucuda kaynak koddan build almak.  
C. Backend ve frontend imajlarını farklı commit'lerden üretmek.  
D. Backend ve frontend için ayrı imajlar üretip bunları aynı sürüm bağlamında etiketlemek.

**Seçiminizi açıklayınız:**

---

### Soru 7

Bir ekip Docker Compose ile çalışan backend, frontend ve PostgreSQL servislerini uzak sunucuya taşımıştır. Deployment sonrası `docker ps` tüm container'ları çalışır gösteriyor, ancak kullanıcılar sipariş ekranında hata alıyor.

Bu durumda en doğru DevOps yorumu hangisidir?

A. `docker ps` çıktısı deployment başarısını değerlendirmek için tek başına yeterlidir.  
B. Kullanıcı hatası görüldüğünde yalnızca frontend kontrol edilmelidir.  
C. Deployment sonrası container durumu yanında uygulama doğrulaması da yapılmalıdır.  
D. Docker Compose kullanılan sistemlerde deployment sonrası doğrulama yapılmaz.

**Seçiminizi açıklayınız:**

---

### Soru 8

Bir ekip Kubernetes'e yeni geçmiştir ve "Biz zaten Docker ile container çalıştırıyoruz, Kubernetes sadece daha büyük bir docker run komutudur" şeklinde düşünmektedir.

Bu ifadeye karşı en doğru kavramsal cevap hangisidir?

A. İfade doğrudur; Kubernetes yalnızca daha gelişmiş bir container çalıştırma komutudur.  
B. Kubernetes, container çalıştırmanın ötesinde bir orkestrasyon katmanıdır.  
C. Kubernetes'te control plane veya worker node gibi mimari bileşenler yoktur.  
D. Kubernetes container teknolojisi yerine yalnızca sanal makineleri yönetir.

**Seçiminizi açıklayınız:**

---

### Soru 9

Kubernetes'te bir uygulama tek bir Pod olarak çalıştırılıyor. Pod silindiğinde yeni Pod farklı IP ile oluşuyor ve istemciler eski IP'ye gitmeye devam ettiği için hata alıyor.

Bu durum için en doğru mimari yorum hangisidir?

A. Pod yönetimi için Deployment, sabit erişim için Service kullanılmalıdır.  
B. İstemciler doğrudan Pod IP'sine bağlanacak şekilde yapılandırılmalıdır.  
C. IP değişimini önlemek için uygulama tek replica ile sınırlandırılmalıdır.  
D. Service yalnızca log toplama amacıyla kullanılmalıdır.

**Seçiminizi açıklayınız:**

---

### Soru 10

Production Kubernetes ortamında aynı uygulama imajı staging ve production için kullanılacaktır. Ancak veritabanı adresi, log seviyesi, parola, hazırlık kontrolü ve CPU/bellek kullanımı ortamdan ortama değişmektedir.

Bu ihtiyaç için en doğru kavramsal eşleştirme hangisidir?

A. Ortama göre değişen tüm bilgiler Docker imajına gömülmelidir.  
B. Secret içindeki base64 veri tek başına yeterli güvenliği sağlar.  
C. Requests ve limits yalnızca uygulama log seviyesini belirlemek için kullanılır.  
D. Yapılandırma, gizli bilgi, sağlık kontrolü ve kaynak yönetimi ayrı Kubernetes mekanizmalarıyla ele alınmalıdır.

**Seçiminizi açıklayınız:**

---

## Cevap Anahtarı

### Soru 1

**Doğru cevap: B**

Jenkins build ve test gibi otomatik doğrulamaları çalıştırır; ancak kodun tasarım kalitesi, okunabilirliği, mimari uygunluğu ve ekip standartları insan incelemesi gerektirir. Pull request, hem Jenkins kontrollerini hem de code review sürecini aynı değişiklik etrafında toplar.

### Soru 2

**Doğru cevap: C**

Kişisel özellik dalı henüz paylaşılmamış veya yalnızca tek geliştirici tarafından kullanılıyorsa rebase, commit geçmişini güncel `main` üzerine taşıyarak daha temiz bir pull request oluşturabilir. Ortak `main` dalında rebase veya force push yapmak ise ekip geçmişini bozabileceği için risklidir.

### Soru 3

**Doğru cevap: D**

`latest` hareketli olduğu için hangi imajın hangi commit'ten üretildiğini belirsizleştirir. Build numarası ve commit SHA içeren sabit etiketler, çalışan sürümü kaynak koddaki belirli bir noktaya bağlar ve rollback için açık referans oluşturur.

### Soru 4

**Doğru cevap: C**

Staging'e otomatik geçiş ve production'a hazır tutma Continuous Delivery yaklaşımına uygundur. Continuous Deployment olması için production geçişinin de başarılı otomatik kontrollerden sonra insan onayı olmadan yapılması gerekir.

### Soru 5

**Doğru cevap: A**

Polling gereksiz sorgu trafiği ve gecikme üretir. Webhook olay gerçekleştiğinde GitHub'ın Jenkins'e HTTP POST göndermesini sağlar; böylece CI süreci olay odaklı ve daha hızlı başlar. Güvenlik için webhook secret, payload doğrulama ve endpoint erişimi dikkate alınmalıdır.

### Soru 6

**Doğru cevap: D**

Monorepo içindeki iki deploy edilebilir birim ayrı imajlar olarak üretilmelidir. Aynı build numarası ve commit SHA ile etiketlemek, backend ve frontend'in aynı sürüm bağlamında staging ve production'a taşınmasını sağlar. Production onayı, Continuous Delivery modelindeki kontrol noktasıdır.

### Soru 7

**Doğru cevap: C**

Container'ın çalışır görünmesi uygulamanın doğru cevap verdiği, veritabanına bağlandığı veya frontend-backend iletişiminin sağlıklı olduğu anlamına gelmez. Deployment sonrası endpoint, log ve bağımlı servis kontrolleri gerekir.

### Soru 8

**Doğru cevap: B**

Kubernetes yalnızca container başlatmaz; cluster içindeki istenen durumun korunması, pod'ların uygun node'lara yerleştirilmesi, self-healing, servis keşfi, ölçekleme ve reconciliation döngüsü gibi orkestrasyon sorumluluklarını üstlenir.

### Soru 9

**Doğru cevap: A**

Pod IP'leri kalıcı kabul edilmemelidir. Deployment replica sayısını, güncelleme ve yeniden oluşturma davranışını yönetir. Service ise pod'lar değişse bile istemciler için sabit DNS/IP erişim katmanı sağlar ve trafiği uygun pod'lara yönlendirir.

### Soru 10

**Doğru cevap: D**

Ortamdan ortama değişen hassas olmayan yapılandırma ConfigMap ile, parola ve token gibi hassas bilgiler Secret ile yönetilmelidir. Probe'lar pod'un canlı ve trafik almaya hazır olup olmadığını belirler. Requests ve limits ise scheduler kararlarını ve kaynak tüketim sınırlarını etkiler. Base64'in şifreleme olmadığı özellikle belirtilmelidir.
