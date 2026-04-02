# Bölüm 3  
# Sürekli Entegrasyon ve Jenkins: Mimari, Çalışma Modeli ve Uygulama Süreci

Önceki bölümde Git ile sürümlenen `user-service` adlı Java Spring Boot REST API deposu temel alınmaktadır. Bu proje, kullanıcı oluşturma, listeleme ve tekil kullanıcı sorgulama işlevlerini sağlayan bir servis olarak tasarlanmıştır. Kod tabanı Maven ile derlenmekte, testler JUnit 5 ve Spring Boot Test ile yürütülmektedir. Sürekli entegrasyon yaklaşımı, bu tür bir projede yalnızca otomatik derleme yapmak için değil, ekip içi entegrasyon riskini denetim altına almak için de kullanılmaktadır.

Depoda kullanılan temel yapı aşağıdaki gibidir:

```text
user-service/
├── src/
│   ├── main/
│   │   └── java/com/example/userservice/
│   │       ├── controller/
│   │       ├── service/
│   │       ├── dto/
│   │       └── UserServiceApplication.java
│   └── test/
│       └── java/com/example/userservice/
│           ├── controller/
│           └── service/
├── pom.xml
└── Jenkinsfile
```

Projede kullanılan Maven tanımı aşağıdaki biçimdedir:

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>user-service</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>user-service</name>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.3.2</version>
    </parent>

    <properties>
        <java.version>17</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

## 3.1 Manuel Build ve Entegrasyon Problemleri

Yazılım geliştirme sürecinde ilk büyük kırılma noktalarından biri, kodun bireysel olarak çalışması ile ekip içinde entegre biçimde çalışması arasındaki farktır. Bir geliştirici kendi makinesinde başarılı biçimde derlenen bir özelliği depoya gönderdiğinde, bu durum kodun ekip ortamında sorunsuz biçimde çalışacağı anlamına gelmez. Manuel build yaklaşımı, özellikle ekip büyüdükçe ve teslimat sıklığı arttıkça güvenilmez hâle gelir.

`user-service` projesi üzerinden tipik bir durum ele alınabilir. Bir geliştirici kullanıcı oluşturma akışına e-posta doğrulaması eklemekte, başka bir geliştirici ise kullanıcı listeleme uç noktasında yeni filtreleme parametreleri tanımlamaktadır. Her iki geliştirici de değişikliklerini kendi makinelerinde test etmiş olabilir. Buna rağmen aşağıdaki riskler oluşur:

- Farklı JDK sürümleriyle derleme yapılması
- Bir geliştiricinin yerel Maven önbelleğinde bulunan bağımlılıklara dayanılması
- Testlerin bir geliştirici tarafından çalıştırılıp diğerinde çalıştırılmaması
- Aynı sınıf veya yapılandırma üzerinde yapılan değişikliklerin entegrasyon sırasında çakışması
- Derleme komutunun ekip içinde standart olmaması

Bu koşullarda manuel entegrasyon süreci çoğunlukla şu biçimde ilerler:

1. Geliştirici kodu `main` dalına veya ortak dala gönderir.
2. Başka biri kodu çekerek yerel makinesinde derlemeye çalışır.
3. Testler ayrı ayrı yürütülür.
4. Hata varsa hata kaynağı derleme ortamı mı, kod mu, bağımlılık mı sorusu gündeme gelir.
5. Sorunun fark edilmesi ile düzeltilmesi arasında gecikme oluşur.

Bu modelde asıl problem, hatanın varlığı değil, hatanın geç fark edilmesidir. Entegrasyon ne kadar gecikirse hata bağlamı o kadar kaybolur. Hangi commit’in hangi yan etkiye yol açtığı ayırt edilmesi güç bir meseleye dönüşür.

`user-service` için basit bir controller değişikliği bile bu tür bir soruna yol açabilir:

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @PostMapping
    public ResponseEntity<UserResponse> createUser(@Valid @RequestBody CreateUserRequest request) {
        UserResponse response = userService.createUser(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }

    @GetMapping("/{id}")
    public ResponseEntity<UserResponse> getUserById(@PathVariable Long id) {
        return ResponseEntity.ok(userService.getById(id));
    }
}
```

Eğer `CreateUserRequest` sınıfında alan adı değişmiş, fakat testler güncellenmemişse yerel ortamda fark edilmeyen bir hata, ortak dalda bütün ekibin akışını durdurabilir. Sürekli entegrasyon yaklaşımı bu riski azaltmak için ortaya çıkmıştır.

## 3.2 Sürekli Entegrasyon Kavramı

Sürekli entegrasyon, geliştiricilerin kodlarını kısa aralıklarla ortak depoya entegre etmesi ve her entegrasyon sonrasında otomatik doğrulama işlemlerinin yürütülmesi ilkesine dayanır. Bu yaklaşımın merkezinde yalnızca otomasyon değil, entegrasyon sıklığının artırılması vardır. Asıl amaç, kod tabanının sürekli olarak çalışabilir durumda tutulmasıdır.

Sürekli entegrasyonun teorik temeli birkaç esas üzerine kuruludur:

- Küçük ve sık değişiklikler büyük ve seyrek değişikliklerden daha düşük entegrasyon maliyeti üretir.
- Derleme ve test süreçleri insan inisiyatifinden çıkarıldığında süreç daha güvenilir hâle gelir.
- Ortak dalın sağlık durumu sürekli gözlenebilir olmalıdır.
- Hata mümkün olan en erken noktada yakalanmalıdır.
- Geliştiricinin geri bildirim döngüsü kısa tutulmalıdır.

Bu çerçevede sürekli entegrasyon, teknik olduğu kadar organizasyonel bir disiplindir. Ekip üyeleri kısa ömürlü dallarla çalışır, değişiklikler küçük parçalar hâlinde birleştirilir, her commit sistem tarafından doğrulanır. Amaç yalnızca “kodun derlenmesi” değildir; kodun ekip standartlarına, test beklentilerine ve teslimat hattının geri kalanına uygunluğunun denetlenmesidir.

`user-service` projesinde sürekli entegrasyonun temel kontrol noktaları şunlardır:

- Kodun çekilmesi
- Maven bağımlılıklarının çözülmesi
- Derleme yapılması
- Birim testlerinin çalıştırılması
- Test raporlarının kaydedilmesi
- Üretilen `jar` dosyasının arşivlenmesi

Bu işlem zinciri, geliştiricinin makinesinde değil, tanımlı bir entegrasyon ortamında yürütüldüğünde ekip için ortak bir doğruluk zemini oluşur.

## 3.3 Jenkins’in Ortaya Çıkışı ve Rolü

Sürekli entegrasyon pratiği, ekiplerin elle yürüttüğü build ve test işlemlerinin sürdürülemez hâle gelmesiyle daha sistematik araçlara ihtiyaç duymuştur. Jenkins bu ihtiyacın en yaygın karşılıklarından biri olarak gelişmiştir. Açık kaynak olması, eklenti mimarisi ve farklı teknoloji yığınlarıyla uyumlu çalışabilmesi, Jenkins’i kurumsal ortamlarda uzun süre güçlü bir seçenek hâline getirmiştir.

Jenkins’in rolü yalnızca komut çalıştırmak değildir. Jenkins, yazılım geliştirme sürecinde şu işlevleri üstlenir:

- Kaynak kod deposundaki değişiklikleri algılar.
- Bu değişiklikleri belirli kurallara göre build işine dönüştürür.
- Derleme ve test işlemlerini tanımlı yürütme düğümlerinde çalıştırır.
- Sonuçları görünür ve izlenebilir kılar.
- Hata durumunda erken geri bildirim üretir.
- Teslimat hattının sonraki aşamalarına veri ve çıktı sağlar.

Bu yönüyle Jenkins, bir “otomasyon sunucusu” olmanın ötesinde, yazılım üretim sürecinin akış denetleyicisidir. `user-service` örneğinde Jenkins, GitHub üzerinde gerçekleşen her değişikliği gözleyerek projenin derlenebilir ve test edilebilir durumda kalmasını sağlar. Ekip için güvenilir referans ortamı Jenkins olur; herhangi bir geliştiricinin kişisel makinesi değil.

## 3.4 Jenkins Mimarisi

Jenkins’in doğru anlaşılması için dışarıdan görünen arayüzün ötesine geçmek gerekir. Bir Jenkins kurulumu, birbirine bağlı yürütme bileşenlerinden oluşur. Bu bileşenler, işin tanımlanmasından fiziksel olarak çalıştırılmasına kadar farklı sorumluluklar taşır.

### Controller (Master)

Controller, Jenkins sisteminin yönetim merkezidir. Tarihsel olarak “Master” adıyla anılmıştır. Güncel terminolojide “Controller” ifadesi tercih edilmektedir. Controller’ın başlıca görevleri şunlardır:

- Yapılandırmaları saklamak
- Job tanımlarını yönetmek
- Build isteklerini sıraya almak
- Agent’larla iletişim kurmak
- Kullanıcı arayüzünü ve API’yi sunmak
- Sonuçları ve build geçmişini tutmak

Controller, çoğu zaman işin kendisini yürütmekten çok, yürütmenin organizasyonunu yapar. Büyük ölçekli kurulumlarda controller üzerinde doğrudan build çalıştırmak tercih edilmez. Çünkü controller’ın kararlı kalması, tüm sistemin çalışabilirliği açısından kritiktir.

`user-service` için GitHub’dan gelen webhook çağrısı önce controller tarafından alınır. Controller, bu çağrıyı ilgili job ile eşleştirir ve build talebini sıraya ekler.

### Agent (Worker)

Agent, build ve test gibi işlerin fiilen çalıştırıldığı yürütme düğümüdür. “Worker” kavramı ile aynı mantığı ifade eder. Agent fiziksel bir sunucu, sanal makine, kapsayıcı veya bulut üstünde geçici olarak açılan bir yürütme ortamı olabilir.

Agent kullanımının temel gerekçeleri şunlardır:

- Build yükünü controller’dan ayırmak
- Farklı işletim sistemi ve araç seti gerektiren işleri bölmek
- Paralel yürütme kapasitesi sağlamak
- İzole build ortamları kurmak

Örneğin `user-service` projesi için JDK 17 ve Maven 3.9 içeren bir Linux agent tanımlanabilir. Jenkins, bu etikete sahip agent üzerinde pipeline’ı çalıştırır. Android, Node.js veya Docker ağırlıklı işler için başka agent’lar tanımlanabilir.

### Executor

Executor, bir agent veya controller üzerinde aynı anda kaç build’in çalışabileceğini belirleyen yürütme yuvasıdır. Tek bir agent birden fazla executor barındırabilir. Her executor aynı anda bir build veya pipeline dalını çalıştırır.

Örnek olarak dört çekirdekli bir makinede iki executor tanımlandığında aynı anda iki bağımsız build koşabilir. Ancak executor sayısını artırmak, otomatik olarak verim artışı anlamına gelmez. Ağır Maven build’leri, yüksek bellek tüketen testler veya Docker image üretimi gibi işlemler kaynak yarışına yol açabilir.

`user-service` gibi orta ölçekli projelerde agent başına tek executor kullanımı daha öngörülebilir bir davranış sağlar. Bu yaklaşım özellikle kapsayıcı tabanlı ephemeral agent kurulumlarında yaygındır.

### Queue

Queue, çalıştırılmak üzere bekleyen işlerin tutulduğu sıradır. Bir build tetiklendiğinde Jenkins işi doğrudan çalıştırmaz; önce queue’ya alır. Ardından uygun executor ve agent bulunduğunda iş yürütülür.

Queue mekanizması aşağıdaki durumlarda kritik hâle gelir:

- Aynı anda çok sayıda commit geldiğinde
- Uygun etikete sahip agent sınırlı olduğunda
- Bir job için eşzamanlı çalıştırma kapatıldığında
- Kaynak kilidi veya sıra politikaları uygulandığında

`user-service` için kısa aralıklarla üç commit gönderildiğinde, Jenkins bu build’leri kuyrukta tutabilir. Eğer `disableConcurrentBuilds()` kullanılmışsa aynı job’ın ikinci örneği ilk build bitmeden başlamaz.

### Workspace

Workspace, job’ın agent üzerinde çalıştığı dizindir. Kaynak kod, build çıktıları, geçici dosyalar ve test raporları bu alanda bulunur. Jenkins her build’i belirli bir workspace içinde yürütür.

Workspace kavramı birçok nedenle önemlidir:

- Kaynak kodun fiziksel olarak bulunduğu alan olması
- Maven hedef dizinlerinin ve raporların bu alanda üretilmesi
- Eski build kalıntılarının yeni build’i etkileme riskini taşıması
- İzolasyon düzeyinin büyük ölçüde workspace yönetimine bağlı olması

`user-service` için workspace örneği şu biçimde olabilir:

```text
/var/lib/jenkins/workspace/user-service
```

Eşzamanlı build durumunda Jenkins benzer adlarla yeni workspace’ler oluşturabilir:

```text
/var/lib/jenkins/workspace/user-service@2
```

Bununla birlikte yalnızca farklı klasör kullanmak tam izolasyon sağlamaz. Gerçek izolasyon çoğu zaman temiz workspace politikaları ve geçici agent kullanımıyla elde edilir.

### Job

Job, Jenkins’te yapılacak işin tanımıdır. Hangi kaynaktan kod alınacağı, hangi komutların çalıştırılacağı, hangi tetikleyicilerin kullanılacağı ve hangi çıktıların saklanacağı job tanımı içinde yer alır.

Job kavramı Jenkins’in en eski soyutlamalarından biridir. Freestyle job, Maven job, pipeline job ve multibranch pipeline bu modelin farklı biçimleridir.

`user-service` için bir job şu görevleri tanımlayabilir:

- GitHub deposundan `main` dalını izlemek
- Push olduğunda build başlatmak
- `mvn clean test` komutunu çalıştırmak
- Test raporlarını toplamak
- Başarılı build’de `jar` dosyasını arşivlemek

### Build

Build, bir job’ın belirli bir zamanda başlatılmış tekil yürütme örneğidir. Job tanımı sabittir; build ise bu tanımın tarihsel bir kaydıdır. Her build’in bir numarası, başlangıç zamanı, süresi, sonucu ve log’u bulunur.

Build sonucu genellikle şu durumlardan birine karşılık gelir:

- `SUCCESS`
- `FAILURE`
- `UNSTABLE`
- `ABORTED`

`UNSTABLE` durumu, build’in teknik olarak tamamlanmış fakat testlerin başarısız olduğu durumlarda anlamlıdır. Bu ayrım, derleme hatası ile kalite hatasını ayırmak açısından önem taşır.

## 3.5 Jenkins Çalışma Modeli

Jenkins’in çalışma modelini anlamanın en verimli yolu süreci commit anından sonuç üretimine kadar izlemektir. `user-service` projesi için tipik akış aşağıdaki gibidir.

### Commit

Geliştirici `feature/email-validation` dalında bir değişiklik yapar. Amaç, kullanıcı oluşturulurken e-posta alanının biçimsel doğrulamasını sağlamaktır.

```java
public record CreateUserRequest(
        @NotBlank String firstName,
        @NotBlank String lastName,
        @Email @NotBlank String email
) {
}
```

Değişiklik commit edilip GitHub’a gönderildiğinde kaynak kod deposunda yeni bir olay oluşur.

### Webhook Tetiklenmesi

GitHub, ilgili depo için tanımlanmış webhook adresine HTTP `POST` isteği gönderir. Bu istek, belirli bir olayın meydana geldiğini bildirir. Push olayı en yaygın örnektir. Jenkins bu çağrıyı alır ve olayın ilgili job ile ilişkisini kurar.

### Job Başlatılması

Controller, webhook’tan gelen bilgiyi kullanarak hangi job’ın veya multibranch pipeline dalının çalıştırılacağını belirler. `user-service` için `main` dalı ile `feature/*` dalları farklı alt işlere karşılık gelebilir.

### Agent Seçimi

Job tanımında veya Jenkinsfile içinde istenen etiketlere uygun agent aranır. Örnek olarak `maven-jdk17` etiketine sahip bir agent seçilebilir. Eğer uygun agent meşgulse build queue’da bekler.

### Workspace Oluşturulması

Jenkins seçilen agent üzerinde build için bir workspace ayırır. Bu alan mevcutsa temizlenebilir, yoksa oluşturulur. Workspace, build’in dosya sistemi bağlamıdır.

### Kod Çekme

Jenkins Git istemcisi aracılığıyla depoyu workspace içine çeker. Eğer pipeline SCM tabanlı tanımlanmışsa `Jenkinsfile` da aynı depodan alınır.

```bash
git clone https://github.com/example/user-service.git
git checkout main
```

Pratikte bu işlem Jenkins içindeki `checkout scm` veya `git` step’iyle yönetilir.

### Build

Maven yaşam döngüsü kullanılarak proje derlenir. Java kaynak kodu `target/classes` içine çevrilir, gerekli bağımlılıklar çözülür ve paketleme yapılır.

```bash
mvn -B clean package
```

Buradaki `-B` parametresi batch mode anlamına gelir ve CI ortamları için daha uygun log üretir.

### Test

Birim testleri ve varsa entegrasyon testleri çalıştırılır. Testler başarısızsa build durumu `FAILURE` veya `UNSTABLE` olabilir. `user-service` için temel bir test aşağıdaki gibidir:

```java
@WebMvcTest(UserController.class)
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private UserService userService;

    @Test
    void shouldReturnUserById() throws Exception {
        UserResponse response = new UserResponse(1L, "Ada", "Lovelace", "ada@example.com");
        given(userService.getById(1L)).willReturn(response);

        mockMvc.perform(get("/api/users/1"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.email").value("ada@example.com"));
    }
}
```

Bu testin otomatik olarak her build’de çalıştırılması, endpoint davranışındaki beklenmedik kırılmaları erken aşamada yakalar.

### Sonuç

Build tamamlandığında Jenkins sonucu kaydeder, log üretir, test raporlarını gösterir ve gerekli ise bildirim yollarını devreye alır. Başarılı build’lerde `jar` dosyası arşivlenebilir. Başarısız build’lerde ekip, hatalı commit’i kısa sürede saptayabilir.

## 3.6 Freestyle Job ve Pipeline

Jenkins tarihsel olarak Freestyle job modeliyle yaygınlaşmıştır. Bu yaklaşım, kullanıcı arayüzü üzerinden adım adım yapılandırılan işlerden oluşur. Basit projelerde kolay kullanım sağlar; ancak sürüm kontrolü, gözden geçirme ve tekrarlanabilirlik açısından sınırlıdır.

Pipeline yaklaşımı ise build mantığının kod olarak tanımlanmasını sağlar. Jenkinsfile dosyası proje deposuna eklenir ve iş akışı kaynak kodla birlikte sürümlenir.

İki yaklaşım arasındaki temel farklar aşağıdaki tabloda görülmektedir:

| Ölçüt | Freestyle Job | Pipeline |
|---|---|---|
| Tanımlama biçimi | Arayüz üzerinden | Kod ile |
| Sürüm kontrolü | Zayıf | Güçlü |
| Kod inceleme süreci | Dolaylı | Doğrudan |
| Karmaşık akışlar | Sınırlı | Uygun |
| Tekrarlanabilirlik | Yapılandırmaya bağlı | Yüksek |
| Çok aşamalı süreçler | Zorlaşır | Doğal biçimde modellenir |
| Dal bazlı farklılıklar | Sınırlı | Kolay yönetilir |

`user-service` gibi ekip içinde aktif geliştirilen projelerde pipeline yaklaşımı daha uygundur. Çünkü build davranışı proje tanımının bir parçası hâline gelir. Build mantığı, uygulama kodu gibi incelenebilir, değiştirilebilir ve geçmişi izlenebilir bir varlığa dönüşür.

## 3.7 Pipeline Kavramı

Pipeline, build ve teslimat sürecinin birbirini izleyen aşamalar hâlinde tanımlanmasıdır. Jenkins’te pipeline tasarımı rastgele değildir; yazılım üretim sürecinin gözlenebilir, tekrarlanabilir ve denetlenebilir parçalara ayrılması amacı taşır.

### Stage

Stage, pipeline içindeki mantıksal aşamayı temsil eder. `Checkout`, `Build`, `Test`, `Package` gibi adlar stage olarak tanımlanır. Stage kullanımı iki nedenle önemlidir:

- Süreci insan tarafından anlaşılır kılar.
- Jenkins arayüzünde hangi aşamada sorun oluştuğunu görünür kılar.

`user-service` için tipik stage dizisi aşağıdaki gibidir:

- `Checkout`
- `Build`
- `Test`
- `Archive`

### Step

Step, stage içindeki tekil yürütme birimidir. `sh`, `git`, `checkout`, `junit`, `archiveArtifacts` gibi eylemler step örneğidir. Step düzeyi, Jenkins’in eklenti mimarisiyle doğrudan ilişkilidir. Her eklenti pipeline’a yeni step’ler kazandırabilir.

Örnek:

```groovy
stage('Build') {
    steps {
        sh 'mvn -B clean package -DskipTests'
    }
}
```

Bu örnekte `sh` bir step’tir. Çalıştırılan kabuk komutu step’in gerçek yükünü taşır.

### Declarative ve Scripted Pipeline

Jenkins iki temel pipeline stili sunar: declarative ve scripted.

Declarative pipeline daha kural tabanlı, yapılandırılmış ve okunabilir bir modeldir. Kurumsal ekiplerde standartlaştırma açısından çoğunlukla ilk tercihtir.

```groovy
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                sh 'mvn -B clean package'
            }
        }
    }
}
```

Scripted pipeline ise Groovy tabanlı daha esnek bir yaklaşımdır. Karmaşık koşullar, dinamik akışlar ve özel mantıklar için güçlüdür; ancak okunabilirlik ve bakım maliyeti daha yüksek olabilir.

```groovy
node('maven-jdk17') {
    stage('Build') {
        sh 'mvn -B clean package'
    }

    if (env.BRANCH_NAME == 'main') {
        stage('Archive') {
            archiveArtifacts artifacts: 'target/*.jar'
        }
    }
}
```

Pipeline mantığının bu biçimde tasarlanmasının temel nedeni, süreç modellemesi ile yürütme esnekliğini dengelemektir. Declarative yaklaşım standart akışların doğrulanmasını ve ekip içinde ortak dil oluşmasını sağlar. Scripted yaklaşım ise daha düşük seviyeli kontrol gereken durumlarda devreye girer. Jenkins’in iki modeli birlikte sunması, onu yalnızca “komut çalıştıran araç” olmaktan çıkarıp akış motoru niteliğine taşır.

## 3.8 Jenkinsfile Kullanımı

Jenkinsfile, pipeline tanımının proje deposunda saklandığı dosyadır. Kaynak kodla aynı repoda tutulması birkaç açıdan önem taşır:

- Build mantığı kodla birlikte sürümlenir.
- Hangi commit’in hangi build tanımını kullandığı netleşir.
- Kod inceleme sürecinde pipeline değişiklikleri de denetlenir.
- Farklı dallar farklı Jenkinsfile sürümleri taşıyabilir.

`user-service` için gerçekçi bir Jenkinsfile aşağıdaki gibi tanımlanabilir:

```groovy
pipeline {
    agent { label 'maven-jdk17' }

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }

    triggers {
        githubPush()
    }

    environment {
        MAVEN_OPTS = '-Dmaven.test.failure.ignore=false'
    }

    stages {
        stage('Checkout') {
            steps {
                deleteDir()
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'mvn -B clean package -DskipTests'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn -B test'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('Archive') {
            when {
                branch 'main'
            }
            steps {
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }
    }

    post {
        success {
            echo 'user-service build basarili.'
        }
        failure {
            echo 'user-service build basarisiz.'
        }
    }
}
```

Bu dosya, bir build’in hangi sırayla ve hangi kurallarla çalışacağını açık biçimde tanımlar. `Checkout` aşamasında depo çekilir, `Build` aşamasında derleme yapılır, `Test` aşamasında test raporları toplanır, `Archive` aşamasında ise yalnızca `main` dalında oluşan paketler saklanır.

## 3.9 Pipeline’ın Çalışma Mantığı

Pipeline’ın değeri yalnızca yazılmış olmasında değil, nasıl çalıştığında yatar. Jenkins pipeline yürütülürken bir dizi iç mekanizma devreye girer.

İlk olarak controller, Jenkinsfile’ı okur ve yapısal doğrulama yapar. Declarative pipeline kullanılıyorsa dosya önce belirli bir şemaya göre çözümlenir. Ardından pipeline, Jenkins’in yürütme modeline uygun adımlara ayrılır.

Bu noktada önemli olan husus, pipeline’ın doğrusal bir kabuk betiği olmamasıdır. Jenkins, stage ve step bilgilerini ayrı ayrı takip eder. Bu sayede:

- Hangi aşamanın ne kadar sürdüğü görülebilir.
- Başarısızlığın hangi adımda olduğu belirlenebilir.
- Bazı aşamalar için koşullu yürütme uygulanabilir.
- Paralel akışlar modellenebilir.
- Sonuçlar Jenkins arayüzünde semantik olarak sunulabilir.

Pipeline tasarımının bu biçimde olması, CI sürecini yalnızca “bir dizi komut” olmaktan çıkarır. Süreç, denetlenebilir bir iş akışına dönüşür. Bu tasarım özellikle ekip ölçeği büyüdüğünde önem kazanır. Tek satırlık bir `ci.sh` betiği ile aynı görünürlük ve izlenebilirlik elde edilemez.

Jenkins ayrıca pipeline yürütmesini belirli adım sınırlarında durum bilgisiyle yönetebilir. Bu özellik, uzun süren süreçlerde dayanıklılık sağlar. Sistemin beklenmedik kesintiler karşısında toparlanabilmesi, Jenkins’in pipeline motorunu geleneksel betik çalıştırma yaklaşımından ayıran önemli bir özelliktir.

## 3.10 Jenkins ve GitHub Entegrasyonu

Modern geliştirme süreçlerinde Jenkins çoğunlukla GitHub ile birlikte çalışır. Kaynak kodun GitHub’da bulunması, Jenkins’in bu depo ile güvenli ve izlenebilir biçimde konuşmasını gerektirir.

`user-service` için GitHub entegrasyonunun temel unsurları şunlardır:

- Jenkins’in depoya erişim yetkisi
- Jenkins job’ının hangi dalı veya dalları izleyeceğinin tanımlanması
- Webhook ile olay bazlı tetikleme
- İstenirse pull request bazlı build doğrulaması

Jenkins, GitHub’a iki farklı modelle bağlanabilir:

- HTTPS ve erişim belirteci kullanarak
- SSH anahtarları kullanarak

Kurumsal ortamlarda kimlik bilgileri Jenkins Credentials Store içinde saklanır. Bu bilgilerin doğrudan Jenkinsfile içine yazılması güvenlik açısından kabul edilemez.

Kaynak kod çekme adımı çoğunlukla şu mantıkla işler:

```groovy
stage('Checkout') {
    steps {
        checkout scm
    }
}
```

`checkout scm` ifadesi, job’ın bağlı olduğu kaynak kod yönetim tanımını kullanır. Bu, özellikle multibranch pipeline projelerinde güçlü bir yaklaşımdır; çünkü branch bağlamı otomatik korunur.

GitHub entegrasyonu ile birlikte Jenkins, yalnızca kodu çeken bir araç olmaktan çıkar; depo üzerinde gerçekleşen olaylara tepki veren bir yürütme katmanı hâline gelir.

## 3.11 Webhook Mekanizması

Webhook, bir sistemdeki olayın başka bir sisteme anlık olarak bildirilmesini sağlayan HTTP tabanlı çağrıdır. Sürekli entegrasyon bağlamında webhook, depodaki değişiklikleri düzenli aralıklarla sorgulamak yerine olay anında bildirmek için kullanılır.

GitHub’da bir push olayı gerçekleştiğinde Jenkins’e benzer bir istek gönderilir:

```http
POST /github-webhook/ HTTP/1.1
Host: jenkins.example.edu
X-GitHub-Event: push
Content-Type: application/json
```

İstek gövdesinde commit bilgileri, dal adı, depo adı ve gönderen kullanıcı gibi alanlar yer alır. Jenkins bu veriyi yorumlayarak ilgili job’ı tetikler.

Webhook mekanizmasının başlıca avantajları şunlardır:

- Gecikme düşer
- Gereksiz polling yükü ortadan kalkar
- Olay ile build başlangıcı arasındaki bağ kuvvetlenir
- Build kayıtları commit geçmişiyle daha tutarlı ilişkilendirilir

`user-service` projesinde bir geliştirici `main` dalına push yaptığında GitHub webhook’u Jenkins controller’a ulaşır, controller ilgili pipeline’ı kuyruğa ekler ve build yaşam döngüsü başlar. Bu mekanizma, CI sürecinin aktif olarak olay güdümlü çalışmasını sağlar.

## 3.12 Workspace ve Build İzolasyonu

Workspace izolasyonu, sürekli entegrasyon sistemlerinin çoğu zaman göz ardı edilen fakat kritik başlıklarından biridir. Bir build’in çıktısının başka bir build’i etkilememesi gerekir. Aksi durumda CI sistemi güvenilirlik üretmez; rastlantısal başarı ve başarısızlıklar üretir.

`user-service` için izolasyon gerektiren başlıca noktalar şunlardır:

- `target/` klasöründeki eski derleme çıktıları
- Geçmiş test raporları
- Geçici dosyalar
- Aynı job’ın ardışık build’leri
- Farklı dalların birbirinden etkilenmesi

Bu nedenle pipeline içinde workspace temizliği yaygın biçimde kullanılır:

```groovy
stage('Checkout') {
    steps {
        deleteDir()
        checkout scm
    }
}
```

`deleteDir()` step’i, bir önceki build’den kalan artıkların temizlenmesini sağlar. Bununla birlikte build izolasyonu yalnızca dizin temizliğiyle sınırlı değildir. Daha güçlü izolasyon yaklaşımları şunlardır:

- Her build için geçici agent kullanımı
- Kapsayıcı tabanlı build ortamları
- Salt okunur araç imajları
- Eşzamanlı build’lerin sınırlandırılması

Maven projelerinde yerel depo önbelleği de dikkat gerektirir. Ortak bir `.m2` önbelleği build sürelerini azaltabilir; ancak bozuk veya yarım kalmış bağımlılık indirmeleri beklenmedik sorunlara yol açabilir. Bu nedenle hız ile güvenilirlik arasında bilinçli tercih yapılmalıdır.

## 3.13 Agent Kullanımı ve Dağıtık Yapı

Jenkins’in dağıtık mimarisi, büyük ölçüde agent kavramı üzerine kuruludur. Tek sunuculu küçük kurulumlarda controller ve yürütme düğümü aynı makinede olabilir. Fakat bu yaklaşım ölçeklenebilir değildir.

Dağıtık yapı aşağıdaki gereksinimlerden doğar:

- Farklı projelerin farklı araç zincirlerine ihtiyaç duyması
- İş yükünün yatay olarak dağıtılması
- Güvenlik sınırlarının ayrılması
- Geçici ve temiz build ortamlarının sağlanması

`user-service` projesi için aşağıdaki agent etiketi tanımlanabilir:

- `maven-jdk17`
- `docker`
- `linux`

Pipeline bu etiketi açık biçimde kullanabilir:

```groovy
pipeline {
    agent { label 'maven-jdk17' }
    stages {
        stage('Build') {
            steps {
                sh 'mvn -B clean package'
            }
        }
    }
}
```

Bu yaklaşım, build’in hangi özelliklere sahip bir düğümde koşacağını tanımlı hâle getirir. Jenkins, kaynakları buna göre planlar.

Dağıtık yapının önemli bir sonucu, controller ile agent arasındaki sorumluluk ayrımıdır. Controller planlar, agent yürütür. Bu ayrım bozulduğunda aşağıdaki sorunlar ortaya çıkar:

- Controller üzerinde aşırı kaynak tüketimi
- Yönetim panelinin yavaşlaması
- Bütün sistemin tek hata noktasına dönüşmesi
- Bakım ve ölçekleme sorunları

Kurumsal düzeyde Jenkins kullanımlarında controller’ın mümkün olduğunca hafif tutulması, build yükünün agent’lara kaydırılması temel ilkedir.

## 3.14 Anti-Pattern Örnekleri

Sürekli entegrasyon sistemleri kurulurken yalnızca çalışan bir yapı kurmak yeterli değildir; hatalı tasarımlardan kaçınmak da gerekir. `user-service` projesi üzerinden sık karşılaşılan anti-pattern’ler aşağıda incelenmiştir.

### Controller Üzerinde Build Çalıştırmak

Küçük kurulumlarda pratik görünse de controller’ı aynı zamanda build sunucusu olarak kullanmak sürdürülebilir değildir. Maven build’leri, testler ve Docker işlemleri controller kaynaklarını tüketir.

### Bütün Pipeline’ı Tek Bir Kabuk Betiğine Gömme

Aşağıdaki örnekte Jenkins görünürde pipeline kullanmakta, gerçekte ise tüm mantığı tek bir betiğe devretmektedir:

```groovy
pipeline {
    agent any
    stages {
        stage('CI') {
            steps {
                sh './ci.sh'
            }
        }
    }
}
```

Bu yaklaşımın sorunu, Jenkins’in stage düzeyindeki görünürlüğünü ortadan kaldırmasıdır. Hata nerede oluştuğu yeterince ayrışmaz, ölçüm ve raporlama kalitesi düşer.

### Kimlik Bilgilerini Kod İçine Yazmak

Aşağıdaki örnek güvenlik açısından açık bir hatadır:

```groovy
sh 'git clone https://username:password@github.com/example/user-service.git'
```

Kimlik bilgileri Jenkins Credentials Store içinde tutulmalı ve pipeline’a güvenli bağlamda enjekte edilmelidir.

### Testleri Devre Dışı Bırakıp Build Başarısını Yeterli Saymak

Sık karşılaşılan bir başka sorun, hız kazanmak amacıyla testleri sistematik biçimde devre dışı bırakmaktır:

```groovy
sh 'mvn -B clean package -DskipTests'
```

Bu komut tek başına kalıcı çözüm değildir. `Build` aşamasında test atlamak kabul edilebilir; fakat pipeline içinde ayrı bir `Test` aşaması bulunmuyorsa CI süreci amacını yitirir.

### Kirli Workspace ile Çalışmak

Eski `target/` çıktıları veya geçmiş branch kalıntıları, build’in yanlış biçimde başarılı görünmesine neden olabilir. Temiz çalışma alanı politikası uygulanmadığında CI güvenilirliği düşer.

### Aşırı Uzun ve Tek Parça Pipeline

Bir pipeline içine derleme, test, statik analiz, paketleme, dağıtım ve operasyonel bakım komutlarını denetimsiz biçimde yığmak bakım maliyetini artırır. CI ve CD sınırları açık tanımlanmalıdır.

## 3.15 CI ve Docker İlişkisi

Sürekli entegrasyon süreci çoğu zaman kapsayıcılaştırma adımının ön koşuludur. Docker, çalıştırma ortamını paketlemek için kullanılır; CI ise paketlenecek yazılımın doğruluğunu sınar. Bu iki katman birbirinin yerine geçmez.

`user-service` için tipik akış şöyledir:

1. Jenkins kaynak kodu çeker.
2. Maven ile projeyi derler.
3. Testleri çalıştırır.
4. Başarılı ise `jar` dosyasını üretir.
5. Sonraki aşamada bu `jar`, Docker image içine yerleştirilir.

Basit bir Dockerfile örneği aşağıdaki gibidir:

```dockerfile
FROM eclipse-temurin:17-jre
WORKDIR /app
COPY target/user-service-0.0.1-SNAPSHOT.jar app.jar
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

Bu dosyanın anlamlı olabilmesi için `target/user-service-0.0.1-SNAPSHOT.jar` artefaktının güvenilir biçimde üretilmiş olması gerekir. Dolayısıyla Docker aşaması, CI tarafından doğrulanmış build çıktısına dayanmalıdır. Aksi durumda kapsayıcılaştırılmış olmak, yazılımın kaliteli olduğu anlamına gelmez.

Bir sonraki aşamada Jenkins pipeline’ına image oluşturma adımı eklenebilir. Ancak bu adımın CI sürecine dahil edilmesi, temel build ve test disiplini kurulmadan yapılırsa yalnızca hatalı sistemlerin daha hızlı paketlenmesine yol açar.

## 3.16 Baştan Sona Gerçek Bir CI Senaryosu

Bu bölümde `user-service` için uçtan uca bir sürekli entegrasyon akışı ele alınmaktadır. Senaryoda geliştirici, kullanıcı oluşturma sırasında e-posta doğrulaması eklemektedir.

### 1. Geliştirme

Geliştirici `feature/email-validation` dalında aşağıdaki değişikliği yapar:

```java
public record CreateUserRequest(
        @NotBlank String firstName,
        @NotBlank String lastName,
        @Email(message = "Gecerli bir e-posta adresi girilmelidir")
        @NotBlank String email
) {
}
```

Aynı değişikliğe karşılık test de eklenir:

```java
@WebMvcTest(UserController.class)
class UserControllerValidationTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private UserService userService;

    @Test
    void shouldRejectInvalidEmail() throws Exception {
        String requestBody = """
                {
                  "firstName": "Ada",
                  "lastName": "Lovelace",
                  "email": "invalid-email"
                }
                """;

        mockMvc.perform(post("/api/users")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(requestBody))
                .andExpect(status().isBadRequest());
    }
}
```

### 2. Push

Geliştirici commit’i GitHub’a gönderir. Örnek commit mesajı:

```text
feat: validate e-mail field in user creation request
```

### 3. GitHub Olayı

GitHub, push olayını Jenkins webhook adresine iletir. Bu olay branch bilgisini ve commit kimliğini içerir.

### 4. Jenkins Job Eşleştirmesi

Jenkins controller, ilgili repoya bağlı multibranch pipeline job’ını bulur. `feature/email-validation` dalı için uygun alt job tetiklenir.

### 5. Queue ve Agent Seçimi

Build queue’ya alınır. `maven-jdk17` etiketli agent müsaitse iş bu düğüme atanır.

### 6. Workspace Hazırlığı

Agent üzerinde temiz bir workspace oluşturulur. Önceki build kalıntıları silinir.

### 7. Checkout

Repo workspace içine çekilir ve ilgili commit checkout edilir.

### 8. Derleme

Maven derlemesi başlatılır:

```bash
mvn -B clean package -DskipTests
```

Bu aşamada Java kaynak kodu derlenir ve uygulama paketlenir.

### 9. Test

Birim testleri çalıştırılır:

```bash
mvn -B test
```

Yeni eklenen doğrulama testi de bu aşamada yürütülür. Eğer geliştirici `@Email` anotasyonunu eklememiş olsaydı test başarısız olurdu. Böylece hata, kod birleşmeden önce görünür hâle gelirdi.

### 10. Sonuçların Kaydı

Jenkins test raporlarını toplar, build log’unu saklar ve sonucu kullanıcı arayüzünde yayınlar. Başarısızlık durumunda ilgili commit hızlıca saptanabilir.

### 11. Ana Dala Birleşme

Pull request kabul edilip `main` dalına birleştiğinde aynı süreç yeniden çalışır. `Archive` aşaması bu kez devreye girer ve `jar` artefaktı saklanır.

### 12. Sonraki Teslimat Aşaması

Başarılı `main` build’i, Docker image üretimi veya dağıtım hattının sonraki aşamaları için güvenilir girdi oluşturur.

Bu senaryo, Jenkins’in yalnızca build otomasyonu değil, ekipte ortak kalite eşiği oluşturma işlevi gördüğünü açık biçimde ortaya koyar. Asıl kazanım, hatanın insan hafızasına değil, sistematik geri bildirim mekanizmasına emanet edilmesidir.

## 3.17 Değerlendirme

Sürekli entegrasyon, yazılım geliştirme sürecinde teknik borcun görünmez biçimde birikmesini engelleyen temel disiplinlerden biridir. Jenkins ise bu disiplinin somutlaştırıldığı, genişletilebilir ve kurumsal kullanıma uygun bir yürütme platformudur. Değerini yalnızca otomasyon sağlamasından değil, entegrasyon sürecini gözlenebilir ve tekrarlanabilir kılmasından alır.

`user-service` projesi üzerinden incelenen akış, birkaç temel ilkeyi açık biçimde ortaya koymaktadır:

- Manuel build süreçleri ekip ölçeğinde güvenilir değildir.
- Sürekli entegrasyonun merkezi konusu otomasyon kadar erken geri bildirimdir.
- Jenkins mimarisi, controller, agent, executor, queue ve workspace gibi ayrı sorumluluklara dayanır.
- Pipeline yaklaşımı build sürecini kodla birlikte sürümlenebilir hâle getirir.
- GitHub entegrasyonu ve webhook mekanizması, CI sürecini olay güdümlü bir yapıya dönüştürür.
- Build izolasyonu ve dağıtık agent kullanımı, güvenilirlik ve ölçeklenebilirlik açısından belirleyicidir.
- Docker gibi sonraki teslimat aşamalarının sağlıklı işlemesi, önce güçlü bir CI temelinin kurulmasına bağlıdır.

Bu çerçevede Jenkins, bir komut yürütme aracı değil, yazılım geliştirme sürecinin entegrasyon omurgasıdır. Başarılı bir kullanım için Jenkins’in arayüzüne değil, arkasındaki çalışma modeline hâkim olmak gerekir. Bu bilgi düzeyi olmadan kurulan CI sistemleri kısa sürede kırılgan, opak ve bakım yükü yüksek yapılara dönüşür. Bu bilgi düzeyi sağlandığında ise Jenkins, ekip içi koordinasyonu güçlendiren ve yazılım kalitesini sistematik biçimde yükselten bir altyapı görevi üstlenir.
