# Bölüm 4  
# Docker: Konteynerleşme, Mimari, Bileşenler ve Uygulama Süreci

Önceki bölümde sürekli entegrasyon hattına alınan `user-service` adlı Java Spring Boot REST API projesi temel alınmaktadır. Jenkins ile otomatik derlenen ve test edilen bu uygulamanın farklı ortamlarda aynı davranışı üretmesi, yalnızca kaynak kodun sürümlenmesiyle sağlanamaz. Uygulamanın çalıştığı işletim sistemi bileşenleri, kütüphaneler, ortam değişkenleri ve çalışma komutları da denetimli biçimde paketlenmelidir. Docker, bu ihtiyaca karşılık veren konteynerleşme platformudur.

Docker, tek başına bir komut satırı aracı değildir. Kapsayıcı çalıştırma motoru, imaj formatı, katmanlı dosya sistemi, ağ soyutlaması, kalıcı veri mekanizmaları, kayıt depoları, çoklu servis tanımı ve güvenlik pratikleriyle birlikte ele alınması gereken geniş bir ekosistemdir. Bu bölümde Docker’ın kuramsal zemini, iç mimarisi ve uygulama süreci ayrıntılı biçimde incelenmektedir.

## 4.1 Konteynerleşme İhtiyacı

Yazılım geliştirme sürecinde aynı uygulamanın farklı ortamlarda farklı davranması uzun yıllardır temel sorunlardan biri olmuştur. Geliştirici makinesinde çalışan bir uygulama, test ortamında veya üretim ortamında aynı sonucu vermeyebilir. Bunun nedeni çoğu zaman kaynak kod değil, çalışma bağlamıdır.

`user-service` için aşağıdaki farklılıklar tipik sorunlar üretir:

- Farklı JDK sürümleri
- Farklı işletim sistemi paketleri
- Ortam değişkenlerinin eksik veya hatalı tanımlanması
- Ağ yapılandırmalarındaki farklılıklar
- Yerel dosya sistemi bağımlılıkları

Geleneksel çözüm çoğu zaman sanal makine kullanmak olmuştur. Sanal makineler güçlü yalıtım sunar; ancak her makine tam bir işletim sistemi içerdiği için ağırdır, yavaş açılır ve kaynak tüketimi yüksektir. Konteyner yaklaşımı, uygulamayı işletim sistemi çekirdeğini paylaşan fakat kullanıcı alanı düzeyinde ayrıştırılmış ortamlarda çalıştırarak daha hafif bir model sunar.

Docker bu yaklaşımı geliştirici deneyimi açısından erişilebilir hâle getirmiştir. Uygulama, bağımlılıkları ve çalışma komutu ile birlikte taşınabilir bir paket hâline gelir. Aynı imaj, geliştirici bilgisayarında, CI sunucusunda ve üretim ortamında kullanılabilir.

## 4.2 Konteyner Kavramı

Konteyner, işletim sistemi düzeyinde yalıtılmış bir çalışma ortamıdır. Sanal makineden temel farkı, ayrı bir çekirdek taşımamasıdır. Aynı çekirdeği paylaşan süreçler, Linux çekirdeğinin sunduğu yalıtım mekanizmalarıyla birbirinden ayrılır.

Konteynerlerin dayandığı temel çekirdek mekanizmaları şunlardır:

- `Namespaces`: Süreçlerin PID, ağ, mount, hostname ve kullanıcı alanı gibi kaynakları sınırlı ve izole görmesini sağlar.
- `cgroups`: CPU, bellek, disk I/O gibi kaynak kullanımını sınırlar ve ölçer.
- `Capabilities`: Süper kullanıcı yetkilerini parçalayarak süreçlere daha dar ayrıcalıklar verir.
- `Union file system`: Dosya sistemi katmanlarının üst üste bindirilerek okunması ve yazılmasını sağlar.

Bu mekanizmalar birlikte kullanıldığında, bir süreç grubu ayrı bir makinede çalışıyormuş gibi davranabilir. Docker’ın önemi, bu düşük seviyeli çekirdek özelliklerini doğrudan yönetmeyi gerektirmeden standart araçlar ve formatlar sunmasıdır.

## 4.3 Docker’ın Ortaya Çıkışı ve Rolü

Docker, konteyner teknolojisinin kendisini icat etmemiştir; ancak onu yazılım geliştirme ve DevOps pratikleri açısından kullanılabilir, paketlenebilir ve yaygınlaştırılabilir bir yapıya dönüştürmüştür. Asıl katkısı, konteyner çalıştırmayı standartlaştırması ve imaj tabanlı dağıtım modelini geliştirici iş akışının merkezine yerleştirmesidir.

Docker’ın rolü birkaç başlık altında toplanabilir:

- Uygulamanın çalışma bağlamını imaj olarak tanımlamak
- Bu imajdan izole konteyner örnekleri üretmek
- Ağ ve veri depolama soyutlamaları sağlamak
- İmajların kayıt depolarına gönderilip dağıtılmasını kolaylaştırmak
- Geliştirme, test ve dağıtım süreçlerinde aynı paketleme modelini kullanmak

`user-service` için Docker kullanıldığında, uygulama “bir JAR dosyası” olmaktan çıkar ve tanımlı bir çalışma ortamına sahip taşınabilir bir teslimat birimine dönüşür.

## 4.4 Docker Mimarisi

Docker mimarisi birkaç katmandan oluşur. Kullanıcı tarafında görünen `docker` komutunun arkasında, istekleri alan, imajları yöneten, konteyner oluşturan ve sistem kaynaklarıyla etkileşen daha geniş bir yürütme modeli vardır.

Temel mimari aşağıdaki akışla açıklanabilir:

```text
Docker CLI / API
        |
        v
     dockerd
        |
        v
    containerd
        |
        v
       runc
        |
        v
    Container Process
```

Bu akışa imaj deposu, ağ sürücüleri, hacimler, build motoru ve registry bileşenleri eşlik eder.

### Docker Client

Docker Client, kullanıcının doğrudan etkileşim kurduğu katmandır. En yaygın biçimi `docker` komut satırı aracıdır. `docker build`, `docker run`, `docker ps`, `docker logs` gibi komutlar bu istemci üzerinden çalıştırılır.

Client, işlemleri kendi başına gerçekleştirmez. Asıl görevi Docker daemon’a istek göndermektir. Bu iletişim çoğu Linux kurulumunda Unix socket üzerinden yapılır:

```text
/var/run/docker.sock
```

Bu socket dosyasına erişim, sistem üzerinde çok yüksek yetki anlamına geldiği için güvenlik açısından kritik kabul edilir.

### Docker Daemon (`dockerd`)

Docker daemon, Docker Engine’in merkezî servisidir. İmajları oluşturur, konteynerleri başlatır, ağları ve hacimleri yönetir, istemciden gelen API çağrılarını işler. Arka planda çalışan bu servis, Docker ekosisteminin yürütme omurgasıdır.

Daemon’ın başlıca sorumlulukları şunlardır:

- İmaj build sürecini başlatmak
- Kayıt depolarından imaj çekmek
- Konteynerleri oluşturmak ve yaşam döngülerini yönetmek
- Ağ ve volume tanımlarını sürdürmek
- Olay ve log bilgisini saklamak

### Docker Engine

Docker Engine, pratikte Docker daemon, API ve ilgili çalışma zamanı bileşenlerinin tamamını ifade eden genel addır. Sistem yöneticisi açısından “Docker kurulu” denildiğinde çoğunlukla Docker Engine kastedilir.

### containerd

`containerd`, konteyner yaşam döngüsünü yönetmek için kullanılan düşük seviyeli çalışma zamanı bileşenidir. İmaj çekme, snapshot yönetimi, konteyner başlatma ve durdurma gibi işleri daha alt seviyede yürütür. Docker, güncel mimaride bu katmandan yararlanır.

Bu ayrım önemlidir; çünkü Docker, konteyner dünyasının tek çalışma zamanı değildir. `containerd` ve OCI tabanlı araçlar Docker dışındaki sistemlerde de kullanılabilir.

### runc

`runc`, OCI uyumlu konteynerleri fiilen başlatan düşük seviyeli yürütme aracıdır. Linux çekirdeği tarafından sunulan namespace ve cgroup yapılarını kullanarak süreçleri izole ortamda çalıştırır.

Docker, doğrudan çekirdek seviyesinde her şeyi tek başına uygulamak yerine katmanlı bir yürütme modeli kullanır. Bu model, standartlaşmayı ve birlikte çalışabilirliği artırır.

### BuildKit

BuildKit, modern Docker build süreçlerinin arka planındaki build motorudur. Klasik `docker build` davranışına göre daha verimli önbellekleme, paralel işlem ve daha gelişmiş build tanımları sunar.

BuildKit’in başlıca katkıları şunlardır:

- Katman önbelleğini daha iyi kullanmak
- Çok aşamalı build süreçlerini hızlandırmak
- Gereksiz dosyaları build bağlamı dışına almak
- Gizli bilgilerin build sırasında daha güvenli kullanılmasına imkân vermek

### Docker Registry

Registry, Docker imajlarının saklandığı ve dağıtıldığı sunucudur. Docker Hub herkese açık en yaygın registry örneğidir. Kurumlar genellikle özel registry kullanır.

`user-service` için iş akışı çoğu zaman şu biçimdedir:

1. İmaj yerelde build edilir.
2. Uygun etiket verilir.
3. Registry’ye gönderilir.
4. Başka bir ortam bu imajı çekerek çalıştırır.

## 4.5 Docker Ekosistemindeki Temel Bileşenler

Docker dünyasını anlamak için aşağıdaki nesnelerin birbirinden kesin biçimde ayrılması gerekir.

### Image

Image, uygulamanın ve çalışma ortamının salt okunur şablonudur. İmaj, çalıştırılabilir konteynerlerin temelini oluşturur. İçinde dosya sistemi katmanları, metadata, varsayılan komut ve yapılandırma bilgileri bulunur.

Örnek:

```bash
docker pull eclipse-temurin:17-jre
```

Bu komut çalıştırıldığında bir konteyner değil, JRE taban imajı çekilmiş olur.

### Container

Container, imajın çalışan örneğidir. İmaj değişmez bir tanımken konteyner canlı süreçtir. Dosya sistemi üzerinde yazılabilir bir üst katman taşır, ağ arayüzü edinir, log üretir ve yaşam döngüsüne sahiptir.

Örnek:

```bash
docker run -d -p 8080:8080 user-service:1.0.0
```

Bu komut, `user-service:1.0.0` imajından çalışan bir konteyner oluşturur.

### Dockerfile

Dockerfile, imajın nasıl üretileceğini tanımlayan bildirimsel metin dosyasıdır. Bir imajın kaynak kodu olarak düşünülebilir. İçinde taban imaj, kopyalanacak dosyalar, kurulacak paketler, açılacak portlar ve çalıştırma komutu tanımlanır.

### Layer

Layer, imajın katmanlı yapısındaki her bir dosya sistemi farkını temsil eder. Docker imajları tek parça değildir; bir dizi katmanın üst üste bindirilmesiyle oluşur. Katmanlı yapı sayesinde aynı taban imajı kullanan farklı uygulamalar depolama ve indirme maliyeti açısından verimli hâle gelir.

### Repository, Tag ve Digest

Docker imajları çoğu zaman `repository:tag` biçiminde anılır:

```text
example/user-service:1.0.0
```

- `repository`: İmajın mantıksal adı
- `tag`: İnsan tarafından okunabilir sürüm etiketi
- `digest`: İmaj içeriğini kriptografik olarak tanımlayan değişmez kimlik

Etiketler değiştirilebilir. Digest ise içerik temelli olduğu için daha güvenilir referanstır.

### Network

Network, konteynerler arası iletişimi yöneten sanal ağ soyutlamasıdır. Docker, varsayılan ve kullanıcı tanımlı ağlar oluşturabilir.

### Volume

Volume, konteyner ömründen bağımsız kalıcı veri alanıdır. Konteyner silinse bile volume içindeki veri korunabilir.

### Bind Mount

Bind mount, ana makinedeki belirli bir dizinin konteyner içine bağlanmasıdır. Geliştirme süreçlerinde kaynak kod senkronizasyonu için yararlıdır; ancak taşınabilirlik açısından volume kadar kontrollü değildir.

### Compose

Compose, birden fazla servisin tek dosyada tanımlanıp birlikte çalıştırılmasını sağlar. Uygulama ve veritabanı gibi ilişkili servisleri bir arada yönetmek için kullanılır.

### Docker Desktop

Docker Desktop, özellikle Windows ve macOS ortamlarında Docker deneyimini kolaylaştıran masaüstü paketidir. Docker Engine, Compose, GUI araçları ve bazı yardımcı bileşenleri birlikte sunar.

### Swarm

Docker Swarm, Docker’ın yerleşik küme ve orkestrasyon çözümüdür. Günümüzde geniş ölçekte Kubernetes daha yaygın olsa da Swarm, Docker ekosisteminin tarihsel ve kavramsal önemli bileşenlerinden biridir.

## 4.6 İmaj Yapısı ve Katman Mantığı

Docker imajlarının en önemli özelliği katmanlı olmalarıdır. Her Dockerfile talimatı genellikle yeni bir katman üretir. Bu katmanlar tekrar kullanılabilir, önbelleğe alınabilir ve farklı imajlar arasında paylaşılabilir.

Örnek bir mantık şu biçimdedir:

```dockerfile
FROM eclipse-temurin:17-jre
WORKDIR /app
COPY target/user-service.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Bu tanımda:

- `FROM` taban imaj katmanlarını getirir.
- `WORKDIR` metadata ve dizin bağlamı oluşturur.
- `COPY` uygulama dosyasını yeni katmana ekler.
- `ENTRYPOINT` çalıştırma davranışını tanımlar.

Katman mantığının pratik sonuçları şunlardır:

- Aynı taban imajı kullanan projeler daha hızlı çekilir.
- Yalnızca değişen katmanlar yeniden build edilir.
- Uygun Dockerfile sıralaması build hızını belirgin biçimde etkiler.

## 4.7 Dockerfile Talimatları

Dockerfile, imaj üretiminin temel tanımıdır. En sık kullanılan talimatlar aşağıda açıklanmaktadır.

### `FROM`

Taban imajı belirler.

```dockerfile
FROM eclipse-temurin:17-jre
```

### `WORKDIR`

Sonraki komutların çalışacağı varsayılan dizini ayarlar.

```dockerfile
WORKDIR /app
```

### `COPY`

Dosyaları build bağlamından imaja kopyalar.

```dockerfile
COPY target/user-service-0.0.1-SNAPSHOT.jar app.jar
```

### `RUN`

Build aşamasında komut çalıştırır. Paket kurmak veya dosya üretmek için kullanılır.

```dockerfile
RUN addgroup --system spring && adduser --system spring --ingroup spring
```

### `CMD`

Varsayılan komutu tanımlar. Çalıştırma anında kolayca ezilebilir.

### `ENTRYPOINT`

Konteynerin ana yürütme komutunu tanımlar. Uygulama konteynerlerinde çoğu zaman tercih edilir.

### `ENV`

Ortam değişkeni tanımlar.

```dockerfile
ENV SPRING_PROFILES_ACTIVE=prod
```

### `EXPOSE`

Konteynerin hangi portu dinlediğini belgesel amaçla belirtir.

```dockerfile
EXPOSE 8080
```

### `ARG`

Build zamanında verilen değişkenleri kullanmak için tanımlanır.

### `USER`

Konteyner içindeki süreçlerin hangi kullanıcıyla çalışacağını belirler. Güvenlik açısından kritik talimattır.

## 4.8 `user-service` İçin Çok Aşamalı Dockerfile

Spring Boot uygulamalarında en yaygın ve verimli yaklaşım, build ile çalıştırma ortamını ayıran çok aşamalı Dockerfile kullanmaktır. Bu modelde Maven ve JDK içeren ağır build ortamı, son imaj içinde yer almaz.

`user-service` için uygun bir Dockerfile aşağıdaki gibidir:

```dockerfile
FROM maven:3.9.8-eclipse-temurin-17 AS build
WORKDIR /workspace

COPY pom.xml .
RUN mvn -B dependency:go-offline

COPY src src
RUN mvn -B clean package -DskipTests

FROM eclipse-temurin:17-jre
WORKDIR /app

RUN addgroup --system spring && adduser --system spring --ingroup spring
USER spring:spring

COPY --from=build /workspace/target/user-service-0.0.1-SNAPSHOT.jar app.jar

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

Bu Dockerfile’ın başlıca avantajları şunlardır:

- Son imaj küçük kalır.
- Maven ve kaynak kod son imaja taşınmaz.
- Güvenlik yüzeyi daralır.
- Build önbelleği daha verimli kullanılabilir.

İmaj build komutu aşağıdaki gibidir:

```bash
docker build -t user-service:1.0.0 .
```

## 4.9 Build Context ve Önbellek Davranışı

Docker build işlemi yalnızca Dockerfile’ı okumaz; aynı zamanda build context adı verilen dosya kümesini daemon’a gönderir. Bu nedenle proje kök dizinindeki gereksiz dosyaların context içine dâhil edilmesi build süresini ve güvenlik riskini artırır.

Bu nedenle `.dockerignore` dosyası önemlidir:

```text
.git
target
.idea
*.log
```

Önbellek davranışı, Dockerfile sıralamasından güçlü biçimde etkilenir. Örneğin önce `pom.xml` kopyalanıp bağımlılıklar çözülür, ardından `src/` klasörü eklenirse kaynak kod değişiklikleri olduğunda bağımlılık katmanı yeniden üretilmez.

Yanlış sıralama örneği:

```dockerfile
COPY . .
RUN mvn -B clean package
```

Bu yaklaşım, en küçük değişiklikte tüm katmanların yeniden build edilmesine yol açabilir.

## 4.10 Konteyner Yaşam Döngüsü

Bir konteynerin yaşamı, yalnızca “çalışıyor” veya “çalışmıyor” ikiliğine indirgenemez. Docker, konteynerler için belirli durumlar tanımlar:

- `created`
- `running`
- `paused`
- `restarting`
- `exited`
- `dead`

Temel yaşam döngüsü komutları şunlardır:

```bash
docker create user-service:1.0.0
docker start <container_id>
docker stop <container_id>
docker restart <container_id>
docker rm <container_id>
```

Tek komutta oluşturup başlatmak için:

```bash
docker run -d --name user-service -p 8080:8080 user-service:1.0.0
```

Bu komut aşağıdaki işlemleri kapsar:

- Konteyner oluşturur
- Ağ bağlamını ayarlar
- Port eşlemesini yapar
- Yazılabilir üst katmanı ekler
- Tanımlı komutu başlatır

## 4.11 Konteyner İçinde Çalışan Süreç Mantığı

Her konteynerin merkezinde ana bir süreç bulunur. Docker, konteyneri bu sürecin yaşamına bağlar. Ana süreç sona erdiğinde konteyner de sona erer. Bu nedenle konteyner içinde art arda çalışan uzun kabuk betikleri veya arka plan süreci başlatıp ana süreci sonlandıran tasarımlar sorun üretir.

Spring Boot tabanlı `user-service` için ana süreç çoğu zaman aşağıdaki komuttur:

```dockerfile
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

Bu komut konteynerin PID 1 sürecidir. PID 1 davranışı Linux’ta sinyal işleme ve çocuk süreç yönetimi açısından farklılık taşıdığı için uygulama konteynerlerinin tasarımında önemlidir.

## 4.12 Ağ Bileşenleri

Docker, konteyner ağını soyutlamak için farklı ağ sürücüleri sunar. Ağ bileşenlerini anlamak, çok servisli uygulamaların davranışını çözmek açısından zorunludur.

### Bridge Network

Tek sunuculu Docker kurulumlarında en yaygın ağ türüdür. Aynı bridge ağına bağlı konteynerler birbirleriyle sanal ağ üzerinden haberleşebilir.

Varsayılan `bridge` ağı kullanılabilir; ancak kullanıcı tanımlı bridge ağları daha iyi DNS çözümleme ve daha düzenli izolasyon sağlar.

```bash
docker network create user-service-net
```

### Host Network

Konteyner, ana makinenin ağ yığınını doğrudan kullanır. İzolasyon düşer, performans kazanımı bazı durumlarda artabilir. Linux odaklı özel senaryolarda kullanılır.

### None Network

Konteyner ağ erişimi olmadan çalışır. Ağ gerektirmeyen işler veya sert izolasyon senaryoları için anlamlıdır.

### Overlay Network

Birden fazla Docker host üzerinde çalışan servislerin sanal ağ üzerinden haberleşmesini sağlar. Swarm gibi çok düğümlü yapılarda önem kazanır.

### Port Mapping

Konteyner içindeki portu dış dünyaya açmak için kullanılır:

```bash
docker run -p 8080:8080 user-service:1.0.0
```

Sol taraftaki port ana makineye, sağ taraftaki port konteyner içine aittir.

### DNS ve Servis Adı Çözümleme

Kullanıcı tanımlı ağlarda Docker, konteyner adlarını DNS adı gibi çözebilir. Bu özellik Compose tabanlı uygulamalarda servislerin birbirini isimle bulmasını sağlar.

## 4.13 Veri Depolama Bileşenleri

Konteynerler varsayılan olarak geçicidir. Konteyner silindiğinde yazılabilir katmanda üretilen veri kaybolur. Bu nedenle kalıcı verinin konteynerin içine değil, harici depolama mekanizmalarına taşınması gerekir.

### Writable Layer

Her konteyner, imajın üstüne eklenen yazılabilir katmanda çalışır. Bu alan hızlı ve pratiktir; ancak kalıcı depolama için uygun değildir.

### Volume

Docker tarafından yönetilen kalıcı veri alanıdır. Veritabanları ve uygulama verisi için en güvenilir seçeneklerden biridir.

```bash
docker volume create postgres-data
```

Konteyner çalıştırılırken:

```bash
docker run -v postgres-data:/var/lib/postgresql/data postgres:16
```

### Bind Mount

Ana makinedeki bir dizini konteyner içine bağlar:

```bash
docker run -v $(pwd):/workspace maven:3.9.8-eclipse-temurin-17
```

Geliştirme sırasında güçlüdür; fakat ana makineye sıkı bağ oluşturduğu için üretim ortamlarında dikkatli kullanılmalıdır.

### tmpfs Mount

Bellek tabanlı geçici dosya sistemi sunar. Diskte kalmaması gereken hassas veya geçici veri için kullanılabilir.

## 4.14 Registry, Repository ve İmaj Dağıtımı

Konteynerleşme süreci yalnızca yerel build ile tamamlanmaz. İmajın paylaşılması ve farklı ortamlara taşınması için registry mekanizması gerekir.

### Docker Hub

Genel kullanıma açık en yaygın registry’dir. Resmî taban imajların büyük bölümü buradan sağlanır.

### Özel Registry

Kurumlar çoğu zaman imajlarını özel registry üzerinde saklar. Bunun nedenleri şunlardır:

- Erişim denetimi
- İç ağ içinde yüksek hız
- Uyumluluk ve denetim gereksinimleri
- Güvenlik politikaları

### Etiketleme

İmajın anlamlı sürüm bilgileriyle etiketlenmesi gerekir:

```bash
docker tag user-service:1.0.0 registry.example.edu/user-service:1.0.0
```

### Push ve Pull

```bash
docker push registry.example.edu/user-service:1.0.0
docker pull registry.example.edu/user-service:1.0.0
```

İmajın farklı ortamlarda değişmeden kullanılabilmesi, CI/CD süreçlerinin temel avantajlarından biridir.

## 4.15 Konteyner Gözlemleme ve İşletim Komutları

Docker kullanımında build kadar işletim de önemlidir. Aşağıdaki komutlar günlük operasyonlarda sık kullanılır.

Çalışan konteynerleri listelemek:

```bash
docker ps
```

Tüm konteynerleri görmek:

```bash
docker ps -a
```

İmajları listelemek:

```bash
docker images
```

Konteyner loglarını izlemek:

```bash
docker logs -f user-service
```

Konteyner içine komut çalıştırmak:

```bash
docker exec -it user-service sh
```

Detaylı metadata görmek:

```bash
docker inspect user-service
```

Kaynak kullanımını gözlemlemek:

```bash
docker stats
```

Bu komutlar, uygulamanın yalnızca çalıştırılması için değil, teşhis ve hata ayıklama için de gereklidir.

## 4.16 Docker Compose

Gerçek uygulamalar çoğu zaman tek konteynerden oluşmaz. `user-service` uygulamasına PostgreSQL veritabanı ve gerekirse bir ters vekil eklendiğinde, ilişkili servisleri tek tek çalıştırmak yönetim yükü doğurur. Compose bu sorunu çözmek için kullanılır.

Compose, çoklu servisleri YAML tabanlı tek bir dosyada tanımlar. Servisler, ağlar, hacimler ve ortam değişkenleri birlikte yönetilir.

`user-service` için örnek bir `compose.yaml` dosyası aşağıdaki gibidir:

```yaml
services:
  user-service:
    build:
      context: .
      dockerfile: Dockerfile
    image: user-service:1.0.0
    container_name: user-service
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/users
      SPRING_DATASOURCE_USERNAME: users
      SPRING_DATASOURCE_PASSWORD: users
    depends_on:
      - postgres
    networks:
      - backend

  postgres:
    image: postgres:16-alpine
    container_name: postgres
    environment:
      POSTGRES_DB: users
      POSTGRES_USER: users
      POSTGRES_PASSWORD: users
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - backend

volumes:
  postgres-data:

networks:
  backend:
```

Compose ile sistemi başlatmak için:

```bash
docker compose up -d
```

Durdurmak için:

```bash
docker compose down
```

Volume’leri de silmek için:

```bash
docker compose down -v
```

Compose, geliştirici ortamı, test ortamı ve küçük ölçekli dağıtımlar için çok etkilidir. Çok düğümlü üretim orkestrasyonu söz konusu olduğunda daha ileri çözümler gündeme gelir.

## 4.17 Güvenlik Boyutu

Docker kullanımında güvenlik, yalnızca imajın çalışmasıyla ilgili değil, çalışma biçimiyle ilgilidir. Konteynerler tam sanal makine değildir; çekirdeği paylaştıkları için yanlış yapılandırmalar önemli riskler doğurabilir.

Temel güvenlik ilkeleri şunlardır:

- Gereksiz büyük taban imajlardan kaçınmak
- Root kullanıcıyla çalışmamak
- İmaj içine parola veya anahtar gömmemek
- Gereksiz portları açmamak
- Güncel ve güvenilir taban imaj kullanmak
- İmajları düzenli taramak
- En az ayrıcalık ilkesini uygulamak

`user-service` için güvenli kullanıcı tanımı şu biçimde yapılabilir:

```dockerfile
RUN addgroup --system spring && adduser --system spring --ingroup spring
USER spring:spring
```

Gizli bilgilerin Dockerfile içine yazılması yanlış bir yaklaşımdır:

```dockerfile
ENV DB_PASSWORD=supersecret
```

Bunun yerine çalışma anında dışarıdan ortam değişkeni, secret yönetimi veya platforma özgü güvenli mekanizmalar kullanılmalıdır.

## 4.18 Rootless ve İmtiyaz Yönetimi

Docker süreçleri çoğu zaman yüksek ayrıcalıklarla ilişkilendirilir. Bu nedenle rootless çalışma modelleri önem kazanmıştır. Rootless Docker, daemon ve konteyner süreçlerini kök kullanıcı ayrıcalığı olmadan çalıştırmayı hedefler.

Bu yaklaşım her durumda tam performans veya tüm özellikleri sunmayabilir; ancak güvenlik açısından değerlidir. Özellikle paylaşımlı geliştirme ortamlarında veya hassas sistemlerde rootless model anlamlıdır.

## 4.19 Kaynak Sınırlandırma

Konteynerler hafif olmaları nedeniyle sınırsız kaynak kullanacakları anlamına gelmez. Kaynak sınırları tanımlanmadığında tek bir konteyner ana makineyi baskılayabilir.

Örnek:

```bash
docker run -d \
  --memory="512m" \
  --cpus="1.0" \
  -p 8080:8080 \
  user-service:1.0.0
```

Bu sınırlar özellikle çok servisli ortamlarda ve CI düğümlerinde önemlidir.

## 4.20 Loglama ve Gözlenebilirlik

Konteyner içinde log dosyası tutmak çoğu zaman iyi yaklaşım değildir. Uygulama loglarının standart çıktı ve hata akışına yazılması tercih edilir. Docker bu akışları toplayabilir ve farklı log sürücülerine yönlendirebilir.

Spring Boot uygulamaları doğal olarak konsol logu ürettiği için `user-service` bu modele uygundur. Konteyner logları şu komutla izlenebilir:

```bash
docker logs -f user-service
```

Gerçek üretim ortamlarında bu loglar merkezi sistemlere aktarılır. Docker, burada tek başına tam gözlenebilirlik çözümü değil, veri kaynağıdır.

## 4.21 Docker ve CI/CD İlişkisi

Önceki bölümde kurulan Jenkins hattı, Docker ile daha güçlü bir teslimat modeline dönüştürülebilir. CI hattı önce uygulamayı derler ve test eder, ardından güvenilir build çıktısından imaj üretir.

`user-service` için Jenkins pipeline içinde Docker adımı şu biçimde genişletilebilir:

```groovy
pipeline {
    agent { label 'maven-jdk17-docker' }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Test') {
            steps {
                sh 'mvn -B test'
            }
        }

        stage('Package') {
            steps {
                sh 'mvn -B clean package -DskipTests'
            }
        }

        stage('Docker Build') {
            steps {
                sh 'docker build -t registry.example.edu/user-service:${BUILD_NUMBER} .'
            }
        }
    }
}
```

Bu yaklaşımda önemli olan ilke, hatalı veya test edilmemiş yazılımın imaj hâline getirilmemesidir. Docker, kalite kontrolünün yerine geçmez; doğrulanmış çıktının paketlenmesini sağlar.

## 4.22 Docker Kullanımında Yaygın Anti-Pattern’ler

### Uygulamayı Root Olarak Çalıştırmak

Varsayılan kullanıcıyla bırakılan birçok imaj root olarak çalışır. Bu durum saldırı yüzeyini büyütür.

### Çok Büyük Taban İmaj Kullanmak

İhtiyaç duyulmayan araçları içeren büyük imajlar indirme süresini, güvenlik riskini ve bakım maliyetini artırır.

### Aynı İmajı Hem Build Hem Runtime İçin Kullanmak

Maven, kaynak kod ve derleme araçlarını son imaj içine bırakmak gereksizdir. Çok aşamalı build tercih edilmelidir.

### Gizli Bilgileri İmaja Yazmak

Parola, token ve anahtar gibi değerlerin Dockerfile veya imaj katmanında bulunması ciddi güvenlik açığıdır.

### Veriyi Konteyner Katmanına Yazmak

Veritabanı veya kritik veri yalnızca konteyner yazılabilir katmanında tutulursa konteyner silindiğinde veri kaybı yaşanır.

### `latest` Etiketine Güvenmek

`latest` etiketi açık ve sabit sürüm bilgisi sağlamaz. Üretim süreçlerinde açık sürüm etiketleri veya digest tercih edilmelidir.

### Tek Konteyner İçinde Birden Fazla Birincil Süreç Çalıştırmak

Konteyner tasarımında tek sorumluluk ilkesi önemlidir. Uygulama, veritabanı ve yardımcı süreçleri aynı konteyner içine yığmak bakım zorluğu doğurur.

## 4.23 Baştan Sona Uygulama Senaryosu

`user-service` için Docker tabanlı tipik iş akışı aşağıdaki gibidir:

1. Geliştirici Spring Boot uygulamasında değişiklik yapar.
2. Kod GitHub’a gönderilir.
3. Jenkins build ve test süreçlerini çalıştırır.
4. Testler başarılı ise Docker imajı üretilir.
5. İmaj `registry.example.edu/user-service:42` gibi bir etiketle registry’ye gönderilir.
6. Test ortamı bu imajı çekerek konteyneri ayağa kaldırır.
7. Gerekli ise Compose veya orkestrasyon katmanı üzerinden veritabanı ile birlikte çalıştırılır.

Yerel geliştirme açısından örnek komut akışı şu biçimdedir:

```bash
mvn -B clean package
docker build -t user-service:dev .
docker run -d --name user-service -p 8080:8080 user-service:dev
curl http://localhost:8080/api/users/1
docker logs user-service
docker stop user-service
docker rm user-service
```

Bu akış, kaynak koddan çalışan konteynere kadar olan hattı görünür ve tekrarlanabilir hâle getirir.

## 4.24 Orkestrasyona Geçiş

Docker, tekil host ve sınırlı sayıda servis için güçlü bir araçtır. Fakat düğüm sayısı arttığında, servis keşfi, otomatik iyileşme, yatay ölçekleme, rolling update ve secret yönetimi gibi ihtiyaçlar daha gelişmiş orkestrasyon katmanlarını gündeme getirir.

Bu nedenle Docker bilgisi çoğu zaman Kubernetes gibi platformlara geçişin ön koşuludur. İmaj, konteyner, ağ ve hacim kavramları anlaşılmadan orkestrasyon katmanlarını sağlıklı değerlendirmek mümkün değildir.

## 4.25 Değerlendirme

Docker, uygulamayı yalnızca paketleyen bir araç değil, çalıştırma bağlamını standartlaştıran bir platformdur. Değeri; imaj, konteyner, katman, ağ, hacim, registry ve çoklu servis yönetimi gibi bileşenlerin birlikte çalışmasından doğar.

`user-service` örneği üzerinden değerlendirildiğinde Docker’ın sağladığı başlıca kazanımlar şunlardır:

- Uygulamanın farklı ortamlarda tutarlı biçimde çalışması
- Build ile runtime ortamının ayrıştırılması
- Taşınabilir, sürümlenebilir teslimat birimi elde edilmesi
- Ağ ve veri depolama ihtiyaçlarının standart soyutlamalarla yönetilmesi
- CI hattından üretim hattına uzanan ortak paketleme modeli kurulması

Docker’ın etkili kullanımı için komut ezberlemek yeterli değildir. İmajın nasıl üretildiği, konteynerin nasıl çalıştığı, verinin nerede tutulduğu, ağın nasıl kurulduğu ve güvenliğin nasıl sağlandığı bütünlüklü biçimde anlaşılmalıdır. Bu bütünlük sağlandığında Docker, DevOps pratiğinde tekrar üretilebilirlik, taşınabilirlik ve işletim disiplini sağlayan temel araçlardan biri hâline gelir.
