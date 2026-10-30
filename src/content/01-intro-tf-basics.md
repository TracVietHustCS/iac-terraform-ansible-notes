=== overview | Bắt đầu | Tổng quan và cách học
>> Bộ ghi chú về Infrastructure as Code (IaC) với hai công cụ phổ biến nhất: Terraform để dựng hạ tầng và Ansible để cấu hình máy. Đi từ nguyên tắc, workflow, state, module, đến CI/CD, kiểm thử, bảo mật và cách kết hợp hai công cụ.

::: warn Đọc trước khi dùng
Đây là ghi chú **tự học, tự soạn**, không phải tài liệu chính thức của HashiCorp, IBM hay Red Hat. Cú pháp, tên provider và tính năng thay đổi theo phiên bản (nhất là các tính năng mới của Terraform như `import` block, `terraform test`, khoá state bằng S3). Ghi chú không tra cứu trực tuyến nên có thể lỗi thời: hãy đối chiếu tài liệu hiện hành và chạy thử ở môi trường lab. `terraform apply`, `destroy` và `ansible-playbook` có thể thay đổi hoặc xoá tài nguyên thật.
:::

## Bạn sẽ làm được gì sau khi học xong

- Giải thích IaC giải quyết vấn đề gì, và khác nhau giữa mô hình khai báo (declarative) và thủ tục (procedural).
- Dùng Terraform đúng workflow: `init`, `plan`, `apply`; hiểu state, backend, module, nhiều môi trường.
- Đưa Terraform vào CI/CD an toàn: plan trên pull request, apply sau khi duyệt, kiểm tra chính sách và bảo mật.
- Viết playbook, role, inventory Ansible idempotent; quản lý bí mật bằng Vault; triển khai cuốn chiếu.
- Chọn đúng công cụ cho đúng việc và kết hợp Terraform với Ansible.

## Cấu trúc

| Nhóm | Trang | Nội dung |
|---|---|---|
| Bắt đầu | [Nguyên tắc IaC](#iac-concepts) | Khai báo, idempotent, drift, bất biến |
| Terraform | [Workflow](#tf-workflow), [HCL](#tf-hcl), [Biến và output](#tf-variables), [Resource](#tf-resources), [State](#tf-state), [Module](#tf-modules), [Ví dụ AWS](#tf-aws), [Nhiều môi trường](#tf-envs), [CI/CD và kiểm thử](#tf-pipeline), [Xử lý sự cố](#tf-troubleshooting) | Từ cơ bản tới vận hành nhóm |
| Ansible | [Cơ bản](#an-basics), [Playbook](#an-playbooks), [Role](#an-roles), [Mẫu thực tế](#an-patterns) | Cấu hình máy bằng SSH, không agent |
| Kết hợp | [Terraform và Ansible](#tf-vs-ansible), [Bảo mật IaC](#iac-security) | Phối hợp và rủi ro |
| Tham chiếu | [Cheat sheet](#iac-cheatsheet), [Luyện tập Terraform](#quiz-tf), [Luyện tập Ansible](#quiz-ansible) | Tra nhanh và tự kiểm tra |

## Môi trường lab (không cần cloud)

Phần lớn lab chạy được trên máy của bạn:

| Cần | Cách |
|---|---|
| Terraform | Cài theo hướng dẫn chính thức. Có thể dùng **OpenTofu** (bản fork mã nguồn mở, lệnh là `tofu`, cú pháp gần như tương thích) |
| Provider không cần cloud | `local`, `random`, `null`, và `docker` để dựng container thật |
| Ansible | `pipx install ansible` (hoặc `pip install ansible`), chạy thử với `localhost` hoặc một VM Multipass/Docker có SSH |
| Cloud (tuỳ chọn) | Tài khoản AWS free tier cho trang [Ví dụ AWS](#tf-aws); luôn đặt cảnh báo chi phí và `terraform destroy` sau khi thử |

```
terraform version
ansible --version
```

::: tip Giấy phép Terraform
Từ giữa năm 2023, HashiCorp chuyển Terraform sang giấy phép Business Source License (không còn là mã nguồn mở theo định nghĩa OSI), và cộng đồng tạo ra nhánh **OpenTofu** do Linux Foundation quản lý. Với việc học và dùng nội bộ thường không có khác biệt thực tế; nếu bạn xây sản phẩm thương mại xung quanh Terraform, hãy đọc điều khoản hiện hành.
:::

Liên quan: máy ảo và container mà Ansible cấu hình xem [Linux và mạng](https://tracviethustcs.github.io/linux-network-notes/) và [Docker](https://tracviethustcs.github.io/docker-container-notes/); phần triển khai ứng dụng trên Kubernetes dùng GitOps ([Flux, Argo CD](https://tracviethustcs.github.io/k8s-study-notes/#gitops)); tự động hoá pipeline xem [CI/CD](https://tracviethustcs.github.io/cicd-release-notes/).

=== iac-concepts | Bắt đầu | Nguyên tắc Infrastructure as Code
>> IaC nghĩa là mô tả hạ tầng bằng tệp văn bản được quản lý phiên bản, để việc dựng và thay đổi hạ tầng lặp lại được, xem lại được và kiểm thử được như mã nguồn.

## Vì sao cần IaC

Dựng hạ tầng bằng tay (click trên console, SSH vào chỉnh) gây ra: không lặp lại được (mỗi máy một khác), không ai biết **vì sao** có thay đổi, khó khôi phục khi sự cố, và kiến thức nằm trong đầu một vài người. IaC đổi điều đó: hạ tầng nằm trong Git, thay đổi qua pull request có review, và có thể dựng lại từ đầu bằng một lệnh.

## Khai báo và thủ tục

| | Khai báo (declarative) | Thủ tục (procedural) |
|---|---|---|
| Bạn mô tả | **Trạng thái cuối mong muốn** | **Các bước** cần làm |
| Công cụ tính | Công cụ tự suy ra cần tạo, sửa, xoá gì | Bạn tự lo thứ tự và điều kiện |
| Ví dụ | Terraform, CloudFormation, Kubernetes manifest | Script shell, một số playbook viết kiểu "chạy lệnh" |
| Ưu | Chạy lại an toàn, thấy trước thay đổi (plan) | Linh hoạt cho việc đặc thù |

Ansible nằm giữa hai bên: module của nó mang tính khai báo (`state: present`), nhưng playbook là danh sách tác vụ theo thứ tự.

## Các khái niệm nền

- **Idempotent**: chạy một thao tác nhiều lần cho cùng kết quả như chạy một lần. Với IaC, đây là điều kiện để dám chạy lại. Module `apt` với `state: present` idempotent; `command: useradd bob` thì không (lần hai báo lỗi).
- **Drift** (lệch cấu hình): thực tế khác với mã, do ai đó sửa tay, hoặc hệ thống tự thay đổi. Phát hiện bằng `terraform plan`, chạy playbook ở chế độ `--check`, hoặc công cụ GitOps tự đối chiếu liên tục.
- **Hạ tầng bất biến (immutable)** so với **có thể biến đổi (mutable)**: bất biến là thay vì sửa máy đang chạy thì dựng máy mới từ image rồi bỏ máy cũ (image dựng bằng Packer, hoặc container); biến đổi là cập nhật tại chỗ (thường dùng Ansible). Bất biến dễ dự đoán và quay lui hơn; biến đổi tiện cho máy sống lâu và thay đổi nhỏ.
- **Blast radius** (bán kính ảnh hưởng): một thay đổi hỏng ảnh hưởng tới bao nhiêu thứ. Chia nhỏ state và môi trường để một lỗi không làm hỏng tất cả.
- **Nguồn sự thật duy nhất**: Git là nơi mô tả hạ tầng đúng; sửa tay ngoài Git là nợ cần trả.

## Quy tắc làm việc

1. Mọi thay đổi hạ tầng đi qua **Git và review**.
2. **Xem trước khi áp dụng** (`terraform plan`, `ansible-playbook --check --diff`), và đọc kỹ phần xoá hoặc thay thế.
3. **Không sửa tay** tài nguyên do IaC quản lý; nếu bắt buộc thì ghi lại và đưa vào mã ngay.
4. **Bí mật không nằm trong mã** (xem [Bảo mật IaC](#iac-security)).
5. Tách **môi trường** và **state** để giảm blast radius.
6. Kiểm thử và chạy tự động trong **pipeline**, không từ laptop cá nhân khi đã có nhóm.
7. Ghim phiên bản công cụ và provider để kết quả lặp lại được.

## Công cụ trong bản đồ

| Việc | Công cụ phổ biến |
|---|---|
| Dựng hạ tầng (cloud, mạng, dịch vụ) | Terraform, OpenTofu, CloudFormation, Pulumi, Crossplane |
| Cấu hình máy (gói, file, dịch vụ) | Ansible, Chef, Puppet, Salt |
| Dựng image máy ảo | Packer |
| Triển khai ứng dụng lên Kubernetes | Helm, Kustomize, Flux, Argo CD |

Hai ghi chú này tập trung vào Terraform và Ansible vì chúng bổ sung cho nhau: Terraform "dựng nhà", Ansible "bày đồ vào nhà" (xem [Terraform và Ansible](#tf-vs-ansible)).

=== tf-workflow | Terraform | Cài đặt và workflow
>> Terraform đọc mô tả hạ tầng bằng HCL, so sánh với state, lập kế hoạch thay đổi rồi thực hiện qua provider. Vòng lặp `init`, `plan`, `apply` là cốt lõi của mọi thứ.

## Terraform làm gì

```ascii
   *.tf (mã HCL)  --+
                    |--> terraform plan --> so sánh --> kế hoạch: +tạo ~sửa -xoá
   state            --+        ^                          |
   (cái đã tạo)               |                          v
                       API của nhà cung cấp  <--- terraform apply (qua provider)
                       (AWS, Azure, Docker...)
```

- **Configuration** (`.tf`): mô tả trạng thái mong muốn.
- **Provider**: plugin nói chuyện với một API (AWS, Azure, GCP, Kubernetes, Docker, GitHub...).
- **State**: bản ghi Terraform đã tạo gì và thuộc tính của chúng (xem [State](#tf-state)).
- **Plan**: so mã với state và thực tế, sinh danh sách thay đổi cần làm.

## Workflow cơ bản

```
terraform init        # tải provider và module, khởi tạo backend (chạy lại khi đổi provider/module/backend)
terraform fmt         # định dạng mã chuẩn
terraform validate    # kiểm tra cú pháp và tham chiếu (không gọi API)
terraform plan        # xem trước thay đổi
terraform apply       # thực hiện (hỏi xác nhận)
terraform destroy     # xoá mọi thứ do cấu hình này quản lý
```

Trong CI nên tách plan và apply, lưu plan thành tệp để apply đúng cái đã duyệt:

```
terraform plan -out=tfplan
terraform show tfplan                 # xem lại kế hoạch
terraform apply tfplan                # áp dụng đúng kế hoạch đó
```

Ký hiệu trong plan: `+` tạo mới, `-` xoá, `~` sửa tại chỗ, `-/+` **thay thế** (xoá rồi tạo lại: chú ý nếu là database hay ổ đĩa), `<=` đọc data source. Dòng cuối `Plan: 2 to add, 1 to change, 0 to destroy.` là thứ phải đọc kỹ nhất.

## Cấu trúc thư mục

```
.
├── main.tf              # tài nguyên chính
├── variables.tf         # khai báo biến
├── outputs.tf           # giá trị xuất ra
├── providers.tf         # cấu hình provider và phiên bản
├── terraform.tfvars     # giá trị biến (không chứa bí mật nếu commit)
├── .terraform.lock.hcl  # khoá phiên bản provider: COMMIT file này
└── .terraform/          # thư mục làm việc: KHÔNG commit
```

Mọi file `.tf` trong cùng thư mục được ghép thành một cấu hình; tên file chỉ là quy ước.

## Ghim phiên bản

```
terraform {
  required_version = ">= 1.6.0, < 2.0.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"          # >= 5.0, < 6.0
    }
  }
}
```

- `required_version` giới hạn phiên bản Terraform; `version` giới hạn provider.
- `.terraform.lock.hcl` ghi lại phiên bản và mã băm chính xác của provider đã chọn: **commit** để mọi người và CI dùng cùng bản. Nâng cấp có chủ đích bằng `terraform init -upgrade`.
- Mỗi lần nâng provider lớn có thể có thay đổi phá vỡ: đọc changelog và chạy plan trên môi trường thử.

## .gitignore điển hình

```
.terraform/
*.tfstate
*.tfstate.*
crash.log
*.tfplan
terraform.tfvars        # nếu chứa giá trị nhạy cảm
override.tf
```

::: lab Vòng lặp đầu tiên, không cần cloud
```
mkdir -p ~/tf1 && cd ~/tf1
cat > main.tf <<'EOF'
terraform {
  required_providers {
    local  = { source = "hashicorp/local" }
    random = { source = "hashicorp/random" }
  }
}
resource "random_pet" "name" { length = 2 }
resource "local_file" "hello" {
  filename = "${path.module}/hello-${random_pet.name.id}.txt"
  content  = "Xin chào từ Terraform\n"
}
output "file" { value = local_file.hello.filename }
EOF
terraform init
terraform plan
terraform apply -auto-approve
ls; cat hello-*.txt
terraform plan                 # không còn thay đổi: idempotent
rm hello-*.txt; terraform plan # phát hiện drift: file đã bị xoá tay
terraform apply -auto-approve  # tạo lại
terraform destroy -auto-approve
```
Quan sát: lần plan thứ hai nói gì, và lần plan sau khi xoá tay file nói gì? Đây là drift mà plan phát hiện.
:::

=== tf-hcl | Terraform | Ngôn ngữ HCL: khối, kiểu dữ liệu và biểu thức
>> HCL (HashiCorp Configuration Language) là ngôn ngữ khai báo đọc được của Terraform. Chỉ có vài loại khối và một số biểu thức cần nắm để đọc hiểu hầu hết mã Terraform.

## Khối và đối số

```
resource "aws_instance" "web" {        # khối: loại, nhãn loại tài nguyên, nhãn tên
  ami           = "ami-0abcdef"         # đối số: tên = giá trị
  instance_type = "t3.micro"
  tags = {                              # map
    Name = "web"
  }
  root_block_device {                   # khối lồng nhau
    volume_size = 20
  }
}
```

| Khối | Dùng để |
|---|---|
| `terraform` | Cấu hình Terraform: phiên bản, provider cần, backend |
| `provider` | Cấu hình một provider (vùng, xác thực) |
| `resource` | Khai báo một tài nguyên cần quản lý |
| `data` | Đọc thông tin đã có, không tạo |
| `variable` | Đầu vào |
| `output` | Giá trị đầu ra |
| `locals` | Giá trị tính toán nội bộ |
| `module` | Gọi module khác |

Tham chiếu tới tài nguyên: `LOẠI.TÊN.THUỘC_TÍNH`, ví dụ `aws_instance.web.id`; tới biến: `var.region`; tới local: `local.name`; tới data: `data.aws_ami.ubuntu.id`; tới module: `module.vpc.vpc_id`.

## Kiểu dữ liệu

| Kiểu | Ví dụ |
|---|---|
| `string`, `number`, `bool` | `"abc"`, `3`, `true` |
| `list(T)` | `["a", "b"]` |
| `set(T)` | Danh sách không trùng, không có thứ tự |
| `map(T)` | `{ env = "dev", team = "x" }` |
| `object({...})` | `{ name = string, size = number }` |
| `tuple([...])` | Danh sách với kiểu từng phần tử |

## Biểu thức

```
locals {
  name_prefix = "${var.project}-${var.env}"              # nội suy chuỗi
  is_prod     = var.env == "prod"
  size        = local.is_prod ? "m5.large" : "t3.micro"  # điều kiện
  upper_names = [for n in var.names : upper(n)]          # vòng for tạo list
  by_name     = { for s in var.servers : s.name => s }   # tạo map
  first_az    = var.azs[0]
  merged_tags = merge(var.common_tags, { Name = local.name_prefix })
}
```

Chuỗi nhiều dòng dùng heredoc:

```
user_data = <<-EOT
  #!/bin/bash
  echo "xin chào" > /tmp/hello
EOT
```

## Hàm hay dùng

| Nhóm | Hàm |
|---|---|
| Chuỗi | `format`, `join`, `split`, `replace`, `lower`, `upper`, `trimspace` |
| Tập hợp | `length`, `merge`, `lookup`, `concat`, `flatten`, `distinct`, `keys`, `values`, `toset`, `tolist` |
| Số và điều kiện | `min`, `max`, `ceil`, `coalesce`, `try`, `can` |
| Tệp | `file`, `templatefile`, `filebase64`, `fileexists` |
| Mã hoá | `base64encode`, `jsonencode`, `yamlencode`, `sha256` |
| Mạng | `cidrsubnet`, `cidrhost`: chia dải IP |

`cidrsubnet("10.0.0.0/16", 8, 2)` cho `10.0.2.0/24`. Dùng `terraform console` để thử biểu thức tương tác:

```
terraform console
> cidrsubnet("10.0.0.0/16", 8, 2)
> [for n in ["a","b"] : upper(n)]
```

## Quy ước viết tốt

- Đặt tên tài nguyên bằng chữ thường và gạch dưới, mô tả vai trò (`aws_security_group.web`, không phải `sg1`).
- Một khối `locals` cho giá trị lặp lại; tránh lặp chuỗi.
- Chạy `terraform fmt` (và trong CI `terraform fmt -check`).
- Chú thích bằng `#` hoặc `//`; giải thích **vì sao**, không lặp lại cái mã đã nói.

=== tf-variables | Terraform | Biến, local và output
>> Biến làm cấu hình tái sử dụng được, output chuyển thông tin ra ngoài (hoặc sang module khác). Kiểm tra giá trị đầu vào và đánh dấu nhạy cảm là hai thói quen đáng có.

## Khai báo biến

```
variable "env" {
  type        = string
  description = "Tên môi trường: dev, stg hoặc prod"
  default     = "dev"

  validation {
    condition     = contains(["dev", "stg", "prod"], var.env)
    error_message = "env phải là dev, stg hoặc prod."
  }
}

variable "instance_count" {
  type    = number
  default = 2
}

variable "db_password" {
  type      = string
  sensitive = true          # ẩn khỏi output của plan/apply
}

variable "subnets" {
  type = list(object({
    name = string
    cidr = string
  }))
}
```

- Biến không có `default` thì **bắt buộc** phải truyền giá trị.
- `validation` cho thông báo lỗi sớm và rõ, trước khi gọi API.
- `sensitive = true` chỉ ẩn khỏi **hiển thị trên màn hình**; giá trị **vẫn nằm trong state** dưới dạng văn bản rõ: xem [State](#tf-state) và [Bảo mật IaC](#iac-security).

## Cách truyền giá trị và thứ tự ưu tiên

Từ thấp tới cao (cái sau thắng cái trước):

1. Giá trị `default` trong khai báo.
2. Biến môi trường `TF_VAR_tên` (ví dụ `TF_VAR_db_password`).
3. `terraform.tfvars` (hoặc `terraform.tfvars.json`).
4. Các file `*.auto.tfvars` (theo thứ tự chữ cái).
5. Tuỳ chọn dòng lệnh `-var` và `-var-file` (theo thứ tự xuất hiện).

```
terraform apply -var-file=envs/prod.tfvars -var 'instance_count=4'
export TF_VAR_db_password='...'       # bí mật: dùng biến môi trường, không ghi vào file commit
```

Ví dụ `envs/prod.tfvars`:

```
env            = "prod"
instance_count = 4
subnets = [
  { name = "a", cidr = "10.0.1.0/24" },
  { name = "b", cidr = "10.0.2.0/24" },
]
```

## Locals

```
locals {
  name = "${var.project}-${var.env}"
  common_tags = {
    Project = var.project
    Env     = var.env
    Owner   = "platform"
  }
}
```

`locals` là giá trị tính toán **nội bộ** của module, không phải đầu vào: dùng để tránh lặp và đặt tên cho biểu thức phức tạp.

## Output

```
output "web_url" {
  value       = "http://${aws_instance.web.public_dns}"
  description = "Địa chỉ truy cập web"
}

output "db_endpoint" {
  value     = aws_db_instance.main.endpoint
  sensitive = true
}
```

```
terraform output                 # liệt kê
terraform output -raw web_url    # in giá trị thô, tiện dùng trong script
terraform output -json
```

Output là cách module **trả kết quả** cho nơi gọi nó, và cách một cấu hình chia sẻ thông tin với cấu hình khác (qua `terraform_remote_state` hoặc data source).

## Dữ liệu từ bên ngoài (data source)

```
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]            # Canonical
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-noble-24.04-amd64-server-*"]
  }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"
}
```

`data` **đọc** thông tin đã tồn tại (AMI mới nhất, VPC đã có, tài khoản AWS hiện tại) mà không quản lý nó. Cẩn thận với data source có "mới nhất": kết quả đổi theo thời gian có thể gây thay thế tài nguyên ngoài ý muốn.

::: lab Biến, validation và thứ tự ưu tiên
```
mkdir -p ~/tf2 && cd ~/tf2
cat > main.tf <<'EOF'
variable "env" {
  type = string
  default = "dev"
  validation {
    condition     = contains(["dev", "stg", "prod"], var.env)
    error_message = "env phải là dev, stg hoặc prod."
  }
}
locals { name = "app-${var.env}" }
output "name" { value = local.name }
EOF
terraform init -input=false >/dev/null
terraform apply -auto-approve | grep name
echo 'env = "stg"' > terraform.tfvars; terraform apply -auto-approve | grep name
TF_VAR_env=prod terraform apply -auto-approve | grep name
terraform apply -auto-approve -var env=dev | grep name
terraform apply -auto-approve -var env=test        # lỗi validation
```
Ghi lại kết quả mỗi lần và đối chiếu với thứ tự ưu tiên ở trên.
:::
