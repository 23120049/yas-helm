# YAS Keycloak Realm Reconciler

## Mục đích

`yas-realm-reconciler` duy trì desired state của realm `Yas` trong Keycloak. Cơ chế này thay thế việc phụ thuộc lâu dài vào `KeycloakRealmImport`, vì `KeycloakRealmImport` chỉ import một lần và không tự sửa realm khi dữ liệu trong Keycloak bị mất hoặc bị thay đổi.

Reconciler bảo đảm:

- Realm `Yas` tồn tại.
- Cấu hình realm trong Keycloak được đồng bộ từ Helm chart.
- Hai client bắt buộc `backoffice-bff` và `storefront-bff` tồn tại sau khi reconcile.
- ArgoCD báo sync thất bại nếu không thể đăng nhập Keycloak hoặc không thể đưa realm về desired state.

## Các thành phần

### Desired realm template

File `keycloak/templates/keycloak-yas-realm-import.yaml` định nghĩa named template Helm `keycloak.yasRealm`. Đây là nguồn dữ liệu duy nhất cho cấu hình realm, gồm users, roles, clients, redirect URIs, authentication flows và các thiết lập khác.

Không nên tạo một bản realm JSON riêng vì hai bản cấu hình có thể bị lệch nhau.

### Desired-state ConfigMap

File `keycloak/templates/yas-realm.configmap.yaml` render named template trên thành JSON và lưu vào:

```text
ConfigMap/yas-realm-desired-state
└── yas-realm.json
```

Job mount file này read-only tại:

```text
/opt/yas/realm/yas-realm.json
```

### PostSync reconciler Job

File `keycloak/templates/yas-realm-reconciler.job.yaml` tạo Job `yas-realm-reconciler`. Job dùng `kcadm.sh` trong Keycloak image để gọi Keycloak Admin API.

Các annotation quan trọng:

```yaml
argocd.argoproj.io/hook: PostSync
argocd.argoproj.io/hook-delete-policy: BeforeHookCreation,HookSucceeded
```

- `PostSync`: chỉ chạy sau khi các resource thông thường của application Keycloak đã được apply.
- `BeforeHookCreation`: xóa hook Job cũ trước khi tạo Job mới, tránh lỗi vì trường Job là immutable.
- `HookSucceeded`: tự xóa Job sau khi chạy thành công.

Vì vậy, không thấy Job bằng `kubectl get jobs` sau một lần sync thành công là hành vi bình thường. Lịch sử hook vẫn có thể xem trong operation của application `keycloak` trên ArgoCD.

## Luồng hoạt động

```text
Commit yas-helm
       |
       v
ArgoCD sync application keycloak
       |
       +--> Apply Keycloak CR, Secret và desired-state ConfigMap
       |
       v
Chạy PostSync Job yas-realm-reconciler
       |
       +--> Chờ Keycloak Admin API sẵn sàng
       |
       +--> Đăng nhập realm master bằng Secret keycloak-credentials
       |
       +--> Kiểm tra realm Yas
              |
              +--> Chưa tồn tại: tạo realm từ yas-realm.json
              |
              +--> Đã tồn tại: partial import với OVERWRITE
       |
       +--> Xác minh realm Yas và hai BFF clients
       |
       +--> Thành công: ArgoCD xóa Job
       |
       +--> Thất bại: Job và ArgoCD sync báo lỗi
```

## Tính idempotent

Reconciler có thể chạy nhiều lần với cùng desired state:

- Nếu realm chưa tồn tại, Job tạo realm.
- Nếu một importer khác vừa tạo realm cùng lúc, Job chuyển sang reconcile realm đã tồn tại.
- Nếu realm đã tồn tại, Job dùng `partialImport` với `ifResourceExists=OVERWRITE` để đưa các resource có trong file import về cấu hình mong muốn.
- Cuối cùng Job luôn kiểm tra lại realm và hai client bắt buộc.

Kết quả mong muốn của nhiều lần chạy là giống nhau. Tuy nhiên, `OVERWRITE` có thể cập nhật hoặc tạo lại các resource do Helm quản lý. Không nên chỉnh thủ công các client, role hoặc user được định nghĩa trong desired realm rồi kỳ vọng thay đổi đó tồn tại sau lần sync tiếp theo.

## Bootstrap import và reconciler

Trong `keycloak/values.yaml` có hai cơ chế riêng:

```yaml
realmImport:
  enabled: false
  name: yas-realm-kc-v2

realmReconciler:
  enabled: true
```

`realmImport` chỉ dùng để recovery/bootstrap một lần khi cần chạy lại Keycloak Operator Realm Import. Sau khi endpoint discovery của realm trả HTTP 200, phải tắt nó bằng `enabled: false`.

`realmReconciler` là cơ chế vận hành lâu dài và nên giữ `enabled: true`.

Không đổi tên `yas-realm-kc-v2` để reconcile định kỳ. Việc tăng hậu tố `v3`, `v4` chỉ là biện pháp recovery thủ công, không phải workflow bình thường.

## Cách cập nhật realm

1. Sửa named template `keycloak.yasRealm` trong `keycloak/templates/keycloak-yas-realm-import.yaml`.
2. Nếu thêm hostname mới, cập nhật các values tương ứng như `backofficeRedirectUrls`, `storefrontRedirectUrls` hoặc `apiRedirectUrls`.
3. Render và lint chart trước khi commit:

```bash
helm lint yas-helm/deploy/keycloak/keycloak \
  -f yas-gitops/values/infra/keycloak.yaml

helm template keycloak yas-helm/deploy/keycloak/keycloak \
  --namespace infra \
  -f yas-gitops/values/infra/keycloak.yaml
```

4. Commit và push `yas-helm`.
5. Sync application `keycloak` trên ArgoCD.
6. Kiểm tra hook PostSync thành công.

## Kiểm tra sau khi deploy

Kiểm tra OpenID discovery endpoint:

```bash
curl -i \
  http://identity.yas.local.com/realms/Yas/.well-known/openid-configuration
```

Kết quả mong muốn là `HTTP/1.1 200 OK`.

Kiểm tra desired-state ConfigMap:

```bash
kubectl get configmap yas-realm-desired-state -n infra
```

Kiểm tra realm import tĩnh đã được tắt:

```bash
kubectl get keycloakrealmimport -n infra
```

Sau khi migration hoàn tất, không nên còn `yas-realm-kc-v2`.

Kiểm tra client bằng Keycloak Admin Console:

```text
Realm: Yas
Clients: backoffice-bff, storefront-bff
```

## Xem log Job

Job thành công bị xóa ngay bởi `HookSucceeded`, nên cách chính để xem kết quả là mở application `keycloak` trong ArgoCD và xem PostSync hook của operation gần nhất.

Khi debug cần giữ Job lại, có thể tạm thời bỏ `HookSucceeded` khỏi annotation, sync lại, rồi chạy:

```bash
kubectl logs -n infra job/yas-realm-reconciler
```

Sau khi debug phải khôi phục `HookSucceeded` để tránh tích lũy Job cũ.

## Xử lý sự cố

### ArgoCD đứng ở PostSync

Kiểm tra pod của Job:

```bash
kubectl get pods -n infra \
  -l app.kubernetes.io/component=realm-reconciler
```

Sau đó xem log pod hoặc Job. Các nguyên nhân thường gặp:

- Keycloak chưa Ready.
- Service `keycloak-service.infra.svc.cluster.local` không truy cập được.
- Secret `keycloak-credentials` thiếu hoặc sai.
- Desired realm JSON không hợp lệ.
- Partial import gặp resource không tương thích với phiên bản Keycloak hiện tại.

### Discovery endpoint trả 404

Điều này nghĩa là realm `Yas` không tồn tại. Sync lại application `keycloak` để chạy reconciler. Nếu reconciler không chạy, kiểm tra ArgoCD operation và xác nhận `realmReconciler.enabled: true`.

### Backoffice vẫn không đăng nhập được dù discovery trả 200

Kiểm tra lần lượt:

- Client `backoffice-bff` tồn tại và enabled.
- Redirect URI có `http://dev-backoffice.yas.local.com/*`.
- Client secret của Keycloak khớp `KEYCLOAK_BACKOFFICE_BFF_CLIENT_SECRET`.
- BFF đã restart sau khi ConfigMap hoặc Secret thay đổi.

### Job thành công nhưng không thấy bằng kubectl

Đây là hành vi mong muốn của `HookSucceeded`, không phải lỗi.

## Bảo mật

- Job đọc tài khoản master admin từ `Secret/keycloak-credentials`; mật khẩu không được ghi trực tiếp trong command hoặc log.
- ConfigMap desired state hiện chứa dữ liệu realm export, bao gồm client secrets và credential hash. Vì vậy quyền đọc ConfigMap trong namespace `infra` phải được giới hạn.
- Không log toàn bộ `yas-realm.json` trong pipeline hoặc ticket hỗ trợ.
- Với môi trường production, nên chuyển secret nhạy cảm khỏi realm ConfigMap sang Kubernetes Secret hoặc secret manager, đồng thời dùng service account Keycloak có quyền tối thiểu thay cho master admin.

## Giới hạn hiện tại

- Reconciler chạy khi application Keycloak được ArgoCD sync; nó không phải tiến trình chạy liên tục.
- Nếu realm bị xóa nhưng không có Git change hoặc sync mới, cần chủ động sync application `keycloak` để kích hoạt Job.
- `partialImport OVERWRITE` phù hợp với desired-state management nhưng có thể ghi đè thay đổi thủ công.
- Job xác minh bắt buộc hai BFF clients; các invariant khác có thể được bổ sung khi hệ thống cần kiểm tra chặt hơn.
