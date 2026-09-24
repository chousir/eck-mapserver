# ECK 離線地圖底圖 — Ansible Playbook

在 air-gapped 的 k8s 叢集裝好 ECK 之後，用這個 playbook 一次部署 Kibana Maps 用的自架底圖：
**tileserver-gl**（讀 OpenFreeMap 全球 `.mbtiles`）加上 **nginx TLS sidecar**。部署完成後，
在 Kibana 設定兩行（見下方「ECK 部署時要加入的 Kibana 設定」），並讓使用者的瀏覽器信任 CA，
就能在 Kibana Maps 使用底圖。

背景與所有已驗證的細節請見 `ECK離線地圖底圖規劃書.md`。playbook 本體在 **`eck-map-playbook/`**。

## 版本對應（本案驗證組合）

| 元件 | 版本 | 備註 |
|---|---|---|
| Elasticsearch / Kibana | **9.4.2** | |
| ECK operator | **3.4** | |
| eck-stack Helm chart | 0.19.1 | Kibana 設定放在 `eck-kibana.config` |
| kubespray | v2.31.0 | 叢集部署工具 |
| cert-manager | v1.15.3 | kubespray v2.31.0 預設版本（`cert_manager_enabled: true`） |
| MetalLB | v0.13.9 | kubespray v2.31.0 預設版本；固定 IP 用 `metallb.universe.tf/loadBalancerIPs` annotation |
| Ansible | ansible 11.13.0（ansible-core 2.18） | 與 kubespray v2.31.0 的 `requirements.txt` 相同，可直接沿用 kubespray 的 venv；不需額外 collection |
| tileserver-gl | v5.6.0 | `config.json` 的內建樣式路徑綁定此版本，**勿任意升級** |
| nginx（TLS sidecar） | 1.30-alpine | |
| OpenFreeMap mbtiles | `20260810_211301_pt` | ~95GB，SHA256 見規劃書 §1.1 |
| 打包環境 | Debian 13 (amd64) | 用來 pull/save image 的連外機器 |

> cert-manager / MetalLB 版本是 kubespray v2.31.0 的預設值；若 kubespray inventory 有另外覆寫，
> 以實際部署的版本為準。

## 環境前提（固定，playbook 不另外檢查）

本案環境都是用 kubespray v2.31.0 部署的，以下項目視為固定、已經準備好，所以 playbook 直接使用，
不做額外檢查：

- `k8s-controller01` 上有 `kubectl`，kubeconfig 在 `/root/.kube/config`（kubespray 預設）。playbook 以
  `become: true` 執行，SSH 帳號必須是 root 或可以 sudo。
- MetalLB 已安裝，且有可用的 IP pool（Service 固定是 `LoadBalancer`）。
- cert-manager 已安裝，而且有 CA 型 ClusterIssuer `ca-issuer`（`spec.ca.secretName: ca-key-pair`）。
- 兩個 image 可以透過 Nexus 的 containerd mirror 拉取（見「Air-gap 前置作業」§1）。
- render 出來的 manifest 放在 `k8s-controller01` 的 `/etc/k8s/eck-map/`。

## 部署前必須修改的參數

只有 `inventory/hosts`：`k8s-controller01` 的 `ansible_host=` / `ansible_user=`（SSH 位址或帳號跟主機名稱
不同時才需要加）。其他都已經有符合本案環境的預設值。

視情況才需要修改（`inventory/group_vars/all.yml`）：

| 參數 | 什麼時候要改 |
|---|---|
| `eck_map_tile_vip` | 預設留空，由 MetalLB 從 pool 自動分配 IP。想固定 IP、或想在跑 playbook 之前就先寫好 Kibana 設定時才要填（見下方「Service IP」） |
| `eck_map_mbtiles_sha256` | 搬完檔案後想驗證一次時才填（會讀完整的 95GB） |
| `eck_map_mbtiles_path` | 檔案沒放在預設的 `/var/lib/tileserver/tiles.mbtiles` 時 |
| `eck_map_image_registry` | 預設留空（用原始 image 名稱，透過 Nexus mirror 拉取）。只有 image 放在不同前綴下時才要填 |
| `eck_map_k8s_node_hostname` | inventory 名稱跟 k8s node name（`kubernetes.io/hostname`）不同時 |

## ECK 部署時要加入的 Kibana 設定

playbook 跑完最後會印出實際的 `map.tilemap.url`，照著填入即可。Kibana 伺服器端**不需要**
額外信任任何憑證，因為底圖圖磚是**使用者瀏覽器**直接向 tileserver 取得的（見下一節）。

**Kibana CR（直接用 ECK）：**

```yaml
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: kibana
spec:
  version: 9.4.2
  config:
    map.includeElasticMapsService: false      # air-gap 連不到 EMS，關掉避免 Maps 一直等到逾時
    map.tilemap.url: "https://<tile-IP>/styles/basic-preview/{z}/{x}/{y}.png"
    map.tilemap.options.minZoom: 0
    map.tilemap.options.maxZoom: 18           # 資料 maxzoom=14，tileserver 會自動放大 (over-zoom)
    map.tilemap.options.attribution: >-       # ODbL 授權要求的標示，必填
      &copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a>
      contributors &copy; <a href="https://www.openmaptiles.org/">OpenMapTiles</a>
      tiles by <a href="https://openfreemap.org">OpenFreeMap</a>
  # 選配：讓 Kibana 也使用同一個 CA 簽發的憑證，使用者只需匯入一次 CA（見下一節）
  # http:
  #   tls:
  #     certificate:
  #       secretName: kibana-http-tls
```

**eck-stack Helm values：** 內容相同，`config:` 放在 `eck-kibana.config`（選配的 `http:` 放在
`eck-kibana.http`）。

套用設定後，在 Kibana → Maps → Add layer → **Configured Tile Map Service** 加入底圖圖層。
關掉 EMS 後，新開的 Map 預設是空白的，這是正常現象。

## TLS 與瀏覽器信任

**為什麼一定要處理**：Kibana 走 HTTPS，所以底圖也必須是 HTTPS（否則會被當成 mixed content
擋掉）。而瀏覽器載入圖磚時，遇到**不受信任的憑證不會跳出警告，會直接失敗**，結果就是底圖整片
空白。所以真正要信任 CA 的是**使用者的瀏覽器/作業系統**，不是 Kibana。

**本案的 issuer**：`ca-issuer` 是 CA 型 ClusterIssuer（`spec.ca.secretName: ca-key-pair`），它簽出的
憑證都鏈到同一張 CA，所以使用者只要匯入這張 CA 一次即可。不要改成 `selfSigned` 型的 issuer：
那種 issuer 簽出的每張憑證都是自己簽自己，沒有固定的 CA 可以匯入，底圖會是空白。

**取得 CA 並匯入瀏覽器**：playbook 跑完後，會把憑證 Secret 裡的 `ca.crt` 複製到 Ansible 控制端的
`eck-map-playbook/tileserver-ca.crt`（變數 `eck_map_ca_export_path`）。請把它匯入使用者電腦：
Windows 用 GPO 或 `certutil -addstore -f Root tileserver-ca.crt`；Firefox 使用自己的憑證庫，
需要另外匯入，或啟用 `security.enterprise_roots.enabled`。

**讓 Kibana 也使用同一個 CA（建議做）**：在 ES/Kibana 所在的 namespace 用同一個 ClusterIssuer
簽一張 Kibana 的憑證，然後在 Kibana CR 指定 `spec.http.tls.certificate.secretName`（見上一節）。
這樣使用者只要匯入一次 CA，Kibana 本身和底圖就都會被信任。

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: kibana-http
  namespace: <elastic namespace>
spec:
  secretName: kibana-http-tls
  issuerRef: { name: ca-issuer, kind: ClusterIssuer }
  ipAddresses: ["<Kibana-VIP>"]
```

**為什麼不直接用 ECK 自己的 CA**：ECK 為每個 Kibana/ES 各自產生一張自簽 CA
（`<name>-kb-http-ca-internal`），預設每年自動輪替。用它來簽 tileserver 憑證，得把 CA 私鑰
複製到 cert-manager 的 namespace，而且每次輪替都要重做，使用者也要重新匯入。統一改用
cert-manager 的 CA，會比較好管理。

**憑證續簽**：`eck_map_cert_duration` 預設是 `8760h`，可以調長，但不能超過 CA 本身的效期。
cert-manager 續簽後會更新 Secret，nginx sidecar 每 6 小時（`eck_map_nginx_reload_interval_seconds`）
會自動 `nginx -s reload` 一次，所以不需要手動重啟。

## Service IP（自動分配或固定）

- **`eck_map_tile_vip` 留空（預設）**：playbook 會先建立 Service，等 MetalLB 分配 IP，再用這個 IP
  簽憑證（SAN），最後印出這個 IP 和對應的 `map.tilemap.url`。
- **要注意的地方**：Kibana 的 `map.tilemap.url` 和憑證都綁定這個 IP。只要 Service 沒被刪除，MetalLB
  就會一直保留同一個 IP，重跑 playbook 不會變。但如果 Service 或 namespace 被刪掉重建，IP 可能會
  改變，這時 playbook 會自動重簽憑證，**但 Kibana 設定必須手動修改**。
- **建議**：第一次部署後，把印出來的 IP 填回 `eck_map_tile_vip`，用 annotation 固定下來。之後即使
  重建也會拿到同一個 IP（前提是這個 IP 仍在 MetalLB 的 pool 裡，且沒被其他 Service 使用）。
- 如果希望在跑 playbook 之前就先把 Kibana 設定寫進 ECK，一開始就填入 `eck_map_tile_vip` 即可。

## Air-gap 前置作業（執行 playbook 前必須完成）

playbook 不會搬運任何檔案或 image。第 2 項沒做完，preflight 會直接 fail 並印出原因；第 1 項請用下面的 `crictl pull` 自行確認。

### 1. 確認節點拉得到兩個 image

本案的內部 Nexus docker registry 前面有 nginx 反向代理，而且 kubespray 已把 containerd 的
registry mirror（`containerd_registries_mirrors`）指向它，所以節點可以直接用**原始 image 名稱**拉取，
不需要重新 tag，也不需要設定 `eck_map_image_registry`：

- `maptiler/tileserver-gl:v5.6.0`
- `nginx:1.30-alpine`

只要確認這兩個 image 已經在 Nexus 裡（照平常放 image 進 Nexus 的方式處理；需要從外網打包的話，
在 Debian 13 (amd64) 上 `docker pull` + `docker save`）。部署前在 `k8s-controller01` 上先試拉一次：

```bash
sudo crictl pull docker.io/maptiler/tileserver-gl:v5.6.0
sudo crictl pull docker.io/library/nginx:1.30-alpine
```

兩個都成功，Pod 就不會卡在 `ImagePullBackOff`。如果 image 是放在另一個前綴下（例如
`registry.internal:5000/nginx:1.30-alpine`），再把 `eck_map_image_registry` 設成該前綴即可。

### 2. 把 `tiles.mbtiles` 放到 `k8s-controller01`

```bash
sudo mkdir -p /var/lib/tileserver
# 把 tiles.mbtiles 放進 /var/lib/tileserver/
sudo chmod o+rx /var/lib/tileserver
sudo chmod o+r  /var/lib/tileserver/tiles.mbtiles
```

- tileserver-gl 容器以非 root 身分執行，所以檔案必須 other 可讀、目錄必須 other 可進入。
- 整個目錄會以唯讀方式掛進容器，請不要放其他檔案。
- 建議搬完後做一次 SHA256 驗證：暫時設定 `eck_map_mbtiles_sha256`，preflight 就會比對（會讀完整的
  95GB，平常重跑時請保持空白）。

## 使用方式

請在 **`eck-map-playbook/`** 目錄下執行（`ansible.cfg` 只會從目前的工作目錄自動載入）：

```bash
cd eck-map-playbook
ansible-playbook site.yml
```

playbook 依序執行：

1. **preflight**：確認 `tiles.mbtiles` 存在、權限正確（有設定 `eck_map_mbtiles_sha256` 時也會比對 checksum）。
2. **deploy**：套用 `00-service.yml` → 等 MetalLB 分配 IP → 用這個 IP 套用 `10-config.yml`（憑證 +
   ConfigMap）→ 等憑證簽發完成 → 套用 `20-workload.yml`（PV/PVC + Deployment）。
3. **verify**：等 rollout 完成，檢查 `https://<IP>/styles.json` 有 `basic-preview`，把 `ca.crt` 匯出到
   `eck-map-playbook/tileserver-ca.crt`，並印出要填進 Kibana 的設定。

最後把印出的 `map.tilemap.url` 寫進 ECK 的 Kibana 設定，並把 `tileserver-ca.crt` 匯入使用者的瀏覽器。

重跑也是同一個指令。設定有變動時，Deployment 的 `eck-map/config-hash` annotation 會跟著改變，Pod 就會
自動滾動更新。如果要 dry-run，在 `k8s-controller01` 上執行
`kubectl apply -f /etc/k8s/eck-map/ --dry-run=server`（要先跑過一次 playbook）。

## 目錄結構

```
eck-map/
├── ECK離線地圖底圖規劃書.md          原始規劃文件（背景與驗證細節）
└── eck-map-playbook/                所有指令都在這裡執行
    ├── ansible.cfg
    ├── site.yml
    ├── inventory/
    │   ├── hosts                    目標主機（group: k8s_controller）
    │   └── group_vars/all.yml       選填參數
    └── roles/kubectl/eck-map/
        ├── defaults/main.yml        全部變數與預設值
        ├── tasks/
        │   ├── main.yml             preflight → deploy → verify
        │   ├── preflight.yml        tiles.mbtiles 檔案與權限
        │   ├── deploy.yml           render + kubectl apply（順序不可打亂）
        │   └── verify.yml           rollout、HTTPS smoke test、匯出 ca.crt、印出 Kibana 設定
        └── templates/               依 apply 順序編號
            ├── 00-service.yml.j2    Namespace + Service（先建立，拿到 IP 才能簽憑證）
            ├── 10-config.yml.j2     Certificate + nginx.conf + tileserver config.json
            └── 20-workload.yml.j2   PV/PVC + Deployment（tileserver + nginx-tls）
```

## 全部變數（`roles/kubectl/eck-map/defaults/main.yml`）

| 變數 | 預設值 | 說明 |
|---|---|---|
| `eck_map_namespace` | `gis` | |
| `eck_map_manifest_dir` | `/etc/k8s/eck-map` | render 後 manifest 在目標主機上的位置 |
| `eck_map_mbtiles_path` | `/var/lib/tileserver/tiles.mbtiles` | PV 的 `local.path`（取目錄）與 config.json 的檔名都由此推導 |
| `eck_map_mbtiles_sha256` | `""` | 選填，設定後才會做 checksum |
| `eck_map_pv_storage_size` | `150Gi` | 建議 ≥ 實際檔案大小的 1.2 倍 |
| `eck_map_image_registry` | `""` | 空值代表用原始 image 名稱；有值時會加在 image 名稱前面 |
| `eck_map_cert_issuer_name` | `ca-issuer` | CA 型 ClusterIssuer |
| `eck_map_cert_duration` | `8760h` | 憑證效期，不能超過 CA 的效期 |
| `eck_map_ca_export_path` | `eck-map-playbook/tileserver-ca.crt` | 在控制端匯出的 CA |
| `eck_map_tile_vip` | `""` | 空值代表由 MetalLB 自動分配；有值代表用 annotation 固定 |
| `eck_map_k8s_node_hostname` | `{{ inventory_hostname }}` | PV nodeAffinity 與 Deployment nodeSelector 使用 |
| `eck_map_tileserver_resources` | requests `1Gi`/`500m`、limits `2Gi` | 依節點資源調整 |

其他固定值（資源名稱、port 443/8443/8080、control-plane toleration、nginx 每 6 小時 reload、各種等待
時間）直接寫在 template 和 task 裡。

## 已知限制

- **固定在單一節點**：PV 與 Pod 都固定在放 `tiles.mbtiles` 的那台節點，沒有 HA。
- **PV 建立後幾乎不能修改**：之後要改 `eck_map_mbtiles_path` 或 `eck_map_pv_storage_size`，必須手動
  刪除 PV/PVC 再重建（`Retain`，不會遺失資料）。
- **smoke test 不驗證瀏覽器信任**：`verify.yml` 使用 `validate_certs: false`，只能證明服務正常。
  瀏覽器是否信任憑證，請依規劃書 §6.5 手動確認。
