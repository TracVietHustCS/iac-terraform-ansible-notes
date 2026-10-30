=== tf-vs-ansible | Kết hợp | Terraform và Ansible: phân vai và phối hợp
>> Hai công cụ giải quyết hai bài toán khác nhau. Terraform dựng hạ tầng (mạng, máy, dịch vụ), Ansible cấu hình bên trong máy. Dùng đúng vai thì chúng bổ sung nhau; dùng lẫn lộn thì gây rối.

## So sánh

| | Terraform | Ansible |
|---|---|---|
| Bài toán chính | **Provision**: tạo, sửa, xoá tài nguyên (VPC, VM, DB, DNS) | **Configuration**: cài gói, ghi file, chạy dịch vụ bên trong máy |
| Mô hình | Khai báo, có **state** và **plan** | Phần lớn khai báo qua module, **không state**, chạy tuần tự |
| Cách kết nối | Gọi API của nhà cung cấp | SSH hoặc WinRM tới máy |
| Biết trước thay đổi | `plan` rất đầy đủ | `--check` có giới hạn tuỳ module |
| Quản lý vòng đời | Có (xoá bằng `destroy`) | Không lưu "đã tạo gì": không tự dọn thứ không còn trong playbook |
| Điểm mạnh | Hạ tầng cloud, phụ thuộc, dựng và phá toàn bộ | Cấu hình máy, quy trình, triển khai cuốn chiếu, máy vật lý |
| Điểm yếu | Cấu hình sâu bên trong máy là việc không phù hợp | Tạo hạ tầng cloud lớn khó quản lý vòng đời |

Quy tắc đơn giản: **nếu thứ đó là một tài nguyên của nhà cung cấp có API, dùng Terraform; nếu là trạng thái bên trong một máy, dùng Ansible** (hoặc đưa vào image).

## Các mô hình kết hợp

### 1. Terraform dựng, cloud-init cấu hình nhẹ

Với máy đơn giản, truyền script khởi tạo qua `user_data` (cloud-init): cài gói, ghi file, chạy dịch vụ lúc khởi động. Không cần công cụ thứ hai.

### 2. Terraform dựng máy, Ansible cấu hình

```ascii
   terraform apply  --->  tạo VM, mạng, load balancer  --->  output: danh sách IP
                                                                    |
                                                          sinh inventory / inventory động
                                                                    |
                                                         ansible-playbook  --->  cấu hình máy
```

Truyền thông tin sang Ansible bằng:

- **Inventory động** (plugin `amazon.aws.aws_ec2`...) lọc theo tag mà Terraform đã gắn.
- Hoặc Terraform ghi tệp inventory bằng `local_file` và `templatefile`:

```
resource "local_file" "inventory" {
  filename = "${path.module}/inventory.ini"
  content  = templatefile("${path.module}/inventory.tftpl", {
    web_ips = aws_instance.web[*].public_ip
  })
}
```

```
# inventory.tftpl
[web]
%{ for ip in web_ips ~}
${ip}
%{ endfor ~}
```

Tránh chạy Ansible bằng `provisioner "local-exec"` trong Terraform: kết quả nằm ngoài state, khó chạy lại và gỡ lỗi. Tách thành hai bước trong pipeline (Terraform rồi Ansible) rõ ràng hơn.

### 3. Dựng image bất biến (Packer), Terraform triển khai

Packer dựng **image máy** (AMI) có cấu hình sẵn, thường dùng Ansible làm bước cấu hình trong lúc build. Terraform sau đó chỉ tạo máy từ image. Máy không bao giờ bị cấu hình tại chỗ; thay đổi nghĩa là build image mới và thay máy. Dễ dự đoán và quay lui hơn.

```ascii
   Packer (+ Ansible)  --->  image đã cấu hình  --->  Terraform: tạo/thay máy từ image  --->  máy chạy
```

### 4. Container và Kubernetes

Khi ứng dụng chạy trong container, phần lớn việc cấu hình máy biến mất: Terraform dựng cluster (hoặc dùng dịch vụ quản lý như EKS) và hạ tầng xung quanh, [GitOps](https://tracviethustcs.github.io/k8s-study-notes/#gitops) (Flux, Argo CD) triển khai ứng dụng. Ansible còn vai trò cài đặt node khi dựng cluster on-prem ([Chuẩn bị node](https://tracviethustcs.github.io/k8s-study-notes/#setup-node)).

## Chọn gì cho tình huống nào

| Tình huống | Gợi ý |
|---|---|
| Dựng VPC, subnet, database, load balancer trên cloud | Terraform |
| Cài gói và cấu hình trên một đội máy ảo/vật lý sống lâu | Ansible |
| Triển khai cuốn chiếu ứng dụng lên các VM | Ansible (`serial`) hoặc image bất biến thay máy |
| Cấu hình rất ít, tạo máy là xong | Terraform + cloud-init |
| Nhiều máy giống nhau, muốn thay vì sửa | Packer + Terraform |
| Ứng dụng trên Kubernetes | Terraform cho hạ tầng, GitOps cho ứng dụng |
| Quản lý SaaS (GitHub, Datadog, Cloudflare, Vault) | Terraform (có provider) |

## Sai lầm hay gặp

- Dùng Terraform `remote-exec` như một Ansible kém: cấu hình sâu trong máy không có idempotency và không có vòng đời rõ.
- Dùng Ansible để tạo hạ tầng cloud lớn: không có state nên không biết cái gì đã tạo, khó xoá và khó phát hiện drift.
- Hai công cụ cùng quản lý một thứ (Terraform tạo security group và Ansible cũng sửa nó): sửa chồng nhau gây drift. Mỗi tài nguyên **một chủ**.
- Chạy lại Ansible trên máy sống lâu mà không ai kiểm tra: máy trôi dần khỏi playbook. Lên lịch chạy (hoặc `--check`) để phát hiện.

::: lab Terraform sinh inventory cho Ansible (không cần cloud)
```
mkdir -p ~/combo && cd ~/combo
cat > main.tf <<'EOF'
terraform {
  required_providers {
    local = { source = "hashicorp/local" }
  }
}
variable "hosts" {
  default = ["web1", "web2", "web3"]
}
resource "local_file" "inventory" {
  filename = "${path.module}/inventory.ini"
  content  = <<-EOT
    [web]
    %{ for h in var.hosts ~}
    ${h} ansible_connection=local
    %{ endfor ~}
  EOT
}
EOF
terraform init >/dev/null && terraform apply -auto-approve >/dev/null
cat inventory.ini
ansible -i inventory.ini web -m ping
ansible-inventory -i inventory.ini --graph
terraform destroy -auto-approve >/dev/null
```
Quan sát: danh sách máy do Terraform quyết định, Ansible chỉ đọc. Phân vai này giúp thêm một máy vào `var.hosts` là Ansible tự thấy ở lần chạy sau.
:::

=== iac-security | Kết hợp | Bảo mật IaC
>> IaC gom quyền thay đổi cả hạ tầng vào mã và pipeline: nó vừa giúp kiểm soát vừa tạo ra mục tiêu hấp dẫn. Bảo mật IaC là bảo vệ bí mật, quyền của pipeline, state và chuỗi cung ứng của chính các công cụ.

## Bí mật

| Rủi ro | Biện pháp |
|---|---|
| Mật khẩu, khoá API nằm trong mã `.tf`, playbook, `tfvars` | **Không đặt bí mật trong repo.** Dùng biến môi trường `TF_VAR_...`, kho bí mật (Vault, AWS Secrets Manager, SSM), Ansible Vault |
| Bí mật nằm trong **state** của Terraform | Mã hoá state, hạn chế quyền đọc state (xem bên dưới); ưu tiên để dịch vụ **tự sinh** mật khẩu và lưu ở kho bí mật rồi chỉ truyền tham chiếu |
| Bí mật lộ trong log CI | `sensitive`, `no_log: true`, che biến trong CI; không in biến môi trường |
| Khoá dài hạn của cloud trong CI | **OIDC** và danh tính ngắn hạn; xoay vòng khoá nếu bắt buộc phải dùng |
| Commit nhầm bí mật | Quét bí mật trước khi commit và trong CI (gitleaks, trufflehog); nếu đã lộ thì **coi như đã bị lộ** và xoay vòng ngay |

Biến `sensitive = true` của Terraform chỉ ẩn khỏi hiển thị, không mã hoá trong state ([State](#tf-state)).

## Quyền của pipeline và của người

- **Quyền tối thiểu**: vai trò `plan` chỉ đọc; vai trò `apply` chỉ trong phạm vi môi trường đó. Vai trò production chỉ nhánh `main` (và môi trường có duyệt) mới assume được.
- **Tách tài khoản/môi trường**: production ở tài khoản riêng, state riêng, danh tính riêng.
- **Tránh "quản trị viên cho mọi thứ"**: pipeline có quyền `AdministratorAccess` nghĩa là ai chiếm được pipeline là chiếm cả tài khoản. Cấp quyền theo tài nguyên Terraform cần quản lý.
- **Bảo vệ nhánh chính**: bắt buộc review, bắt buộc CI qua, cấm push thẳng; đây là rào chắn trước khi mã IaC tác động hạ tầng.
- Với pull request từ nhánh fork hoặc bên ngoài: **không** cho chạy với bí mật hay quyền ghi.

## Bảo vệ state

- Backend từ xa, **mã hoá lúc lưu**, **bật versioning**, chặn truy cập công khai.
- Quyền ghi chỉ cho pipeline; quyền đọc hạn chế vì state chứa dữ liệu nhạy cảm.
- Bật nhật ký truy cập (CloudTrail, S3 access logs) để biết ai chạm vào state.
- Khoá state ngăn hai lần apply đồng thời.

## Cấu hình hạ tầng an toàn (policy as code)

Quét cấu hình **trước khi apply** để chặn lỗi phổ biến: bucket công khai, security group mở `0.0.0.0/0` cho SSH hay database, thiếu mã hoá đĩa và database, IMDSv1, thiếu logging.

| Công cụ | Ghi chú |
|---|---|
| Trivy (`trivy config`), Checkov | Quét tệp Terraform, CloudFormation, Kubernetes manifest, Dockerfile |
| OPA / Conftest, Sentinel | Chính sách tự viết cho quy định riêng của tổ chức |
| `ansible-lint` | Kiểm tra playbook, gồm một số quy tắc bảo mật (quyền file, `no_log`) |

Xem cách tích hợp trong pipeline ở [Terraform trong CI/CD](#tf-pipeline).

## Chuỗi cung ứng của chính công cụ

- **Ghim phiên bản** Terraform, provider (lock file), module (`version` hay `ref`), collection và role Ansible.
- Provider và module từ Registry/Galaxy là mã chạy với quyền của bạn: ưu tiên nguồn chính thức hoặc đã kiểm chứng, đọc mã khi cần, và tránh phụ thuộc vào tác giả cá nhân không ai duy trì.
- Cân nhắc **mirror nội bộ** cho provider và module để kiểm soát và không phụ thuộc mạng ngoài.
- Cập nhật phiên bản có chủ đích (Renovate/Dependabot), có plan và review.

## Chống thay đổi ngoài ý muốn

- `prevent_destroy` cho dữ liệu quan trọng (database, bucket sao lưu).
- Bảo vệ xoá ở chính tài nguyên cloud (`deletion_protection` của RDS, MFA delete của S3).
- Duyệt thủ công cho production, và đọc kỹ các dòng `-` và `-/+` trong plan.
- Phát hiện drift theo lịch để nhận ra thay đổi tay ngoài Git ([CI/CD](#tf-pipeline)).
- Bật ghi nhật ký kiểm toán của cloud (CloudTrail) để truy vết ai đã làm gì.

::: warn Một tệp state hay một khoá trong repo đủ để mất cả hạ tầng
Rủi ro lớn nhất của IaC thường không nằm ở công cụ mà ở nơi mọi người quên bảo vệ: bucket state công khai, khoá cloud dài hạn trong biến CI, pipeline có quyền quản trị, và bí mật đã commit. Rà soát bốn chỗ này trước tiên.
:::

::: lab Quét thử một cấu hình sai
```
mkdir -p ~/sec && cd ~/sec
cat > main.tf <<'EOF'
resource "aws_security_group" "bad" {
  name = "bad"
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
resource "aws_s3_bucket" "data" {
  bucket = "my-data-bucket-lab"
}
EOF
trivy config .                    # hoặc: checkov -d .   (cần cài công cụ)
grep -rn "password\|secret\|AKIA" . || echo "không thấy bí mật trong mã"
```
Ghi lại các cảnh báo quét được, và với mỗi cảnh báo, bạn sửa mã thế nào cho an toàn hơn?
:::

=== iac-cheatsheet | Tham chiếu | Cheat sheet Terraform và Ansible
>> Tra nhanh theo câu hỏi "tôi muốn làm gì". Chi tiết ở các trang tương ứng.

## Terraform: lệnh

| Muốn | Lệnh |
|---|---|
| Khởi tạo | `terraform init` (thêm `-upgrade` để nâng provider, `-migrate-state` khi đổi backend) |
| Định dạng, kiểm tra | `terraform fmt -recursive`, `terraform validate` |
| Xem trước, lưu kế hoạch | `terraform plan`, `terraform plan -out=tfplan` |
| Áp dụng | `terraform apply`, `terraform apply tfplan`, `-auto-approve` (chỉ trong CI có kiểm soát) |
| Phá huỷ | `terraform destroy` |
| Ép tạo lại một tài nguyên | `terraform apply -replace="ADDR"` |
| Xem output | `terraform output`, `terraform output -raw NAME`, `-json` |
| Thử biểu thức | `terraform console` |
| Chỉ làm mới state | `terraform plan -refresh-only` |
| Phát hiện drift trong CI | `terraform plan -detailed-exitcode` (0: không đổi, 1: lỗi, 2: có thay đổi) |
| Kiểm thử | `terraform test` |
| Đồ thị | `terraform graph \| dot -Tpng > g.png` |

## Terraform: state

| Muốn | Lệnh |
|---|---|
| Liệt kê, xem | `terraform state list`, `terraform state show ADDR` |
| Đổi địa chỉ | `terraform state mv A B` (hoặc khối `moved`) |
| Gỡ khỏi quản lý (không xoá thật) | `terraform state rm ADDR` (hoặc khối `removed`) |
| Đưa tài nguyên có sẵn vào | `terraform import ADDR ID` (hoặc khối `import`) |
| Sao lưu, tải state | `terraform state pull > backup.tfstate` |
| Gỡ khoá kẹt | `terraform force-unlock LOCK_ID` |
| Chuyển backend | `terraform init -migrate-state` |

## Terraform: cú pháp nhanh

| Cần | Viết |
|---|---|
| Nhiều bản, định danh bằng khoá | `for_each = toset(var.names)` hoặc `{ for x in list : x.name => x }` |
| Nhiều bản giống nhau, bật/tắt | `count = var.enabled ? 1 : 0` |
| Tránh bị xoá nhầm | `lifecycle { prevent_destroy = true }` |
| Thay thế không gián đoạn | `lifecycle { create_before_destroy = true }` |
| Bỏ qua thay đổi bên ngoài | `lifecycle { ignore_changes = [tags] }` |
| Đổi tên không xoá-tạo lại | `moved { from = A  to = B }` |
| Ghim phiên bản | `required_version`, `required_providers { ... version = "~> 5.0" }` |
| Chia dải IP | `cidrsubnet(var.cidr, 8, 1)` |
| Biến nhạy cảm | `sensitive = true` (vẫn nằm trong state) |

## Terraform: thứ tự ưu tiên biến

Mặc định, rồi `TF_VAR_*`, rồi `terraform.tfvars`, rồi `*.auto.tfvars`, rồi `-var` và `-var-file` (cái sau thắng).

## Ansible: lệnh

| Muốn | Lệnh |
|---|---|
| Kiểm tra kết nối | `ansible all -m ping` |
| Chạy một module | `ansible web -b -m apt -a "name=nginx state=present"` |
| Xem fact | `ansible HOST -m setup -a "filter=ansible_os_family"` |
| Chạy playbook | `ansible-playbook -i inv site.yml` |
| Chạy thử, xem khác biệt | `ansible-playbook site.yml --check --diff` |
| Giới hạn máy, thẻ | `--limit web1`, `--tags config`, `--skip-tags slow` |
| Bắt đầu từ tác vụ | `--start-at-task "TÊN"`, `--step` |
| Chi tiết khi lỗi | `-v`, `-vv`, `-vvv` |
| Kiểm tra cú pháp, phong cách | `--syntax-check`, `ansible-lint site.yml` |
| Xem inventory | `ansible-inventory --graph`, `ansible-inventory --host H` |
| Vault | `ansible-vault create/edit/view/encrypt/rekey`, `encrypt_string` |
| Collection | `ansible-galaxy collection install -r requirements.yml` |

## Ansible: cú pháp nhanh

| Cần | Viết |
|---|---|
| Chỉ chạy khi có điều kiện | `when: ansible_os_family == "Debian"` |
| Vòng lặp | `loop: [a, b]` rồi dùng `{{ item }}` |
| Lưu kết quả | `register: ket_qua` |
| Chạy khi có thay đổi | `notify: Tên handler` |
| Idempotent cho `command` | `args: { creates: /path }` hoặc `changed_when: false` |
| Quyền cao | `become: true` |
| Chạy trên máy khác | `delegate_to: lb1` |
| Cuốn chiếu | `serial: "25%"`, `max_fail_percentage: 0` |
| Thử lại tới khi đạt | `until: ketqua.status == 200`, `retries: 20`, `delay: 3` |
| Try/catch | `block:`, `rescue:`, `always:` |
| Ẩn bí mật khỏi log | `no_log: true` |

## Trạng thái hay dùng của module

`present` / `absent` (có hoặc không), `started` / `stopped` / `restarted` / `reloaded` (dịch vụ), `latest` (luôn mới nhất, cẩn thận), `directory` / `file` / `link` / `touch` (module `file`).

=== quiz-tf | Luyện tập | Luyện tập 1: Terraform
>> 6 câu tình huống về plan, state, vòng đời tài nguyên, module và CI/CD. Chọn đáp án rồi bấm "Kiểm tra" để xem giải thích. Kết quả được lưu trong trình duyệt.

## Câu hỏi

@quiz
Q: Bạn đổi tên khối `resource "aws_instance" "web"` thành `"app"` trong mã, rồi `terraform plan` hiện: "1 to destroy, 1 to add". Cách làm đúng để giữ nguyên máy thật mà chỉ đổi tên trong mã?
- Chạy `terraform apply -replace` để đồng bộ
- Xoá file state rồi chạy lại `terraform init`
+ Thêm khối `moved { from = aws_instance.web  to = aws_instance.app }` (hoặc `terraform state mv`) để Terraform đổi địa chỉ trong state thay vì xoá-tạo lại
- Thêm `lifecycle { ignore_changes = all }`
E: Terraform nhận diện tài nguyên bằng địa chỉ trong state; đổi tên trong mã nghĩa là "cái cũ không còn, cái mới xuất hiện". `moved` (hoặc `state mv`) khai báo rằng đây là cùng một tài nguyên. Xoá state làm Terraform quên mọi thứ đã tạo; `ignore_changes` không liên quan tới đổi địa chỉ.
Q: Hai kỹ sư cùng chạy `terraform apply` vào cùng một thời điểm và state bị ghi đè dẫn tới hỏng. Biện pháp phòng ngừa đúng nhất?
- Chỉ cho phép một người biết mật khẩu tài khoản cloud
- Lưu file state trong Git để có lịch sử
+ Dùng backend từ xa có khoá state (và mã hoá, versioning), để hai lần apply không chạy đồng thời
- Chạy `terraform apply -auto-approve` nhanh hơn
E: Backend từ xa có khoá (S3 với khoá, DynamoDB, HCP Terraform...) đảm bảo mỗi lúc chỉ một lần chạy ghi state. State không nên nằm trong Git vì chứa dữ liệu nhạy cảm và không có cơ chế khoá.
Q: Bạn dùng `count` để tạo 3 subnet từ một danh sách, rồi xoá phần tử ở giữa danh sách. Plan muốn thay thế cả phần tử cuối. Cách thiết kế nào tránh vấn đề này?
- Thêm `depends_on` cho từng subnet
+ Dùng `for_each` với khoá có ý nghĩa (ví dụ tên subnet) để mỗi tài nguyên có danh tính ổn định, thay vì chỉ số
- Đổi sang `terraform apply -target`
- Dùng `count = length(var.subnets) + 1`
E: Với `count`, tài nguyên được định danh bằng chỉ số; xoá phần tử ở giữa làm các phần tử sau bị đánh số lại và Terraform coi là thay đổi. `for_each` định danh theo khoá nên chỉ phần tử bị xoá mới bị tác động.
Q: Chọn TẤT CẢ phát biểu đúng về state của Terraform.
+ State có thể chứa dữ liệu nhạy cảm dưới dạng văn bản rõ, kể cả giá trị của biến đánh dấu `sensitive`
+ Không nên chỉnh sửa tay file state; dùng các lệnh `terraform state`
- Nên commit file state vào Git để mọi người cùng dùng
+ `terraform plan -refresh-only` giúp thấy drift giữa state và thực tế mà không đổi mã
E: `sensitive` chỉ ẩn khỏi hiển thị, không mã hoá trong state. Commit state vào Git làm lộ dữ liệu và không có khoá. Nên bảo vệ state bằng backend từ xa mã hoá, và dùng `refresh-only` để quan sát drift.
Q: Pipeline production của bạn dùng khoá AWS dài hạn lưu trong secret của CI và vai trò có quyền quản trị toàn bộ. Cải thiện đúng nhất?
- Đổi tên secret cho khó đoán
+ Dùng OIDC để nhận quyền ngắn hạn, tách vai trò `plan` (chỉ đọc) và `apply`, giới hạn quyền theo tài nguyên cần quản lý và chỉ cho nhánh chính assume vai trò apply
- Cho mọi lập trình viên dùng chung khoá để dễ gỡ lỗi
- Lưu khoá trong file `.tfvars` đã mã hoá base64
E: Khoá dài hạn là mục tiêu bị đánh cắp; OIDC cấp token tạm thời cho đúng repo và nhánh. Quyền tối thiểu và tách `plan`/`apply` giảm hậu quả nếu pipeline bị chiếm. Base64 không phải mã hoá.
Q: Trong CI bạn chạy `terraform plan` rồi ở bước sau `terraform apply` (không dùng tệp plan). Rủi ro chính là gì, và cách khắc phục?
- Không có rủi ro vì hai lệnh chạy ngay liền nhau
+ Apply có thể tính ra kế hoạch khác với cái đã được review (mã, state hoặc thực tế đã đổi giữa chừng); lưu `plan -out=tfplan` và `apply tfplan` để áp dụng đúng kế hoạch đã duyệt
- `apply` luôn chậm hơn nếu không có tệp plan
- Terraform không cho chạy `apply` sau `plan` nếu không có `-auto-approve`
E: Cái được review phải là cái được thực thi. Lưu plan thành tệp và apply đúng tệp đó bảo đảm điều này (kèm lưu ý bảo vệ tệp plan vì có thể chứa giá trị nhạy cảm).

=== quiz-ansible | Luyện tập | Luyện tập 2: Ansible và kết hợp công cụ
>> 6 câu tình huống về idempotency, handler, cuốn chiếu, Vault và phối hợp Terraform với Ansible.

## Câu hỏi

@quiz
Q: Playbook của bạn có tác vụ `ansible.builtin.command: useradd deploy`. Lần chạy đầu thành công, lần hai báo lỗi vì user đã tồn tại. Cách sửa tốt nhất?
- Thêm `ignore_errors: true` cho tác vụ đó
+ Dùng module `ansible.builtin.user` với `name: deploy` và `state: present`, vì module tự kiểm tra trạng thái nên idempotent
- Chạy playbook chỉ một lần duy nhất
- Đổi `command` thành `shell`
E: Module chuyên biệt mô tả trạng thái mong muốn và chỉ hành động khi chưa khớp. `command`/`shell` luôn chạy lệnh nên không idempotent. `ignore_errors` che lỗi chứ không giải quyết nguyên nhân; nếu buộc dùng `command`, thêm `creates` hoặc `changed_when`.
Q: Bạn cập nhật `nginx.conf` bằng `template` và muốn nginx chỉ reload khi cấu hình thật sự đổi. Cơ chế đúng?
- Thêm tác vụ `service: state=restarted` ở cuối play
+ Dùng `notify` ở tác vụ template và định nghĩa handler `reload nginx`; handler chỉ chạy khi tác vụ báo `changed`
- Dùng `when: always`
- Đặt `register` rồi so sánh thủ công
E: Handler chạy một lần ở cuối play và chỉ khi có tác vụ `notify` nó thực sự thay đổi. Restart vô điều kiện làm gián đoạn dịch vụ mỗi lần chạy playbook dù không có gì đổi.
Q: Bạn cần nâng cấp ứng dụng trên 20 web server mà luôn còn ít nhất phần lớn máy phục vụ, và dừng toàn bộ nếu một đợt lỗi. Cấu hình phù hợp?
- Chạy playbook song song trên cả 20 máy bằng `forks: 20`
+ Dùng `serial` (ví dụ `"25%"`) để cập nhật từng đợt, `max_fail_percentage: 0` để dừng khi có lỗi, và gỡ máy khỏi load balancer bằng `delegate_to`
- Dùng `ignore_errors: true` cho cả play
- Chạy lần lượt bằng `--limit` thủ công từng máy
E: `serial` chia thành đợt nhỏ, `max_fail_percentage: 0` ngăn lỗi lan sang các đợt sau, và `delegate_to` cho phép điều khiển load balancer từ máy đích. `forks` chỉ quyết định song song hoá, không phải an toàn.
Q: Chọn TẤT CẢ thói quen đúng khi xử lý bí mật trong Ansible.
+ Mã hoá file biến nhạy cảm bằng Ansible Vault và giữ mật khẩu vault ngoài Git
+ Dùng `no_log: true` cho tác vụ xử lý bí mật để tránh lộ trong log
- Đặt mật khẩu dạng văn bản rõ trong `group_vars/all.yml` vì repo là riêng tư
+ Cân nhắc lấy bí mật từ HashiCorp Vault hoặc kho bí mật của cloud thay vì lưu trong repo
E: Repo riêng tư vẫn có thể bị sao chép, fork, lộ log hoặc cấp quyền rộng. Bí mật nên được mã hoá hoặc lấy lúc chạy từ kho chuyên dụng, mật khẩu vault không nằm trong Git, và log không in giá trị nhạy cảm.
Q: Nhóm muốn: tạo VPC, 3 VM và load balancer trên cloud, rồi cài và cấu hình nginx trên các VM. Phân công công cụ hợp lý nhất?
- Dùng Ansible cho cả hai việc vì nó cũng có module cloud
- Dùng Terraform `remote-exec` cho cả cấu hình bên trong VM
+ Terraform tạo hạ tầng (có state và plan), rồi Ansible (hoặc cloud-init, hoặc image dựng sẵn) cấu hình bên trong VM; truyền danh sách máy qua inventory động hoặc output
- Viết một script bash gọi API cloud rồi SSH vào từng máy
E: Terraform mạnh ở vòng đời tài nguyên cloud nhờ state và plan; Ansible mạnh ở cấu hình bên trong máy. Dùng `remote-exec` như công cụ cấu hình mất idempotency và nằm ngoài state; Ansible tạo hạ tầng lớn thì khó quản lý vòng đời và drift.
Q: Sau khi chạy playbook thành công, bạn chạy lại ngay lần hai để kiểm tra và thấy nhiều tác vụ vẫn báo `changed`. Điều này cho biết gì và nên làm gì?
- Bình thường, Ansible luôn báo `changed` ở mọi lần chạy
+ Có tác vụ không idempotent (thường do `command`/`shell` hoặc nội dung tạo ra thay đổi mỗi lần); tìm các tác vụ đó và dùng module chuyên biệt, `creates`/`changed_when`, hoặc làm nội dung ổn định
- Playbook đã hoàn hảo và nên đẩy lên production ngay
- Cần tăng `forks`
E: Lần chạy thứ hai trên máy đã khớp phải báo `changed=0`. Nếu không, bạn không thể chạy lại một cách an toàn hay dùng `--check` để phát hiện drift. Các thủ phạm thường là `command`/`shell` và template chứa giá trị đổi liên tục (thời gian, giá trị ngẫu nhiên).
