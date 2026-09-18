# ECK 離線地圖底圖 — Ansible Playbook

把 `ECK離線地圖底圖規劃書.md`(§4)裡手動 `kubectl apply` 的部署流程,改寫成可重複執行、
順序安全的 Ansible playbook:在 k8s 上部署 **tileserver-gl**(讀 OpenFreeMap `.mbtiles`)+
**nginx TLS sidecar**,提供 Kibana Maps 使用的自架底圖。

playbook 本體在 **`eck-map-playbook/`** 子目錄下(不是這個 repo 的根目錄)。

**範圍只到 tileserver 本身**(Namespace / PV+PVC / ConfigMap ×2 / cert-manager
Certificate+Issuer / Deployment / Service)。Kibana / `eck-stack` 的 Helm values(規劃書
§4.5 `map.tilemap.url`)**不在這個 playbook 裡**,由另一個既有的 elastic-stack 專案處理。

這個環境是 **air-gapped**(叢集連不到外網),所以以下三件事 playbook 完全不碰,必須在
執行前手動做完:

1. 兩個 image 要自己搬進內部 registry。
2. `tiles.mbtiles`(~95GB)要自己搬到目標主機上,並且權限要對。
3. cert-manager 要叢集裡已經裝好(裝 cert-manager 本身在 air-gap 下也是一次鏡像搬運,
   不在本 playbook 範圍)。

沒做完這三件事,`ansible-playbook site.yml` 的 preflight 階段會直接 fail 並印出明確訊息。

## Air-gap 前置作業(執行 playbook 前必做)

### 1. 把兩個 image 搬進內部 registry

需要的 image(版本見 `roles/kubectl/eck-map/defaults/main.yml`):

- `maptiler/tileserver-gl:v5.6.0`
- `nginx:1.27-alpine`

在一台**能連外網**的機器上:

```bash
docker pull maptiler/tileserver-gl:v5.6.0
docker pull nginx:1.27-alpine
docker save -o images.tar \
  maptiler/tileserver-gl:v5.6.0 \
  nginx:1.27-alpine
# 把 images.tar 搬進 air-gap 網段(隨身碟/內部傳檔管道)
```

在 air-gap 網段內、能推 registry 的機器上:

```bash
docker load -i images.tar

docker tag maptiler/tileserver-gl:v5.6.0 registry.internal:5000/maptiler/tileserver-gl:v5.6.0
docker tag nginx:1.27-alpine            registry.internal:5000/nginx:1.27-alpine

docker push registry.internal:5000/maptiler/tileserver-gl:v5.6.0
docker push registry.internal:5000/nginx:1.27-alpine
```

(`registry.internal:5000`只是預設值,實際 registry 位址/repo 名稱不同就對應改
`inventory/group_vars/all.yml` 裡的 `eck_map_image_registry` 等變數 —— 見下方變數表。
也可以用 `skopeo copy` 取代 `docker save/load`。)

### 2. 把 `tiles.mbtiles` 放到目標主機上

- 檔案(~95GB)要自己搬到 `k8s-controller01` 上,playbook **不會**幫忙傳檔案。
- 預設路徑是 `~/openfreemap/tiles.mbtiles`(SSH 連線帳號的家目錄下);路徑不同就設定
  `eck_map_mbtiles_path`。
- 檔案跟其所在目錄都要 **other 可讀**(檔案 `o+r`、目錄 `o+rx`),因為 tileserver-gl
  容器以非 root UID 執行:

  ```bash
  chmod o+r  ~/openfreemap/tiles.mbtiles
  chmod o+rx ~/openfreemap
  ```

- 如果主機是 SELinux enforcing,即使上面權限對了,家目錄路徑(`user_home_t`)還是可能
  被擋。需要的話:`chcon -Rt container_file_t <目錄>`,或乾脆把檔案放到
  `/var/lib/...` 之類本來就對容器開放的路徑。
- 建議搬檔案後跑一次 SHA256 驗證(見下面 `eck_map_mbtiles_sha256`),確認搬檔過程沒有
  損毀 —— 這台 playbook 的 preflight 可以幫你比對,但**只在你設定了這個變數時才會做**
  (讀完整 95GB,平時重跑不要開)。

### 3. 確認 cert-manager 已安裝

playbook 只在 preflight 檢查 `certificates.cert-manager.io` 這個 CRD 存不存在,**不會**
幫你安裝。air-gap 下要自己先把 cert-manager 的 image 也搬進內部 registry、部署好。

## 目錄結構

```
eck-map/                             這個 repo 的根目錄
├── ECK離線地圖底圖規劃書.md          原始規劃文件(來源依據)
└── eck-map-playbook/                playbook 本體 —— 所有指令都要在這裡執行
    ├── ansible.cfg
    ├── site.yml                     入口 playbook
    ├── inventory/
    │   ├── hosts                    目標主機(預設 group: k8s_controller)
    │   └── group_vars/all.yml       環境相關變數 ── 部署前必看
    └── roles/kubectl/eck-map/
        ├── defaults/main.yml        完整變數清單與預設值
        ├── handlers/main.yml        ConfigMap/Certificate 變更時 rollout restart
        ├── tasks/
        │   ├── main.yml             串接 preflight → apply → verify
        │   ├── preflight.yml        變數檢查、kubectl 連線、cert-manager、mbtiles 檔案/權限
        │   ├── render_apply.yml     依序 render + kubectl apply(順序不可打亂)
        │   └── verify.yml           部署後唯讀檢查(rollout status、log、tile smoke test)
        └── templates/               7 份 k8s manifest(Jinja2),依 apply 順序編號
            ├── 00-namespace.yml.j2
            ├── 05-issuer.yml.j2     SelfSigned Issuer(僅測試連通性用)
            ├── 10-certificate.yml.j2
            ├── 15-nginx-config.yml.j2
            ├── 20-config.yml.j2     tileserver-gl config.json(必要,見規劃書 §2)
            ├── 30-pv.yml.j2         PersistentVolume + PersistentVolumeClaim
            └── 40-deployment.yml.j2 Deployment(tileserver + nginx-tls)+ Service
```

## 前置需求(playbook 假設已具備,不會幫你裝)

- `k8s-controller01` 可 SSH 連線,且該主機上 `kubectl` 已在 PATH、kubeconfig 可用。
- 叢集已安裝 **cert-manager**(見上方 air-gap 前置作業 §3)。
- 兩個 image 已推到你設定的內部 registry(`eck_map_image_registry` 等變數,見上方
  air-gap 前置作業 §1)。
- `tiles.mbtiles` 已放在該主機上並權限正確(見上方 air-gap 前置作業 §2)。

## 使用方式

務必在 **`eck-map-playbook/`** 目錄下執行,`ansible.cfg`(裡面的相對路徑
`inventory = inventory/hosts`)只會從目前工作目錄自動載入:

```bash
cd eck-map-playbook

# 1. 編輯 inventory/hosts:若 k8s-controller01 的 SSH 位址/帳號不同,補上
#    ansible_host= / ansible_user=

# 2. 編輯 inventory/group_vars/all.yml:
#    - eck_map_tile_vip 必填,沒有預設值(MetalLB 分配的 IP)
#    - image registry/repo/tag 改成實際推上去的值(對應 air-gap 前置作業 §1)
#    - 確認 eck_map_mbtiles_path 與實際放檔案的路徑一致(對應 air-gap 前置作業 §2)

# 3. 先跑 preflight,不會碰叢集,只做檢查
ansible-playbook site.yml --tags preflight

# 4. 正式部署(preflight → apply → verify 全部執行)
ansible-playbook site.yml
```

之後可單獨重跑局部階段,但要注意 `preflight` 這個 tag 掛在 `[preflight, always]`
上,**每次執行都會先跑一次**(包含變數檢查、kubectl 連線、mbtiles stat —— 如果
`eck_map_mbtiles_sha256` 有設定,連 SHA256 都會重算一次,~95GB 會很慢,所以平常
重跑請保持這個變數空白):

```bash
ansible-playbook site.yml --tags apply     # preflight 之後只重新 apply manifests
ansible-playbook site.yml --tags verify    # preflight 之後只重新跑部署後檢查
```

`ansible-playbook --check` 對這個 role 沒有效果(刻意不依賴 `kubernetes.core`
collection,全部走 raw `kubectl apply`)。要 dry-run,`ansible-playbook site.yml
--tags apply` 之後,manifest 會 render 在**目標主機**(`k8s-controller01`)上的
`~/.eck-map/manifests`(即 `eck_map_remote_manifest_dir`),要 SSH 上去對它跑:
`kubectl apply -f ~/.eck-map/manifests --dry-run=server`。

## 關鍵變數(完整清單見 `eck-map-playbook/roles/kubectl/eck-map/defaults/main.yml`)

依環境不同,最常需要調整的變數:

| 變數 | 預設值 | 說明 |
|---|---|---|
| `eck_map_tile_vip` | `""`(**必填**) | Certificate 的 `ipAddresses` SAN,也是 Service LoadBalancer 應該拿到的 IP |
| `eck_map_mbtiles_path` | `~/openfreemap/tiles.mbtiles` | 目標主機上的檔案路徑;PV 的 `local.path`(取目錄)與 config.json 的檔名都從這個變數推導,只需改一處 |
| `eck_map_mbtiles_sha256` | `""` | 選填,設定才會做 checksum(讀完整 95GB,較慢,建議只在放檔案後跑一次) |
| `eck_map_pv_storage_size` | `150Gi` | 建議 ≥1.2× 實際檔案大小 |
| `eck_map_image_registry` / `..._tileserver_image_*` / `..._nginx_image_*` | `registry.internal:5000/...` | 只是引用,不會幫你 push,對應 air-gap 前置作業 §1 |
| `eck_map_image_pull_policy` | `IfNotPresent` | air-gap 環境務必保持這個值;改成 `Always` 會導致 kubelet 嘗試連外網 registry 失敗 |
| `eck_map_cert_manager_create_selfsigned_issuer` | `true` | 正式環境務必改 `false`,並設定 `eck_map_cert_issuer_name`/`_kind` 指向內部 CA 的 ClusterIssuer |
| `eck_map_k8s_node_hostname` | `{{ inventory_hostname }}` | k8s 節點的 `kubernetes.io/hostname` label 值,與 SSH inventory 別名分開,避免兩者未來不同步;PV 的 `nodeAffinity` 與 Deployment 的 `nodeSelector` 都用這個值 |
| `eck_map_tolerations` | pin 在 `node-role.kubernetes.io/control-plane:NoSchedule` | 如果目標節點不是 control-plane(或是舊版叢集用 `node-role.kubernetes.io/master` 這個 key),要改成對應的 taint key,不然 Pod 會一直 Pending |
| `eck_map_namespace` | `gis` | 跟叢集裡既有 namespace 撞名就要改 |
| `eck_map_service_type` / `eck_map_service_port` | `LoadBalancer` / `443` | 沒有 MetalLB 或其他 LB 實作的叢集要改成 `NodePort` 或其他值 —— 但注意 `verify.yml` 裡等 LB IP、比對 `eck_map_tile_vip`、HTTPS smoke test 這幾步都是 `when: eck_map_service_type == 'LoadBalancer'` 才跑,改了 type 這些檢查會被跳過,要自己手動驗證 |
| `eck_map_tileserver_requests_memory` / `_cpu` / `_limits_memory` | `1Gi` / `500m` / `2Gi` | 依節點實際資源調整,資源太緊會 OOMKilled 或排程不上去 |
| `eck_map_cert_wait_timeout` / `eck_map_rollout_status_timeout` / `eck_map_lb_ip_retries` / `_delay` | `120s` / `180s` / `12` / `5` | 叢集/映像拉取較慢時可以調大,避免 apply/verify 階段誤判失敗 |
| `eck_map_kubectl_bin` / `_context` / `_kubeconfig` | `kubectl` / 空 | PATH 或 kubeconfig 非預設時的逃生口(`_kubeconfig` 用絕對路徑,argv 不會展開 `~`)|

## 已知限制 / 風險

- **SELinux**:若 `k8s-controller01` 是 enforcing,家目錄路徑(`user_home_t`)可能被擋,
  即使檔案權限正確。preflight 目前只檢查 DAC 權限位元,抓不到 SELinux 問題;需要的話用
  `chcon -Rt container_file_t <目錄>`,或改用 `/var/lib/...` 之類的路徑。
- **單節點 pin 死**:PV 的 `nodeAffinity` 與 Deployment 的 `nodeSelector` 都固定在放
  `tiles.mbtiles` 的那台節點(照規劃書設計)。調高 `eck_map_replicas` 不會有 HA 效果,
  所有 replica 還是跟同一台節點共存亡。
- **PV 建立後欄位幾乎不可變**:之後改 `eck_map_mbtiles_path`/`eck_map_pv_storage_size`,
  `kubectl apply` 不會幫你搬移/調整已綁定的 `local` PV,要手動刪掉重建(不會遺失資料,
  `Retain` policy 且只是參照外部檔案)。
- 自動化的 smoke test(`verify.yml` 的 HTTPS/`styles.json` 檢查)只證明伺服器本身正常
  運作,**不證明瀏覽器會信任這張憑證**(尤其預設的 SelfSigned Issuer)。瀏覽器信任鏈要
  照規劃書 §6.5 手動確認。

## 參考

原始規劃與所有已驗證細節(檔案大小/SHA256、config.json 為何必要、mixed content 問題、
TLS SAN 要用 IP 不是 DNS 等)見 `ECK離線地圖底圖規劃書.md`。
