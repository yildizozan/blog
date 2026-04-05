---
title: "Azure DevOps Agents on Kubernetes"
description: "Azure DevOps pipeline agent'larını Kubernetes üzerinde container olarak çalıştırma: Dockerfile hazırlama, K8s'e deploy ve private NuGet config yönetimi."
pubDate: "Apr 05 2026"
heroImage: ""
---

Azure DevOps pipeline'larını çalıştıran agent'lar genellikle bir ya da birkaç sanal makineye kurulur. Bu yöntem başlangıçta işe yarasa da zamanla sorunlar çıkmaya başlar: agent sayısını artırmanız gerektiğinde yeni VM oluşturup elle yapılandırmanız gerekir, pipeline yoğunluğuna göre kapasite ayarlamak zordur, makine bakımları agent'ları etkiler. Kubernetes'in sağladığı scaling ve pod yönetimi bu sorunların büyük bölümünü ortadan kaldırır.

Bu yazıda, Azure DevOps agent'larını K8s üzerinde container olarak nasıl çalıştıracağımızı adım adım anlatacağım. HuaweiCloud CCE üzerinde kurduğumuzda karşılaştığımız sorunları ve çözümleri de paylaşacağım.

## Gereklilikler

Kullandığımız Azure DevOps Server versiyonu 2020 olduğundan, bu versiyonla uyumlu agent versiyonumuz **2.181.2**'dir. Agent versiyonları Azure DevOps Server versiyonuna göre değişir; kendi versiyonunuza uygun agent'ı `{AZP_URL}/_apis/distributedtask/packages/agent` endpoint'inden sorgulayabilirsiniz.

Mevcut Linux base image versiyonumuz Ubuntu focal 20.04'tür.

## Container Hazırlayalım

Microsoft'un Docker üzerinde agent çalıştırmak için hazırladığı [resmi rehber](https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/docker?view=azure-devops)den yararlanacağız. Rehberde Ubuntu 22.04 için verilmiş örneği baz alarak ilerleyeceğiz.

Proje oluşturup içerisine şu iki dosyayı oluşturalım:

### Dockerfile

`Dockerfile` agent'ın çalışacağı Ubuntu imajını hazırlıyor: gerekli paketleri kuruyor, Azure CLI'ı ekliyor ve agent'ı root olmayan bir kullanıcıyla çalıştıracak şekilde yapılandırıyor.

```dockerfile
FROM ubuntu:22.04
ENV TARGETARCH="linux-x64"
# Also can be "linux-arm", "linux-arm64".

RUN apt update && \
  apt upgrade -y && \
  apt install -y curl git jq libicu70

# Install Azure CLI
RUN curl -sL https://aka.ms/InstallAzureCLIDeb | bash

WORKDIR /azp/

COPY ./start.sh ./
RUN chmod +x ./start.sh

# Create agent user and set up home directory
RUN useradd -m -d /home/agent agent
RUN chown -R agent:agent /azp /home/agent

USER agent
# Another option is to run the agent as root.
# ENV AGENT_ALLOW_RUNASROOT="true"

ENTRYPOINT [ "./start.sh" ]
```

### start.sh

`start.sh` container başladığında Azure DevOps'a bağlanıyor, agent binary'sini indiriyor, yapılandırıyor ve çalıştırıyor. Container kapandığında ise agent'ı pool'dan temizliyor.

```bash
#!/bin/bash
set -e

if [ -z "${AZP_URL}" ]; then
  echo 1>&2 "error: missing AZP_URL environment variable"
  exit 1
fi

if [ -n "$AZP_CLIENTID" ]; then
  echo "Using service principal credentials to get token"
  az login --allow-no-subscriptions --service-principal --username "$AZP_CLIENTID" --password "$AZP_CLIENTSECRET" --tenant "$AZP_TENANTID"
  # adapted from https://learn.microsoft.com/en-us/azure/databricks/dev-tools/user-aad-token
  AZP_TOKEN=$(az account get-access-token --query accessToken --output tsv)
  echo "Token retrieved"
fi

if [ -z "${AZP_TOKEN_FILE}" ]; then
  if [ -z "${AZP_TOKEN}" ]; then
    echo 1>&2 "error: missing AZP_TOKEN environment variable"
    exit 1
  fi

  AZP_TOKEN_FILE="/azp/.token"
  echo -n "${AZP_TOKEN}" > "${AZP_TOKEN_FILE}"
fi

unset AZP_CLIENTSECRET
unset AZP_TOKEN

if [ -n "${AZP_WORK}" ]; then
  mkdir -p "${AZP_WORK}"
fi

cleanup() {
  trap "" EXIT

  if [ -e ./config.sh ]; then
    print_header "Cleanup. Removing Azure Pipelines agent..."

    # If the agent has some running jobs, the configuration removal process will fail.
    # So, give it some time to finish the job.
    while true; do
      ./config.sh remove --unattended --auth "PAT" --token $(cat "${AZP_TOKEN_FILE}") && break

      echo "Retrying in 30 seconds..."
      sleep 30
    done
  fi
}

print_header() {
  lightcyan="\033[1;36m"
  nocolor="\033[0m"
  echo -e "\n${lightcyan}$1${nocolor}\n"
}

# Let the agent ignore the token env variables
export VSO_AGENT_IGNORE="AZP_TOKEN,AZP_TOKEN_FILE"

print_header "1. Determining matching Azure Pipelines agent..."

AZP_AGENT_PACKAGES=$(curl -LsS \
    -u user:$(cat "${AZP_TOKEN_FILE}") \
    -H "Accept:application/json" \
    "${AZP_URL}/_apis/distributedtask/packages/agent?platform=${TARGETARCH}&top=1")

AZP_AGENT_PACKAGE_LATEST_URL=$(echo "${AZP_AGENT_PACKAGES}" | jq -r ".value[0].downloadUrl")

if [ -z "${AZP_AGENT_PACKAGE_LATEST_URL}" -o "${AZP_AGENT_PACKAGE_LATEST_URL}" == "null" ]; then
  echo 1>&2 "error: could not determine a matching Azure Pipelines agent"
  echo 1>&2 "check that account "${AZP_URL}" is correct and the token is valid for that account"
  exit 1
fi

print_header "2. Downloading and extracting Azure Pipelines agent..."

curl -LsS "${AZP_AGENT_PACKAGE_LATEST_URL}" | tar -xz & wait $!

source ./env.sh

trap "cleanup; exit 0" EXIT
trap "cleanup; exit 130" INT
trap "cleanup; exit 143" TERM

print_header "3. Configuring Azure Pipelines agent..."

# Despite it saying "PAT", it can be the token through the service principal
./config.sh --unattended \
  --agent "${AZP_AGENT_NAME:-$(hostname)}" \
  --url "${AZP_URL}" \
  --auth "PAT" \
  --token $(cat "${AZP_TOKEN_FILE}") \
  --pool "${AZP_POOL:-Default}" \
  --work "${AZP_WORK:-_work}" \
  --replace \
  --acceptTeeEula & wait $!

print_header "4. Running Azure Pipelines agent..."

chmod +x ./run.sh

# To be aware of TERM and INT signals call ./run.sh
# Running it with the --once flag at the end will shut down the agent after the build is executed
./run.sh "$@" & wait $!
```

## Kubernetes'e Deploy

Container imajını build edip registry'ye push ettikten sonra Kubernetes'e deploy edebiliriz. Temel bir `deploy.yaml` şöyle görünür:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: azdevops-agent
  labels:
    app: azdevops-agent
spec:
  replicas: 1
  selector:
    matchLabels:
      app: azdevops-agent
  template:
    metadata:
      labels:
        app: azdevops-agent
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: cce.cloud.com/cce-nodepool
                operator: In
                values:
                - company-azdevops-nodepool
      securityContext:
        supplementalGroups:
          - 10001
      containers:
      - name: agent
        image: yildizozan/ado-agent:16
        env:
          - name: AZP_URL
            value: "https://azdevops.company.com/DefaultCollection"
          - name: AZP_TOKEN
            value: "top_secret_token"
          - name: AZP_POOL
            value: k8s
        securityContext:
          privileged: true
        volumeMounts:
        - name: data
          mountPath: /azp
        - name: dockersock
          mountPath: "/var/run/docker.sock"
      - name: sleep
        image: busybox
        command: ['sh', '-c', 'sleep 1000000']
        volumeMounts:
        - name: data
          mountPath: /azp
      volumes:
      - name: data
        emptyDir: {}
      - name: dockersock
        hostPath:
          path: /var/run/docker.sock
```

Bu manifest ile agent pod olarak ayağa kalkıp Azure DevOps pool'una bağlanır. Bir pipeline tetiklendiğinde agent işi alır ve tamamlar.

## NuGet Config

Pipeline'ları çalıştırmaya başladığımızda şöyle bir hatayla karşılaştık:

```
(0,0): Error NU1101: Unable to find package Company.Core. No packages exist with this id in source(s): nuget.org
sproj : error NU1101: Unable to find package Company.Core. No packages exist with this id in source(s): nuget.org
```

Sorun şu: agent container'ı yeni ve temiz; şirket içi private NuGet artifact repository'mizi tanımıyor. Sadece nuget.org'u biliyor. Şirket ağındaki NuGet package'larını çekebilmek için NuGet konfigürasyonunu container'a taşımamız gerekiyor.

Çözüm: mevcut bir agent makinesinden `nuget.config` dosyasını alıp Kubernetes secret olarak saklamak:

```bash
kubectl create secret generic nuget-config --from-file nuget.config
```

Ardından bu secret'ı deployment'a volume olarak bağlıyoruz:

```yaml
volumeMounts:
- name: nuget-config
  mountPath: /home/agent/.nuget/NuGet/NuGet.Config
  subPath: nuget.config
volumes:
- name: nuget-config
  secret:
    secretName: nuget-config
```

## Volume İzin Sorunu ve fsGroup

NuGet config'i ekleyip deploy ettikten sonra pipeline yeniden çalıştı ama bu sefer farklı bir hatayla karşılaştık: agent, mount edilen config dosyasını okuyamıyordu.

Sorunun kaynağı şu: Kubernetes, volume mount işlemlerini varsayılan olarak `root` kullanıcısıyla yapar. Bizim agent'ımız ise `agent` kullanıcısıyla çalışıyor (UID 1000). Dosya root'a ait olduğundan agent okuyamıyor.

Çözüm olarak `securityContext`'e `fsGroup` eklememiz gerekiyor. `fsGroup`, mount edilen volume'ların belirtilen group ID'siyle sahiplendirilmesini sağlıyor. Değer olarak agent kullanıcısının UID'si olan `1000`'i veriyoruz:

```yaml
securityContext:
  fsGroup: 1000  # agent kullanıcısının UID'si
  supplementalGroups:
    - 10001
```

## deploy.yaml Son Hali

Tüm bu değişiklikleri içeren final manifest:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: azdevops-agent
  labels:
    app: azdevops-agent
spec:
  replicas: 1
  selector:
    matchLabels:
      app: azdevops-agent
  template:
    metadata:
      labels:
        app: azdevops-agent
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: cce.cloud.com/cce-nodepool
                operator: In
                values:
                - company-azdevops-nodepool
      securityContext:
        fsGroup: 1000  # agent kullanıcısının UID'si
        supplementalGroups:
          - 10001
      containers:
      - name: agent
        image: yildizozan/ado-agent:16
        env:
          - name: AZP_URL
            value: "https://azdevops.company.com/DefaultCollection"
          - name: AZP_TOKEN
            value: "top_secret_token"
          - name: AZP_POOL
            value: k8s
        securityContext:
          privileged: true
        volumeMounts:
        - name: data
          mountPath: /azp
        - name: dockersock
          mountPath: "/var/run/docker.sock"
        - name: nuget-config
          mountPath: /home/agent/.nuget/NuGet/NuGet.Config
          subPath: nuget.config  # subPath yazmazsak dizin olarak mount eder
      - name: sleep
        image: busybox
        command: ['sh', '-c', 'sleep 1000000']
        volumeMounts:
        - name: data
          mountPath: /azp
      volumes:
      - name: data
        emptyDir: {}
      - name: dockersock
        hostPath:
          path: /var/run/docker.sock
      - name: nuget-config
        secret:
          secretName: nuget-config
```

## Sonuç

Azure DevOps agent'larını Kubernetes üzerinde çalıştırmak başlangıçta birkaç ek adım gerektiriyor ancak uzun vadede VM yönetiminin getirdiği yükten kurtarıyor. Replicas sayısını artırarak anında yeni agent ekleyebilir, iş yüküne göre scale edebilirsiniz.

Bu kurulumda karşılaştığımız iki temel sorun şunlardı:

- **Private NuGet repository**: Agent container'ı şirket içi package kaynağını tanımıyor. `nuget.config`'i Kubernetes secret olarak saklayıp volume mount ile çözüldü.
- **Volume izin sorunu**: Root ile mount edilen dosyayı `agent` kullanıcısı okuyamıyor. `securityContext.fsGroup` ile çözüldü.

Eğer siz de on-premise Azure DevOps Server kullanıyorsanız agent versiyonuna dikkat edin; server versiyonuyla uyumsuz agent binary container'a indirilemez ve kurulum sessizce başarısız olabilir.
