# Bölüm 10 — Kubernetes Ortamında Production Pratikleri: ConfigMap, Secret, Storage, Probe'lar ve CI/CD Entegrasyonu

## 1. Production Ortamında Yeni Gereksinimler

Bölüm 9'da user-service uygulamasını basit bir şekilde Kubernetes'e dağıttık. Uygulamada sabit bir image vardı, ortam değişkenleri uygulamaya gömülüydü.

Gerçek production ortamına geçerken, yeni gereksinimler ortaya çıkıyor.

### Aynı Image, Farklı Ortam

user-service aynı Docker image'ını, geliştirme, test, staging ve production ortamlarında çalıştırmak istiyoruz. Ancak her ortamda farklı ayarlar gerekir:

- **Database host**: Geliştirme ortamında localhost, production'da managed database service
- **Log seviyesi**: Geliştirmede DEBUG, production'da INFO veya ERROR
- **API endpoints**: Test ortamında mock API'lar, production'da gerçek hizmetler
- **Cache ayarları**: Her ortamda farklı Redis instance'ı

Aynı image kullanarak bu farklılıkları nasıl yönetebiliriz? Kod içine environment'lara özel şeyler gömemez.

### Yapılandırma Ayrımı

Docker image, uygulamanın kodu ve bağımlılıklarını içerir. Yapılandırma (configuration) bu image dışında tutulmalı. Docker image'ı build ettikten sonra, dışarıdan yapılandırma gelmeli.

Bu, "12 Factor App" prensiplerinden biridir: konfigürasyonu ortamdan okuyun.

### Güvenlik

user-service, PostgreSQL veritabanına bağlanması gerekiyor. Bağlantı için kullanıcı adı ve parola gerekir. Bunu image'ın içine gömemez - bu ciddi güvenlik açığıdır.

Paraola ve API key'ler gibi hassas bilgiler, ayrı bir şekilde yönetilmeli. Kubernetes, bunu Secret mekanizması ile sağlıyor.

### Kalıcılık

PostgreSQL, verileri disk üzerinde saklıyor. Eğer PostgreSQL pod'u yeni bir node'a taşınırsa, verileri de ile gitmesi gerekir. Aksi takdirde, tüm veriler kaybolur.

Kubernetes, bunu PersistentVolume mekanizmu ile çözdü.

### Sağlık Kontrolü

Deployment pod'u başlatıyor. Ancak pod başladı = uygulama hazır mı? Belki uygulama still başlıyoruz, veya database bağlantısı başarısız oldu.

Kubernetes'e "uygulamanın gerçekten hazır olmasını kontrol et" demek gerekir.

---

## 2. ConfigMap Kavramı

### Neden Gereklidir

ConfigMap, Kubernetes'de konfigürasyon verilerini tutmak için kullanılan bir kaynaktır.

Örneğin, user-service'in çeşitli ayarlarını ConfigMap'te sakladığımız:

- Database host
- Servis port'u
- Log seviyesi
- Feature flags
- Timeout değerleri

Bu veriler, pod çalışırken environment variable olarak pod'a enjekte edilir.

### Environment'dan Farkı

Geleneksel ortamlarda, konfigürasyonu shell environment'ına yazarız:

```bash
export DB_HOST=localhost
export DB_PORT=5432
export LOG_LEVEL=INFO
```

Sonra uygulama başlatırız ve uygulama bu değişkenleri okur.

Kubernetes'te de benzer mantık var, ama Version kontrol altında yapılır. ConfigMap'i Git'te saklayabilir, kimin ne değiştiğini görebilir, geri dönebilirsiniz.

ConfigMap şu açıdan farklı:

1. **Deklaratif**: YAML ile tanımlanır
2. **Merkezi yönetim**: Tüm orchestration sistemi üzerinden yönetilir
3. **Versiyon kontrolü**: Git'te tutulabilir
4. **Dinamik güncelleme**: Pod yeniden başlatıldığında yeni değerler okunabilir

### Ne Tür Veriler Burada Tutulur

ConfigMap'te, **non-sensitive** veriler tutulur:

- Veritabanı host name'i (IP olmamak üzere)
- Port numaraları
- Feature toggle'ları
- Log seviyesi
- Cache timeout'ları
- URL'ler

Parola, API key, token gibi **sensitive** veriler ConfigMap'te tutulmaz. Onlar Secret'a gider.

---

## 3. User-Service için ConfigMap Örneği

user-service, PostgreSQL veritabanına bağlanıyor. Bağlantı ayarlarını ConfigMap'te tutelim:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: user-service-config
  namespace: production
data:
  # Spring profili
  SPRING_PROFILES_ACTIVE: "prod"
  
  # Database ayarları (host ve port)
  DB_HOST: "postgres"
  DB_PORT: "5432"
  DB_NAME: "user_service_db"
  
  # Server ayarları
  SERVER_PORT: "8080"
  SERVER_SERVLET_CONTEXT_PATH: "/api"
  
  # Logging
  LOGGING_LEVEL_ROOT: "INFO"
  LOGGING_LEVEL_COM_EXAMPLE_USERSERVICE: "DEBUG"
  
  # Performance
  SPRING_JPA_HIBERNATE_JDBC_BATCH_SIZE: "10"
  CONNECTION_POOL_SIZE: "20"
  
  # Features
  FEATURE_AUDIT_ENABLED: "true"
  FEATURE_CACHE_ENABLED: "true"
  CACHE_TTL_SECONDS: "3600"
```

Bu ConfigMap oluşturulduktan sonra:

```bash
kubectl create configmap user-service-config --from-file=... -n production
```

veya kubectl apply ile:

```bash
kubectl apply -f configmap.yaml
```

### ConfigMap İçeriğini Doğrulama

ConfigMap oluşturduktan sonra:

```bash
kubectl get configmap -n production
NAME                     DATA   AGE
user-service-config      14     2m
```

İçeriğini görmek:

```bash
kubectl describe configmap user-service-config -n production
Name:         user-service-config
Namespace:    production
Labels:       <none>
Annotations:  <none>

Data
====
DB_HOST:
----
postgres
DB_NAME:
----
user_service_db
DB_PORT:
----
5432
...
```

---

## 4. Secret Kavramı

### Neden ConfigMap'ten Ayrı Düşünülmelidir

ConfigMap'te, non-sensitive veriler tutulur. Ancak hassas bilgiler (parola, API key, müşteri sırrı) tutulmamalı.

Neden? Çünkü ConfigMap'terin içeriği şifrelenmez. Cluster yöneticileri, `kubectl describe` ile içeriği görebilir.

```bash
kubectl describe configmap my-config
Data:
====
password: mySecretPassword  # Şifrelenmemiş, açık metin
api_key: xyz123abc         # Şifrelenmemiş
```

Secret ise, verileri base64 ile şifreler (gerçek şifreleme değil, ama durumu teknik açıdan iyileştirir). Ayrıca, etcd'ye deposu sırasında TLS ile şifrelenebilir.

### Parola, Token, API Key Örnekleri

Secret tipik olarak şunları içerir:

- Database kullanıcı adı ve parolası
- API key'leri
- OAuth token'ları
- TLS sertifikası
- Private key'ler
- SSH key'ler

### Base64 Gerçeği ve Sınırlılığı

Secret içindeki veriler base64 ile kodlanır, ancak bu şifreleme değildir. Base64, sadece veriyi başka bir formata dönüştürür.

```bash
echo "mypassword" | base64
bXlwYXNzd29yZAo=
```

Bu, geri decode edilebilir:

```bash
echo "bXlwYXNzd29yZAo=" | base64 -d
mypassword
```

Base64, güvenlik sağlamaz. Ama etcd'ye kaydedilen verilerde şifreleme yapılıyorsa (TLS, encryption at rest), bu sayede veriler geri plandaki depolamada korunur.

Daha iyi güvenlik istiyorsanız, HashiCorp Vault gibi harici bir secret yöneticisi kullanılmalı.

---

## 5. User-Service için Secret Örneği

user-service, PostgreSQL'e parolayla bağlanıyor. Parolayı Secret'ta tutelim:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: user-service-secret
  namespace: production
type: Opaque
data:
  # Base64 ile kodlanmış
  DB_USER: dXNlcl9zZXJ2aWNl        # user_service
  DB_PASSWORD: c3RyMG5nUGFzNXcwcmQ=  # str0ngPas5w0rd
  API_KEY: YWJjZGVmZ2hpamtsbW5vcA==  # abcdefghijklmnop
  ENCRYPTION_KEY: dGhpcyBpcyBhIHNlY3JldA==  # this is a secret
```

**Önemli not:** Buradaki base64'ler örnek amaçlıdır. Gerçek production'da, Secret'ları versiyonlama sistemine koymayın. Bunları dışarıdan bir secret manager'dan alın (CI/CD pipeline tarafından).

Secret oluşturmak için, base64 encoding:

```bash
echo -n "user_service" | base64
dXNlcl9zZXJ2aWNl

echo -n "str0ngPas5w0rd" | base64
c3RyMG5nUGFzNXcwcmQ=
```

veya kubectl ile doğrudan:

```bash
kubectl create secret generic user-service-secret \
  --from-literal=DB_USER=user_service \
  --from-literal=DB_PASSWORD=str0ngPas5w0rd \
  -n production
```

---

## 6. ConfigMap ve Secret'ın Pod'a Aktarılması

ConfigMap ve Secret oluşturduk. Şimdi, bunları pod'a nasıl aktarırız?

Üç yöntem var: **envFrom**, **env**, ve **volume mount**.

### Yöntem 1: envFrom (Toplu Aktarım)

ConfigMap'teki tüm veriler, pod'un environment variable'larına dönüştürülür:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
    spec:
      containers:
        - name: user-service
          image: registry.example.com/user-service:1.0.0
          
          # ConfigMap'teki tüm veriler env variable olarak eklenir
          envFrom:
            - configMapRef:
                name: user-service-config
            - secretRef:
                name: user-service-secret
          
          ports:
            - containerPort: 8080
```

Pod başlatıldığında:

```bash
kubectl exec -it user-service-abc123 -- env | grep DB
DB_HOST=postgres
DB_PORT=5432
DB_NAME=user_service_db
DB_USER=user_service
DB_PASSWORD=str0ngPas5w0rd
```

### Yöntem 2: env (Seçmeli Aktarım)

Belirli veriler aktarılır:

```yaml
containers:
  - name: user-service
    image: registry.example.com/user-service:1.0.0
    
    env:
      # ConfigMap'ten
      - name: DB_HOST
        valueFrom:
          configMapKeyRef:
            name: user-service-config
            key: DB_HOST
      
      - name: DB_PORT
        valueFrom:
          configMapKeyRef:
            name: user-service-config
            key: DB_PORT
      
      # Secret'tan
      - name: DB_PASSWORD
        valueFrom:
          secretKeyRef:
            name: user-service-secret
            key: DB_PASSWORD
      
      # Doğrudan değer
      - name: ENV_NAME
        value: "production"
    
    ports:
      - containerPort: 8080
```

Bu yaklaşım, seçmeli ve kontrollü. Hangi değişkeni hangi kaynaktan aldığını açıkça görüyoruz.

### Yöntem 3: Volume Mount (Dosya Olarak)

ConfigMap/Secret'ın verileri, pod içinde dosya olarak mount edilebilir:

```yaml
containers:
  - name: user-service
    image: registry.example.com/user-service:1.0.0
    
    volumeMounts:
      - name: config-volume
        mountPath: /etc/config
      - name: secret-volume
        mountPath: /etc/secrets

volumes:
  - name: config-volume
    configMap:
      name: user-service-config
  - name: secret-volume
    secret:
      secretName: user-service-secret
```

Pod içinde:

```bash
cd /etc/config
ls
DB_HOST  DB_PORT  DB_NAME  ...

cat DB_HOST
postgres

cd /etc/secrets
ls
DB_USER  DB_PASSWORD  ...
```

Bu yöntem, konfigürasyon dosyalarını (JSON, YAML) temsil eden uygulamalar için faydalı.

### Hangi Yöntemini Seçmeli?

- **envFrom**: Tüm ConfigMap'i kullanıyorsanız, basit ve hızlı
- **env**: Belirli değişkenler istiyorsanız, seçici
- **Volume Mount**: Konfigürasyon dosyaları (properties, conf) istiyorsanız

---

## 7. Stateless ve Stateful Ayrımı

### User-Service Stateless

user-service, bir REST API'dır. Request'i alıyor, işliyor, response döndürüyor. Ardından bu request'i hatırlamıyor.

Bir request'i pod-1'e göndersek, sonraki request'i pod-2'ye göndersek, sorun yok. Her pod bağımsız.

Bu, **stateless** yapı. Stateless uygulamalar, kolayca ölçeklenebilir ve high-available yapılabilir.

### PostgreSQL Stateful

PostgreSQL, veri tabanıdır. Veri diskte saklanır. Pod silinirse, veri de silinir.

StatefulSet ile Pods:

1. Her pod'un kalıcı bir kimliği (pod-0, pod-1, pod-2)
2. Her pod'un ayrı depolama alanı
3. Pod silinirse yeniden başlatıldığında, aynı depolama alanına bağlanır

PostgreSQL gibi stateful servisleri, StatefulSet ile çalıştırırız. Bölüm 9'da gördüğümüz Deployment, stateless uygulamalar içindir.

### Bu Farkın Deployment Tasarımına Etkisi

**Stateless (user-service)**:

```yaml
kind: Deployment
replicas: 3
```

Herhangi bir pod silinebilir, ölçekleme kolay.

**Stateful (PostgreSQL)**:

```yaml
kind: StatefulSet
serviceName: postgres
replicas: 1  # Genellikle 1
```

Tek bir instance (veya uydu'lı yüksek kullanılabilirlik). Pod silinirse veri aynı kalır.

---

## 8. PersistentVolume ve PersistentVolumeClaim

### Neden Gerekir

Kubernetes pod'lar geçici. Pod silinirse, içindeki veriler de silinir.

PostgreSQL gibi bir veritabanı, verileri kalıcı depolama alanında saklıyor. Pod yeniden başlansa bile, verileri orada kalmalı.

Kubernetes, bunu PersistentVolume (PV) ve PersistentVolumeClaim (PVC) ile sağlıyor.

### Storage Soyutlaması

PV, fiziksel depolamayı temsil eder. NFS, iSCSI, cloud block storage (AWS EBS, Azure Disk) olabilir.

PVC, uygulamanın depolama talebini temsil eder. "10GB storage istiyorum" der, Kubernetes uygun bir PV bulur ve bağlar.

Uygulamacı, "bu NFS'i bağla, o cloud disk'i bağla" demek zorunda değil. PVC ile talep eden, Kubernetes otomatik olarak batır.

### PVC Bağlama Mantığı

Storage class tanısı:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
  replication-type: regional-pd
```

PVC oluşturma:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast
  resources:
    requests:
      storage: 20Gi
```

Pod'da kullanma:

```yaml
volumes:
  - name: postgres-storage
    persistentVolumeClaim:
      claimName: postgres-pvc
```

---

## 9. Probe Kavramı

### Liveness Probe

Kubernetes'e "bu pod'un container'ı canlı mı?" sorusunun cevabı liveness probe ile verilir.

Eğer probe başarısız olursa, Kubernetes container'ı yeniden başlatır.

Örnek:

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3
```

Kubernetes:
1. 30 saniye bekler (container başlıyorsa)
2. Her 10 saniyede `GET /health` çağrısı yapar
3. Eğer 3 kez başarısız olursa, container yeniden başlatır

### Readiness Probe

Kubernetes'e "bu pod talep almaya hazır mı?" sorusunun cevabı readiness probe ile verilir.

Eğer probe başarısız olursa, pod Service'ten çıkarılır. Traffic gelmez.

Örnek:

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 2
```

Kubernetes:
1. 5 saniye bekler
2. Her 5 saniyede `GET /ready` çağrısı yapır
3. Eğer 2 kez başarısız olursa, pod kaldırılır

### Startup Probe

Uygulamanın başlanması uzun sürüyor (migration, cache loading). Başlama süresi bilinmeyebilir.

Startup probe, başlama tamamlanmasını kontrol eder. Tamamlandıktan sonra liveness ve readiness probe'lar çalışır.

```yaml
startupProbe:
  httpGet:
    path: /health
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
```

Kubernetes:
- Her 10 saniyede kontrol eder
- 30 başarısız denemeye izin verir (300 saniye = 5 dakika)
- Başarılı olunca liveness/readiness probe'a geçer

### Neden Production için Kritik Olduğu

Probe'lar olmadan:

1. Container crash'leyebilir, baştan başlatılmaz
2. Pod hata veriyorsa de talep alıyor, kullanıcılar mutsuz
3. Deployment update sırasında hazır olmayan pod'lar talep alabilir

Probe'lar, sistem güvenilirliğini sağlar.

---

## 10. User-Service için Probe Tanımları

user-service, Spring Boot uygulaması. Spring Actuator health endpoint'leri sağlıyor.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
    spec:
      containers:
        - name: user-service
          image: registry.example.com/user-service:1.0.0
          ports:
            - containerPort: 8080
          
          # Startup probe: başlama süresi kontrol
          startupProbe:
            httpGet:
              path: /actuator/health
              port: 8080
              scheme: HTTP
            initialDelaySeconds: 5
            periodSeconds: 10
            failureThreshold: 30  # 5 dakika
          
          # Liveness probe: container canlı mı?
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
              scheme: HTTP
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          
          # Readiness probe: talep almaya hazır mı?
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 2
          
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
          
          envFrom:
            - configMapRef:
                name: user-service-config
            - secretRef:
                name: user-service-secret
```

### Probe Yanıtları

Spring Actuator sağlayan endpoints:

- `/actuator/health`: Genel sağlık (UP/DOWN)
- `/actuator/health/liveness`: Container canlılığı
- `/actuator/health/readiness`: Database bağlantısı gibi bağımlılıklar

Başarılı durum: HTTP 200 OK
Başarısız durum: HTTP 503 Service Unavailable

---

## 11. Resource Requests ve Limits

### Neden Gerekir

Kubernetes, cluster'daki kaynakları (CPU, bellek) yönetir. Scheduler, pod'ları nereye yerleştireceğini kaynağa göre belirler.

Eğer requests ve limits koymassanız:

1. Scheduler, pod'u nereye yerleştireceğini bilemez
2. Bir pod aşırı kaynak tüketebilir, diğer pod'ları öldürebilir ("noisy neighbor" problemi)
3. Cluster'ın kaynak planlaması başarısız olabilir

### Scheduler Üzerindeki Etkisi

user-service pod'u için:

```yaml
resources:
  requests:
    cpu: 100m      # 0.1 CPU (1m = 1 milicore)
    memory: 256Mi  # 256 megabyte
  limits:
    cpu: 500m      # 0.5 CPU
    memory: 512Mi  # 512 megabyte
```

Scheduler:
- İstediği kaynak: CPU 100m, bellek 256Mi
- Sınırı: CPU 500m, bellek 512Mi

Node'da 1000m CPU varsa, 10 tane user-service pod'u yerleştirebilir (10 * 100m = 1000m).

Eğer requests koymazsanız, Kubernetes "0 kaynak istiyoruz" gibi davranabilir ve disk disk pod'u aşırı yükleyebilir.

### Aşırı Tüketim ve Noisy Neighbor Problemi

Pod-1 bellekte 512Mi limitlenmiş, ama bir bellek leak'i var. Pod, belleği aşırı kullanmaya çalışıyor. Kubernetes onu durdurur (OOMKilled).

Limit olmadan, Pod-1 tüm belleği kullanır, diğer pod'lar star kalır.

Limits, pod'u kontrol altında tutar.

---

## 12. Namespace ile Ortam Ayrımı

### Staging ve Production

user-service'i test etmek istiyoruz. Bunu production'a zarar vermeden yapabiliriz. Namespace kullanarak:

```bash
kubectl create namespace staging
kubectl create namespace production
```

Ayrı ConfigMap, Secret, Deployment her namespace'de:

```bash
kubectl apply -f user-service-config.yaml -n staging
kubectl apply -f user-service-config.yaml -n production
```

Production'daki pod'lar staging'i etkilemez, tersi de.

### Mantıksal Ayrımın Önemi

RBAC (Role Based Access Control) ile:

- Geliştirme ekibi: `staging` namespace'e erişim
- DevOps ekibi: `production` namespace'e erişim

Konfigürasyonlar farklı:

Production:

```yaml
data:
  SPRING_PROFILES_ACTIVE: "prod"
  REPLICA_COUNT: "5"
  LOG_LEVEL: "ERROR"
```

Staging:

```yaml
data:
  SPRING_PROFILES_ACTIVE: "staging"
  REPLICA_COUNT: "2"
  LOG_LEVEL: "DEBUG"
```

---

## 13. Jenkins → Kubernetes Deployment Akışı

Bölüm 7'de, Jenkins pipeline kurmıştuk. Şimdi, bunu Kubernetes ile entegre edeceğiz.

### Image Build

Jenkins build kısmı (değişmemiş):

```groovy
stage('Build') {
  steps {
    sh 'mvn clean package'
    sh 'docker build -t registry.example.com/user-service:${BUILD_NUMBER} .'
    sh 'docker push registry.example.com/user-service:${BUILD_NUMBER}'
  }
}
```

Image'ın tag'ı build numarasına göre: 1.0.0, 1.0.1, 1.0.2, ...

### Registry Push

Image, container registry'e yüklendi.

### Deployment Image Update

Kubernetes'te, image'ı güncelleme:

```groovy
stage('Deploy to Kubernetes') {
  steps {
    sh '''
      kubectl set image deployment/user-service \
        user-service=registry.example.com/user-service:${BUILD_NUMBER} \
        -n production
    '''
  }
}
```

kubectl, deployment'ı günceller. Kubernetes otomatik olarak rolling update başlatır.

### Rollout Status Takibi

Pipeline, deployment'ın hazır olmasını bekler:

```groovy
stage('Verify Deployment') {
  steps {
    sh '''
      kubectl rollout status deployment/user-service -n production --timeout=5m
    '''
  }
}
```

Eğer 5 dakika içinde deployment hazır olmazsa, pipeline hata verir.

### Başarısız Durum İçin Rollback

Pipeline hata alırsa, önceki sürüme geri dön:

```groovy
post {
  failure {
    sh '''
      echo "Deployment failed, rolling back to previous version"
      kubectl rollout undo deployment/user-service -n production
    '''
  }
}
```

---

## 14. Kubectl set image ve/veya Manifest Tabanlı Güncelleme

İki yaklaşım var.

### Yaklaşım 1: kubectl set image

Hızlı ve command-line tabanlı:

```bash
kubectl set image deployment/user-service \
  user-service=registry.example.com/user-service:1.1.0 \
  -n production
```

Avantaj: Hızlı, single komut
Dezavantaj: Versiyon kontrol yok, tekrar edilebilir değil

### Yaklaşım 2: Manifest Güncelleme

Git'te manifest vardır:

```yaml
# deployment.yaml
image: registry.example.com/user-service:1.0.0
```

Pipeline tarafından güncellenir:

```groovy
stage('Update Manifest') {
  steps {
    sh '''
      sed -i "s|registry.example.com/user-service:.*|registry.example.com/user-service:${BUILD_NUMBER}|g" deployment.yaml
      git add deployment.yaml
      git commit -m "Update user-service image to ${BUILD_NUMBER}"
      git push
    '''
  }
}

stage('Deploy') {
  steps {
    sh 'kubectl apply -f deployment.yaml -n production'
  }
}
```

Avantaj: Git'te geçmiş, denetlenebilir, repeatable
Dezavantaj: Daha karmaşık, Git'te veri saklama ihtiyacı

### Hangi Yaklaşım?

- **kubectl set image**: Test, geliştirme, quick fix
- **Manifest güncelleme**: Production, audit trail gerekirse

---

## 15. Production Rollout Stratejilerine Giriş

### Rolling Update (Default)

Eski pod'lar silinip yeni pod'lar başlatılır. Sırası ile.

Avantaj: Simple, zero-downtime
Dezavantaj: Eski ve yeni versiyon aynı anda çalışıyor (compatibility gerekli)

### Blue-Green Deployment

İki ortam: mavi (eski versiyon), yeşil (yeni versiyon).

İkisi de çalışıyor. Testler yapılıyor. Uygunsa, traffic yeşile switch edilir.

Avantaj: Instant switchover, instant rollback
Dezavantaj: Çift kaynaklar gerekli

### Canary Deployment

Yeni versiyon, small percentage ile başlar (örneğin %5).

- %95 traffic eski versiyona
- %5 traffic yeni versiyona

Eğer yeni versiyon iyi çalışıyorsa, gradually artırılır. Problem varsa, rollback edilir.

Avantaj: Riski minimize eder
Dezavantaj: Konfigürasyon karmaşık (istio gibi service mesh gerekir)

Bu bölümde yalnızca giriş seviyesi. Production'da bu stratejiler, service mesh (Istio, Linkerd) ile birlikte kullanılır.

---

## 16. Anti-Pattern: Yapmaması Gerekenler

### Secret'ı Plain Text Yazmak

**Yanlış:**

```yaml
data:
  DB_PASSWORD: str0ngPas5w0rd
```

Burada, parola plain text'dir. Git'te görülebilir.

**Doğru:**

Base64 ile encode et:

```bash
echo -n "str0ngPas5w0rd" | base64 # c3RyMG5nUGFzNXcwcmQ=
```

Manifest'te:

```yaml
data:
  DB_PASSWORD: c3RyMG5nUGFzNXcwcmQ=
```

Daha iyi: Vault gibi harici secret manager kullan.

### Config'i Image İçine Gömmek

**Yanlış:**

```dockerfile
FROM openjdk:11
RUN echo "DB_HOST=prod-db.example.com" >> /app.env
COPY app.jar /app.jar
```

Config, image'a gömülü. Farklı ortanta için yeni image build gerekir.

**Doğru:**

Config, Kubernetes ConfigMap/Secret'tan gelmeli.

### Probe Kullanmamak

**Yanlış:**

Deployment, probe olmadan:

```yaml
containers:
  - name: user-service
    image: ...
```

Pod crash'leyebilir, baştan başlatılmaz. Hazır olmayan pod talep alabilir.

**Doğru:**

Probe'ları tanımla.

### Resource Limit Koymamak

**Yanlış:**

```yaml
containers:
  - name: user-service
    image: ...
```

Pod aşırı kaynak tüketebilir. Diğer pod'lar starve kalır.

**Doğru:**

Request ve limit tanımla.

### Stateful Servisleri Rastgele Deployment Gibi Ele Almak

**Yanlış:**

PostgreSQL'i Deployment ile çalıştırma. Pod silinirse veri silinir.

**Doğru:**

StatefulSet kullanma. PVC ile depolama bağlama.

---

## 17. Baştan Sona Örnek Production Senaryosu

Kompleks bir senaryo: user-service + PostgreSQL.

### Adım 1: Storage Class ve PVC

```yaml
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: postgres-storage
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: postgres-storage
  resources:
    requests:
      storage: 20Gi
```

### Adım 2: ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: user-service-config
  namespace: production
data:
  SPRING_PROFILES_ACTIVE: prod
  DB_HOST: postgres
  DB_PORT: "5432"
  DB_NAME: user_service_db
  LOGGING_LEVEL_ROOT: INFO
```

### Adım 3: Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: user-service-secret
  namespace: production
type: Opaque
data:
  DB_USER: dXNlcl9zZXJ2aWNl
  DB_PASSWORD: c3RyMG5nUGFzNXcwcmQ=
  POSTGRES_DB: dXNlcl9zZXJ2aWNlX2Ri
  POSTGRES_PASSWORD: c3RyMG5nUGFzNXcwcmQ=
```

### Adım 4: PostgreSQL StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:13
          ports:
            - containerPort: 5432
          
          env:
            - name: POSTGRES_DB
              valueFrom:
                secretKeyRef:
                  name: user-service-secret
                  key: POSTGRES_DB
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: user-service-secret
                  key: POSTGRES_PASSWORD
          
          volumeMounts:
            - name: postgres-storage
              mountPath: /var/lib/postgresql/data
  
  volumeClaimTemplates:
    - metadata:
        name: postgres-storage
      spec:
        accessModes: [ "ReadWriteOnce" ]
        storageClassName: postgres-storage
        resources:
          requests:
            storage: 20Gi
```

### Adım 5: PostgreSQL Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: production
  labels:
    app: postgres
spec:
  clusterIP: None  # Headless service for StatefulSet
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
```

### Adım 6: User-Service Deployment (Production Ready)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  namespace: production
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
        version: "1.0.0"
    spec:
      containers:
        - name: user-service
          image: registry.example.com/user-service:1.0.0
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8080
          
          envFrom:
            - configMapRef:
                name: user-service-config
            - secretRef:
                name: user-service-secret
          
          startupProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
            failureThreshold: 30
          
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 2
          
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
```

### Adım 7: User-Service Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: user-service
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: user-service
  ports:
    - port: 8080
      targetPort: 8080
```

### Adım 8: Jenkins Pipeline (Production Deploy)

```groovy
pipeline {
  agent any
  
  parameters {
    choice(
      name: 'TARGET_ENVIRONMENT',
      choices: ['staging', 'production'],
      description: 'Deployment ortamı'
    )
  }
  
  stages {
    stage('Build') {
      steps {
        sh '''
          mvn clean package
          docker build -t registry.example.com/user-service:${BUILD_NUMBER} .
          docker push registry.example.com/user-service:${BUILD_NUMBER}
        '''
      }
    }
    
    stage('Deploy to Kubernetes') {
      steps {
        sh '''
          kubectl set image deployment/user-service \
            user-service=registry.example.com/user-service:${BUILD_NUMBER} \
            -n ${TARGET_ENVIRONMENT}
        '''
      }
    }
    
    stage('Verify Deployment') {
      steps {
        sh '''
          kubectl rollout status deployment/user-service \
            -n ${TARGET_ENVIRONMENT} --timeout=5m
        '''
      }
    }
    
    stage('Smoke Tests') {
      steps {
        sh '''
          kubectl run smoke-test --rm -it \
            --image=curlimages/curl:latest \
            -n ${TARGET_ENVIRONMENT} \
            -- curl http://user-service:8080/health
        '''
      }
    }
  }
  
  post {
    failure {
      sh '''
        echo "Deployment failed, rolling back..."
        kubectl rollout undo deployment/user-service \
          -n ${TARGET_ENVIRONMENT}
      '''
    }
  }
}
```

### Adım 9: Deployment İzleme

```bash
# Deployment durumu
kubectl get deployment user-service -n production

# Pod'ları izle
kubectl get pods -n production --watch

# Rollout durumu
kubectl rollout status deployment/user-service -n production

# Detaylı bilgi
kubectl describe deployment user-service -n production

# Log'ları kontrol et
kubectl logs deployment/user-service -n production --tail=100

# ConfigMap'ı kontrol et
kubectl get configmap user-service-config -n production

# Secret'ı kontrol et (içeriği görmez)
kubectl get secret user-service-secret -n production

# PostgreSQL pod'u
kubectl get statefulset postgres -n production
```

### Adım 10: Başarısız Durumda Geri Dönüş

```bash
# Önceki sürüme dön
kubectl rollout undo deployment/user-service -n production

# Belirli revizyon'a dön
kubectl rollout undo deployment/user-service \
  --to-revision=2 -n production

# Rollout geçmişi
kubectl rollout history deployment/user-service -n production
```



