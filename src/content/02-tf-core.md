=== tf-resources | Terraform | Resource, phụ thuộc và meta-argument
>> Resource là đơn vị cơ bản của Terraform. Hiểu cách Terraform suy ra thứ tự, và các công cụ để tạo nhiều bản (`count`, `for_each`) và kiểm soát vòng đời, giúp tránh hầu hết những bất ngờ khi `apply`.

## Phụ thuộc

Terraform dựng **đồ thị phụ thuộc** từ các tham chiếu giữa tài nguyên, rồi tạo song song những thứ không phụ thuộc nhau và theo thứ tự những thứ phụ thuộc.

```
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "a" {
  vpc_id     = aws_vpc.main.id        # tham chiếu => phụ thuộc ngầm: VPC trước, subnet sau
  cidr_block = "10.0.1.0/24"
}
```

Khi sự phụ thuộc **không** thể hiện qua tham chiếu (ví dụ cần một quyền IAM tồn tại trước nhưng không dùng thuộc tính nào của nó), dùng `depends_on` tường minh, nhưng chỉ khi cần vì nó làm giảm tính song song:

```
resource "aws_instance" "app" {
  # ...
  depends_on = [aws_iam_role_policy.app]
}
```

Xem đồ thị: `terraform graph | dot -Tpng > graph.png` (cần Graphviz).

## count và for_each

Tạo nhiều bản của một tài nguyên:

```
# count: nhiều bản đánh số 0, 1, 2...
resource "aws_instance" "web" {
  count         = var.instance_count
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"
  tags          = { Name = "web-${count.index}" }
}

# for_each: mỗi phần tử một khoá ổn định
resource "aws_subnet" "this" {
  for_each          = { for s in var.subnets : s.name => s }
  vpc_id            = aws_vpc.main.id
  cidr_block        = each.value.cidr
  availability_zone = each.value.az
  tags              = { Name = each.key }
}
```

| | `count` | `for_each` |
|---|---|---|
| Định danh | Chỉ số (`web[0]`, `web[1]`) | Khoá (`subnet["a"]`) |
| Xoá phần tử ở giữa | Các phần tử sau **bị đánh số lại**, có thể bị thay thế | Chỉ phần tử đó bị xoá |
| Dùng khi | Bản giống hệt nhau, hoặc bật/tắt (`count = var.enabled ? 1 : 0`) | Mỗi bản có danh tính riêng |

Quy tắc thực tế: ưu tiên `for_each` với khoá có ý nghĩa; chỉ dùng `count` cho số lượng thuần tuý hoặc cờ bật/tắt. `for_each` cần tập khoá **biết được lúc plan**.

## lifecycle

```
resource "aws_db_instance" "main" {
  # ...
  lifecycle {
    prevent_destroy       = true             # từ chối plan nào xoá tài nguyên này
    create_before_destroy = true             # khi phải thay thế: tạo cái mới trước rồi mới xoá cái cũ
    ignore_changes        = [tags["LastScan"]]   # bỏ qua thay đổi do hệ thống khác sửa
    replace_triggered_by  = [aws_launch_template.app]
  }
}
```

| Thuộc tính | Dùng để |
|---|---|
| `prevent_destroy` | Bảo vệ tài nguyên quan trọng (database, bucket dữ liệu) khỏi bị xoá nhầm |
| `create_before_destroy` | Thay thế không gián đoạn (thường với thứ đứng sau load balancer) |
| `ignore_changes` | Chấp nhận có thứ do bên ngoài thay đổi (autoscaling đổi số lượng, tag do công cụ khác) |
| `replace_triggered_by` | Thay thế tài nguyên khi một tài nguyên khác đổi |

## Một số khối đáng biết

**dynamic block**: tạo các khối lồng nhau lặp theo dữ liệu.

```
resource "aws_security_group" "web" {
  name   = "web"
  vpc_id = aws_vpc.main.id
  dynamic "ingress" {
    for_each = var.allowed_ports
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = var.allowed_cidrs
    }
  }
}
```

**precondition và postcondition**: kiểm tra giả định ngay trong `lifecycle` (có thông báo lỗi rõ).

**provisioner** (`local-exec`, `remote-exec`): chạy lệnh sau khi tạo tài nguyên. Tài liệu của Terraform coi đây là **phương án cuối cùng**: kết quả không nằm trong state, khó lặp lại. Việc cấu hình máy nên giao cho `cloud-init`, image dựng sẵn hoặc Ansible ([Terraform và Ansible](#tf-vs-ansible)).

## Thay thế và xoá có chủ đích

```
terraform apply -replace="aws_instance.web[0]"     # ép tạo lại một tài nguyên
terraform destroy -target=aws_instance.web         # chỉ dùng khi thật cần
```

`-target` bỏ qua đồ thị đầy đủ, nên chỉ dùng cho tình huống đặc biệt, không phải thói quen. Lệnh `terraform taint` cũ đã được thay bằng `-replace`.

::: lab count và for_each khi xoá phần tử ở giữa
```
mkdir -p ~/tf3 && cd ~/tf3
cat > main.tf <<'EOF'
terraform {
  required_providers {
    local = { source = "hashicorp/local" }
  }
}
variable "names" { default = ["a", "b", "c"] }
resource "local_file" "by_count" {
  count    = length(var.names)
  filename = "${path.module}/count-${var.names[count.index]}.txt"
  content  = var.names[count.index]
}
resource "local_file" "by_key" {
  for_each = toset(var.names)
  filename = "${path.module}/key-${each.key}.txt"
  content  = each.key
}
EOF
terraform init >/dev/null && terraform apply -auto-approve | tail -2
terraform plan -var='names=["a","c"]'        # bỏ "b"
```
Quan sát: với `count`, bỏ `b` thì `count-c.txt` có bị đổi không? Với `for_each` thì sao? Đây là lý do nên chọn `for_each` khi mỗi phần tử có danh tính riêng.
:::

=== tf-state | Terraform | State: lưu trữ, khoá và các thao tác
>> State là bộ nhớ của Terraform: nó ghi Terraform đã tạo những gì và ánh xạ mã sang tài nguyên thật. Hầu hết sự cố nghiêm trọng của Terraform đều liên quan tới state.

## State chứa gì và vì sao quan trọng

Tệp `terraform.tfstate` (JSON) lưu danh sách tài nguyên đã quản lý, thuộc tính của chúng, và ánh xạ giữa khối trong mã và ID thật (ví dụ `aws_instance.web` là `i-0abc...`). Khi plan, Terraform đọc state, làm mới thông tin từ API rồi so với mã.

Hệ quả:

- **Mất state** nghĩa là Terraform "quên" mọi thứ nó đã tạo: lần apply sau sẽ cố tạo lại tất cả.
- **State chứa dữ liệu nhạy cảm** dưới dạng văn bản rõ (mật khẩu database, khoá, `sensitive` cũng không được mã hoá trong state). Bảo vệ state như bí mật.
- **Không sửa tay** file state. Dùng các lệnh `terraform state`.
- **Không commit** state vào Git.

## Backend từ xa

Với làm việc nhóm, state phải nằm ở **backend từ xa** có **khoá** (để hai người không apply cùng lúc) và **mã hoá, sao lưu**.

```
terraform {
  backend "s3" {
    bucket       = "myorg-terraform-state"
    key          = "prod/network/terraform.tfstate"
    region       = "ap-southeast-1"
    encrypt      = true
    use_lockfile = true          # khoá bằng chính S3 (các bản Terraform mới)
    # dynamodb_table = "tf-locks"   # cách cũ: khoá bằng DynamoDB
  }
}
```

- Bật **versioning** cho bucket để khôi phục state cũ khi hỏng; chặn truy cập công khai; giới hạn quyền ghi bằng IAM.
- Cách khoá bằng S3 (`use_lockfile`) có ở các bản Terraform gần đây; trước đó dùng bảng DynamoDB. Kiểm tra theo phiên bản bạn dùng.
- Các backend khác: Azure Blob, GCS, Terraform Cloud/HCP Terraform, GitLab, Consul, PostgreSQL.
- Đổi backend rồi chạy `terraform init -migrate-state` để chuyển state.

Mỗi **key** là một state riêng. Chia state theo ranh giới hợp lý (mạng, dữ liệu, ứng dụng, từng môi trường) để giảm blast radius và thời gian plan.

## Chia sẻ thông tin giữa các state

```
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "myorg-terraform-state"
    key    = "prod/network/terraform.tfstate"
    region = "ap-southeast-1"
  }
}
# dùng: data.terraform_remote_state.network.outputs.vpc_id
```

Cách này gắn chặt hai cấu hình và đòi quyền đọc toàn bộ state. Thay thế thường tốt hơn: dùng data source tra cứu tài nguyên bằng tag, hoặc SSM Parameter Store.

## Các lệnh state

```
terraform state list                              # liệt kê tài nguyên trong state
terraform state show aws_instance.web             # chi tiết một tài nguyên
terraform state mv aws_instance.web aws_instance.app    # đổi địa chỉ (khi đổi tên/di chuyển trong mã)
terraform state rm aws_instance.legacy            # gỡ khỏi state mà không xoá tài nguyên thật
terraform import aws_instance.web i-0abc123       # đưa tài nguyên đã có vào state
terraform plan -refresh-only                      # chỉ cập nhật state theo thực tế, xem drift
terraform force-unlock LOCK_ID                    # gỡ khoá bị kẹt (cẩn thận)
```

## Đổi tên và tái cấu trúc không phá huỷ

Đổi tên tài nguyên trong mã mà không báo cho Terraform sẽ bị hiểu là "xoá cái cũ, tạo cái mới". Khai báo trong mã bằng khối `moved` (Terraform 1.1 trở lên) để giữ nguyên tài nguyên thật:

```
moved {
  from = aws_instance.web
  to   = aws_instance.app
}
```

Để **bỏ** một tài nguyên khỏi quản lý mà không xoá nó thật, dùng khối `removed` (Terraform 1.7 trở lên) với `lifecycle { destroy = false }`, hoặc `terraform state rm`.

## Import tài nguyên đã có

Hạ tầng dựng tay từ trước có thể đưa vào Terraform. Từ Terraform 1.5 có khối `import` khai báo ngay trong mã và có thể sinh cấu hình mẫu:

```
import {
  to = aws_s3_bucket.logs
  id = "myorg-logs-bucket"
}
```
```
terraform plan -generate-config-out=generated.tf     # sinh mã mẫu từ tài nguyên thật (xem lại rồi chỉnh)
terraform apply
```

Sau khi import, chạy `plan` cho tới khi **không còn khác biệt**: mã phải khớp thực tế, nếu không lần apply sau sẽ đổi tài nguyên thật.

## Drift và làm mới

`terraform plan` tự làm mới thông tin từ API. `plan -refresh-only` cho thấy chỉ phần drift (thực tế khác state). Khi có drift, chọn một trong hai: **đưa thực tế về lại mã** (apply) hoặc **sửa mã cho khớp thực tế** (nếu thay đổi tay là đúng).

::: warn Không bao giờ xoá state "cho gọn"
Xoá hoặc đổi tên file state thật sự khiến Terraform mất liên kết với tài nguyên đã tạo, dẫn tới tạo trùng hoặc rác không ai quản lý. Với backend từ xa, bật versioning và hạn chế ai có quyền ghi vào bucket. Trước thao tác rủi ro (`state mv`, `state rm`, đổi backend), sao lưu state: `terraform state pull > backup.tfstate`.
:::

::: lab State, drift và state mv
```
mkdir -p ~/tf4 && cd ~/tf4
cat > main.tf <<'EOF'
terraform {
  required_providers {
    local = { source = "hashicorp/local" }
  }
}
resource "local_file" "a" {
  filename = "${path.module}/a.txt"
  content  = "một"
}
EOF
terraform init >/dev/null && terraform apply -auto-approve >/dev/null
terraform state list && terraform state show local_file.a | head -8
rm a.txt                                  # giả lập ai đó xoá tay tài nguyên
terraform plan -refresh-only              # thấy drift: tài nguyên không còn trong thực tế
terraform apply -auto-approve >/dev/null  # tạo lại file, đưa thực tế về khớp mã
# đổi tên trong mã nhưng giữ tài nguyên: dùng moved
sed -i 's/local_file" "a"/local_file" "b"/' main.tf
printf 'moved {\n  from = local_file.a\n  to   = local_file.b\n}\n' >> main.tf
terraform plan                            # không xoá-tạo lại, chỉ di chuyển địa chỉ
terraform state pull | head -5            # xem state thô (cẩn thận với dữ liệu nhạy cảm)
terraform destroy -auto-approve
```
Quan sát: plan sau khi thêm `moved` nói gì về `local_file.a` và `local_file.b`?
:::

=== tf-modules | Terraform | Module: đóng gói và tái sử dụng
>> Module là một thư mục chứa mã Terraform được gọi như một hàm: nhận biến, trả output. Dùng module để tái sử dụng và chuẩn hoá, nhưng đừng đóng gói sớm quá hay trừu tượng quá mức.

## Module là gì

Mọi cấu hình Terraform đều là một module: thư mục bạn chạy lệnh là **root module**; những module nó gọi là **child module**.

```
modules/
└── web_service/
    ├── main.tf          # tài nguyên
    ├── variables.tf     # đầu vào
    └── outputs.tf       # đầu ra

envs/
└── prod/
    └── main.tf          # gọi module
```

```
# modules/web_service/variables.tf
variable "name"          { type = string }
variable "instance_type" {
  type    = string
  default = "t3.micro"
}
variable "subnet_ids"    { type = list(string) }

# modules/web_service/main.tf
resource "aws_launch_template" "this" {
  name_prefix   = "${var.name}-"
  instance_type = var.instance_type
}
# ... autoscaling group, security group, v.v.

# modules/web_service/outputs.tf
output "asg_name" { value = aws_autoscaling_group.this.name }

# envs/prod/main.tf
module "web" {
  source        = "../../modules/web_service"
  name          = "shop-web"
  instance_type = "m5.large"
  subnet_ids    = module.network.private_subnet_ids
}
```

Sau khi thêm hoặc đổi `source`, chạy `terraform init` để tải module. Truy cập đầu ra của module: `module.web.asg_name`.

## Nguồn module

| `source` | Ví dụ |
|---|---|
| Đường dẫn cục bộ | `"../../modules/web_service"` |
| Terraform Registry | `"terraform-aws-modules/vpc/aws"` kèm `version = "~> 5.0"` |
| Git | `"git::https://github.com/org/repo.git//modules/vpc?ref=v1.2.0"` |
| Bucket/HTTP | `"s3::https://..."` |

Với module từ Registry hoặc Git: **ghim phiên bản** (`version` hay `?ref=tag`) và đọc mã trước khi dùng, vì module chạy với quyền của bạn.

## Thiết kế module tốt

- **Một việc rõ ràng** (một VPC, một dịch vụ web), đặt tên theo mục đích.
- **Giao diện nhỏ**: ít biến, có `description` và `type`, giá trị mặc định hợp lý; ẩn chi tiết bên trong.
- **Output đủ dùng** cho nơi gọi (ID, ARN, endpoint), không xuất cả đống.
- **Không đặt provider cứng** bên trong module tái sử dụng; để nơi gọi truyền vào.
- **Có ví dụ và README**, và kiểm thử ([CI/CD và kiểm thử](#tf-pipeline)).
- **Phiên bản hoá** bằng tag Git (semver), ghi changelog.

## Những lỗi thường gặp khi dùng module

- **Module "bọc" quá dày**: một module chỉ bọc một resource kèm hàng chục biến chuyển tiếp không thêm giá trị, làm mã khó đọc hơn viết resource trực tiếp.
- **Trừu tượng hoá quá sớm**: tách module khi mới có một nơi dùng. Đợi tới lần lặp thứ hai hoặc ba, khi bạn thấy phần nào thật sự chung.
- **Module quá to** (cả hạ tầng một cục): đổi nhỏ cũng plan rất lâu và blast radius lớn.
- **Đổi tên module hoặc tài nguyên bên trong** mà quên khối `moved`: bị tạo lại.
- **Dùng nhiều tầng module lồng nhau**: khó lần theo.

## Composition: ghép các module nhỏ

Thay vì một module khổng lồ, ghép nhiều module nhỏ ở root:

```
module "network" { source = "../modules/network" }
module "database" {
  source    = "../modules/database"
  vpc_id    = module.network.vpc_id
  subnet_ids = module.network.private_subnet_ids
}
module "app" {
  source        = "../modules/web_service"
  subnet_ids    = module.network.private_subnet_ids
  database_url  = module.database.endpoint
}
```

::: lab Tạo và gọi một module cục bộ
```
mkdir -p ~/tf5/modules/greeting ~/tf5/root && cd ~/tf5
cat > modules/greeting/main.tf <<'EOF'
terraform {
  required_providers {
    local = { source = "hashicorp/local" }
  }
}
variable "who" { type = string }
variable "dir" { type = string }
resource "local_file" "g" {
  filename = "${var.dir}/hello-${var.who}.txt"
  content  = "Xin chào, ${var.who}!\n"
}
output "path" { value = local_file.g.filename }
EOF
cat > root/main.tf <<'EOF'
module "an" {
  source = "../modules/greeting"
  who    = "an"
  dir    = path.module
}
module "binh" {
  source = "../modules/greeting"
  who    = "binh"
  dir    = path.module
}
output "files" { value = [module.an.path, module.binh.path] }
EOF
cd root && terraform init >/dev/null && terraform apply -auto-approve | tail -4
terraform state list
terraform destroy -auto-approve >/dev/null
```
Quan sát địa chỉ tài nguyên trong `state list`: chúng có dạng `module.an.local_file.g` và `module.binh.local_file.g`. Khi nào việc đóng gói thế này thực sự đáng làm?
:::

=== tf-aws | Terraform | Ví dụ thực tế: mạng và máy chủ trên AWS
>> Một ví dụ vừa đủ để thấy các khái niệm làm việc cùng nhau: VPC, subnet, security group, instance, kèm cách xác thực và gắn tag.

::: warn Tài nguyên AWS tốn tiền
Ví dụ dưới đây tạo tài nguyên thật trên tài khoản AWS. Dùng tài khoản thử nghiệm, đặt cảnh báo chi phí, dùng loại instance nhỏ nằm trong free tier nếu có, và luôn `terraform destroy` sau khi thử. Các tên AMI, tham số provider có thể thay đổi: đối chiếu tài liệu của provider.
:::

## Cấu hình provider và xác thực

```
provider "aws" {
  region = var.region
  default_tags {
    tags = {
      Project = var.project
      Env     = var.env
      Managed = "terraform"
    }
  }
}
```

`default_tags` gắn tag cho mọi tài nguyên hỗ trợ tag: tiện cho quản lý chi phí và tìm tài nguyên do Terraform tạo.

Cách xác thực (từ an toàn nhất trở xuống):

1. **Danh tính ngắn hạn qua vai trò (assume role, OIDC)**: trong CI dùng OIDC để nhận quyền tạm thời, không lưu khoá ([CI/CD và kiểm thử](#tf-pipeline)).
2. **AWS SSO / profile** cho máy cá nhân: `export AWS_PROFILE=dev`.
3. Biến môi trường `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`: tránh khoá dài hạn.
4. **Không** ghi khoá vào file `.tf`.

## Ví dụ: một web server sau security group

```
variable "region"  { default = "ap-southeast-1" }
variable "project" { default = "lab" }
variable "env"     { default = "dev" }

data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd-gp3/ubuntu-noble-24.04-amd64-server-*"]
  }
}

resource "aws_vpc" "main" {
  cidr_block           = "10.10.0.0/16"
  enable_dns_hostnames = true
  tags = { Name = "${var.project}-${var.env}" }
}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
}

resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(aws_vpc.main.cidr_block, 8, 1)
  map_public_ip_on_launch = true
  tags = { Name = "${var.project}-public" }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
}

resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}

resource "aws_security_group" "web" {
  name   = "${var.project}-web"
  vpc_id = aws_vpc.main.id
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_instance" "web" {
  ami                    = data.aws_ami.ubuntu.id
  instance_type          = "t3.micro"
  subnet_id              = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.web.id]
  user_data = <<-EOT
    #!/bin/bash
    apt-get update -y && apt-get install -y nginx
  EOT
  tags = { Name = "${var.project}-web" }
}

output "web_ip" { value = aws_instance.web.public_ip }
```

Đối chiếu với kiến thức mạng: subnet "public" là subnet có route `0.0.0.0/0` tới Internet Gateway; security group chỉ cho phép (stateful) cổng 80. Xem [Cầu nối sang cloud](https://tracviethustcs.github.io/linux-network-notes/#net-cloud) và [AWS SAP-C02 notes](https://tracviethustcs.github.io/aws-sap-study-notes/).

Điều ví dụ **cố ý bỏ qua** (cần có ở môi trường thật): SSH qua Session Manager thay vì mở cổng 22, nhiều AZ, subnet riêng với NAT, mã hoá đĩa, IMDSv2 bắt buộc (`metadata_options`), logging. Đó là lúc nên dùng module đã được cộng đồng kiểm chứng (như `terraform-aws-modules/vpc/aws`) thay vì tự viết từng resource.

## Thói quen tốt khi làm việc với cloud

- Mọi tài nguyên có tag đủ để biết ai tạo, cho dự án nào, môi trường nào.
- Đọc `plan` kỹ, đặc biệt các dòng `-/+` (thay thế) và `-` (xoá).
- Chạy trong tài khoản/sub-account riêng cho từng môi trường; credentials của CI chỉ có quyền cần thiết.
- Đặt `prevent_destroy` cho dữ liệu; dùng snapshot trước khi thay đổi lớn.
- Với giá trị "mới nhất" (AMI), ghim hoặc chấp nhận có thay thế định kỳ.

::: lab Chạy thử có kiểm soát chi phí
```
mkdir -p ~/tf-aws && cd ~/tf-aws        # dán ví dụ ở trên vào main.tf, thêm khối terraform/provider
export AWS_PROFILE=lab
terraform init
terraform fmt -check && terraform validate
terraform plan -out=tfplan
terraform show tfplan | grep -E "Plan:|will be created"
terraform apply tfplan
curl -s "http://$(terraform output -raw web_ip)" | head -3     # có thể cần chờ vài phút cho nginx
terraform destroy
```
Trả lời: plan tạo bao nhiêu tài nguyên? Thứ tự tạo suy ra từ đâu? Sau `destroy` còn tài nguyên nào tính phí không (kiểm tra console)?
:::
