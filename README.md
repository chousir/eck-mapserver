# ECK 離線地圖底圖 — Ansible Playbook

把 `ECK離線地圖底圖規劃書.md`(§4)裡手動 `kubectl apply` 的部署流程,改寫成可重複執行、
順序安全的 Ansible playbook:在 k8s 上部署 **tileserver-gl**(讀 OpenFreeMap `.mbtiles`)+
**nginx TLS sidecar**,提供 Kibana Maps 使用的自架底圖。

**範圍只到 tileserver 本身**(Namespace / PV+PVC / ConfigMap ×2 / cert-manager
Certificate+Issuer / Deployment / Service)。Kibana / `eck-stack` 的 Helm values(規劃書
§4.5 `map.tilemap.url`)**不在這個 playbook 裡**,由另一個既有的 elastic-stack 專案處理。

## 這個 playbook 不會做的事

- **不會**下載、build、load、tag、push 任何 image —— 兩個 image
  (`maptiler/tileserver-gl:v5.6.0`、`nginx:1.27-alpine`)要自己 pull 好、推到內部
  registry,playbook 只是引用 image 名稱/tag 的變數。
- **不會**搬運 `tiles.mbtiles`(~95GB)—— 要自己先放到目標主機
  (`k8s-controller01`)上,playbook 只用 `stat` 確認檔案存在、權限正確(必要時可選
  checksum 驗證),沒放好會直接 fail 並印出明確訊息,不會嘗試傳檔案。
- **不會**安裝 cert-manager —— 視為叢集已具備的前提條件,只在 preflight 檢查
  CRD 是否存在。

## 目錄結構

```
eck-map/
├── ECK離線地圖底圖規劃書.md          原始規劃文件(來源依據)
├── ansible.cfg
├── site.yml                        入口 playbook
├── inventory/
│   ├── hosts                       目標主機(預設 group: k8s_controller)
│   └── group_vars/all.yml          環境相關變數 ── 部署前必看
└── roles/kubectl/eck-map/
    ├── defaults/main.yml           完整變數清單與預設值
    ├── handlers/main.yml           ConfigMap/Certificate 變更時 rollout restart
    ├── tasks/
    │   ├── main.yml                串接 preflight → apply → verify
    │   ├── preflight.yml           變數檢查、kubectl 連線、cert-manager、mbtiles 檔案/權限
    │   ├── render_apply.yml        依序 render + kubectl apply(順序不可打亂)
    │   └── verify.yml              部署後唯讀檢查(rollout status、log、tile smoke test)
    └── templates/                  7 份 k8s manifest(Jinja2),依 apply 順序編號
        ├── 00-namespace.yml.j2
        ├── 05-issuer.yml.j2        SelfSigned Issuer(僅測試連通性用)
        ├── 10-certificate.yml.j2
        ├── 15-nginx-config.yml.j2
        ├── 20-config.yml.j2        tileserver-gl config.json(必要,見規劃書 §2)
        ├── 30-pv.yml.j2            PersistentVolume + PersistentVolumeClaim
        └── 40-deployment.yml.j2    Deployment(tileserver + nginx-tls)+ Service
```

## 前置需求(playbook 假設已具備,不會幫你裝)

- `k8s-controller01` 可 SSH 連線,且該主機上 `kubectl` 已在 PATH、kubeconfig 可用。
- 叢集已安裝 **cert-manager**。
- 兩個 image 已推到你設定的內部 registry(`eck_map_image_registry` 等變數)。
- `tiles.mbtiles` 已放在該主機上(預設路徑 `~/openfreemap/tiles.mbtiles`,可用變數
  `eck_map_mbtiles_path` 覆寫),且該檔案與其所在目錄要 **other 可讀/可執行**
  (tileserver-gl 容器以非 root UID 執行,權限不足會 `EACCES`)。

## 使用方式

務必在專案根目錄(`eck-map/`)下執行,`ansible.cfg` 只會從目前工作目錄自動載入:

```bash
cd eck-map

# 1. 編輯 inventory/hosts:若 k8s-controller01 的 SSH 位址/帳號不同,補上
#    ansible_host= / ansible_user=

# 2. 編輯 inventory/group_vars/all.yml:
#    - eck_map_tile_vip 必填,沒有預設值(MetalLB 分配的 IP)
#    - image registry/repo/tag 改成實際推上去的值
#    - 確認 eck_map_mbtiles_path 與實際放檔案的路徑一致

# 3. 先跑 preflight,不會碰叢集,只做檢查
ansible-playbook site.yml --tags preflight

# 4. 正式部署(preflight → apply → verify 全部執行)
ansible-playbook site.yml

# 之後可單獨重跑
ansible-playbook site.yml --tags apply     # 只重新 apply manifests
ansible-playbook site.yml --tags verify    # 只重新跑部署後檢查
```

`ansible-playbook --check` 對這個 role 沒有效果(刻意不依賴 `kubernetes.core`
collection,全部走 raw `kubectl apply`)。要 dry-run,手動對 render 出來的 manifest
跑:`kubectl apply -f ~/.eck-map/manifests --dry-run=server`。

## 關鍵變數(完整清單見 `roles/kubectl/eck-map/defaults/main.yml`)

| 變數 | 預設值 | 說明 |
|---|---|---|
| `eck_map_tile_vip` | `""`(**必填**) | Certificate 的 `ipAddresses` SAN,也是 Service LoadBalancer 應該拿到的 IP |
| `eck_map_mbtiles_path` | `~/openfreemap/tiles.mbtiles` | 目標主機上的檔案路徑;PV 的 `local.path`(取目錄)與 config.json 的檔名都從這個變數推導,只需改一處 |
| `eck_map_mbtiles_sha256` | `""` | 選填,設定才會做 checksum(讀完整 95GB,較慢,建議只在放檔案後跑一次) |
| `eck_map_pv_storage_size` | `150Gi` | 建議 ≥1.2× 實際檔案大小 |
| `eck_map_image_registry` / `..._tileserver_image_*` / `..._nginx_image_*` | `registry.internal:5000/...` | 只是引用,不會幫你 push |
| `eck_map_cert_manager_create_selfsigned_issuer` | `true` | 正式環境務必改 `false`,並設定 `eck_map_cert_issuer_name`/`_kind` 指向內部 CA 的 ClusterIssuer |
| `eck_map_k8s_node_hostname` | `{{ inventory_hostname }}` | k8s 節點的 `kubernetes.io/hostname` label 值,與 SSH inventory 別名分開,避免兩者未來不同步 |
| `eck_map_kubectl_bin` / `_context` / `_kubeconfig` | `kubectl` / 空 | PATH 或 kubeconfig 非預設時的逃生口 |

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
