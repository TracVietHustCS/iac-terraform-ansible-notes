=== tf-envs | Terraform | Quản lý nhiều môi trường
>> Dev, staging và production cần cùng một mã nhưng state, quyền và giá trị riêng. Cách tổ chức thư mục và state quyết định blast radius và mức độ dễ sai.

## Mục tiêu

- Cùng một mã (module) cho mọi môi trường, chỉ khác **giá trị** (kích thước, số lượng, tên miền).
- **State riêng** cho từng môi trường: hỏng ở dev không chạm tới production.
- **Quyền riêng**: người và pipeline chỉ làm được trong môi trường họ được phép.
- Đưa thay đổi lên theo thứ tự: dev, rồi staging, rồi production.

## Ba cách phổ biến

| Cách | Mô tả | Ưu | Nhược |
|---|---|---|---|
| **Thư mục theo môi trường** + module dùng chung | `envs/dev`, `envs/prod` đều gọi `modules/...` | Rõ ràng, state và backend tách hẳn, dễ gán quyền khác nhau | Lặp một ít mã gọi module |
| **Workspace** | Một mã, nhiều state trong cùng backend (`terraform workspace new prod`) | Ít lặp mã | Dễ nhầm workspace; cùng backend và cùng quyền; khó thể hiện khác biệt cấu hình |
| **Một mã, tệp tfvars khác nhau** | `-var-file=envs/prod.tfvars` kèm backend cấu hình riêng lúc init | Gọn | Dễ chạy nhầm tệp; cần kỷ luật và tự động hoá |

Hướng được dùng nhiều nhất cho nhóm: **thư mục theo môi trường**, mỗi nơi có backend riêng, và các thành phần dùng chung đóng thành module. Workspace hợp với môi trường tạm thời giống hệt nhau (nhánh thử nghiệm), ít hợp với khác biệt dev và prod thật.

## Bố cục repo điển hình

```
infra/
├── modules/
│   ├── network/
│   ├── database/
│   └── web_service/
├── envs/
│   ├── dev/
│   │   ├── main.tf           # gọi module, truyền giá trị dev
│   │   ├── backend.tf        # key = "dev/terraform.tfstate"
│   │   └── terraform.tfvars
│   ├── stg/
│   └── prod/
│       ├── main.tf
│       ├── backend.tf        # key = "prod/terraform.tfstate", bucket/tài khoản riêng
│       └── terraform.tfvars
└── .github/workflows/        # pipeline: plan khi PR, apply khi merge
```

Tách tiếp theo **tầng** trong mỗi môi trường khi hạ tầng lớn: `network`, `data`, `app` mỗi cái một state riêng. Tầng ít đổi (mạng) tách khỏi tầng hay đổi (ứng dụng) để plan nhanh và rủi ro nhỏ. Tầng sau đọc output của tầng trước (data source, hoặc `terraform_remote_state`).

## Backend theo môi trường

```
# envs/prod/backend.tf
terraform {
  backend "s3" {
    bucket  = "myorg-tfstate-prod"        # bucket riêng, tài khoản riêng
    key     = "app/terraform.tfstate"
    region  = "ap-southeast-1"
    encrypt = true
  }
}
```

State của production nằm ở **tài khoản và bucket riêng**, chỉ pipeline production (và rất ít người) được ghi. Giá trị trong backend không dùng được biến, nên cố định từng thư mục hoặc truyền lúc init: `terraform init -backend-config=backend-prod.hcl`.

## Đưa thay đổi qua các môi trường

1. Thay đổi mã module (hoặc phiên bản module) trong một nhánh.
2. PR chạy `plan` cho dev (và staging); review kết quả.
3. Merge: pipeline `apply` dev, rồi kiểm thử.
4. Tiếp theo staging, cuối cùng production **có bước duyệt thủ công** (xem [CI/CD](#tf-pipeline)).

Module dùng chung được ghim phiên bản (`ref=v1.4.0`): nâng phiên bản ở dev trước, thấy ổn rồi mới nâng ở prod. Như vậy "promotion" là đổi **số phiên bản**, không phải sao chép mã.

## Công cụ hỗ trợ

- **Terragrunt**: lớp bọc giảm lặp cấu hình backend và biến giữa nhiều thư mục.
- **HCP Terraform (Terraform Cloud)**, Spacelift, env0, Scalr: nền tảng quản lý state, quyền, plan/apply theo workspace và chính sách.
- **Atlantis**: tự chạy plan/apply theo bình luận trên pull request.

Chọn khi thật sự gặp vấn đề lặp hoặc quy mô; với nhóm nhỏ, thư mục theo môi trường cộng pipeline CI thường đủ.

::: lab Hai môi trường, hai state
```
mkdir -p ~/tf6/modules/app ~/tf6/envs/dev ~/tf6/envs/prod && cd ~/tf6
cat > modules/app/main.tf <<'EOF'
terraform {
  required_providers {
    local = { source = "hashicorp/local" }
  }
}
variable "env"   { type = string }
variable "size"  { type = number }
resource "local_file" "cfg" {
  filename = "${path.root}/app-${var.env}.conf"
  content  = "env=${var.env}\nsize=${var.size}\n"
}
EOF
for e in dev prod; do
  s=1; [ $e = prod ] && s=4
  printf 'module "app" {\n  source = "../../modules/app"\n  env    = "%s"\n  size   = %s\n}\n' $e $s > envs/$e/main.tf
done
(cd envs/dev  && terraform init >/dev/null && terraform apply -auto-approve >/dev/null && cat app-dev.conf)
(cd envs/prod && terraform init >/dev/null && terraform apply -auto-approve >/dev/null && cat app-prod.conf)
ls envs/dev envs/prod                         # mỗi môi trường có terraform.tfstate riêng
(cd envs/dev && terraform destroy -auto-approve >/dev/null)   # prod không bị ảnh hưởng
ls envs/prod/app-prod.conf
(cd envs/prod && terraform destroy -auto-approve >/dev/null)
```
Quan sát: phá huỷ dev có chạm tới prod không, và vì sao? Điều gì sẽ xảy ra nếu cả hai dùng chung một file state?
:::

=== tf-pipeline | Terraform | Terraform trong CI/CD, kiểm thử và chính sách
>> Đưa Terraform vào pipeline để thay đổi hạ tầng được review, kiểm tra tự động và áp dụng bằng danh tính có kiểm soát, thay vì chạy từ laptop.

## Quy trình chuẩn

```ascii
  Pull request                                    Merge vào main
  ------------                                    ---------------
  fmt -check -> validate -> tflint                 plan (lưu thành tệp)
       |                                                  |
  quét bảo mật/chính sách (Trivy, Checkov, OPA)    DUYỆT thủ công (với prod)
       |                                                  |
  terraform plan  -> đăng kết quả vào PR           apply đúng tệp plan đã duyệt
       |
  người review đọc plan
```

Nguyên tắc: **review nhìn thấy plan**, **cái được apply chính là cái đã duyệt**, và việc apply chạy bằng **danh tính của pipeline** với quyền tối thiểu.

## Pipeline mẫu với GitHub Actions

```
name: terraform
on:
  pull_request:
    paths: ["infra/**"]
  push:
    branches: [main]
    paths: ["infra/**"]

permissions:
  id-token: write        # để lấy token OIDC
  contents: read
  pull-requests: write   # để đăng comment plan

jobs:
  plan:
    runs-on: ubuntu-latest
    defaults: {run: {working-directory: infra/envs/dev}}
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with: {terraform_version: "1.9.8"}
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::111111111111:role/gha-terraform-plan
          aws-region: ap-southeast-1
      - run: terraform fmt -check -recursive
      - run: terraform init -input=false
      - run: terraform validate
      - run: terraform plan -input=false -out=tfplan
      - uses: actions/upload-artifact@v4
        with: {name: tfplan, path: infra/envs/dev/tfplan}

  apply:
    if: github.ref == 'refs/heads/main'
    needs: plan
    runs-on: ubuntu-latest
    environment: dev            # môi trường có thể yêu cầu người duyệt
    defaults: {run: {working-directory: infra/envs/dev}}
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with: {terraform_version: "1.9.8"}
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::111111111111:role/gha-terraform-apply
          aws-region: ap-southeast-1
      - uses: actions/download-artifact@v4
        with: {name: tfplan, path: infra/envs/dev}
      - run: terraform init -input=false
      - run: terraform apply -input=false tfplan
```

Phiên bản action, số phiên bản Terraform và ARN trong ví dụ chỉ là minh hoạ: dùng giá trị của bạn, và ghim action theo mã băm commit nếu cần chặt chẽ hơn. Chi tiết về pipeline xem [CI/CD](https://tracviethustcs.github.io/cicd-release-notes/).

Điểm quan trọng:

- **Xác thực bằng OIDC** (không lưu khoá AWS dài hạn trong secret của CI): AWS tin token ngắn hạn do GitHub phát hành cho đúng repo và nhánh. Vai trò `plan` (chỉ đọc) tách khỏi vai trò `apply` (ghi), và chỉ nhánh `main` mới assume được vai trò `apply`.
- **Môi trường production** gắn `environment` có **người duyệt bắt buộc** trước khi apply.
- **Khoá state** ngăn hai pipeline apply cùng lúc; đặt `concurrency` ở job để xếp hàng.
- **Lưu tệp plan** làm artifact để bước apply dùng đúng nó. Tệp plan có thể chứa giá trị nhạy cảm: giữ thời hạn artifact ngắn, hạn chế truy cập.

## Kiểm tra tĩnh và chính sách

| Công cụ | Làm gì |
|---|---|
| `terraform fmt -check`, `validate` | Định dạng và cú pháp |
| **TFLint** | Phát hiện lỗi cấu hình, loại instance không tồn tại, vi phạm quy ước |
| **Trivy** (đã hấp thụ tfsec), **Checkov** | Quét cấu hình sai về bảo mật (bucket công khai, security group mở toàn bộ, thiếu mã hoá) |
| **OPA / Conftest**, **Sentinel** (HashiCorp) | Chính sách tự viết: "mọi tài nguyên phải có tag Owner", "cấm instance lớn hơn X" |
| **Infracost** | Ước tính chi phí thay đổi ngay trong PR |

Chạy trên **plan dạng JSON** để chính sách thấy giá trị cuối cùng:

```
terraform plan -out=tfplan
terraform show -json tfplan > tfplan.json
conftest test tfplan.json --policy policy/
trivy config .
```

## Kiểm thử

| Mức | Cách |
|---|---|
| Tĩnh | `fmt`, `validate`, TFLint, quét bảo mật: nhanh, chạy trên mọi PR |
| Kế hoạch | `terraform test` (từ Terraform 1.6) với các `run` kiểm tra bằng `command = plan`; kiểm tra điều kiện và giá trị tính toán không cần tạo gì |
| Tích hợp | `terraform test` với `command = apply` hoặc **Terratest** (Go): thật sự dựng hạ tầng trong tài khoản thử nghiệm, kiểm tra, rồi huỷ |
| Hợp đồng | Kiểm tra output và giao diện module không bị phá vỡ khi nâng phiên bản |

Ví dụ tệp `tests/basic.tftest.hcl`:

```
run "tên_đúng_quy_ước" {
  command = plan
  variables {
    env = "dev"
  }
  assert {
    condition     = local.name == "app-dev"
    error_message = "Tên tài nguyên phải theo dạng app-<env>."
  }
}
```

```
terraform test
```

Kiểm thử tích hợp tốn tiền và thời gian: chạy ít hơn (khi đổi module, hoặc theo lịch đêm) và luôn có bước dọn dẹp chắc chắn.

## Phát hiện drift theo lịch

Chạy `terraform plan -detailed-exitcode` định kỳ (mã thoát 2 nghĩa là có thay đổi): báo động khi có người sửa tay ngoài Git.

```
terraform plan -detailed-exitcode -input=false; echo "exit=$?"     # 0: không đổi, 1: lỗi, 2: có thay đổi
```

::: lab Cổng kiểm tra cục bộ như CI
```
cd ~/tf2                                  # dùng lại cấu hình ở bài Biến (hoặc thư mục Terraform bất kỳ)
terraform fmt -check -recursive; echo "fmt=$?"
terraform init -input=false >/dev/null && terraform validate
terraform plan -input=false -out=tfplan -detailed-exitcode; echo "plan exit=$?"
terraform show -json tfplan | python3 -c "import sys,json; d=json.load(sys.stdin); print([ (c['address'], c['change']['actions']) for c in d.get('resource_changes',[])])"
mkdir -p tests && cat > tests/name.tftest.hcl <<'EOF'
run "name" {
  command = plan
  assert {
    condition     = local.name == "app-dev"
    error_message = "tên sai"
  }
}
EOF
terraform test
```
Ghi lại mã thoát của `plan -detailed-exitcode` khi có thay đổi và khi không còn thay đổi, và cách bạn dùng nó để phát hiện drift.
:::

=== tf-troubleshooting | Terraform | Xử lý sự cố
>> Các lỗi Terraform thường gặp, theo khung: triệu chứng, nguyên nhân, cách xử lý an toàn. Nguyên tắc chung: đọc kỹ thông báo lỗi, giữ nguyên state, và sao lưu trước khi thao tác rủi ro.

## Bật log chi tiết

```
TF_LOG=DEBUG terraform plan 2> debug.log      # mức: TRACE, DEBUG, INFO, WARN, ERROR
TF_LOG_PATH=./tf.log TF_LOG=TRACE terraform apply
```

Log chứa cả giá trị nhạy cảm: đừng dán công khai và xoá sau khi xong.

## Danh sách lỗi hay gặp

| Triệu chứng | Nguyên nhân thường gặp | Cách xử lý |
|---|---|---|
| `Error acquiring the state lock` | Có lần chạy khác đang giữ khoá, hoặc lần chạy trước bị ngắt giữa chừng | Chờ lần kia xong; nếu chắc chắn không còn ai chạy, `terraform force-unlock LOCK_ID` |
| Lỗi xác thực: `NoCredentialProviders`, `ExpiredToken`, `AccessDenied` | Chưa đăng nhập, token hết hạn, vai trò thiếu quyền | Kiểm tra `aws sts get-caller-identity`; đăng nhập lại; xem chính sách IAM của vai trò |
| `already exists` / `EntityAlreadyExists` | Tài nguyên đã tồn tại thật nhưng chưa nằm trong state | `terraform import` (hoặc khối `import`), rồi plan cho tới khi sạch |
| Plan muốn **xoá và tạo lại** tài nguyên | Đổi thuộc tính "bắt buộc thay thế" (`forces replacement`), hoặc đổi tên mà quên `moved`, hoặc đổi chỉ số `count` | Đọc dòng `# forces replacement`; thêm `moved`; dùng `for_each`; với dữ liệu thì `prevent_destroy` |
| `Cycle: ...` | Hai tài nguyên tham chiếu lẫn nhau | Tách thuộc tính gây vòng (ví dụ security group rule thành tài nguyên riêng) |
| `Provider produced inconsistent result after apply` | Lỗi của provider hoặc tài nguyên đổi giữa lúc tạo | Chạy lại plan/apply; nếu lặp lại, kiểm tra phiên bản provider và báo lỗi |
| `Invalid for_each argument` ... `known only after apply` | Khoá của `for_each` phụ thuộc giá trị chỉ biết sau khi apply | Dùng khoá tĩnh, hoặc tách thành hai lần apply, hoặc `-target` tạm thời |
| Plan lúc nào cũng hiện thay đổi dù không sửa gì | Drift do hệ thống khác sửa thuộc tính; thuộc tính do provider chuẩn hoá khác mã | `ignore_changes`, hoặc viết mã khớp giá trị thực tế |
| `Error: Unsupported argument` sau khi nâng provider | Thuộc tính đã bị đổi tên hoặc bỏ | Đọc changelog và hướng dẫn nâng cấp của provider |
| Apply dừng giữa chừng, một phần tài nguyên đã tạo | Lỗi API, hết quota, mất mạng | **Không xoá state.** Sửa nguyên nhân và chạy lại `apply`: những gì đã tạo đã nằm trong state |
| Xoá nhầm tài nguyên | Chạy `destroy`, hoặc plan có dòng `-` bị bỏ qua | Khôi phục từ snapshot/backup; dựng lại bằng mã; về sau dùng `prevent_destroy` và review plan |

## Tình huống: state bị lệch với thực tế

1. Sao lưu: `terraform state pull > state-backup.json`.
2. `terraform plan -refresh-only` để thấy drift.
3. Quyết định: đưa thực tế về lại mã (apply), hay sửa mã cho khớp thực tế, hay `import` thứ đang thiếu trong state, hay `state rm` thứ không còn nên quản lý.
4. Sau khi xử lý, `plan` phải **sạch** (không còn thay đổi ngoài ý muốn).

## Tình huống: tài nguyên mất trong state nhưng còn trong thực tế

Plan sẽ muốn tạo lại và thất bại vì trùng tên. Dùng `terraform import` (hoặc khối `import` với `plan -generate-config-out`) để đưa nó trở lại, rồi chỉnh mã cho khớp.

## Tình huống: chạy nhầm môi trường

Triệu chứng: plan muốn xoá cả hệ thống, vì đang trỏ vào backend hoặc workspace khác. **Dừng, không apply.** Kiểm tra `terraform workspace show`, thư mục hiện tại, `terraform init` đã dùng backend nào. Phòng ngừa: thư mục theo môi trường, tên bucket/tài khoản riêng, pipeline là nơi duy nhất apply production, và `prevent_destroy` cho tài nguyên quan trọng.

## Mẹo gỡ lỗi

```
terraform console                            # thử biểu thức với state hiện tại
terraform graph | dot -Tpng > g.png          # đồ thị phụ thuộc
terraform providers                          # provider nào, phiên bản nào
terraform state list | grep NAME
terraform plan -target=ADDR                  # thu hẹp để debug (không dùng làm thói quen)
terraform validate -json
```

::: lab Gây và sửa hai lỗi
```
mkdir -p ~/tf7 && cd ~/tf7
cat > main.tf <<'EOF'
terraform {
  required_providers {
    local = { source = "hashicorp/local" }
  }
}
resource "local_file" "x" {
  filename = "${path.module}/x.txt"
  content  = "một"
}
EOF
terraform init >/dev/null
# Lỗi 1: tài nguyên có thật nhưng không có trong state
echo "đã tồn tại" > x.txt; terraform plan | grep -E "will be|Plan:"      # muốn ghi đè
# Lỗi 2: đổi tên không có moved => xoá và tạo lại
terraform apply -auto-approve >/dev/null
sed -i 's/"x"/"y"/' main.tf; terraform plan | grep -E "destroyed|created|Plan:"
# Sửa: thêm moved
printf 'moved {\n  from = local_file.x\n  to   = local_file.y\n}\n' >> main.tf; terraform plan | grep -E "moved|Plan:|No changes"
terraform apply -auto-approve >/dev/null; terraform destroy -auto-approve >/dev/null   # dọn dẹp
```
Với mỗi lỗi: thông báo nói gì, nguyên nhân là gì, và thao tác nào là an toàn nhất? Thử thêm: mở hai terminal chạy `terraform apply` cùng lúc trên một backend S3 hoặc cục bộ để thấy khoá state hoạt động.
:::
