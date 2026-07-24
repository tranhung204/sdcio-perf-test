# Bộ kiểm thử SDCIO theo Test.xlsx

## 1. Phạm vi

Bộ này chuẩn bị dữ liệu và step cho 4 testcase trong `Test.xlsx`:

- TC01: nạp 300.000 **prefix entry hợp lệ theo YANG SR Linux** vào `dev1`.
- TC02: thay đổi 1% dữ liệu trên Git và đo thời gian tự đồng bộ.
- TC03: dùng `ConfigSet` áp dụng cùng 300.000 prefix entry lên `dev1` và `dev2`.
- TC04: thay đổi 1% dữ liệu cho hai thiết bị.
- TC04-FAILURE: pause `dev2` để kiểm tra có partial commit hay không.

Dữ liệu dùng `routing-policy/prefix-set`. Mỗi dataset có 300 object, mỗi object chứa 1.000 prefix; tổng cộng 300.000 prefix entry. V1 dùng dải `10.0.0.0/8`; V2 đổi 1% entry sang dải `11.0.0.0/8`.

## 2. Lưu ý quan trọng

1. SDCIO cung cấp `Config` và `ConfigSet`; việc tự kéo YAML từ Git cần một GitOps controller như Argo CD hoặc Flux. Đây không phải chức năng Git watcher tích hợp sẵn của SDCIO.
2. `ConfigSet` chọn nhiều target theo label, nhưng tài liệu công khai không cam kết distributed atomic commit giữa nhiều thiết bị. Vì vậy phải chạy bài failure injection; chỉ test cả hai thiết bị cùng thành công chưa chứng minh atomicity.
3. Mục tiêu 300.000 entry trong 2 phút và update trong 30 giây là mục tiêu benchmark của dự án, không phải thông số được SDCIO công bố. Cần ghi cấu hình CPU/RAM, số replica, tốc độ disk và GitOps polling khi báo cáo.
4. Trước khi chạy 300k, bắt buộc chạy dataset `smoke_dev1_1000`.

Nguồn:
- https://docs.sdcio.dev/user-guide/configuration/config/config/
- https://docs.sdcio.dev/user-guide/configuration/config/configset/
- https://docs.sdcio.dev/user-guide/configuration/discovery/addresses/
- https://docs.sdcio.dev/user-guide/configuration/target-profiles/sync-profile/

## 3. Setup hai router

Việc redeploy sẽ xóa container `dev1` hiện tại. SDCIO có thể reapply intent sau khi target trở lại.

```bash
cd 00_environment
sudo containerlab destroy -t basic-usage-2nodes.clab.yaml --cleanup || true
sudo containerlab deploy -t basic-usage-2nodes.clab.yaml
ping -c 2 172.21.0.200
ping -c 2 172.21.0.201
nc -zvw5 172.21.0.200 57400
nc -zvw5 172.21.0.201 57400
```

Apply DiscoveryRule hai thiết bị:

```bash
kubectl apply -f 00_environment/discovery-2nodes.yaml
kubectl wait --for=condition=Ready target/dev1 --timeout=5m
kubectl wait --for=condition=Ready target/dev2 --timeout=5m
kubectl get target -o wide --show-labels
```

Nếu tên Secret/Profile trong lab khác `srl.nokia.sdcio.dev`, `gnmi-skipverify`, `gnmi-get`, sửa file discovery trước khi apply:

```bash
kubectl get secret,targetconnectionprofile,targetsyncprofile
```

## 4. Setup GitOps

Cài Argo CD nếu chưa có:

```bash
kubectl create namespace argocd --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait -n argocd --for=condition=Available deployment/argocd-server --timeout=10m
kubectl apply -f 00_environment/argocd-reconcile-10s-patch.yaml
kubectl rollout restart deployment/argocd-repo-server deployment/argocd-application-controller -n argocd
```

Tạo repository Git riêng cho bài test, copy thư mục `03_gitops_repo` vào repository, commit/push. Sửa `repoURL` trong `argocd-application.yaml`, rồi:

```bash
kubectl apply -f 00_environment/argocd-application.yaml
kubectl get application -n argocd sdcio-perf -w
```

Polling 10 giây chỉ phục vụ lab. Với mục tiêu dưới 30 giây ổn định, nên cấu hình webhook từ Git server tới Argo CD.

## 5. Precheck và smoke test

```bash
04_scripts/precheck.sh
04_scripts/monitor.sh
```

Ở terminal khác:

```bash
04_scripts/activate_dataset.sh 02_datasets/smoke_dev1_1000 /path/to/git-repo 'SMOKE 1000'
# Lấy start_epoch và revision từ output rồi:
START_EPOCH=<epoch> 04_scripts/wait_argocd_revision.sh <revision>
DATASET=smoke-dev1 VERSION=v1 EXPECTED=1 KIND=config START_EPOCH=<epoch> 04_scripts/wait_sdcio_resources.sh
04_scripts/verify_device_sample.sh dev1 v1
```

## 6. TC01 — 300k trên dev1

```bash
out=$(04_scripts/activate_dataset.sh 02_datasets/tc01_dev1_v1 /path/to/git-repo 'TC01 dev1 v1 300k')
echo "$out"
START=$(awk -F= '/start_epoch/{print $2}' <<<"$out")
REV=$(awk -F= '/revision/{print $2}' <<<"$out")
START_EPOCH=$START TIMEOUT=180 04_scripts/wait_argocd_revision.sh "$REV"
DATASET=tc01-dev1 VERSION=v1 EXPECTED=300 KIND=config START_EPOCH=$START TIMEOUT=180 04_scripts/wait_sdcio_resources.sh
04_scripts/verify_device_sample.sh dev1 v1
# Chạy sau khi kết thúc đo thời gian:
04_scripts/count_device_prefixes.sh dev1
```

PASS theo file gốc khi toàn bộ dữ liệu hợp lệ và có trên device trong <=120 giây. Nên ghi riêng thời gian GitOps, SDC accept và device apply.

## 7. TC02 — update Git <=30s

```bash
out=$(04_scripts/activate_dataset.sh 02_datasets/tc02_dev1_v2_delta1pct /path/to/git-repo 'TC02 dev1 v2 delta 1pct')
START=$(awk -F= '/start_epoch/{print $2}' <<<"$out")
REV=$(awk -F= '/revision/{print $2}' <<<"$out")
START_EPOCH=$START TIMEOUT=60 04_scripts/wait_argocd_revision.sh "$REV"
DATASET=tc01-dev1 VERSION=v2 EXPECTED=300 KIND=config START_EPOCH=$START TIMEOUT=60 04_scripts/wait_sdcio_resources.sh
04_scripts/verify_device_sample.sh dev1 v2
```

## 8. TC03 — 300k trên hai thiết bị

Dọn Config TC01/02 trước để không chồng intent:

```bash
kubectl delete config -l perf.sdcio.dev/dataset=tc01-dev1
```

Apply ConfigSet v1:

```bash
out=$(04_scripts/activate_dataset.sh 02_datasets/tc03_two_devices_v1 /path/to/git-repo 'TC03 two devices v1 300k')
START=$(awk -F= '/start_epoch/{print $2}' <<<"$out")
REV=$(awk -F= '/revision/{print $2}' <<<"$out")
START_EPOCH=$START TIMEOUT=180 04_scripts/wait_argocd_revision.sh "$REV"
DATASET=tc03-two-devices VERSION=v1 EXPECTED=300 KIND=configset START_EPOCH=$START TIMEOUT=180 04_scripts/wait_sdcio_resources.sh
04_scripts/verify_device_sample.sh dev1 v1
04_scripts/verify_device_sample.sh dev2 v1
```

## 9. TC04 — update hai thiết bị

```bash
out=$(04_scripts/activate_dataset.sh 02_datasets/tc04_two_devices_v2_delta1pct /path/to/git-repo 'TC04 two devices v2 delta 1pct')
START=$(awk -F= '/start_epoch/{print $2}' <<<"$out")
REV=$(awk -F= '/revision/{print $2}' <<<"$out")
START_EPOCH=$START TIMEOUT=60 04_scripts/wait_argocd_revision.sh "$REV"
DATASET=tc03-two-devices VERSION=v2 EXPECTED=300 KIND=configset START_EPOCH=$START TIMEOUT=60 04_scripts/wait_sdcio_resources.sh
04_scripts/verify_device_sample.sh dev1 v2
04_scripts/verify_device_sample.sh dev2 v2
```

## 10. Atomic failure injection

Quay lại V1 trước, sau đó chạy:

```bash
04_scripts/activate_dataset.sh 02_datasets/tc03_two_devices_v1 /path/to/git-repo 'Reset atomic v1'
04_scripts/atomic_failure_injection.sh /path/to/git-repo
```

Đánh giá:

- `dev1` đổi sang V2 trong khi `dev2` bị pause: có partial commit, FAIL tiêu chí distributed atomicity.
- Cả hai giữ V1 cho tới khi `dev2` trở lại rồi cùng đổi V2: PASS theo định nghĩa atomic của bài test.
- Không được coi `ConfigSet` là atomic chỉ vì hai thiết bị đều thành công trong trường hợp bình thường.

## 11. Dọn dữ liệu

```bash
04_scripts/cleanup.sh
```

## 12. Chỉ số cần ghi

- Git commit/push timestamp.
- Argo CD detected/synced timestamp.
- SDC resource accepted timestamp.
- Device sample applied timestamp.
- Tổng thời gian end-to-end.
- CPU/RAM và restart của api-server, controller, data-server-controller.
- CPU/RAM của dev1/dev2.
- Số Config/ConfigSet lỗi validation.
- Số deviation NOT-APPLIED/OVERRULED.
- Kết quả partial commit khi dev2 lỗi.
