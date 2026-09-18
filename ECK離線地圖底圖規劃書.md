# ECK 離線地圖底圖規劃書(全球底圖 / tileserver-gl)

> 從《ECK部署規劃書.md》原 §10 移出,範圍由 Taiwan 單一區域改為**全球底圖**。
> 依賴主文件:硬體規格(§0)、PV/節點 Label(§4)、Kibana values(§6)、GeoIP City(§9,geo_point 來源)。

---

## 0. 方案

- 免費 Basic 授權連不到 Elastic Maps Service(EMS),air-gapped 環境本來就無法連外網——自架底圖伺服器:**tileserver-gl v5.6.0**,讀 `.mbtiles`,跑在 k8s-controller。
- 底圖資料:直接下載 [OpenFreeMap](https://openfreemap.org/) 現成的全球 `.mbtiles`,免自行轉檔。
- Kibana 以 `map.tilemap.url` 指向 tileserver,並關閉 `map.includeElasticMapsService`(見 §4.5)。
- Geofencing:手動查詢圈選(geo_bounding_box / geo_shape),不涉付費告警。

---

## 1. 下載 tiles.mbtiles(OpenFreeMap)

### 1.1 確認來源版本

```bash
curl -s "https://btrfs.openfreemap.com/files.txt" | grep "areas/planet/.*tiles.mbtiles$" | tail -5
# 取最新版本目錄,例如 areas/planet/20260810_211301_pt/
curl -s "https://btrfs.openfreemap.com/areas/planet/<版本>/osm_date"      # 該版本的 OSM 資料日期
curl -s "https://btrfs.openfreemap.com/areas/planet/<版本>/SHA256SUMS"    # 官方 checksum
```

**已驗證可用的版本(供對照)**:

| 項目 | 值 |
|---|---|
| 版本 | `20260810_211301_pt` |
| OSM 資料日期 | 2026-08-03 |
| 下載網址 | `https://btrfs.openfreemap.com/areas/planet/20260810_211301_pt/tiles.mbtiles` |
| 大小 | 101,859,651,584 bytes(~94.9GB) |
| SHA256 | `60a37d2324a27c18798570b9193534c1b88a2a564c10a808a03a419e8dd15ea1` |
| 授權 | MIT(程式碼)+ OSM ODbL(資料,須標示 © OpenStreetMap contributors) |

> 每週更新一次,若要用更新版本,以上面指令查最新版取代路徑,並改用該版本自己的 `SHA256SUMS`。
> **換版本時,`tileserver/openfreemap/download_loop.sh` 內寫死的三個常數 `URL`/`EXPECTED`(bytes 數)/結尾 `sha256sum -c -` 用的 hash 要一起換**,只改 `URL` 的話,下載會卡在錯的 `EXPECTED` 判斷式裡,或跑出誤導性的 checksum 失敗。腳本內 `cd` 用的也是寫死的絕對路徑(`/home/user/elastic-stack/tileserver/openfreemap`),換機器部署時這行也要對應調整。

### 1.2 下載(可續傳、自動重試)

95GB 等級的下載,**不要用單純一行 `curl`**——長連線搭配 curl 內建 `--retry` 在 HTTP/2 上重試時,實測會發生 resume 位置與磁碟不同步、把已下載內容覆寫掉的問題(曾一次损失 15GB+ 進度)。改用 `tileserver/openfreemap/download_loop.sh`,重試交給外層迴圈處理,強制 HTTP/1.1、每 30 分鐘乾淨重啟一次連線。

執行:
```bash
mkdir -p tileserver/openfreemap
cd tileserver/openfreemap
chmod +x download_loop.sh
nohup ./download_loop.sh >> download_loop.log 2>&1 &
disown
```

**查進度**:`stat -c%s tileserver/openfreemap/tiles.mbtiles`(目標:101859651584)

**暫停**(不遺失進度):
```bash
pkill -TERM -f download_loop.sh
pkill -TERM -f "curl.*tiles.mbtiles"
```

**繼續**:重新執行同一支 `nohup ./download_loop.sh ...` 即可,自動從磁碟現有大小續傳。

**若看到 `WARNING: 檔案變小了`**:代表又發生 resume 覆寫,立即暫停,重新執行腳本(會自動以磁碟實際大小續傳,不需手動改檔)。

### 1.3 下載完成後驗證

```bash
sha256sum -c <(echo "60a37d2324a27c18798570b9193534c1b88a2a564c10a808a03a419e8dd15ea1  tiles.mbtiles")
file tiles.mbtiles      # 應顯示 SQLite 3.x database (MBTiles tileset)

# 落地一份 tiles.mbtiles.sha256——§3/§4.2 的 air-gapped 匯入流程會用到這個檔案,
# 上面 sha256sum -c 只是驗證,不會自動產生這個檔案:
echo "60a37d2324a27c18798570b9193534c1b88a2a564c10a808a03a419e8dd15ea1  tiles.mbtiles" > tiles.mbtiles.sha256

# 順便查證 mbtiles metadata(§2 提到的 name 字串比對、§4.5 提到的 maxzoom 出處都來自這裡):
sqlite3 tiles.mbtiles "select name, value from metadata where name in ('name','maxzoom','minzoom','format');"
# -> name|OpenFreeMap  maxzoom|14  minzoom|0  format|pbf(已實測確認)
```

---

## 2. tileserver-gl 設定檔(必要,不是選配)

**只給 `--mbtiles` 參數不夠**——v5.6.0 只在 mbtiles metadata 的 `name` 欄位包含字串 `"openmaptiles"` 時,才會自動掛上內建樣式。OpenFreeMap 的 `name` 是 `"OpenFreeMap"`,踩不中這個字串比對,會變成:

```
WARN: MBTiles not in "openmaptiles" format. Serving raw data only...
```

結果 `/styles.json` 回空陣列,Kibana 拿到的一律是 404——不是資料或授權問題,純粹是 tileserver-gl 原始碼裡的字串比對(`src/main.js`,`info.name.toLowerCase().indexOf('openmaptiles')`),已用容器內原始碼與實測 tile 畫面雙重確認。資料本身是完整的 OpenMapTiles schema(vector_layers 對得上內建 `basic-preview` 樣式引用的 `landuse`/`landcover`/`water` 等 source-layer),所以繞過偵測即可正常渲染。

**修法**:給明確的 `config.json`,直接宣告 data source 與樣式,跳過自動偵測。

`tileserver/config.json`:
```json
{
  "options": {
    "paths": {
      "root": "/usr/src/app/node_modules/tileserver-gl-styles",
      "fonts": "fonts",
      "styles": "styles",
      "mbtiles": "/data"
    }
  },
  "styles": {
    "basic-preview": {
      "style": "basic-preview/style.json",
      "tilejson": { "bounds": [-180, -85.05113, 180, 85.05113] }
    }
  },
  "data": {
    "v3": { "mbtiles": "tiles.mbtiles" }
  }
}
```

`options.paths.root` 是 v5.6.0 映像內建樣式的固定路徑(已用該版本映像實測確認,映像版本釘死見 §3,不會漂移)。此檔案本機測試與正式部署都要用(正式部署見 §4.3b/4.4)。

---

## 3. 下載 tileserver-gl docker image

```bash
docker pull --platform linux/amd64 maptiler/tileserver-gl:v5.6.0
docker save maptiler/tileserver-gl:v5.6.0 -o tileserver/images/tileserver-gl_v5.6.0_amd64.tar
sha256sum tileserver/images/tileserver-gl_v5.6.0_amd64.tar > tileserver/images/tileserver-gl_v5.6.0_amd64.tar.sha256
```

**已備妥(已執行並驗證)**:

| 檔案 | 大小 | SHA256 |
|---|---|---|
| `images/tileserver-gl_v5.6.0_amd64.tar` | 364MB | `64320a0f55f585148fd4923e700f2d6ced78be866e8d61b7c6d8e49f33642452` |

> 上面這個 tar 目前權限是 `-rw-------`(僅 owner 可讀),實際搬運/匯入時如果換了操作帳號會讀不到,記得先 `chmod`。

**§4.3c 需要一個 nginx sidecar 幫 tileserver 終止 TLS**(見該節),這個映像也要一起打包,不能漏——一樣釘死明確版本號,不要用 `:latest`:

```bash
docker pull --platform linux/amd64 nginx:1.27-alpine
docker save nginx:1.27-alpine -o tileserver/images/nginx_1.27-alpine_amd64.tar
sha256sum tileserver/images/nginx_1.27-alpine_amd64.tar > tileserver/images/nginx_1.27-alpine_amd64.tar.sha256
```

**搬進 air-gapped 需要的檔案**:

| 檔案 | 用途 |
|---|---|
| `openfreemap/tiles.mbtiles`(~95GB)+ `.sha256` | tileserver 資料 |
| `config.json` | tileserver-gl 樣式設定(§2,必要) |
| `images/tileserver-gl_v5.6.0_amd64.tar` + `.sha256` | 服務容器 |
| `images/nginx_1.27-alpine_amd64.tar` + `.sha256` | TLS sidecar(§4.3c,必要——沒有它 §4.5 的 https 底圖連不上) |

依主文件《ECK部署規劃書.md》§3.3 方式之一保全:推入內部 registry,或用實體介質匯入(§4.1)——**兩個映像都要處理**,不是只有 tileserver-gl。

---

## 4. 部署(k8s)

### 4.1 匯入 docker image

**兩個映像(`tileserver-gl` + §4.3c 的 `nginx` sidecar)都要匯入,以下指令對兩個 tar 各跑一次。**

```bash
# 方式 A(建議,跟主文件 §3.3 方式 A 一致):載入本地 docker、推進內部 registry
docker load -i tileserver-gl_v5.6.0_amd64.tar
docker tag maptiler/tileserver-gl:v5.6.0 registry.internal:5000/maptiler/tileserver-gl:v5.6.0
docker push registry.internal:5000/maptiler/tileserver-gl:v5.6.0

docker load -i nginx_1.27-alpine_amd64.tar
docker tag nginx:1.27-alpine registry.internal:5000/nginx:1.27-alpine
docker push registry.internal:5000/nginx:1.27-alpine

# 方式 B:直接把 tar 匯入節點的 containerd(過渡用)。
# 注意:crictl 沒有 load 子命令——containerd 原生匯入本地 tar 要用 ctr,不是 crictl
# (這跟主文件 §3.3 方式 B 的「crictl pull 從 registry 拉」是不同情境,那邊拉的是遠端映像,
#  這裡是匯入本機已存在的 tar,兩者不能混用)。實際節點上執行前建議先跑一次
# `ctr -n k8s.io images import --help` 確認語法,這條指令目前文件裡沒有實測紀錄。
ctr -n k8s.io images import tileserver-gl_v5.6.0_amd64.tar
ctr -n k8s.io images import nginx_1.27-alpine_amd64.tar
```

### 4.2 mbtiles 放置到 controller

controller 為 5×SSD RAID5,mbtiles 直接放本地(不走 NFS)。單一 tileserver 副本對「內部底圖」通常已足夠,放一台即可(此處以 `k8s-controller01` 為例):

```bash
# k8s-controller01
mkdir -p /var/lib/tileserver
# 將 tiles.mbtiles 放入 /var/lib/tileserver/(scp/實體介質匯入)
sha256sum -c tiles.mbtiles.sha256
```

### 4.3 PV / PVC

**容量依實際檔案調整**(全球底圖遠大於原 Taiwan 版的 20Gi,以下抓 150Gi,建議 ≥1.2×實際檔案大小):

`tileserver-pv.yaml`:
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: tileserver-data
spec:
  capacity:
    storage: 150Gi
  accessModes:
    - "ReadOnlyMany"
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""
  local:
    path: /var/lib/tileserver
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - "k8s-controller01"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: tileserver-data
  namespace: gis
spec:
  accessModes:
    - "ReadOnlyMany"
  storageClassName: ""
  volumeName: tileserver-data
  resources:
    requests:
      storage: 150Gi
```

### 4.3b tileserver-gl 設定(ConfigMap)

`config.json`(§2)體積小、無機密內容,用 ConfigMap 掛載,跟 mbtiles 的 PV 分開,不用把它塞進同一個 150Gi 的 PV:

`tileserver-config.yaml`:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: tileserver-config
  namespace: gis
data:
  config.json: |
    {
      "options": {
        "paths": {
          "root": "/usr/src/app/node_modules/tileserver-gl-styles",
          "fonts": "fonts",
          "styles": "styles",
          "mbtiles": "/data"
        }
      },
      "styles": {
        "basic-preview": {
          "style": "basic-preview/style.json",
          "tilejson": { "bounds": [-180, -85.05113, 180, 85.05113] }
        }
      },
      "data": {
        "v3": { "mbtiles": "tiles.mbtiles" }
      }
    }
```

### 4.3c TLS(nginx sidecar,必要——見 §4.5 mixed content 說明)

**為什麼需要**:正式環境 Kibana 走 HTTPS(`server.publicBaseUrl: "https://<Kibana-VIP>:5601"`,主文件 §6),但 `tileserver-gl` 本身沒有原生 TLS termination。若 `map.tilemap.url`(§4.5)維持純 `http://`,瀏覽器對 HTTPS 頁面裡的 HTTP 資源預設會直接阻擋、或自動升級成 https 後失敗——不管哪種機制,底圖磚都會整片消失,而且不會有明顯錯誤訊息,容易被誤判成別的問題。§5/§7.5 的本機測試測不出這個問題,因為本機 Kibana 也是純 http,兩邊協定一致沒有 mixed content。

**修法**:在同一個 Pod 裡加一個 `nginx` sidecar 終止 TLS、反向代理到 `tileserver` container 的 `localhost:8080`,Service 對外只開 443。

**待確認(部署前必須先解決,不是可以晚點補的優化項)**:cert-manager 要用哪個 `ClusterIssuer`/`Issuer`。**首選是沿用簽 ES/Kibana 憑證用的那個內部 CA `ClusterIssuer`**——好處是瀏覽器/OS 只要匯入這條內部 CA 一次,之後這條 CA 簽的所有服務憑證(ES、Kibana、tileserver…)都自動被信任,而且如果 ES/Kibana 的憑證信任鏈已經在使用者端建立好了,tileserver 直接沿用就不用再處理一次。部署前跟叢集管理者要到這個 issuer 的實際名稱,填進下面的 `issuerRef`。

下面範例的 `SelfSigned` Issuer **只能拿來測連通性,不能當正式部署的預設值**——`SelfSigned` 簽出來的是「自己簽自己」的單張 leaf 憑證,不是一條可重複使用的 CA:瀏覽器/Kibana 載入底圖圖磚是 subresource 請求,對不受信任的憑證**不會跳出可點擊「繼續前往」的警告,會直接靜默失敗**——效果跟沒修 mixed content 之前一樣,底圖一樣整片消失、看起來像沒改到。而且 §6.5 加的「內部 CA 已匯入使用者瀏覽器信任庫」這項檢查,`SelfSigned` 這條路徑本來就沒有 CA 可以匯入,永遠過不了。真的要用 `SelfSigned` 過渡,務必先在 §5.2(本機重現)確認瀏覽器信任問題怎麼處理,再套用到正式環境。

**`<tile-VIP>` 是 MetalLB 分配的 IP,不是網域名稱**——憑證要用 `ipAddresses`,不是 `dnsNames`(用 `dnsNames` 簽的憑證,瀏覽器直接用 IP 連線時會 hostname 驗證失敗,`curl -k` 因為跳過驗證所以測不出這個差異,§6.1/§6.2 的 `-k` 不能拿來確認這件事)。§4.5 的 `map.tilemap.url` 與這裡的 `ipAddresses` 必須是同一個值。

`tileserver-tls.yaml`(下面用 `SelfSigned` 只是示範欄位長相,**實際部署把 `issuerRef` 換成內部 CA issuer**):
```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: tileserver-selfsigned    # 僅供連通性測試;正式部署改用內部 CA ClusterIssuer(見上)
  namespace: gis
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: tileserver-tls
  namespace: gis
spec:
  secretName: tileserver-tls
  issuerRef:
    name: tileserver-selfsigned  # ← 換成內部 CA ClusterIssuer 的名稱
    kind: Issuer                 # ← 若換成 ClusterIssuer,這裡也要改成 kind: ClusterIssuer
  ipAddresses:
    - "<tile-VIP>"     # 換成實際 tile-VIP,必須跟 §4.5 map.tilemap.url 裡的值一致
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: tileserver-nginx-config
  namespace: gis
data:
  nginx.conf: |
    events {}
    http {
      server {
        listen 8443 ssl;
        ssl_certificate     /etc/nginx-tls/tls.crt;
        ssl_certificate_key /etc/nginx-tls/tls.key;
        location / {
          proxy_pass http://127.0.0.1:8080;
        }
      }
    }
```

### 4.4 部署 tileserver-gl

**Namespace 獨立成一個檔案先建,§4.4 結尾的 apply 指令必須先跑這個**(舊版把 `Namespace` 併在 `tileserver.yaml` 裡、卻排在 ConfigMap/PVC 之後 apply,會直接報 `namespaces "gis" not found`):

`tileserver-namespace.yaml`:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: gis
```

`tileserver.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tileserver
  namespace: gis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tileserver
  template:
    metadata:
      labels:
        app: tileserver
    spec:
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          effect: NoSchedule
      nodeSelector:
        kubernetes.io/hostname: k8s-controller01     # 釘到放 mbtiles 的那台
      containers:
        - name: tileserver
          image: registry.internal:5000/maptiler/tileserver-gl:v5.6.0
          args:
            - "--config"
            - "/etc/tileserver-config/config.json"
            - "--port"
            - "8080"
          ports:
            - containerPort: 8080
          resources:
            requests:
              memory: "1Gi"
              cpu: "500m"
            limits:
              memory: "2Gi"
          volumeMounts:
            - name: mapdata
              mountPath: /data
              readOnly: true
            - name: tsconfig
              mountPath: /etc/tileserver-config
              readOnly: true
        - name: nginx-tls                              # §4.3c:終止 TLS,反代到上面的 tileserver:8080
          image: registry.internal:5000/nginx:1.27-alpine
          ports:
            - containerPort: 8443
          volumeMounts:
            - name: nginx-config
              mountPath: /etc/nginx/nginx.conf
              subPath: nginx.conf
              readOnly: true
            - name: tls-cert
              mountPath: /etc/nginx-tls
              readOnly: true
      volumes:
        - name: mapdata
          persistentVolumeClaim:
            claimName: tileserver-data
            readOnly: true
        - name: tsconfig
          configMap:
            name: tileserver-config
        - name: nginx-config
          configMap:
            name: tileserver-nginx-config
        - name: tls-cert
          secret:
            secretName: tileserver-tls
---
apiVersion: v1
kind: Service
metadata:
  name: tileserver
  namespace: gis
spec:
  type: LoadBalancer
  selector:
    app: tileserver
  ports:
    - port: 443
      targetPort: 8443
```

```bash
kubectl apply -f tileserver-namespace.yaml            # 一定要先建 namespace,其餘檔案都引用 namespace: gis
kubectl apply -f tileserver-tls.yaml
kubectl apply -f tileserver-config.yaml
kubectl apply -f tileserver-pv.yaml
kubectl apply -f tileserver.yaml
kubectl -n gis get pods -o wide
kubectl -n gis logs -l app=tileserver -c tileserver | grep -i "warn\|error"   # 不應出現 "not in \"openmaptiles\" format"
```

### 4.5 Kibana 指向自架底圖

**mixed content 提醒**:正式環境 Kibana 走 HTTPS(`server.publicBaseUrl: "https://<Kibana-VIP>:5601"`),`map.tilemap.url` 必須是 `https://`,對應 §4.3c 的 nginx TLS sidecar——不能沿用 §5 本機測試(純 http)的 URL,否則瀏覽器會把底圖圖磚當 mixed content 擋掉或升級失敗,底圖整片消失且不一定有明顯錯誤訊息。

**套用時,把主文件 `eck-stack-values.yaml` 裡那行已經存在、但已過期的 `map.tilemap.url` 註解 placeholder 直接取代掉**(該行路徑是 `/tile/{z}/{x}/{y}.png`,跟這裡實際要用的 `/styles/basic-preview/{z}/{x}/{y}.png` 不同;縮排也對不齊同層的 `xpack.reporting.*`),不要讓新舊兩行 `map.tilemap.url` 並存造成混淆。

`eck-stack-values.yaml` 的 `eck-kibana.config`:
```yaml
  config:
    map.tilemap.url: "https://<tile-VIP>/styles/basic-preview/{z}/{x}/{y}.png"
    map.tilemap.options.minZoom: 0
    map.tilemap.options.maxZoom: 18                  # tiles.mbtiles 的 maxzoom=14(§1.3 用 sqlite3 查證),但 tileserver-gl 會對超過的縮放做 over-zoom(拿 z14 向量圖磚放大渲染,已實測 z16 正常出圖);這裡是「使用者在 Kibana 能縮多近」的上限,不是資料上限,設 14 會讓使用者卡在街廓尺度,画 geofence 多邊形時像是地圖壞了
    map.tilemap.options.attribution: >-
      &copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a>
      contributors &copy; <a href="https://www.openmaptiles.org/">OpenMapTiles</a>
      tiles by <a href="https://openfreemap.org">OpenFreeMap</a>
    map.includeElasticMapsService: false             # air-gapped 連不到 EMS,關掉避免 Maps app 白等逾時
```

> `map.tilemap.options.attribution` 是 ODbL 授權要求的標示義務(§1.1),不是裝飾用選項。

```bash
helm upgrade prod elastic/eck-stack -n elastic-stack --version 0.19.1 -f eck-stack-values.yaml
```

---

## 5. 本機 docker 測試(k8s 部署前先驗證)

用 ES 9.4.2 + Kibana 9.4.2 + tileserver-gl v5.6.0 在本機跑一輪,不用碰到 k8s 就能抓出設定問題(§2 的 config.json 問題就是這樣抓出來的)。三個映像都建議先備好本機快取,起這個 compose 就不需要連網。

`tileserver/docker-compose.local-test.yml` + `tileserver/kibana.local-test.yml` + `tileserver/config.json`(已備妥,見檔案本身的行內註解)。

Kibana 連線帳號用 `kibana_system`(9.4.2 起直接拒絕 `elastic` 這個 superuser 連線,啟動即 FATAL,已實測驗證)。它的密碼是 ES 起來後才存在的一次性資料,起 compose 前還沒有,所以**先只起 ES 跟 tileserver**,設完密碼再起 Kibana——順序顛倒 Kibana 會直接 crash-loop:

```bash
cd tileserver
docker compose -f docker-compose.local-test.yml up -d elasticsearch tileserver

curl -u elastic:changeme-test-only -X POST \
  http://localhost:9200/_security/user/kibana_system/_password \
  -H 'Content-Type: application/json' -d '{"password":"changeme-test-only"}'

docker compose -f docker-compose.local-test.yml up -d kibana

curl -s http://localhost:8080/styles.json                          # 應非空陣列,含 basic-preview
curl -s -u elastic:changeme-test-only http://localhost:5601/api/status
```

瀏覽器開 `http://localhost:5601`(elastic / changeme-test-only)→ Maps → Add layer → **Configured Tile Map Service** → 確認底圖有渲染。

### 5.1 順便驗證圖層疊加(ES 資料圖層)

底圖確認沒問題後,同一張 Map 再疊 ES 資料圖層,確認多圖層能一起呈現(概念說明見 §7.1–7.4,
逐步指令與可直接匯入的示範 Map 見 **§7.5**):

1. 參考 §7.5.1 建 `geo-test` 索引並灌 5 筆點位(該節指令已是本機 compose 連線形式)。
2. 建 `geo-test` 的 data view 時選「不使用時間欄位」(這批測試資料沒有 `@timestamp`)。
   Maps → Add layer → **Documents** → data view 選 `geo-test` → geospatial field 選
   `geo.location` → 5 個點位疊在底圖上。
3. 工具列畫一個多邊形(涵蓋台北那 4 筆)→ 多一個圖層,左側 filter bar 自動出現 geo 查詢。
4. 確認三層(底圖 / 點位 / 多邊形)同時顯示;拖曳左側圖層清單改順序;點 Documents
   圖層 → Layer settings 調 Opacity → 底圖會從半透明的點位層透出來。
5. 右上 Save 存成一張 Map → Dashboards → Create dashboard → Add panel → 選這張 Map
   → 疊加圖層在 Dashboard 內一致呈現。

> 時間軸連動(改上方時間範圍、點位數量跟著變)需索引有時間欄位;正式環境
> GeoIP 產出的索引本來就有,用 §6.3 那步驗即可。

**測完清掉**:
```bash
docker compose -f docker-compose.local-test.yml down -v
```

### 5.2 本機重現 §4.3c 的 TLS sidecar(在犯錯成本低的地方先抓 nginx.conf/憑證問題)

§4.3c 的 nginx sidecar、憑證、瀏覽器信任鏈完全沒有實跑過——air-gapped 正式環境是最不該第一次跑這段的地方。建議先在本機用同一份 `nginx.conf` 過一遍:

1. 本機產生一張自簽憑證(`openssl req -x509 -newkey rsa:2048 -nodes -keyout tls.key -out tls.crt -days 30 -addext "subjectAltName=IP:127.0.0.1"`——注意用 `subjectAltName=IP:...`,不是 CN/DNS,呼應 §4.3c「`<tile-VIP>` 是 IP,憑證要用 ipAddresses」的重點)。
2. 額外起一個 `nginx:1.27-alpine` 容器,掛 §4.3c 那份 `nginx.conf`(把 `proxy_pass` 指到本機 compose 的 `tileserver:8080`)+ 上面產生的 `tls.crt`/`tls.key`,對外開 8443。
3. 把 `kibana.local-test.yml` 的 `map.tilemap.url` 暫時改成 `https://localhost:8443/styles/basic-preview/{z}/{x}/{y}.png`,重啟 Kibana。
4. 瀏覽器開 Kibana Maps,確認底圖圖磚正常渲染(跟 §7.5 的截圖比對)。若瀏覽器擋憑證不信任,這裡先解掉信任問題(匯入這張測試憑證,或改用內部 CA 簽的憑證),而不是等到正式環境才發現。
5. 驗證完把 `map.tilemap.url` 改回本機測試原本的值,或直接 `down -v` 收掉。

這一步主要是驗證 nginx.conf 語法、憑證 SAN 型別(`ipAddresses` vs `dnsNames`)、以及瀏覽器信任鏈這三件事——跟正式環境用哪個 cert-manager issuer 無關,不需要等 issuer 確定才能做。

---

## 6. 測試(正式環境)

### 6.1 服務健康檢查

```bash
TILEIP=$(kubectl -n gis get svc tileserver -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "tile VIP: $TILEIP"
# §4.3c 已加 TLS,Service 對外是 443——優先用 --cacert 指向簽這張憑證的內部 CA(正式部署走的路徑)。
# -k 只證明 nginx sidecar 有在聽、TLS handshake 能過,不能證明瀏覽器會信任這張憑證
# (§4.3c 已說明 SelfSigned 僅供連通性測試,-k 剛好會把這個信任問題蓋過去,兩者不要混用來下結論)。
curl -s --cacert internal-ca.crt "https://$TILEIP/" -o /dev/null -w "%{http_code}\n"        # 應回 200
curl -s --cacert internal-ca.crt "https://$TILEIP/styles.json"                              # 應非空陣列,含 basic-preview(§2)
```

### 6.2 抽查實際 tile

**不要只看 200,也不要只看 z0**——z0 是全球縮成一格,內容量本來就大,200 加上「看起來有東西」都不能證明底圖沒問題。真正有鑑別力的比較是**同一 zoom 下,海面 tile vs. 城市 tile**:海面應該只是純色 + 少量海岸線(數 KB 等級),城市應該有道路網、地名(數十 KB 等級)。差距不明顯(例如兩者都是數 KB)才代表可能只拿到空白樣式。

tile 的 x/y 用經緯度換算(`n=2^z`,`x=floor((lon+180)/360*n)`,`y` 用 Web Mercator 公式),或直接用下面已算好的 z10 座標(台北,約 121.56E 25.03N):

```bash
curl -s --cacert internal-ca.crt -o /tmp/ocean.png -w "海面 z10 http:%{http_code} bytes:%{size_download}\n" \
  "https://$TILEIP/styles/basic-preview/10/863/442.png"              # 對照組:相鄰海域,實測約 2.5KB
curl -s --cacert internal-ca.crt -o /tmp/taipei.png -w "台北市區 z10 http:%{http_code} bytes:%{size_download}\n" \
  "https://$TILEIP/styles/basic-preview/10/857/438.png"               # 實測約 70KB,含路網/地名
```

### 6.3 Kibana Maps 顯示驗證

- Kibana → Maps → Add layer → **Configured Tile Map Service**(讀 `map.tilemap.url`,不是預設底圖,需手動加圖層)→ 確認底圖有渲染(非空白/非 404)。
- 切換到不同大洲/縮放等級,確認全球覆蓋(非僅單一區域)。
- 疊 ES 資料圖層:再 Add layer → **Documents**,指向正式環境有 `geo.location` 的索引 → 確認點位疊在底圖上;調整圖層順序與透明度、切換時間範圍都即時反映(完整說明見 §7)。

### 6.4 Geofencing 手動查詢

資料需有 `geo_point`/`geo_shape` 欄位(§9 產生的 `geo.location`)。正式環境這欄位由 GeoIP pipeline 自動產生;這裡示範查詢用的 `geo-test` 索引則要手動建立測試資料——**明確宣告 `geo.location` 為 `geo_point`**(dynamic mapping 不會自動推斷),灌 5 筆點位,4 筆在下面查詢用的台北範圍內、1 筆(高雄)刻意在範圍外,這樣才有東西可以被篩掉,驗證查詢真的有在過濾:

```bash
curl -k -u "elastic:$PW" -X PUT "https://$ESIP:9200/geo-test" \
  -H 'Content-Type: application/json' -d '{
  "mappings": {
    "properties": {
      "name": { "type": "keyword" },
      "geo": {
        "properties": {
          "location": { "type": "geo_point" }
        }
      }
    }
  }
}'

curl -k -u "elastic:$PW" -X POST "https://$ESIP:9200/geo-test/_bulk?pretty" \
  -H 'Content-Type: application/json' --data-binary @- <<'EOF'
{"index":{}}
{"name":"Taipei 101","geo":{"location":{"lat":25.0330,"lon":121.5654}}}
{"index":{}}
{"name":"Taipei Main Station","geo":{"location":{"lat":25.0478,"lon":121.5170}}}
{"index":{}}
{"name":"Ximending","geo":{"location":{"lat":25.0421,"lon":121.5079}}}
{"index":{}}
{"name":"Tamsui","geo":{"location":{"lat":25.1697,"lon":121.4419}}}
{"index":{}}
{"name":"Kaohsiung (bbox 之外)","geo":{"location":{"lat":22.6273,"lon":120.3014}}}
EOF
```

驗證:`_count` 應回 5;下面的 geo_bounding_box 查詢應回 4(排除高雄那筆)。

```bash
# 矩形範圍
curl -k -u "elastic:$PW" "https://$ESIP:9200/geo-test/_search?pretty" \
  -H 'Content-Type: application/json' -d '{
  "query": { "geo_bounding_box": {
    "geo.location": {
      "top_left":     { "lat": 25.3, "lon": 121.4 },
      "bottom_right": { "lat": 24.9, "lon": 121.7 }
    }
  } }
}'

# 多邊形範圍(geo_shape + inline polygon——geo_polygon 是 ES 7.12 起 deprecated 的舊寫法,
# 這裡直接用跟 §7.5.3 indexed_shape 同族、本機已實測過的 geo_shape 寫法,座標順序是 GeoJSON 的 [lon,lat])
curl -k -u "elastic:$PW" "https://$ESIP:9200/geo-test/_search?pretty" \
  -H 'Content-Type: application/json' -d '{
  "query": { "geo_shape": {
    "geo.location": {
      "shape": {
        "type": "Polygon",
        "coordinates": [[
          [121.4,25.3],[121.7,25.3],[121.7,24.9],[121.4,24.9],[121.4,25.3]
        ]]
      },
      "relation": "within"
    }
  } }
}'
```

Kibana 內操作:Maps 建圖層 → 工具列繪製多邊形/矩形 → 自動轉成 geo 查詢過濾。

### 6.5 部署後檢查清單

- [ ] `tiles.mbtiles` SHA256 與 §1.1 官方值一致。
- [ ] tileserver pod 的 log **沒有** `not in "openmaptiles" format` 警告(§2,ConfigMap 掛好就不會出現)。
- [ ] tileserver pod Running 在 §4.2 放檔案的那台(`kubectl -n gis get pods -o wide`)。
- [ ] `https://<tile-VIP>/` 回 200,`/styles.json` 非空陣列(§4.3c 的 TLS sidecar 有正常工作)。
- [ ] 瀏覽器對 `https://<tile-VIP>/...` 的憑證受信任(內部 CA 已匯入使用者瀏覽器信任庫,或憑證鏈完整)——不受信任的話 Kibana Maps 一樣載不出底圖。
- [ ] Kibana Maps 加 **Configured Tile Map Service** 圖層後底圖正常顯示,任意大洲皆有圖資(非僅局部區域,且非純色空白 tile)。
- [ ] Kibana Maps 能同時疊「底圖 + ES 資料圖層(Documents / Heat map)」,逐層順序與透明度可調,整張 Map 可嵌入 Dashboard(§7)。
- [ ] `geo.location` 已宣告 `geo_point`(主文件 §17.2),Geofencing 查詢回傳預期筆數。
- [ ] PV 容量(§4.3)與實際 `tiles.mbtiles` 大小有餘裕(建議 ≥1.2×)。

---

## 7. Kibana Maps 圖層疊加(ES 資料圖層)

**結論:可以,這套設定現況就支援,不需要額外部署。** `map.includeElasticMapsService: false`
只拿掉 EMS 的底圖來源與 EMS 代管的行政區界檔,不影響任何「從 Elasticsearch 讀資料」的圖層。
tileserver 的底圖(§4.5 的 raster tile)是最底層,ES 資料圖層往上疊。

### 7.1 什麼是 EMS?為什麼要關?關掉後少了什麼

**EMS(Elastic Maps Service)** 是 Elastic 官方代管、跑在公網上的地圖服務
(`*.maps.elastic.co`),Kibana Maps 預設會去連它拿兩樣東西:

1. **底圖(basemap tiles)** —— 地圖的背景(道路、亮色 / 暗色主題),由 Elastic 的 CDN 提供。
2. **行政區界檔(EMS file layers / boundaries)** —— 一組預先做好的邊界 GeoJSON:
   世界各國、一級行政區(州 / 省)、二級行政區(縣 / 郡)、美國郵遞區號…等。
   拿來做 **choropleth(區域著色圖)**:把「每個國家 / 每個縣」依某個指標值上色。

**為什麼關(`map.includeElasticMapsService: false`)**:

- air-gapped 環境沒有對外網路,連不到 `*.maps.elastic.co`。
- 若不關,Kibana 每次開地圖都會去戳 EMS,每個請求卡到逾時(數十秒)才放棄 ——
  Maps app 開起來「白等」、還可能跳錯誤橫幅。§4.5 因此明確關掉。

**關掉後少的就是上面那兩樣**,而且兩樣都有替代:

| 少了什麼 | 替代方案 |
|---|---|
| EMS 底圖 | 換成自架 tileserver 的 `basic-preview`(§4.5 `map.tilemap.url`) |
| EMS 行政區界檔 | 要用的話自備區界 GeoJSON(見 §7.3.5) |

**關掉不影響的**:所有「資料來自 Elasticsearch」的圖層 —— Documents、聚合圖層
(Clusters / Heat map)、Tracks、手繪圖形、上傳的 GeoJSON、以及手動輸入 URL 的
TMS / WMS 圖層。這些都是本機或內網算的,跟 EMS 無關。

### 7.2 先知道:關掉 EMS 後新 Map 是空白的

新開一張 Map **不會**自動帶底圖 —— 畫面一開始是空白灰底。要手動 Add layer →
**Configured Tile Map Service**(讀 `map.tilemap.url`,§4.5)底圖才出現。這是預期行為、
不是壞掉;之後所有資料圖層都疊在這層之上。

### 7.3 可疊加的圖層(逐一說明)

| 圖層類型 | 一句話 | 資料需求 | 授權 |
|---|---|---|---|
| **Documents** | 每筆 ES 文件畫成一個點 / 線 / 面 | data view + `geo_point`/`geo_shape` 欄位 | Basic |
| **Clusters and grids** / **Heat map** | geotile / geohex 聚合成群集、格網、熱區 | 同上 | Basic |
| **Top hits per entity** | 每個實體最近 N 筆文件(以**點**顯示,不連線) | geo 欄位 + 排序欄位 + 實體欄位 | Basic |
| **Tracks** | 同一實體的多個時間點位置**連成軌跡線** | geo 欄位 + 時間欄位 + 實體欄位 | **Gold+**(本方案 Basic ✗) |
| **手繪圖形(Geofence)** | 工具列畫多邊形 / 矩形 | 無 —— 畫出來就是圖層(§6.4) | Basic |
| **上傳 GeoJSON / 檔案** | 靜態參考疊層(廠區、責任區、行政區界) | 一個 `.geojson` 檔 | Basic |

> **授權提醒**:本方案是「免費 Basic 授權」(§0)。**Tracks 圖層與其底層的 ES `geo_line`
> 聚合都需要 Gold 以上授權**,Basic 直接被擋(Kibana wizard 顯示灰底、ES 回
> `current license is non-compliant for [geo-line-agg]` —— 已於 §7.5 本機實測確認)。
> 需要「連成線的軌跡」時,Basic 的作法見 §7.3.3 / §7.5.2。其餘圖層 Basic 全可用。
> EMS 底圖與 EMS 行政區界 wizard 也會因 `map.includeElasticMapsService: false` 一起變灰(§7.1)。

疊的層數沒有硬上限(每個圖層各自發查詢,層多時留意查詢負載)。

#### 7.3.1 Documents —— 原始文件上圖

**呈現什麼**:ES 索引裡「每一筆文件」在地圖上一個圖徵(feature),點 / 線 / 面。
最直接的「raw data 上圖」。點某個圖徵會彈 tooltip 顯示你選的欄位。

**需要什麼**:

- 一個 data view 指到索引。
- 索引 mapping 至少一個欄位是 `geo_point`(經緯度點)或 `geo_shape`(線 / 面)。
  - 正式環境:GeoIP ingest pipeline 從來源 IP 查出的城市座標 `geo.location`(§9),型別 `geo_point`。
  - `geo_point` 必須在 mapping 明確宣告,dynamic mapping 不會自動推斷(§6.4)。
- (可選)一個 `date` 欄位當時間軸,上方 time picker 才篩得動。

**加圖層步驟**:Add layer → **Documents** → 選 data view → 選 geospatial field →
(可選)設 Scaling → Add and continue → 在右側 panel 設樣式與 tooltip。

**可調**:

- **Scaling**:`Limit results to N`(資料少,直接抓)/ `Show clusters when results exceed N`
  (超量自動轉群集)/ `Use vector tiles`(百萬級資料,走 `_mvt`,每磚受 `index.max_result_window` 限制)。
- **Filtering**:全域搜尋列 + 時間範圍會套;圖層本身也可加 query;
  可勾 `Only request data around map extent`(只抓目前畫面範圍)。
- **Tooltip fields**:點擊圖徵時顯示哪些欄位,自己挑。
- **Style**:填色 / 邊框色(固定,或**依欄位值上色** —— 例如依 `severity`、`http.response.status_code`
  套分類色或色階);標記大小(固定,或依數值欄位縮放,例如依 `bytes`);標記形狀(圓點 / icon);
  在標記旁標 label(某欄位值)。

**典型用途**:登入 / 連線來源的地理分布、每個告警或事件的發生點、資產座標。

#### 7.3.2 Clusters and grids / Heat map —— 聚合後上圖

**呈現什麼**:不畫每一筆,先用 geotile / geohex 網格把文件聚合,每格畫一個標記
(群集圓、格子多邊形、或熱區)。格子上可帶指標:count、avg、sum、unique count…。
資料量大時比 Documents 好看又快。

**需要什麼**:跟 Documents 一樣(data view + geo 欄位)。差別只在圖層類型選
**Clusters and grids** 或 **Heat map**,並選聚合指標與網格解析度(會隨 zoom 自動調)。

#### 7.3.3 移動軌跡(Tracks / Top hits per entity)

**想呈現什麼**:「同一個實體」(車、人、裝置、來源 IP)在不同時間的多個位置。

**⚠ Kibana 的 Tracks 圖層(連成線)需要 Gold+,本方案的 Basic 授權用不了。**
`Add layer → Tracks` 在 Basic 會是灰的;它底層的 ES `geo_line` 聚合也會回
`current license is non-compliant for [geo-line-agg]`(§7.5.2 本機實測)。

**Basic 授權下的兩種作法**:

| 作法 | 呈現 | 怎麼做 |
|---|---|---|
| **A. Top hits per entity**(內建、Basic 可用) | 每個實體**最近 N 個點**(不連線;`geo_point` 會被強制 line width = 0) | `Add layer → Top hits per entity` → 選 data view → point field、**entity field**、sort field(通常 `@timestamp`)、每實體幾筆。底層 = `terms(entity)` + 每 bucket `top_hits(sort @timestamp)` |
| **B. 自算 LineString 存 `geo_shape`**(要連成線就用這個) | 每個實體**一條完整折線** | 在寫入端(Logstash / ingest / 自己的程式)把一條軌跡的座標組成 `{"type":"LineString","coordinates":[[lon,lat],…]}`,存進 `geo_shape` 欄位。地圖上用 **Documents** 圖層畫出來(§7.3.1),線色 / 線寬照 Documents 樣式調 |

兩種都可以再疊一個同資料的 Documents 點層(顯示每個回報點),做成「線 + 點」。
時間軸篩選:作法 A 會跟著上方 time picker 動;作法 B 的 LineString 是一整筆,除非
你在寫入時就依時間切段。

**典型用途**:車隊軌跡、人員 / 裝置動線、來源 IP 隨時間的變化路徑。

#### 7.3.4 手繪圖形(Geofence)

工具列直接在地圖上畫多邊形 / 矩形 / 圓。兩種用途:

- **當查詢過濾器**:畫完自動轉成 `geo_bounding_box` / `geo_shape` filter,把其它圖層
  (與 Discover)篩成只剩範圍內的資料(§6.4)。
- **存成索引**:選 `Create index`,把畫的圖形寫進一個新 ES 索引,之後當固定 geofence 疊層重複用。

不需要任何前置資料 —— 畫出來的圖形本身就是圖層。

#### 7.3.5 上傳 GeoJSON —— 靜態參考疊層

**呈現什麼**:一個靜態 `.geojson` 檔畫上去當固定框線 —— 跟 ES 查詢無關,是「地圖上不會變的參考層」。
多邊形(廠區範圍、責任區、行政區界、機房樓層框)、線(管線、路線)、點(基站、閘門、攝影機位置)。

**怎麼用**:Maps → Add layer → **Upload file** → 拖一個 `.geojson` 進去。
Kibana 會把它**寫進一個新的 ES 索引**(`geo_shape` 或 `geo_point`)並建 data view,
之後它就是一個一般的 ES 圖層,只是資料是你上傳的。

**需要什麼**:

- 合法 GeoJSON:`FeatureCollection`,**WGS84 / EPSG:4326** 經緯度座標。
- 每個 feature 的 `properties`(名稱、代碼…)可拿來當 tooltip / 上色 / label。
- 檔案大小上限約 50MB(與 `server.maxPayload` 有關);太大的邊界檔先用 mapshaper 或
  `ogr2ogr -simplify` 簡化。

**可調**:填色 / 邊框 / 透明度 / label,跟 Documents 一樣。

**典型用途**:

- EMS 關掉後想要行政區界 → 從 GADM、Natural Earth 或政府開放資料下載台灣縣市界 GeoJSON → 上傳,
  再用 Choropleth 圖層拿 ES 聚合值對它上色。
- 標出自家園區 / 廠房實體範圍、預先畫好的 geofence 區域。

### 7.4 逐層控制

左側圖層清單(Layers panel)每一層都是獨立物件,點該層展開設定。以下逐項說明
「在哪設 / 效果 / 疊加時怎麼用」。

#### 7.4.1 圖層順序(Layer order)

- **在哪設**:左側圖層清單直接拖曳。清單**由上到下 = 地圖由上蓋到下**。
- **效果**:上層畫在下層之上;底圖永遠要放最底。
- **疊加時怎麼放**:由下往上建議順序 ——
  ① Configured Tile Map Service(底圖)
  ② 面 / 區塊層(責任區、行政區、Heat map)
  ③ 線 / 軌跡層(路線、軌跡 LineString)
  ④ 點層(Documents 點位、Clusters)
  ⑤ 純標籤 / 文字層(放最上,才不會被壓住)。
  順序錯了會出現「點被面蓋掉、看起來沒資料」。

#### 7.4.2 透明度(Opacity)

- **在哪設**:該層 → **Layer settings** → Opacity 滑桿(0–100%)。
- **效果**:整層透明度。
- **疊加時怎麼用**:面狀層(責任區、行政區、Heat map)設 **40–60%** 才看得到底圖街廓和下面的點;
  點狀層通常維持 100%(調透明反而看不清)。

#### 7.4.3 依縮放顯示(Visibility by zoom)

- **在哪設**:該層 → Layer settings → **Visibility** → 設 min / max zoom。
- **效果**:只有在該縮放區間才畫這層。
- **疊加時怎麼用**:同一份資料疊兩層做「隨縮放切換表現」——
  z0–z8 顯示 **Heat map**(全局分布),z9+ 關掉 Heat map、改顯示 **Documents 點位**(逐筆細節)。
  避免小比例尺時上萬個點糊成一團。

#### 7.4.4 每層獨立篩選(Filtering)

- **在哪設**:該層 → **Filtering** → 加 KQL/Lucene query;可勾 `Only request data around map extent`
  (只抓目前畫面範圍內的資料,大索引省流量)。
- **效果**:這層只顯示符合條件的文件。與 Kibana 上方**全域時間軸、全域搜尋列、Dashboard filter
  是 AND 疊加**。
- **疊加時怎麼用**:同一個索引疊兩層 —— 一層 filter `status:error` 塗紅加大,一層不 filter 塗灰,
  就成了「全部事件(灰)+ 異常事件(紅)突顯」。

#### 7.4.5 時間篩選(時序層)

- **效果**:層的 data view 有時間欄位時,會跟著上方 time picker 一起動;改時間範圍,
  點位 / 軌跡跟著增減。
- **在哪設**:該層 → Filtering → **Apply global time to layer data**(可關,讓某層固定顯示全部,
  不受時間軸影響)。
- **疊加時怎麼用**:一層「最近 15 分鐘」的即時點位(醒目)疊在一層「最近 24 小時」的軌跡(淡)之上。
- Tracks 這種時序層特別依賴這個 —— 時間範圍要涵蓋灌入資料的時間,否則軌跡是空的(見 §7.5.2)。

#### 7.4.6 Tooltip 與互動

- **在哪設**:該層 → **Tooltip fields**,勾要顯示的欄位。
- **效果**:點 feature 彈出資訊卡;多層在同一點重疊時,資訊卡會分頁(可切上下層)。
- Documents / Clusters / GeoJSON 層都支援;底圖(raster)沒有 tooltip。

#### 7.4.7 圖例(Legend)

- **效果**:當某層設了「依欄位上色」或「依欄位縮放大小」,地圖右下角**自動**產生對應圖例。
- **在哪設**:顏色 / 大小規則在該層 → **Layer style**;圖例本身跟著規則走,不用另外開。
- 疊多層時每層各自一段圖例,由上而下堆在右下角。

### 7.5 實作範例:三個疊加圖層從零到畫面

以下三個範例的 curl 指令都針對 **§5 的本機 compose**:`http://localhost:9200`、
`-u elastic:changeme-test-only`、明文 http(不用 `-k`)。正式環境把連線換成 §6 的
`https://$ESIP:9200` + `-k` + 實際密碼即可。

**驗證狀態**:每個範例的建索引 / 灌資料 / ⑥ 驗證查詢、以及 §7.5.4 的 saved object
匯入,都已在本機 ES/Kibana 9.4.2 + tileserver-gl v5.6.0 實跑通過(逐步輸出見
`tileserver/local-test-artifacts/verify.log`);④ 的 Maps 點選步驟**已有實際瀏覽器截圖佐證**
(`tileserver/local-test-artifacts/kibana-stacked-map.png` / `kibana-stacked-map-maponly.png`,
四層——底圖 + `geo-test` 點位 + `zones` 責任區 + `track-lines` 軌跡——同時正確疊加顯示)。

前置:`docker compose -f docker-compose.local-test.yml up -d`,三個容器都起來,
Kibana `GET /api/status` 的 `status.overall.level` = `available`。

#### 7.5.1 Documents —— geo-test 點位

**① 建索引(明確宣告 `geo_point`)**

```bash
curl -s -u elastic:changeme-test-only -H 'Content-Type: application/json' \
  -X PUT http://localhost:9200/geo-test -d '{
  "mappings":{"properties":{
    "name":{"type":"keyword"},
    "geo":{"properties":{"location":{"type":"geo_point"}}}
  }}}'
```

**② 灌 5 筆(4 筆台北、1 筆高雄)**

```bash
curl -s -u elastic:changeme-test-only -H 'Content-Type: application/json' \
  -X POST 'http://localhost:9200/geo-test/_bulk?refresh=wait_for' --data-binary @- <<'EOF'
{"index":{}}
{"name":"Taipei 101","geo":{"location":{"lat":25.0330,"lon":121.5654}}}
{"index":{}}
{"name":"Taipei Main Station","geo":{"location":{"lat":25.0478,"lon":121.5170}}}
{"index":{}}
{"name":"Ximending","geo":{"location":{"lat":25.0421,"lon":121.5079}}}
{"index":{}}
{"name":"Tamsui","geo":{"location":{"lat":25.1697,"lon":121.4419}}}
{"index":{}}
{"name":"Kaohsiung","geo":{"location":{"lat":22.6273,"lon":120.3014}}}
EOF
```

**③ 建 data view**:Kibana → Stack Management → Data Views → Create →
index pattern `geo-test` → **Timestamp field 選「I don't want to use the time filter」**
(這批資料沒有時間欄位)。

**④ 加圖層**:Maps → 新 Map → Add layer → **Configured Tile Map Service**(底圖)→
再 Add layer → **Documents** → data view `geo-test` → geospatial field `geo.location` →
Scaling 用 `Limit results to 10,000` → Add and continue → Tooltip fields 勾 `name`。

**⑤ 應該看到**:底圖上 5 個點,4 個擠在台北、1 個在高雄;點下去 tooltip 顯示地名。
Style → Fill color → `By value` → `name` 可讓 5 個點各自顏色。

**⑥ 不開 UI 的驗證**(Documents 圖層底層就是這種 `_search`):

```bash
curl -s -u elastic:changeme-test-only http://localhost:9200/geo-test/_count            # -> 5
curl -s -u elastic:changeme-test-only -H 'Content-Type: application/json' \
  'http://localhost:9200/geo-test/_search?size=0&track_total_hits=true' -d '{
  "query":{"geo_bounding_box":{"geo.location":{
    "top_left":{"lat":25.3,"lon":121.4},"bottom_right":{"lat":24.9,"lon":121.7}}}}}'   # -> hits.total.value = 4
```

#### 7.5.2 移動軌跡 —— Basic 授權作法(Tracks 用不了)

**先確認 Tracks 真的被擋**(本機 Basic 實測):

```bash
curl -s -u elastic:changeme-test-only -H 'Content-Type: application/json' \
  'http://localhost:9200/track-test/_search?size=0' -d '{
  "aggs":{"e":{"terms":{"field":"entity"},
    "aggs":{"t":{"geo_line":{"point":{"field":"location"},"sort":{"field":"@timestamp"}}}}}}}'
# -> "security_exception ... current license is non-compliant for [geo-line-agg]"
```

**① 建索引 + ② 灌資料**(兩個實體、各 5 個時間遞增的點):

```bash
curl -s -u elastic:changeme-test-only -H 'Content-Type: application/json' \
  -X PUT http://localhost:9200/track-test -d '{
  "mappings":{"properties":{
    "entity":{"type":"keyword"},"@timestamp":{"type":"date"},
    "location":{"type":"geo_point"}
  }}}'

curl -s -u elastic:changeme-test-only -H 'Content-Type: application/json' \
  -X POST 'http://localhost:9200/track-test/_bulk?refresh=wait_for' --data-binary @- <<'EOF'
{"index":{}}
{"entity":"car-A","@timestamp":"2026-09-10T01:00:00Z","location":{"lat":25.0478,"lon":121.5170}}
{"index":{}}
{"entity":"car-A","@timestamp":"2026-09-10T01:05:00Z","location":{"lat":25.0465,"lon":121.5280}}
{"index":{}}
{"entity":"car-A","@timestamp":"2026-09-10T01:10:00Z","location":{"lat":25.0440,"lon":121.5400}}
{"index":{}}
{"entity":"car-A","@timestamp":"2026-09-10T01:15:00Z","location":{"lat":25.0390,"lon":121.5540}}
{"index":{}}
{"entity":"car-A","@timestamp":"2026-09-10T01:20:00Z","location":{"lat":25.0330,"lon":121.5654}}
{"index":{}}
{"entity":"car-B","@timestamp":"2026-09-10T01:02:00Z","location":{"lat":25.0421,"lon":121.5079}}
{"index":{}}
{"entity":"car-B","@timestamp":"2026-09-10T01:08:00Z","location":{"lat":25.0550,"lon":121.5060}}
{"index":{}}
{"entity":"car-B","@timestamp":"2026-09-10T01:14:00Z","location":{"lat":25.0720,"lon":121.4980}}
{"index":{}}
{"entity":"car-B","@timestamp":"2026-09-10T01:20:00Z","location":{"lat":25.0950,"lon":121.4850}}
{"index":{}}
{"entity":"car-B","@timestamp":"2026-09-10T01:26:00Z","location":{"lat":25.1200,"lon":121.4650}}
EOF
```

**作法 A —— Top hits per entity(內建,只顯示點)**

- **③ data view**:`track-test`,Timestamp field 選 `@timestamp`。
- **④ 加圖層**:Add layer → **Top hits per entity** → data view `track-test` →
  geospatial field `location` → **Entity** `entity` → **Sort** `@timestamp` desc →
  documents per entity 例如 5。
- **⑤ 看到**:每個實體最近 5 個點(car-A、car-B 各一組)。`geo_point` 不會自動連線
  (line width 被設 0)。上方時間範圍要涵蓋 `2026-09-10T01:00~01:30Z`,否則沒東西。
- **⑥ 驗證**(這就是該圖層送的查詢):

  ```bash
  curl -s -u elastic:changeme-test-only -H 'Content-Type: application/json' \
    'http://localhost:9200/track-test/_search?size=0' -d '{
    "aggs":{"tracks":{"terms":{"field":"entity","size":10},
      "aggs":{"path":{"top_hits":{"sort":[{"@timestamp":"asc"}],"size":20}}}}}}'
  # -> 2 個 bucket (car-A / car-B),各 5 筆
  ```

**作法 B —— 自算 LineString 存 `geo_shape`(要連成線就用這個)**

寫入端把一條軌跡的座標串成 GeoJSON LineString(`[lon,lat]` 順序),存進 `geo_shape`:

```bash
curl -s -u elastic:changeme-test-only -H 'Content-Type: application/json' \
  -X PUT http://localhost:9200/track-lines -d '{
  "mappings":{"properties":{"entity":{"type":"keyword"},"path":{"type":"geo_shape"}}}}'

curl -s -u elastic:changeme-test-only -H 'Content-Type: application/json' \
  -X POST 'http://localhost:9200/track-lines/_bulk?refresh=wait_for' --data-binary @- <<'EOF'
{"index":{}}
{"entity":"car-A","path":{"type":"LineString","coordinates":[[121.5170,25.0478],[121.5280,25.0465],[121.5400,25.0440],[121.5540,25.0390],[121.5654,25.0330]]}}
{"index":{}}
{"entity":"car-B","path":{"type":"LineString","coordinates":[[121.5079,25.0421],[121.5060,25.0550],[121.4980,25.0720],[121.4850,25.0950],[121.4650,25.1200]]}}
EOF
```

- **③ data view**:`track-lines`(無時間欄位)。
- **④ 加圖層**:Add layer → **Documents** → `track-lines` → geospatial field `path` →
  Style → line width 調粗、line color 依 `entity` 上色。
- **⑤ 看到**:car-A、car-B 各一條折線。要「線 + 每個回報點」就再疊一個作法 A 的點層。

#### 7.5.3 責任區多邊形 —— 上傳 GeoJSON / 直接建 `geo_shape`

UI 作法:Maps → Add layer → **Upload file** → 拖一個 `.geojson`(`FeatureCollection`,
WGS84)進去,Kibana 寫進新索引 + 建 data view。等效的 API 作法(方便自動化):

```bash
curl -s -u elastic:changeme-test-only -H 'Content-Type: application/json' \
  -X PUT http://localhost:9200/zones -d '{
  "mappings":{"properties":{"name":{"type":"keyword"},"geometry":{"type":"geo_shape"}}}}'

curl -s -u elastic:changeme-test-only -H 'Content-Type: application/json' \
  -X POST 'http://localhost:9200/zones/_bulk?refresh=wait_for' --data-binary @- <<'EOF'
{"index":{"_id":"taipei-central"}}
{"name":"Taipei Central Zone","geometry":{"type":"Polygon","coordinates":[[[121.44,24.98],[121.60,24.98],[121.60,25.10],[121.44,25.10],[121.44,24.98]]]}}
{"index":{"_id":"tamsui-north"}}
{"name":"Tamsui North Zone","geometry":{"type":"Polygon","coordinates":[[[121.40,25.12],[121.50,25.12],[121.50,25.22],[121.40,25.22],[121.40,25.12]]]}}
EOF
```

- **③ data view**:`zones`(無時間欄位)。
- **④ 加圖層**:Add layer → **Documents** → `zones` → geospatial field `geometry` →
  Layer settings → Opacity `0.4`(半透明才看得到底圖與下面的點)。放在點層下面(§7.4.1)。
- **⑤ 看到**:兩個矩形責任區疊在底圖上、點位透在上面。
- **⑥ 驗證**(用區界多邊形反查哪些點落在區內 —— 就是 §6.4 geofencing 的 indexed-shape 版):

  ```bash
  curl -s -u elastic:changeme-test-only -H 'Content-Type: application/json' \
    'http://localhost:9200/geo-test/_search?track_total_hits=true' -d '{
    "query":{"geo_shape":{"geo.location":{
      "indexed_shape":{"index":"zones","id":"taipei-central","path":"geometry"},
      "relation":"within"}}}}'
  # -> 3 點 (Taipei 101 / Taipei Main Station / Ximending);Tamsui 落在 tamsui-north 那區
  ```

- **行政區界**:要真的縣市界,把上面的多邊形換成 GADM / 政府開放資料下載的
  台灣縣市 GeoJSON(一樣 upload 或 `_bulk`),再用 **Choropleth** 圖層拿 ES 聚合值對它上色。

#### 7.5.4 一鍵匯入疊好的示範 Map

本機測試已把「底圖 + geo-test 點 + zones 責任區 + track-lines 軌跡」四層疊好、存成一個
Maps saved object 匯出成 `tileserver/local-test-artifacts/stacked-map.ndjson`。在任一台
Kibana(需先有本節的 `geo-test` / `zones` / `track-lines` 索引):

```bash
curl -s -u elastic:changeme-test-only -H 'kbn-xsrf: true' \
  -X POST 'http://localhost:5601/api/saved_objects/_import?overwrite=true' \
  --form file=@tileserver/local-test-artifacts/stacked-map.ndjson
```

→ Kibana → Maps → 開「[ECK 離線底圖] 疊加示範」即可看到四層疊加。
(此 NDJSON 已通過建立 / 匯出 / 重匯入 / 圖層-reference 一致性檢查;實際畫面渲染
也已有瀏覽器截圖佐證,見 `tileserver/local-test-artifacts/kibana-stacked-map.png` /
`kibana-stacked-map-maponly.png`。)

### 7.6 放進 Dashboard

整張 Map(含所有疊加圖層)可 Save 後,在 Dashboard 用 Add panel 加入,與其它視覺化並列;
Dashboard 的時間範圍與 filter 會套到 Map 的 ES 圖層。三個範例圖層 + 底圖存成一張 Map 後,
也可整批用 `POST /api/saved_objects/_export`(`type=map`、`includeReferencesDeep=true`)
匯出 NDJSON、在另一台 Kibana `_import`(§7.5.4)。

### 7.7 目前資料的界線

- OpenFreeMap 這份 `.mbtiles` 是 **OpenMapTiles 向量 schema**(道路 / 水體 / 土地利用 / 建物 /
  地名),**沒有衛星影像、沒有地形高程**。要衛星底圖或 hillshade 疊層需另尋資料來源,不在本方案內。
- EMS 關掉後沒有內建世界 / 行政區界多邊形。要做「依區域比較指標」的 choropleth,需自備區界
  GeoJSON(§7.3.5)。
- **`basic-preview` 樣式的地名標籤是英文/拼音**(已從實測截圖確認,例如顯示 `Tamsui` 而非「淡水」)。
  這件事在 air-gapped 環境**沒辦法事後補救**——要中文地名需要同時改 style 的 `text-field` 設定,
  以及 tileserver 映像 `fonts/` 目錄裡要有涵蓋中文的字型 PBF,兩者都得在 §3 打包映像、
  推進 air-gapped 環境**之前**準備好;封存後才發現字型不夠是修不了的。若需要中文地名,
  建議先確認 `/usr/src/app/node_modules/tileserver-gl-styles/fonts/` 底下的字型涵蓋範圍。
- 底圖的 ODbL 標示義務由 `map.tilemap.options.attribution`(§4.5)負責;ES 疊加層不另產生標示需求。
