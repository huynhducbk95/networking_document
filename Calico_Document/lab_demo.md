# Các bước cài đặt lab kiểm thử hệ thống thu thập thông tin và giám sát trạng thái mạng.

### 1. Trên một máy ảo của máy chủ vật lý triển khai Openstack.

**Framework**: Django.
**Thư viện liên quan**:
- Python 3
- SQLAlchemy

**Các bước thực hiện**:
- Cài đặt Django với câu lệnh:
  ```
   pip install django
  ```
- Cài đặt SQLAlchemy với câu lệnh:
  ```
  pip install python-SQLAlchemy
  ```
- Copy file Server_Demo trong thư mục source vào máy ảo và truy cập vào thư mục Server_Demo và chạy câu lệnh sau:
   ```
   python3 manage.py runserver 8080
  ```
 ### 2. Trên một máy chủ vật lý triển khai Docker.
 Tạo ra container với docker file như sau:
