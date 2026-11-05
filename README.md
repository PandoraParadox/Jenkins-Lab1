# Jenkins CI/CD - Static HTML Deployment to IIS

Lab tối thiểu để triển khai `index.html` lên IIS trên `WIN-SRV01` bằng Jenkins,
Ansible và WinRM.

## Cấu trúc

```text
.
├── index.html
├── Jenkinsfile
├── README.md
└── ansible
    ├── deploy-iis.yml
    └── inventory.ini.example
```

## Luồng triển khai

```text
GitHub push
    -> Jenkins Controller
    -> Jenkins Agent có Ansible
    -> WinRM
    -> WIN-SRV01 / IIS :8080
```

Pipeline thực hiện:

1. Checkout source từ GitHub.
2. Kiểm tra `index.html` và syntax Ansible.
3. Cài IIS nếu chưa có, tạo thư mục website và copy HTML.
4. Tạo/cập nhật website `DevOpsLabPortal` trên port `8080`.
5. Gọi `http://localhost:8080` trên Windows và yêu cầu HTTP 200 cùng nội dung
   `DevOpsLabPortal`.

`win_copy` dùng `backup: true`, vì vậy file HTML cũ được Ansible backup trước khi
ghi đè. Đây là cơ chế rollback thủ công đơn giản phù hợp với lab.

## Chuẩn bị GitHub và Jenkins

1. Tạo repository GitHub mới và push toàn bộ thư mục này.
2. Tạo Jenkins Pipeline dạng **Pipeline script from SCM**:
   - SCM: Git
   - Repository URL: URL repository GitHub
   - Script path: `Jenkinsfile`
3. Trên Jenkins Agent cài Ansible và collection:

   ```bash
   ansible-galaxy collection install ansible.windows
   ```

4. Có thể copy inventory mẫu để dùng tên file riêng:

   ```bash
   cp ansible/inventory.ini.example ansible/inventory.ini
   ```

   Pipeline mẫu dùng trực tiếp `inventory.ini.example`, vì vậy có thể sửa
   `ansible_host` trong file này cho lab. Nếu dùng `inventory.ini` riêng thì sửa
   lại đường dẫn `-i` trong `Jenkinsfile`. Không commit file inventory nếu chứa
   thông tin môi trường nội bộ.

5. Tạo Jenkins credential kiểu **Username with password** với ID
   `win-srv01-winrm`. Username/password phải có quyền cài Windows Feature và
   cấu hình IIS.

6. Cấu hình webhook GitHub gọi Jenkins khi push, hoặc bật Poll SCM nếu lab không
   dùng webhook.

## Kiểm tra thủ công

Từ máy có thể truy cập Windows Server:

```text
http://WIN-SRV01:8080/
```

Kết quả mong đợi: HTTP `200` và trang `DevOpsLabPortal`.

## Lưu ý lab

- Mẫu này dùng WinRM HTTPS/5986 với `ntlm`; hãy cấu hình WinRM và firewall trước.
- Không lưu password trong GitHub hoặc inventory. Jenkins credential được truyền
  lúc chạy pipeline.
- `ansible.windows.win_iis_website` yêu cầu collection `ansible.windows`.
