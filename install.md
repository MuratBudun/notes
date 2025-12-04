# QMex Forms - AKS Kurulum Kılavuzu

Bu dokümanda QMex Forms uygulamasının Azure Kubernetes Service (AKS) üzerinde kurulumu adım adım anlatılmaktadır.

## Ön Gereksinimler

- Azure Cloud Shell (Bash) erişimi
- AKS cluster mevcut ve çalışır durumda olmalı

## Değişkenler

Aşağıdaki değişkenleri ortamınıza göre düzenleyin ve Cloud Shell'de tanımlayın:

```bash
CLUSTER_NAME="qmex-forms-k8s-cluster"
RESOURCE_GROUP="qmex-forms-rg"
APPGW_NAME="qmex-forms-appgw"
APPGW_SUBNET_CIDR="10.225.0.0/24"  # Aşağıdaki APPGW_SUBNET_CIDR seçimindeki açıklamaya bakın
SSL_CERT_NAME="qmex-forms-ssl"
STORAGE_ACCOUNT_NAME="qmexformsstorage"
STORAGE_RESOURCE_GROUP="qmex-forms-rg"

# Container Image Ayarları
ACR_NAME="qmexcontainers"                                 # Azure Container Registry adı
TENANT_API_IMAGE_NAME="qmex-forms-tenant-management-api"  # Tenant API image adı
APP_IMAGE_NAME="qmex-forms-app"                           # App image adı
IMAGE_TAG="latest"                                        # Image tag 

# URL Ayarları
WEBSITE_URL="https://qmexforms.com"        # Ana website URL
APP_URL="https://app.qmexforms.com"        # Uygulama URL
APP_HOST="app.qmexforms.com"               # Ingress host (protokol olmadan)

# Secret Değerleri (Hassas Bilgiler - Değiştirin!)
DB_CONNECTION_STRING="Host=your-postgres-server.postgres.database.azure.com; Database=your_database; Username=your_username; Password=YOUR_DB_PASSWORD; MinPoolSize=2; MaxPoolSize=10;"
SENDGRID_API_KEY="SENDGRID_API_KEY"
OIDC_CLIENT_SECRET="OIDC_CLIENT_SECRET"

# Authentication Ayarları (Azure AD B2C)
AUTH_AUTHORITY=""
AUTH_CLIENT_ID="OIDC_CLIENT_ID"
```

### APPGW_SUBNET_CIDR Seçimi

Application Gateway için ayrılacak subnet CIDR değeri, AKS cluster'ınızın mevcut network yapılandırmasıyla **çakışmamalıdır**.

Mevcut AKS network bilgilerini kontrol edin:

```bash
az aks show --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP \
    --query "networkProfile.{podCidr:podCidr, serviceCidr:serviceCidr, networkPlugin:networkPlugin}" -o table
```

Örnek çıktı:
```
PodCidr        ServiceCidr    NetworkPlugin
-------------  -------------  -------------
10.244.0.0/16  10.0.0.0/16    azure
```

Aşağıdaki CIDR aralıkları genellikle güvenlidir (çakışma olmaz):
- `10.225.0.0/24` (varsayılan)
- `10.226.0.0/24`
- `192.168.100.0/24`

**Önemli:** Eğer AKS cluster'ınız bir VNet'e bağlıdıysa, o VNet'teki mevcut subnet'lerle de çakışmadığından emin olun.

---

## Adım 1: Application Gateway ve AGIC Kurulumu

Azure Application Gateway Ingress Controller (AGIC), Kubernetes Ingress kaynaklarını Azure Application Gateway ile entegre eder.

### 1.1 AKS Credentials Alın

```bash
az aks get-credentials --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP --overwrite-existing
```

### 1.2 AGIC Addon'unu Etkinleştirin

Bu komut yeni bir Application Gateway oluşturur ve AGIC addon'unu etkinleştirir:

```bash
az aks enable-addons \
    --name $CLUSTER_NAME \
    --resource-group $RESOURCE_GROUP \
    --addons ingress-appgw \
    --appgw-name $APPGW_NAME \
    --appgw-subnet-cidr $APPGW_SUBNET_CIDR
```

**Not:** Bu işlem 5-10 dakika sürebilir.

### 1.3 Application Gateway Durumunu Kontrol Edin

```bash
# MC Resource Group adını alın
MC_RG=$(az aks show --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP --query nodeResourceGroup -o tsv)

# Application Gateway durumunu kontrol edin
az network application-gateway show \
    --name $APPGW_NAME \
    --resource-group $MC_RG \
    --query "{name:name, provisioningState:provisioningState, operationalState:operationalState}" \
    -o table
```

### 1.4 Application Gateway Durdurulmuşsa Başlatın

AGIC addon'u etkinleştirildiğinde Application Gateway bazen "Stopped" durumunda olabilir:

```bash
# Durumu kontrol et
STATE=$(az network application-gateway show --name $APPGW_NAME --resource-group $MC_RG --query operationalState -o tsv)

if [ "$STATE" == "Stopped" ]; then
    echo "Application Gateway başlatılıyor..."
    az network application-gateway start --name $APPGW_NAME --resource-group $MC_RG
fi
```

### 1.5 AGIC Pod'unun Çalıştığını Doğrulayın

```bash
kubectl get pods -n kube-system -l app=ingress-appgw
```

Beklenen çıktı:
```
NAME                                        READY   STATUS    RESTARTS   AGE
ingress-appgw-deployment-xxxxxxxxx-xxxxx    1/1     Running   0          5m
```

### 1.6 Managed Identity Operator Rolünü Atayın

AGIC'in Key Vault SSL sertifikalarına erişebilmesi için bu rol ataması gereklidir:

```bash
# AGIC Identity Object ID'sini alın
AGIC_OBJECT_ID=$(az aks show \
    --name $CLUSTER_NAME \
    --resource-group $RESOURCE_GROUP \
    --query "addonProfiles.ingressApplicationGateway.identity.objectId" \
    -o tsv)

# Subscription ID'yi alın
SUBSCRIPTION_ID=$(az account show --query id -o tsv)

# Key Vault Secrets Provider Identity Scope'unu oluşturun
KV_IDENTITY_SCOPE="/subscriptions/$SUBSCRIPTION_ID/resourcegroups/$MC_RG/providers/Microsoft.ManagedIdentity/userAssignedIdentities/azurekeyvaultsecretsprovider-$CLUSTER_NAME"

# Rol ataması yapın
az role assignment create \
    --assignee-object-id $AGIC_OBJECT_ID \
    --assignee-principal-type ServicePrincipal \
    --role "Managed Identity Operator" \
    --scope $KV_IDENTITY_SCOPE
```

### 1.7 AGIC Pod'unu Yeniden Başlatın

Rol ataması yapıldıktan sonra AGIC pod'unun yeni izinleri alabilmesi için yeniden başlatılması gerekir:

```bash
kubectl rollout restart deployment ingress-appgw-deployment -n kube-system

# Pod'un Running durumuna geçmesini bekleyin
kubectl get pods -n kube-system -l app=ingress-appgw -w
```

### 1.8 Application Gateway Public IP Adresini Alın

```bash
APPGW_PUBLIC_IP=$(az network public-ip list \
    --resource-group $MC_RG \
    --query "[?contains(name,'appgw')].ipAddress" \
    -o tsv)

echo "Application Gateway IP: $APPGW_PUBLIC_IP"
```

**Not:** Bu IP adresini DNS kaydınızda kullanacaksınız.

---

## Doğrulama Kontrol Listesi

| Adım | Komut | Beklenen Sonuç |
|------|-------|----------------|
| AGIC Addon | `az aks show -n $CLUSTER_NAME -g $RESOURCE_GROUP --query "addonProfiles.ingressApplicationGateway.enabled"` | `true` |
| App Gateway Durumu | `az network application-gateway show -n $APPGW_NAME -g $MC_RG --query operationalState` | `Running` |
| AGIC Pod | `kubectl get pods -n kube-system -l app=ingress-appgw` | `Running` |
| Rol Ataması | `az role assignment list --scope $KV_IDENTITY_SCOPE --query "[].roleDefinitionName"` | `Managed Identity Operator` içermeli |

---

## Sorun Giderme

### AGIC Pod CrashLoopBackOff

Application Gateway'in durumunu kontrol edin:
```bash
az network application-gateway show --name $APPGW_NAME --resource-group $MC_RG --query operationalState -o tsv
```

"Stopped" ise başlatın:
```bash
az network application-gateway start --name $APPGW_NAME --resource-group $MC_RG
```

### LinkedAuthorizationFailed Hatası

AGIC pod loglarında bu hata görülüyorsa:

1. Managed Identity Operator rol atamasını yapın (Adım 1.6)
2. AGIC pod'unu yeniden başlatın (Adım 1.7)

```bash
# Logları kontrol edin
kubectl logs -n kube-system -l app=ingress-appgw --tail=50

# Pod'u yeniden başlatın
kubectl rollout restart deployment ingress-appgw-deployment -n kube-system
```

---

## Adım 2: SSL Sertifikası Yapılandırması

SSL sertifikası Azure Portal üzerinden manuel olarak eklenir.

### 2.1 Application Gateway'e SSL Sertifikası Ekleyin

1. Azure Portal'da Application Gateway'i açın (`qmex-forms-appgw-op`)
2. Sol menüden **Settings > Listeners** bölümüne gidin
3. **+ Add listener** veya mevcut listener'ı düzenleyin
4. **Certificate** bölümünde PFX dosyanızı yükleyin
5. Sertifika adını `qmex-forms-ssl` olarak belirleyin (Ingress annotation'larıyla eşleşmelidir)

### 2.2 Sertifika Yüklendikten Sonra Doğrulama

```bash
az network application-gateway ssl-cert list \
    --gateway-name $APPGW_NAME \
    --resource-group $MC_RG \
    --query "[].name" -o table
```

Beklenen çıktı:
```
Result
--------------
qmex-forms-ssl
```

---

## Adım 3: Storage Account ve File Share Yapılandırması

QMex Forms uygulaması Azure File Share kullanarak kalıcı depolama sağlar.

### 3.1 Storage Account Firewall Ayarları

AKS'in storage account'a erişebilmesi için firewall ayarlarını kontrol edin:

```bash
az storage account show \
    --name $STORAGE_ACCOUNT_NAME \
    --resource-group $STORAGE_RESOURCE_GROUP \
    --query "networkRuleSet.defaultAction" \
    -o tsv
```

Eğer `Deny` dönüyorsa, AKS erişimi için firewall'u açın:

```bash
az storage account update \
    --name $STORAGE_ACCOUNT_NAME \
    --resource-group $STORAGE_RESOURCE_GROUP \
    --default-action Allow
```

**Not:** Güvenlik için, üretim ortamında belirli IP'lere veya VNet'lere erişim kısıtlaması yapılması önerilir.

### 3.2 Storage Account Key'i Alın

```bash
STORAGE_KEY=$(az storage account keys list \
    --account-name $STORAGE_ACCOUNT_NAME \
    --resource-group $STORAGE_RESOURCE_GROUP \
    --query "[0].value" -o tsv)
```

Bu değer bir sonraki adımda secret oluştururken kullanılacaktır.

### 3.3 Gerekli File Share'lerin Kontrolü

Uygulama aşağıdaki file share'leri kullanır:
- `global` - Tüm tenant'lar için ortak dosyalar
- `tenant` - Tenant'a özel dosyalar
- `temp` - Geçici dosyalar
- `queue` - Kuyruk işlemleri için

Mevcut file share'leri kontrol edin:

```bash
az storage share list \
    --account-name $STORAGE_ACCOUNT_NAME \
    --account-key $STORAGE_KEY \
    --query "[].name" \
    -o table
```

### 3.4 Eksik File Share'leri Oluşturun

Eğer file share'ler mevcut değilse oluşturun:

```bash
# Global file share
az storage share create \
    --account-name $STORAGE_ACCOUNT_NAME \
    --account-key $STORAGE_KEY \
    --name global \
    --quota 100

# Tenant file share
az storage share create \
    --account-name $STORAGE_ACCOUNT_NAME \
    --account-key $STORAGE_KEY \
    --name tenant \
    --quota 100

# Temp file share
az storage share create \
    --account-name $STORAGE_ACCOUNT_NAME \
    --account-key $STORAGE_KEY \
    --name temp \
    --quota 50

# Queue file share
az storage share create \
    --account-name $STORAGE_ACCOUNT_NAME \
    --account-key $STORAGE_KEY \
    --name queue \
    --quota 10
```

---

## Doğrulama Kontrol Listesi (Storage)

| Adım | Komut | Beklenen Sonuç |
|------|-------|----------------|
| Firewall | `az storage account show --name $STORAGE_ACCOUNT_NAME -g $STORAGE_RESOURCE_GROUP --query "networkRuleSet.defaultAction"` | `Allow` |
| File Shares | `az storage share list --account-name $STORAGE_ACCOUNT_NAME --account-key $STORAGE_KEY --query "[].name" -o tsv` | global, tenant, temp, queue |

---

## Adım 4: Kubernetes Secret'larının Oluşturulması

**Önemli:** Secret'lar hassas bilgiler içerdiğinden YAML dosyasında saklanmamalıdır. Deployment öncesinde aşağıdaki komutlarla manuel olarak oluşturulmalıdır.

### 4.1 Namespace Oluşturun (Eğer yoksa)

```bash
kubectl create namespace qmex-forms-ns --dry-run=client -o yaml | kubectl apply -f -
```

### 4.2 Storage Secret Oluşturun

```bash
# Önceki adımda alınan değişkenleri kullanın
# Secret oluşturun
kubectl create secret generic qmex-forms-storage-secret \
    --namespace qmex-forms-ns \
    --from-literal=azurestorageaccountname="$STORAGE_ACCOUNT_NAME" \
    --from-literal=azurestorageaccountkey="$STORAGE_KEY"
```

### 4.3 Uygulama Config Secret Oluşturun

Başlangıçta tanımlanan değişkenleri kullanarak secret oluşturun:

```bash
kubectl create secret generic qmex-forms-app-config \
    --namespace qmex-forms-ns \
    --from-literal=DB_CONNECTION_STRING_PROD="$DB_CONNECTION_STRING" \
    --from-literal=DB_CONNECTION_STRING_TEST="" \
    --from-literal=SENDGRID_API_KEY_PROD="$SENDGRID_API_KEY" \
    --from-literal=SENDGRID_API_KEY_TEST="" \
    --from-literal=OIDC_CLIENT_SECRET="$OIDC_CLIENT_SECRET"
```

### 4.4 Secret'ların Doğrulanması

```bash
kubectl get secrets -n qmex-forms-ns
```

Beklenen çıktı:
```
NAME                         TYPE     DATA   AGE
qmex-forms-storage-secret    Opaque   2      1m
qmex-forms-app-config        Opaque   5      1m
```

### 4.5 Secret Güncelleme (Gerekirse)

Mevcut bir secret'ı güncellemek için önce silin, sonra yeniden oluşturun:

```bash
kubectl delete secret qmex-forms-app-config -n qmex-forms-ns
# Ardından 4.3 adımını tekrarlayın
```

---

## Doğrulama Kontrol Listesi (Secrets)

| Adım | Komut | Beklenen Sonuç |
|------|-------|----------------|
| Storage Secret | `kubectl get secret qmex-forms-storage-secret -n qmex-forms-ns` | Secret mevcut |
| App Config Secret | `kubectl get secret qmex-forms-app-config -n qmex-forms-ns` | Secret mevcut |

---

## Adım 5: Kubernetes Deployment

Tüm ön gereksinimler (AGIC, SSL, Storage, Secrets) tamamlandıktan sonra deployment yapılabilir.

### 5.1 Deployment YAML Dosyasını Uygulama

`deploy.yml` dosyasındaki placeholder'ları değişkenlerle değiştirip uygulayın:

```bash
# Tüm değişkenleri export edin (envsubst için gerekli)
export ACR_NAME TENANT_API_IMAGE_NAME APP_IMAGE_NAME IMAGE_TAG
export WEBSITE_URL APP_URL APP_HOST SSL_CERT_NAME
export AUTH_AUTHORITY AUTH_CLIENT_ID

# Değişkenleri yerleştirip kubectl'e pipe edin
envsubst < deploy.yml | kubectl apply -f -
```

**Not:** `envsubst` komutu aşağıdaki placeholder'ları ortam değişkenleriyle değiştirir:
- `${ACR_NAME}` - Container Registry adı
- `${TENANT_API_IMAGE_NAME}` - Tenant API image adı
- `${APP_IMAGE_NAME}` - App image adı
- `${IMAGE_TAG}` - Image tag
- `${WEBSITE_URL}` - Ana website URL
- `${APP_URL}` - Uygulama URL
- `${APP_HOST}` - Ingress host
- `${SSL_CERT_NAME}` - SSL sertifika adı
- `${AUTH_AUTHORITY}` - Azure AD B2C authority URL
- `${AUTH_CLIENT_ID}` - Azure AD B2C client ID

### 5.2 Pod Durumlarını Kontrol Edin

```bash
kubectl get pods -n qmex-forms-ns -w
```

Tüm pod'ların `Running` durumuna geçmesini bekleyin.

### 5.3 Ingress Durumunu Kontrol Edin

```bash
kubectl get ingress -n qmex-forms-ns
```

### 5.4 HTTPS Erişimini Test Edin

DNS kaydınız yapılandırıldıktan sonra:

```bash
curl -I https://app.qmexforms.com
```

---

## Doğrulama Kontrol Listesi (Final)

| Adım | Komut | Beklenen Sonuç |
|------|-------|----------------|
| Pods | `kubectl get pods -n qmex-forms-ns` | Tüm pod'lar Running |
| Ingress | `kubectl get ingress -n qmex-forms-ns` | ADDRESS atanmış |
| HTTPS | `curl -I https://app.qmexforms.com` | HTTP/2 200 |
