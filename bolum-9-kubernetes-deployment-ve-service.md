# Bölüm 9 — Kubernetes Üzerinde Uygulama Çalıştırma: Deployment, Replica Yönetimi, Service ve Ölçekleme

## 1. Neden Tek Pod Yeterli Değildir

Bölüm 8'de, user-service uygulamasını tek bir pod olarak Kubernetes'te çalıştırmayı öğrendik. Manifest şöyle görünüyordu:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: user-service-pod
  labels:
    app: user-service
spec:
  containers:
    - name: user-service
      image: registry.example.com/user-service:1.0.0
      ports:
        - containerPort: 8080
```

Bu yapı, basit kurulumlarda çalışabilir. Ancak üretim ortamında birkaç ciddi sorunla karşılaşırız.

### Kırılganlık Sorunu

Tek bir pod çalışıyor. Pod'ta bir hata olursa ne olur? Pod kapanır. Kubernetes, pod'u yeniden başlatmaya çalışır, ancak başarısız kapanışlar tekrarlanırsa, pod'u başlatmayı bırakır.

Eğer pod kapanırsa, sistem tamamen kullanılamaz hâle gelir. Bunu önlemek için, birden fazla pod çalıştırmalıyız. Eğer bir pod kapanırsa, diğerleri çalışmaya devam eder.

### Güncelleme Zorluğu

user-service'in yeni sürümüne (1.1.0) güncellememiz gerekiyor. Tek pod varsa ne yapacağız?

1. Pod'u sileriz
2. Yeni image ile eğer pod'u çalıştırırız

Bu süre boyunca, sistem kesintiye uğrar. Kullanıcılar hizmet alamaz.

Doğru yaklaşım, yeni pod'u başlattıktan sonra, eski pod'u kapatmak. Bu şekilde, hizmet kesintisiz devam eder. Ancak bunu manual olarak yapmak hataya açıktır.

### Çoğaltma Gereksinimi

Talep yükseldiğinde, tek pod yetmez. Birden fazla pod çalıştırmalıyız. Eğer bunu manual olarak yaparsak:

```bash
kubectl apply -f user-service-pod-1.yaml
kubectl apply -f user-service-pod-2.yaml
kubectl apply -f user-service-pod-3.yaml
```

Bu, ölçeklenebilir bir yaklaşım değildir. Eğer 10 pod'a çıkarmamız gerekirse, 10 tane dosya oluşturmülü mü?

Bu sorunları çözmek için Deployment harika bir araçtır.

---

## 2. Deployment Kavramı

### Deployment Nedir?

Deployment, Kubernetes'te pod'ları yönetmek için kullanılan üst düzey bir yapıdır. Deployment, bir veya birden fazla pod'un oluşturulması ve yönetilmesini otomatikleştirir.

Deployment şunları yönetir:

1. Kaç tane pod çalışması gerektiği (replicas)
2. Pod yaşam döngüsü üzerindeki değişiklikler (güncelleme, rollback)
3. Pod başarısız olduğunda yeniden başlatılması

Pod gibi, Deployment de YAML ile tanımlanır. Ancak Deployment, pod tanımını içinde tutar.

### Neden Doğrudan Pod Yerine Deployment Kullanılır?

Pod'u doğrudan Kubernetes'te çalıştırmak, el ile bağcık kullanmak gibi: işi yapabilir, fakat çok hata yapmaya engel değil.

Deployment kullanmak, bir oto-pilot sistemi kullanmak gibi: belirttiğiniz durumu korumaya çalışan, otomatik olarak gözlem yapan bir sistem.

Deployment'ın temel avantajları:

1. **Çoğaltma**: Deployment, "3 tane pod çalıştır" şeklinde tanımlayabilirsiniz. Kubernetes otomatik olarak 3 pod oluşturur.

2. **Self-healing**: İçindeki pod kapanırsa, Deployment otomatik olarak yeni pod başlatır.

3. **Scaling**: Deployment'ı "2 tane pod çalıştır" olarak değiştirirseniz, Kubernetes otomatik olarak uyum sağlar.

4. **Rolling Update**: Yeni image'a geçerken, hizmet kesintisiz yapılabilir.

---

## 3. ReplicaSet Mantığı

Deployment'ı anlamak için, ReplicaSet'i anlamalı.

### ReplicaSet Nedir?

ReplicaSet, belirli sayıda pod'un çalışmasını garanti eder. Eğer siz "3 tane user-service pod'u çalışması gerekiyor" derseniz, ReplicaSet bunu sağlar.

Pod kapanırsa, ReplicaSet yeniden başlatır. Pod sayısı 3'ün üzerinde çıkarsa, ReplicaSet fazla pod'ları kapatır.

### Desired Replica ve Actual Replica

ReplicaSet, iki sayıyı izler:

**Desired (İstenilen)**: "kaç tane pod olmalı"
**Actual (Mevcut)**: "kaç tane pod seçiliyorum"

Örneğin:

```
Desired: 3
Actual: 2
```

Bu durumda, ReplicaSet fark etmiş ve yeni pod başlatır. Kısa bir süre sonra:

```
Desired: 3
Actual: 3
```

İstenilen duruma getirildi.

### Deployment ile ReplicaSet İlişkisi

Deployment, ReplicaSet tarafından kontrol edilir. Deployment, "bu kadar replica istiyorum, bu kadar pod istiyorum" der. ReplicaSet, bunu uygular.

Genellikle bu detayı bilmeniz gerekmez. Deployment ile çalışırsınız, ReplicaSet arka planda çalışır.

---

## 4. User-Service için Deployment Manifesti

Şimdi user-service uygulamasını Deployment ile çalıştıralım. Manifest şöyledir:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  labels:
    app: user-service
    version: "1.0.0"
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
              protocol: TCP
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
```

### Manifest Alanlarını Açıklama

**apiVersion: apps/v1**
- Deployment, `apps` API grubundadır

**kind: Deployment**
- Nesi oluşturduğumuz

**metadata**
- Deployment'ın adı ve etiketleri

**spec**

- **replicas: 3**: 3 tane pod çalışsın
- **strategy**: Güncelleme stratejisi (RollingUpdate: sırası ile güncelle)
- **selector**: Bu Deployment'ı hangi pod'lar oluşturacağını tanımlar
- **template**: Pod şablonu. Bu şablon ile pod'lar oluşturulur

**template.spec**
- Konteyner tanımı
- image: Docker image
- ports: Konteyner'in açığa çıkardığı portlar
- resources: CPU ve bellek talep/sınırı
- livenessProbe: Pod canlı mı? (5 saniyede bir kontrol)
- readinessProbe: Pod talebe yanıt vermeye hazır mı? (10 saniyede bir kontrol)

---

## 5. Label ve Selector Mantığı

Labels, Kubernetes'te çok önemlidir. Label'lar, kaynakları (pod, deployment, service) etiketlemek için kullanılır.

### Label Nedir?

Label, bir key-value çiftidir:

```yaml
labels:
  app: user-service
  version: "1.0.0"
  environment: production
```

Bu label'lar, pod'u tanımlar ve onu başka kaynaklardan ayırır.

### Selector Nedir?

Selector, label'lara göre kaynakları seçmek için kullanılır.

Deployment'ın selector'ü:

```yaml
selector:
  matchLabels:
    app: user-service
```

Bu, "app: user-service" label'ı olan tüm pod'ları seç" anlamına gelir.

### Deployment ile Pod İlişkisi

Deployment, selector kullanarak hangi pod'ları yönetecağını belirler. Pod template'de de aynı label'lar tanımlanır:

```yaml
template:
  metadata:
    labels:
      app: user-service
```

Deployment oluşturulduğunda:

1. Pod template'den 3 tane pod oluşturur
2. Her pod'a "app: user-service" label'ı verilir
3. Selector, bu label'ları bulur ve Deployment tarafından yönetilmesini sağlar

Eğer bir pod'un label'ını değiştirirseniz, Deployment artık onu yönetmez ve yeni pod oluşturur.

### Service ile Pod İlişkisi

Service de selector kullanarak hangi pod'lara trafik göndereceğini belirler. Bunu sonra göreceğiz.

---

## 6. Self-Healing Davranışı

Deployment'ın önemli bir özelliği, pod kapanırsa otomatik olarak yeniden başlatmadır.

### Pod Silinirse Ne Olur?

Deployment ile 3 pod çalışıyor:

```bash
kubectl get pods
NAME                            READY   STATUS    RESTARTS   AGE
user-service-abc123            1/1     Running   0          5m
user-service-def456            1/1     Running   0          5m
user-service-ghi789            1/1     Running   0          5m
```

Bir pod'u manuel olarak silersiniz:

```bash
kubectl delete pod user-service-abc123
```

Pod silinir. Ancak kısa bir süre sonra:

```bash
kubectl get pods
NAME                            READY   STATUS    RESTARTS   AGE
user-service-def456            1/1     Running   0          6m
user-service-ghi789            1/1     Running   0          6m
user-service-jkl012            1/1     Running   0          10s
```

Yeni bir pod (user-service-jkl012) başlatılmış. Neden? Çünkü ReplicaSet, "3 pod olmalı, şu anda 2 var" dedi ve yeni pod başlattı.

### Sistem Neden Yeniden Oluşturur?

Deployment'ın hedefi, "her zaman 3 tane pod olsun" demektir. Eğer pod kapanırsa, bu hedef karşılanmaz. Deployment, bunu düzeltmeye çalışır.

Yukarıda, pod manuel olarak siliniydi. Gerçek ortamda, pod hata verip kapanabilir. Deployment'ın bu otomatik iyileştirme özelliği, sistemin dayanıklılığını arttırır.

---

## 7. Scaling

Scaling, pod sayısını artırmak veya azaltmak anlamına gelir.

### Yatay Ölçekleme Mantığı

user-service'e gelen talep arttığını varsayalım. Şu anda 3 pod var, ama sistem yeterli hızda yanıt vermiyor.

Yatay ölçekleme, daha fazla pod başlatmaktır. Örneğin 5 pod'a çıkarmak. Bu, beş tane server yerine beş tane uygulama instance'ı anlamına gelir.

Dikey ölçekleme (vertical scaling) ise, mevcut pod'ların kaynaklarını artırmak: CPU ve belleği artırmak. Ancak bu daha az verimli ve sınırlıdır.

### Replica Sayısının Artırılması

Manifest'i güncelleyebilirsiniz:

```yaml
spec:
  replicas: 5
```

Daha hızlı bir yol, kubectl komutunu kullanmak:

```bash
kubectl scale deployment user-service --replicas=5
```

Kubernetes hemen 5 pod çalıştırmaya başlar:

```bash
kubectl get pods
NAME                            READY   STATUS    RESTARTS   AGE
user-service-abc123            1/1     Running   0          20m
user-service-def456            1/1     Running   0          20m
user-service-ghi789            1/1     Running   0          20m
user-service-jkl012            1/1     Running   0          15m
user-service-mno345            1/1     Running   0          10s
user-service-pqr678            1/1     Running   0          10s
```

5 pod artık çalışıyor.

### Gerçek Gereksinim Bağlamı

Talep düştüğünde ne olur? 2 pod'a düşürebiliriz:

```bash
kubectl scale deployment user-service --replicas=2
```

Üretim ortamlarında, talep tahminlenebilirse, bu işlem manuel yapılabilir. Ancak talep öngörülemezse, otomatik ölçekleme (Horizontal Pod Autoscaler, HPA) kullanılır. Bu, ileriki bölümlerde anlatılacak.

---

## 8. Rolling Update

Yeni image sürümüne (1.1.0) geçmenin gerekliliği var.

### Image Sürümü Değiştiğinde Ne Olur?

Deployment manifest'ini güncelleyin:

```yaml
spec:
  containers:
    - name: user-service
      image: registry.example.com/user-service:1.1.0
```

Ardından apply edin:

```bash
kubectl apply -f deployment.yaml
```

Kubernetes ne yapıyor olduğunu gözlemleyin:

```bash
kubectl get pods --watch
NAME                            READY   STATUS    RESTARTS   AGE
user-service-abc123            1/1     Running   0          25m
user-service-def456            1/1     Running   0          25m
user-service-ghi789            1/1     Running   0          25m
user-service-xyz999            0/1     Pending   0          5s
user-service-xyz999            0/1     ContainerCreating   0          5s
user-service-xyz999            1/1     Running             0          10s
user-service-abc123            1/1     Terminating         0          25m
user-service-abc123            0/1     Terminating         0          26m
user-service-abc123            0/1     Terminated          0          26m
```

Işlemler:

1. Yeni image ile bir pod oluşturulur (user-service-xyz999)
2. Yeni pod, ready durumuna gelir, sistem hazır hale gelene kadar bekler
3. Eski pod'lardan biri (user-service-abc123) silinir
4. Bu işlem, tüm eski pod'lar silinene kadar devam eder

Bu sırada, sistem hizmet vermeye devam eder. Eğer 3 pod varsa, güncelleme sırasında her zaman en az 1-2 pod çalışmakta devam eder.

### Sıfır Kesintiye Yakın Güncelleme Mantığı

Manifest'deki strategy alanı bunu kontrol eder:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

- **maxSurge: 1**: Aynı anda en fazla 1 tane ek pod çalışabilir (toplam 4)
- **maxUnavailable: 0**: Hizmet vermeyen pod 0 olmalı (yani daima 3 pod hizmet vermeye hazır)

Bu ayarlar, güncelleme sırasında hizmet kesintisini minimum düzeyde tutar.

---

## 9. Rollout History ve Rollback

Kubernetes, her Deployment güncellememesinin geçmişini tutanı.

### Rollout Geçmişi

Tüm deployment geçmişini görmek:

```bash
kubectl rollout history deployment user-service
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
3         <none>
```

Belirli bir revizyon'un detaylarını görmek:

```bash
kubectl rollout history deployment user-service --revision=2
user-service-5c6b8f6c4d
Containers:
 user-service:
  Image: registry.example.com/user-service:1.0.0
```

### Neden Deployment Geçmişi Önemlidir?

Yeni sürüm hata içeriyorsa veya performans sorunları yaratıyorsa ne yapılır?

Rollback, önceki sürüme geri dönmek anlamında. Örneğin, sürüm 1.1.0 hata içeriyorsa, 1.0.0'a dönebilirsiniz:

```bash
kubectl rollout undo deployment user-service
deployment.apps/user-service rolled back

kubectl rollout status deployment user-service
deployment "user-service" successfully rolled out
```

Kubernetes, yeni pod'lar başlatır ve eski image'ı çalıştırır. Sistem, güncelleme öncesi durumuna gelir.

Belirli bir revizyon'a geri dönmek:

```bash
kubectl rollout undo deployment user-service --to-revision=1
```

Bu güvenlik ağı, üretim ortamlarında çok değerlidir. Hata yapıldığında, sistem hızlı bir şekilde düzeltilir.

---

## 10. Service Kavramı

Deployment ile user-service'i çalıştırıyoruz. 3 pod düğümde çalışıyor, her birinin farklı bir IP'si var.

### Pod IP'lerinin Neden Kalıcı Olmadığı

Pod'un IP'si, pod yaşam boyu değişmez. Ancak pod silinirse, IP de gider. Yeni pod, yeni IP alır.

Deployment güncelleme sırasında, eski pod'lar silinir, yeni pod'lar başlatılır. Her yeni pod, farklı bir IP alır.

Eğer bir uygulama, user-service'i "192.168.1.100 IP'deki pod'u çağır" şekline göre yazılmışsa, pod güncellendikten sonra, IP değişir ve bağlantı kopacak.

### Sabit Erişim Katmanı Gereksinimi

Biz istiyoruz ki, başvuruları bir "sabit" adrese göndersinler. Bu adres, arkaseında hangi pod'ların çalışıyor olduğunu açığa çıkarmasın.

Bu işlevi sağlayan, Service'dir. Service şöyle çalışır:

1. Stable bir IP adresi sağlar
2. Service'e gelen trafiğe, arkadaki pod'lara yönlendirir
3. Pod'lar değişse bile, Service IP'si aynı kalır

---

## 11. Service Türleri

Service'in üç ana türü vardır. Biz, üretim ortamında çoğunlukla bunlardan birini seçeriz.

### ClusterIP

Service, yalnızca Kubernetes cluster'ı içinde erişilebilir. Dışarıdan direkt erişim yok.

**Kullanım alanı**: Cluster içi iletişim. Örneğin, order-service, user-service'i çağırması gerekirse.

Manifest örneği:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: user-service
spec:
  type: ClusterIP
  selector:
    app: user-service
  ports:
    - port: 8080
      targetPort: 8080
      protocol: TCP
```

Service'e erişim:

```bash
curl http://user-service:8080
```

Cluster içinde bu DNS adı çalışır.

### NodePort

Service, cluster düğümlerinin (node'ların) belli bir port'unda erişilebilir.

**Kullanım alanı**: Cluster dışından erişim. Test amaçlı veya küçük sistemler.

Manifest örneği:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: user-service
spec:
  type: NodePort
  selector:
    app: user-service
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 30080
      protocol: TCP
```

Service'e erişim:

```bash
curl http://node-ip:30080
```

Burada node-ip, Kubernetes düğümünün IP'sidir (örneğin 192.168.1.50).

### LoadBalancer

Service, bulut sağlayıcısının load balancer'ını kullanır. Dış IP adresi sağlanır.

**Kullanım alanı**: Üretim ortamlarında cluster dışından erişim. Cloud ortamları (AWS, Azure, GCP) tarafından desteklenir.

Manifest örneği:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: user-service
spec:
  type: LoadBalancer
  selector:
    app: user-service
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
```

Servis yapılandırıldığında, cloud sağlayıcısı bir dış IP adresi atayacak:

```bash
kubectl get service user-service
NAME           TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)        AGE
user-service   LoadBalancer   10.0.0.100    203.0.113.1      80:31234/TCP   5m
```

Service'e erişim:

```bash
curl http://203.0.113.1
```

---

## 12. User-Service için Service Tanımı

Deployment'imizi Service ile ortaya alalım. user-service içinde erişilebilir bir Service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: user-service
  labels:
    app: user-service
spec:
  type: ClusterIP
  selector:
    app: user-service
  ports:
    - name: http
      port: 8080
      targetPort: 8080
      protocol: TCP
  sessionAffinity: None
```

### Manifest Alanlarını Açıklama

**selector: matchLabels: app: user-service**
- Service, "app: user-service" label'ı olan pod'ları seçer

**ports**
- **port**: Service'in dinlediği port
- **targetPort**: Pod'un port'u (konteyner'in dinlediği port)
- **protocol**: TCP

**type: ClusterIP**
- Service, yalnızca cluster içinde erişilebilir

Service oluşturulduktan sonra:

```bash
kubectl apply -f service.yaml
kubectl get service user-service
NAME           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
user-service   ClusterIP   10.0.0.100      <none>        8080/TCP   2m
```

Kubernetes, Service'e `10.0.0.100` cluster IP'sini atadı. Cluster içindeki pod'lar:"

```bash
curl http://user-service:8080
```

ile erişebilirler.

---

## 13. Çoklu Replica ile Trafik Dağıtımı

Deployment'ımızda 3 pod var. Service, gelen trafiği bu 3 pod'a nasıl dağıtıyor?

### Service Yük Dağıtımı

Service, kube-proxy aracılığıyla çalışır. Kube-proxy, Kubernetes'in ağ denetleyicisidir.

Service'e gelen istek:

```
Client -> Service (10.0.0.100:8080) -> Kube-proxy -> Pod1, Pod2 veya Pod3
```

Kube-proxy, istek alımında hangi pod'a yönlendireceğini seçer. Varsayılan olarak "round-robin" kullanılır: sırası ile pod'lar seçilir.

Örnek:
1. İstek1 -> Pod1
2. İstek2 -> Pod2
3. İstek3 -> Pod3
4. İstek4 -> Pod1 (sıra başa döner)

Bu şekilde, yük pod'lar arasında dengeli şekilde dağıtılır.

### Gerçek Hayat Senaryosu

user-service'e saniyede 1000 istek geliyor. 3 pod varsa, her pod saniyede yaklaşık 333 istek alır. Eğer scaling yapıp 5 pod'a çıkarırsanız, her pod saniyede 200 istek alır.

Daha fazla pod = her pod daha az iş = daha hızlı yanıt = daha iyi sistem performansı.

---

## 14. Port-Forwarding ve Test

Development sırasında, siz cluster dışında çalışıyor olabilirsiniz. Service'e erişmek istiyor ama cluster dışında erişim yok.

### Port-Forwarding Nedir?

Port-forwarding, lokal makinenizdeki bir port'u, Kubernetes Service veya Pod'a yönlendirir.

Örneğin, Service'i lokal 8080 port'unda erişmek:

```bash
kubectl port-forward service/user-service 8080:8080
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080
```

Şimdi, lokal makinede:

```bash
curl http://localhost:8080
```

Kubernetes Service'e erişebilirsiniz. Trafik, kubectl aracılığıyla yönlendirilir.

### Pod'a Direkt Bağlanmak

Service yerine, doğrudan pod'a bağlanmak da mümkünde (test amaçlı):

```bash
kubectl port-forward pod/user-service-abc123 8080:8080
```

Bu, yalnızca o spesifik pod'a erişim sağlar.

### Interaktif Shell Açmak

Pod'un içine girip komut çalıştırmak:

```bash
kubectl exec -it user-service-abc123 -- /bin/bash
```

Pod'un içindeki shell'e erişebilirsiniz. Örneğin:

```bash
curl http://localhost:8080/health
```

ile uygulama sağlığını kontrol edebilirsiniz.

---

## 15. Anti-Pattern: Yapmaması Gerekenler

### Pod'a Sabit IP Gibi Davranmak

**Yanlış:**

Uygulama, spesifik pod'un IP'sini hardcode eder. Örneğin:

```java
String userServiceUrl = "http://192.168.1.100:8080";
```

Bu, pod güncellendikten sonra kopacak.

**Doğru:**

Service kullan. Örneğin:

```java
String userServiceUrl = "http://user-service:8080";
```

Service, arkada pod'ları yönetir.

### Label Kullanmamak

**Yanlış:**

Tüm pod'lar aynı namespace'de çalışıyor, birbirleriyle çakışıyor.

**Doğru:**

Clear label'lar kullanmak. Farklı uygulamalar:

```yaml
labels:
  app: user-service
  team: backend
  environment: production
```

Bu, sorgulamayı ve yönetimi güzelleştirir.

### Service Olmadan Erişim Tasarlamak

**Yanlış:**

Pod IP'sine direkt bağlantı yapmak.

**Doğru:**

Her zaman Service kullanmak. Service, stabilitesini ve ölçeklenebilirliğini sağlar.

### Deployment Yerine Sürekli Pod Çalıştırmak

**Yanlış:**

```bash
kubectl run pod1 ...
kubectl run pod2 ...
kubectl run pod3 ...
```

**Doğru:**

Deployment kullanmak. Deployment, replika yönetimini otomatikleştirir.

---

## 16. Baştan Sona Örnek Senaryo

Şimdi, bir senaryoyu adım adım izleyelim.

### Adım 1: Deployment Oluşturma

user-service deployment manifest'i oluşturun (user-service-deployment.yaml):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  labels:
    app: user-service
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
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
```

Deployment oluşturun:

```bash
kubectl apply -f user-service-deployment.yaml
deployment.apps/user-service created
```

Pod'ları kontrol edin:

```bash
kubectl get pods
NAME                            READY   STATUS    RESTARTS   AGE
user-service-5c6b8f6c4d-abc12   1/1     Running   0          2m
user-service-5c6b8f6c4d-def34   1/1     Running   0          2m
user-service-5c6b8f6c4d-ghi56   1/1     Running   0          2m
```

### Adım 2: Service Oluşturma

Service manifest'i (user-service-service.yaml):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: user-service
spec:
  type: ClusterIP
  selector:
    app: user-service
  ports:
    - port: 8080
      targetPort: 8080
```

Service oluşturun:

```bash
kubectl apply -f user-service-service.yaml
service/user-service created
```

Service detaylarını kontrol edin:

```bash
kubectl get service user-service
NAME           TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
user-service   ClusterIP   10.0.0.100    <none>        8080/TCP   1m
```

### Adım 3: Replica Artırma

Talep yükseldi, 5 pod çalıştırmalı:

```bash
kubectl scale deployment user-service --replicas=5
deployment.apps/user-service scaled
```

Pod'ları kontrol edin:

```bash
kubectl get pods
NAME                            READY   STATUS    RESTARTS   AGE
user-service-5c6b8f6c4d-abc12   1/1     Running   0          10m
user-service-5c6b8f6c4d-def34   1/1     Running   0          10m
user-service-5c6b8f6c4d-ghi56   1/1     Running   0          10m
user-service-5c6b8f6c4d-jkl78   1/1     Running   0          2m
user-service-5c6b8f6c4d-mno90   1/1     Running   0          2m
```

5 pod çalışıyor.

### Adım 4: Yeni Image'a Geçme

user-service 1.1.0 sürümü hazırlandı. Deployment güncelleyin:

```bash
kubectl set image deployment/user-service user-service=registry.example.com/user-service:1.1.0
deployment.apps/user-service image updated
```

Rolling update başlar:

```bash
kubectl rollout status deployment user-service
Waiting for deployment "user-service" to roll out...
Waiting for deployment "user-service" to roll out...
deployment "user-service" successfully rolled out
```

Tüm pod'lar yeni image'la çalışıyor:

```bash
kubectl get pods
NAME                            READY   STATUS    RESTARTS   AGE
user-service-7b9c8d2e1f-abc12   1/1     Running   0          5m
user-service-7b9c8d2e1f-def34   1/1     Running   0           5m
user-service-7b9c8d2e1f-ghi56   1/1     Running   0           4m
user-service-7b9c8d2e1f-jkl78   1/1     Running   0           3m
user-service-7b9c8d2e1f-mno90   1/1     Running   0           3m
```

(Pod adları değişti, çünkü ReplicaSet yeni revision'da yeni pod oluşturdu)

### Adım 5: Rollback

Yeni sürümde hata bulundu. Geri dönmek gerekiyor:

```bash
kubectl rollout undo deployment user-service
deployment.apps/user-service rolled back
```

Kubernetes, 1.0.0 image'ına geri döner:

```bash
kubectl rollout status deployment user-service
deployment "user-service" successfully rolled out
```

Pod'lar tekrar güncellenmiş:

```bash
kubectl get pods
NAME                            READY   STATUS    RESTARTS   AGE
user-service-5c6b8f6c4d-abc12   1/1     Running   0          3m
user-service-5c6b8f6c4d-def34   1/1     Running   0           3m
...
```

Sistem, 1.0.0 sürümüne dönmüştür.

---

