=== an-basics | Ansible | Ansible cơ bản: kiến trúc, inventory và ad-hoc
>> Ansible cấu hình máy qua SSH mà không cần cài agent. Bạn mô tả trạng thái mong muốn của máy bằng YAML, Ansible kiểm tra và chỉ thay đổi những gì chưa khớp.

## Kiến trúc

```ascii
   Control node (máy bạn/CI)                         Managed nodes (máy được quản lý)
  +-----------------------------+   SSH (Linux)      +-----------------------+
  | ansible, playbook, inventory| -----------------> | chỉ cần SSH + Python  |
  | module được đẩy sang, chạy, |   WinRM (Windows)  | không cài agent       |
  | rồi xoá                     |                    +-----------------------+
  +-----------------------------+
```

- **Agentless**: máy đích chỉ cần SSH và Python. Ansible đẩy module nhỏ sang, chạy rồi thu kết quả.
- **Inventory**: danh sách máy và nhóm.
- **Module**: đơn vị làm việc (`apt`, `copy`, `service`, `template`...), mỗi module cố gắng idempotent.
- **Playbook**: tệp YAML chứa các play, mỗi play áp một danh sách tác vụ lên một nhóm máy.
- **Role**: gói playbook tái sử dụng (xem [Role](#an-roles)).
- **Fact**: thông tin Ansible thu thập về máy đích (hệ điều hành, IP, RAM) để dùng trong điều kiện.

## Cài đặt và cấu hình

```
pipx install ansible              # hoặc: pip install ansible; hoặc gói của distro
ansible --version
```

Tệp `ansible.cfg` trong thư mục dự án:

```
[defaults]
inventory = ./inventory/hosts.ini
remote_user = deploy
host_key_checking = True
forks = 10
stdout_callback = yaml

[privilege_escalation]
become = True
become_method = sudo

[ssh_connection]
pipelining = True
```

Máy đích cần xác thực bằng **khoá SSH** (xem [SSH](https://tracviethustcs.github.io/linux-network-notes/#lx-ssh)). Tắt `host_key_checking` chỉ là tạm thời trong lab; ở môi trường thật hãy giữ nó bật.

## Inventory

```
# inventory/hosts.ini
[web]
web1 ansible_host=10.0.1.11
web2 ansible_host=10.0.1.12

[db]
db1 ansible_host=10.0.2.21

[prod:children]
web
db

[all:vars]
ansible_user=deploy
ansible_python_interpreter=/usr/bin/python3
```

Cùng nội dung ở dạng YAML:

```
all:
  children:
    web:
      hosts:
        web1: {ansible_host: 10.0.1.11}
        web2: {ansible_host: 10.0.1.12}
    db:
      hosts:
        db1: {ansible_host: 10.0.2.21}
```

**Biến theo nhóm và theo máy** đặt trong thư mục cạnh inventory:

```
inventory/
├── hosts.ini
├── group_vars/
│   ├── all.yml          # mọi máy
│   └── web.yml          # nhóm web
└── host_vars/
    └── web1.yml         # riêng máy web1
```

**Inventory động**: với cloud, máy thay đổi liên tục nên dùng plugin để lấy danh sách từ API (ví dụ `amazon.aws.aws_ec2` nhóm theo tag). Kiểm tra bằng `ansible-inventory --graph`.

```
ansible-inventory --graph
ansible-inventory --host web1
ansible web -m ping
```

## Lệnh ad-hoc

Chạy một module trên một nhóm máy mà không cần viết playbook, dùng cho việc nhanh hoặc kiểm tra:

```
ansible all -m ping                                    # kiểm tra kết nối SSH và Python
ansible web -b -m apt -a "name=nginx state=present"    # -b: nâng quyền (become)
ansible web -m service -a "name=nginx state=restarted" -b
ansible all -m setup -a "filter=ansible_os_family"     # xem fact
ansible all -m command -a "uptime"
ansible web -m copy -a "src=index.html dest=/var/www/html/ mode=0644" -b
```

Mẫu chọn máy: `web`, `web:db` (hợp), `web:&prod` (giao), `all:!db` (loại), `web[0]`, `--limit web1`.

::: lab Chạy Ansible trên chính máy bạn
```
mkdir -p ~/an1 && cd ~/an1
printf '[local]\nlocalhost ansible_connection=local\n' > hosts.ini
ansible -i hosts.ini local -m ping
ansible -i hosts.ini local -m setup -a "filter=ansible_distribution*"
ansible -i hosts.ini local -m command -a "uptime"
ansible -i hosts.ini local -m copy -a "content='xin chào\n' dest=/tmp/an1.txt"
ansible -i hosts.ini local -m copy -a "content='xin chào\n' dest=/tmp/an1.txt"     # lần hai: changed=false
ansible-inventory -i hosts.ini --graph
```
Quan sát: lần chạy `copy` thứ hai báo `changed` hay `ok`? Đó là idempotency.
:::

=== an-playbooks | Ansible | Playbook: tác vụ, biến, điều kiện và handler
>> Playbook là tệp YAML mô tả các bước cấu hình. Vài khái niệm (module, biến, điều kiện, handler) đủ để viết phần lớn công việc hằng ngày.

## Một playbook hoàn chỉnh

```
# site.yml
- name: Cài đặt và cấu hình nginx
  hosts: web
  become: true
  vars:
    http_port: 8080
  tasks:
    - name: Cài gói nginx
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true
        cache_valid_time: 3600
      when: ansible_os_family == "Debian"

    - name: Sinh cấu hình site từ template
      ansible.builtin.template:
        src: site.conf.j2
        dest: /etc/nginx/conf.d/site.conf
        owner: root
        group: root
        mode: "0644"
      notify: Reload nginx

    - name: Bảo đảm nginx chạy và bật khi boot
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true

  handlers:
    - name: Reload nginx
      ansible.builtin.service:
        name: nginx
        state: reloaded
```

```
ansible-playbook -i inventory/hosts.ini site.yml --check --diff    # chạy thử, xem sẽ đổi gì
ansible-playbook -i inventory/hosts.ini site.yml
ansible-playbook site.yml --limit web1 --tags config -v
```

## Các thành phần

| Thành phần | Ý nghĩa |
|---|---|
| **play** | Một khối: áp danh sách tác vụ lên một nhóm máy (`hosts`) |
| **task** | Một lệnh gọi module, có `name` mô tả rõ |
| **module** | Đơn vị làm việc; dùng **tên đầy đủ** (`ansible.builtin.apt`) cho rõ nguồn gốc |
| `become` | Chạy với quyền cao (sudo) |
| `vars`, `vars_files`, `group_vars` | Biến |
| `handlers` | Tác vụ chỉ chạy khi được `notify` và có thay đổi |
| `tags` | Nhãn để chạy một phần |

## Module hay dùng

| Việc | Module |
|---|---|
| Gói | `ansible.builtin.apt`, `ansible.builtin.dnf`, `ansible.builtin.package` |
| File | `ansible.builtin.copy`, `ansible.builtin.template`, `ansible.builtin.file` (quyền, thư mục, symlink), `ansible.builtin.lineinfile`, `ansible.builtin.blockinfile` |
| Dịch vụ | `ansible.builtin.service`, `ansible.builtin.systemd` |
| Người dùng | `ansible.builtin.user`, `ansible.builtin.group`, `ansible.posix.authorized_key` |
| Mã nguồn | `ansible.builtin.git`, `ansible.builtin.unarchive`, `ansible.builtin.get_url` |
| Lệnh | `ansible.builtin.command` (không qua shell), `ansible.builtin.shell` (qua shell, dùng khi cần pipe) |
| Firewall | `community.general.ufw`, `ansible.posix.firewalld` |
| Docker | `community.docker.docker_container`, `docker_image` |
| Chờ và kiểm tra | `ansible.builtin.wait_for`, `ansible.builtin.uri`, `ansible.builtin.assert` |

**Ưu tiên module chuyên biệt hơn `command`/`shell`**: module biết kiểm tra trạng thái và idempotent, còn `command` luôn báo `changed`. Khi buộc dùng `command`, thêm `creates:`, `removes:` hoặc `changed_when:` để nó idempotent:

```
- name: Khởi tạo database một lần
  ansible.builtin.command: /opt/app/bin/initdb
  args:
    creates: /var/lib/app/.initialized
```

## Biến, vòng lặp và điều kiện

```
- name: Tạo nhiều người dùng
  ansible.builtin.user:
    name: "{{ item.name }}"
    groups: "{{ item.groups | default('') }}"
    shell: /bin/bash
  loop:
    - {name: alice, groups: sudo}
    - {name: bob}

- name: Lưu kết quả để dùng sau
  ansible.builtin.command: nginx -v
  register: nginx_version
  changed_when: false

- name: In ra
  ansible.builtin.debug:
    msg: "Phiên bản: {{ nginx_version.stderr }}"
  when: nginx_version.rc == 0
```

- Biến dùng cú pháp Jinja2 `{{ ten }}`; khi giá trị bắt đầu bằng biến phải đặt trong dấu nháy.
- `register` lưu kết quả tác vụ; `when` đặt điều kiện (biểu thức Jinja2 không cần `{{ }}`).
- **Thứ tự ưu tiên biến** (từ thấp tới cao): giá trị mặc định của role, `group_vars`/`host_vars`, biến của play, biến của tác vụ, rồi cuối cùng `-e` (extra vars) thắng tất cả. Tránh phụ thuộc vào các tầng quá nhiều: đặt biến ở **một nơi rõ ràng**.

## Template Jinja2

```
# templates/site.conf.j2
server {
    listen {{ http_port }};
    server_name {{ ansible_fqdn }};
{% for upstream in app_servers %}
    # upstream {{ loop.index }}: {{ upstream }}
{% endfor %}
    location / { proxy_pass http://{{ app_servers[0] }}; }
}
```

## Handler và thứ tự

Handler chỉ chạy **một lần ở cuối play** và chỉ khi có tác vụ `notify` nó **thực sự thay đổi** gì đó: nhờ vậy nginx chỉ reload khi cấu hình đổi. Cần chạy sớm hơn thì dùng `ansible.builtin.meta: flush_handlers`.

## Chạy thử và gỡ lỗi

```
ansible-playbook site.yml --syntax-check        # kiểm tra cú pháp
ansible-playbook site.yml --check --diff        # mô phỏng, hiện khác biệt (không phải module nào cũng hỗ trợ)
ansible-playbook site.yml -v                    # -vv, -vvv: chi tiết hơn
ansible-playbook site.yml --start-at-task "Sinh cấu hình site từ template"
ansible-playbook site.yml --step                # hỏi xác nhận từng tác vụ
ansible-lint site.yml                           # kiểm tra theo thực hành tốt
```

::: lab Playbook đầu tiên chạy trên localhost
```
mkdir -p ~/an2 && cd ~/an2
printf '[local]\nlocalhost ansible_connection=local\n' > hosts.ini
cat > site.yml <<'EOF'
- name: Cấu hình thử trên máy này
  hosts: local
  vars:
    greeting: "Xin chào từ Ansible"
  tasks:
    - name: Tạo thư mục làm việc
      ansible.builtin.file:
        path: /tmp/an2
        state: directory
        mode: "0755"
    - name: Ghi file từ template inline
      ansible.builtin.copy:
        dest: /tmp/an2/hello.txt
        content: "{{ greeting }} - máy {{ ansible_hostname }}\n"
      notify: Báo đã đổi
    - name: Chạy lệnh một lần
      ansible.builtin.command: touch /tmp/an2/marker
      args:
        creates: /tmp/an2/marker
  handlers:
    - name: Báo đã đổi
      ansible.builtin.debug:
        msg: "file đã thay đổi"
EOF
ansible-playbook -i hosts.ini site.yml --check --diff
ansible-playbook -i hosts.ini site.yml
ansible-playbook -i hosts.ini site.yml        # lần hai: changed=0, handler không chạy
```
Quan sát: ở lần chạy thứ hai `changed` bằng bao nhiêu, và handler có chạy không?
:::

=== an-roles | Ansible | Role, collection và Vault
>> Role gom tác vụ, template, biến và handler thành một đơn vị tái sử dụng. Collection là cách phân phối module và role. Vault bảo vệ bí mật trong repo.

## Cấu trúc role

```
roles/
└── nginx/
    ├── tasks/main.yml        # các tác vụ chính
    ├── handlers/main.yml     # handler
    ├── templates/site.conf.j2
    ├── files/                # file tĩnh để copy
    ├── defaults/main.yml     # biến mặc định, ưu tiên THẤP: người dùng role ghi đè dễ
    ├── vars/main.yml         # biến nội bộ role, ưu tiên cao
    └── meta/main.yml         # phụ thuộc và thông tin
```

```
# roles/nginx/defaults/main.yml
nginx_port: 80
nginx_server_name: "{{ ansible_fqdn }}"

# roles/nginx/tasks/main.yml
- name: Cài nginx
  ansible.builtin.package:
    name: nginx
    state: present
- name: Cấu hình site
  ansible.builtin.template:
    src: site.conf.j2
    dest: /etc/nginx/conf.d/site.conf
    mode: "0644"
  notify: Reload nginx
- name: Chạy nginx
  ansible.builtin.service:
    name: nginx
    state: started
    enabled: true
```

Dùng role trong playbook:

```
- hosts: web
  become: true
  roles:
    - role: nginx
      vars:
        nginx_port: 8080
    - role: hardening
```

Nên đặt biến mà người dùng role có thể muốn đổi vào `defaults/`, còn `vars/` cho thứ nội bộ không nên bị ghi đè. **Đặt tiền tố tên role** cho mọi biến (`nginx_port`) để tránh đụng nhau giữa các role.

## Collection và Galaxy

**Collection** là gói phân phối module, plugin và role; tên dạng `namespace.collection` (`community.docker`, `amazon.aws`, `ansible.posix`). Module trong collection gọi bằng tên đầy đủ.

```
ansible-galaxy collection install community.docker
ansible-galaxy collection install -r requirements.yml
```

```
# requirements.yml: ghim phiên bản để lặp lại được
collections:
  - name: community.docker
    version: ">=3.0.0,<4.0.0"
  - name: amazon.aws
roles:
  - name: geerlingguy.nginx
    version: "3.2.0"
```

Role và collection từ Galaxy do cộng đồng viết: **đọc mã và ghim phiên bản** trước khi dùng, vì chúng chạy với quyền root trên máy của bạn.

## Ansible Vault

Vault mã hoá file hoặc từng giá trị để bí mật có thể nằm trong Git.

```
ansible-vault create group_vars/prod/vault.yml          # tạo file mã hoá
ansible-vault edit group_vars/prod/vault.yml
ansible-vault view group_vars/prod/vault.yml
ansible-vault encrypt_string 's3cr3t' --name 'db_password'    # mã hoá một giá trị, dán vào YAML
ansible-vault rekey group_vars/prod/vault.yml           # đổi mật khẩu vault
```

```
ansible-playbook site.yml --ask-vault-pass
ansible-playbook site.yml --vault-password-file ~/.vault_pass     # file ngoài Git, quyền 600
```

Thói quen tốt:

- **Tách biến nhạy cảm vào `vault.yml`**, còn `vars.yml` tham chiếu chúng (`db_password: "{{ vault_db_password }}"`): đọc mã vẫn thấy biến nào tồn tại.
- Mật khẩu vault **không nằm trong Git**; trong CI lấy từ kho bí mật.
- `no_log: true` cho tác vụ xử lý bí mật, nếu không giá trị có thể lộ trong log:

```
- name: Tạo user database
  community.postgresql.postgresql_user:
    name: app
    password: "{{ db_password }}"
  no_log: true
```

Với môi trường lớn, cân nhắc lấy bí mật trực tiếp từ HashiCorp Vault hoặc kho bí mật của cloud (lookup plugin), để bí mật có vòng đời và nhật ký riêng thay vì nằm trong repo dù đã mã hoá. Xem thêm [Bảo mật IaC](#iac-security).

## Kiểm thử role

- `ansible-lint`: kiểm tra phong cách và lỗi thường gặp.
- `yamllint`: cú pháp YAML.
- **Molecule**: dựng container hoặc VM thử, chạy role, chạy lại để kiểm tra **idempotent** (lần hai không còn `changed`), rồi chạy kiểm tra xác minh.

::: lab Viết một role nhỏ
```
mkdir -p ~/an3/roles/motd/{tasks,templates,defaults} && cd ~/an3
printf '[local]\nlocalhost ansible_connection=local\n' > hosts.ini
echo 'motd_message: "Chào mừng"' > roles/motd/defaults/main.yml
echo 'Thông báo: {{ motd_message }} ({{ ansible_hostname }})' > roles/motd/templates/motd.j2
cat > roles/motd/tasks/main.yml <<'EOF'
- name: Ghi motd thử
  ansible.builtin.template:
    src: motd.j2
    dest: /tmp/motd-lab
    mode: "0644"
EOF
printf -- '- hosts: local\n  roles:\n    - role: motd\n      vars: {motd_message: "Xin chào từ role"}\n' > play.yml
ansible-playbook -i hosts.ini play.yml && cat /tmp/motd-lab
ansible-playbook -i hosts.ini play.yml | grep -E "changed|ok="     # lần hai: idempotent
echo 'mật khẩu thử' > .vp && chmod 600 .vp
ansible-vault encrypt_string --vault-password-file .vp 'abc123' --name 'secret'
rm -f .vp
```
Quan sát: biến `motd_message` lấy giá trị từ đâu (defaults hay vars truyền vào)? Chuỗi đã mã hoá của Vault trông thế nào?
:::

=== an-patterns | Ansible | Mẫu thực tế: cuốn chiếu, xử lý lỗi và hiệu năng
>> Các mẫu giúp Ansible chạy an toàn ở quy mô nhiều máy: cập nhật từng đợt, xử lý lỗi, chờ dịch vụ sẵn sàng và giữ playbook idempotent.

## Cập nhật cuốn chiếu (rolling)

Mặc định Ansible chạy mỗi tác vụ trên **mọi máy** của play trước khi sang tác vụ kế. Để cập nhật từng đợt nhỏ (giữ dịch vụ sống), dùng `serial`:

```
- name: Nâng cấp ứng dụng từng đợt
  hosts: web
  become: true
  serial: "25%"                  # hoặc số: 2, hoặc danh sách [1, 3, "100%"]
  max_fail_percentage: 0         # một máy lỗi là dừng, không đụng đợt tiếp
  tasks:
    - name: Gỡ khỏi load balancer
      ansible.builtin.command: /usr/local/bin/lb-disable {{ inventory_hostname }}
      delegate_to: lb1           # chạy trên máy khác, không phải máy đích

    - name: Cập nhật gói ứng dụng
      ansible.builtin.apt:
        name: myapp={{ app_version }}
        state: present
      notify: Restart myapp

    - name: Chờ ứng dụng sẵn sàng
      ansible.builtin.uri:
        url: "http://127.0.0.1:8000/health"
        status_code: 200
      register: health
      until: health.status == 200
      retries: 20
      delay: 3

    - name: Đưa lại vào load balancer
      ansible.builtin.command: /usr/local/bin/lb-enable {{ inventory_hostname }}
      delegate_to: lb1

  handlers:
    - name: Restart myapp
      ansible.builtin.service:
        name: myapp
        state: restarted
```

- `serial` chia máy thành đợt; `max_fail_percentage` hoặc `any_errors_fatal` dừng cả quá trình khi có lỗi.
- `delegate_to` chạy tác vụ trên máy khác (load balancer, máy giám sát); `run_once` chạy một lần cho cả nhóm.
- `until`/`retries`/`delay` thử lại tới khi đạt điều kiện: dùng để chờ dịch vụ thật sự sẵn sàng thay vì `sleep` cố định.

## Xử lý lỗi

```
- name: Triển khai có đường lui
  block:
    - name: Cập nhật cấu hình
      ansible.builtin.template: {src: app.conf.j2, dest: /etc/app/app.conf}
    - name: Reload
      ansible.builtin.service: {name: app, state: reloaded}
  rescue:
    - name: Khôi phục cấu hình cũ
      ansible.builtin.copy:
        src: /etc/app/app.conf.bak
        dest: /etc/app/app.conf
        remote_src: true
    - name: Báo lỗi
      ansible.builtin.fail:
        msg: "Triển khai thất bại, đã khôi phục"
  always:
    - name: Ghi nhật ký
      ansible.builtin.debug:
        msg: "kết thúc"
```

`block`/`rescue`/`always` giống try/catch/finally. `ignore_errors: true` bỏ qua lỗi của một tác vụ (dùng tiết kiệm), `failed_when` và `changed_when` tự định nghĩa thế nào là lỗi và thay đổi.

## Giữ playbook idempotent

Đây là chất lượng quan trọng nhất của một playbook. Kiểm tra bằng cách **chạy hai lần**: lần hai phải báo `changed=0`.

| Nên | Tránh |
|---|---|
| Module chuyên biệt (`apt`, `user`, `lineinfile`) | `shell` và `command` thuần |
| Khi buộc dùng `command`: `creates`, `removes`, `changed_when` | Lệnh luôn báo `changed` |
| `state: present`, `started`, `latest` có chủ đích | Phụ thuộc vào trạng thái từ lần chạy trước |
| Mô tả trạng thái cuối | Viết như kịch bản "làm bước A rồi B" |
| `template` cho cấu hình toàn bộ file | `lineinfile` rải rác nhiều chỗ làm file khó đoán |

## Hiệu năng

- `pipelining = True` (giảm số lần SSH), `forks` đủ lớn (mặc định 5), giữ kết nối SSH (`ControlPersist`).
- `gather_facts: false` khi không cần fact; hoặc `gather_subset`.
- Dùng `async` và `poll` cho tác vụ chạy lâu trên nhiều máy.
- Chiến lược `free` cho phép máy chạy xong tác vụ nào trước thì đi tiếp, không đợi cả nhóm.

## Cấu hình khi máy "tự dựng"

Ngoài kiểu đẩy (push) từ control node, có `ansible-pull`: máy tự kéo playbook từ Git theo lịch và chạy cục bộ (hợp với nhiều máy, máy ra vào liên tục). Nền tảng như AWX hoặc Ansible Automation Platform thêm giao diện, RBAC, lịch chạy và nhật ký cho việc dùng theo nhóm.

## Một playbook hardening cơ bản

```
- name: Siết cơ bản cho máy chủ mới
  hosts: all
  become: true
  tasks:
    - name: Chỉ cho đăng nhập bằng khoá
      ansible.builtin.lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?PasswordAuthentication'
        line: 'PasswordAuthentication no'
        validate: '/usr/sbin/sshd -t -f %s'
      notify: Reload sshd
    - name: Không cho root đăng nhập
      ansible.builtin.lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?PermitRootLogin'
        line: 'PermitRootLogin no'
        validate: '/usr/sbin/sshd -t -f %s'
      notify: Reload sshd
    - name: Bật cập nhật bảo mật tự động
      ansible.builtin.apt:
        name: unattended-upgrades
        state: present
  handlers:
    - name: Reload sshd
      ansible.builtin.service:
        name: ssh
        state: reloaded
```

`validate:` chạy `sshd -t` **trước khi** ghi file vào chỗ thật: cấu hình sai sẽ không được áp dụng, tránh tự khoá mình khỏi máy ([Hardening](https://tracviethustcs.github.io/linux-network-notes/#lx-harden), [SSH](https://tracviethustcs.github.io/linux-network-notes/#lx-ssh)). Tên dịch vụ SSH là `ssh` trên Debian/Ubuntu và `sshd` trên họ RHEL.

::: lab Thử cuốn chiếu và block/rescue ngay trên máy
```
mkdir -p ~/an4 && cd ~/an4
cat > hosts.ini <<'EOF'
[nodes]
n1 ansible_connection=local
n2 ansible_connection=local
n3 ansible_connection=local
n4 ansible_connection=local
EOF
cat > roll.yml <<'EOF'
- hosts: nodes
  serial: 2
  gather_facts: false
  tasks:
    - name: Bắt đầu đợt
      ansible.builtin.debug:
        msg: "đang xử lý {{ inventory_hostname }}"
    - block:
        - name: Cố ý lỗi trên n3
          ansible.builtin.fail:
            msg: "lỗi giả lập"
          when: inventory_hostname == "n3"
      rescue:
        - name: Xử lý lỗi
          ansible.builtin.debug:
            msg: "đã khôi phục {{ inventory_hostname }}"
EOF
ansible-playbook -i hosts.ini roll.yml
```
Quan sát: các máy được xử lý theo đợt nào? `rescue` chạy trên máy nào? Thử đổi `serial` và thêm `max_fail_percentage: 0` để xem khác biệt.
:::
