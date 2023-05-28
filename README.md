# Giới thiệu

bài lab tổng hợp với thao tác build và deploy một ưng dụng Java Maven lên môi trường Kubernetes

project demo được lấy tại github: https://github.com/liamhubian/techmaster-obo-web

# Mục tiêu bài lab

# Chuẩn bị

Để hoàn thành bài lab, người học cần chuẩn bị:

- Cài đặt kubectl trên client
- Cluster Kubernetes đang hoạt động và người học có khả năng kết nối tới cluster này
- Có thể sử dụng các công cụ như minikube, docker-desktop kubernetes, KinD,... trên môi trường lab
- thực hiện cấu hình alias trên terminal bằng command "alias k=kubectl"

# Các bước thực hiện

### Bước 0: (optional) chuẩn bị Cluster

> **Note**
>
> - Học viên chuẩn bị môi trường Kubernetes cho bài lab
> - Với học viên sử dụng KinD, hãy tận dụng file kind.conf được chuẩn bị sẵn cho bài lab
> - `kind create cluster --name project --config kind.conf`
### Toàn bộ file .ymal và Dockerfile trong bài tập này là dùng file được thầy Lâm cung cấp.

### Bước 1: triển khai cơ sở dữ liệu MySQL
> - Thêm cấu hình env vào file templates/mysql.yaml
    với `key: MYSQL_ROOT_PASSWORD - value: "123"`
> - Triển khai Statefulset chứa database MySQL:
    `k create -f templates/mysql.yaml`
> - Thực hiện restore data cho dự án:
    Copy obo.sql file từ repo techmaster-obo-web vào trong worker container chứa pod triển khai MySQL - thư mục /hostPath (thư mục đã khai báo trong file mysql.yaml) - kiểm tra bằng công cụ K9s để thấy tên container chứa MySQL:
        `docker cp obo.sql project-worker:/hostPath`
    Truy cập Shell của container chứa MySQL thông qua công cụ K9s.
    Sau đó truy cập MySQL qua câu lệnh `mysql -u root -p`. Tạo thêm database có tên obo `create database obo;`. Thoát MySQL `exit`.
    Đổ/Bung... dữ liệu từ file obo.spl trong thư mục hostPath:
        ```
        cd hostPath
        mysql -u root -p obo < obo.sql
        ```
    Các câu lệnh `mysql ...` nhập mật khẩu là `123`
> - Như vậy là đã có dữ liệu được lưu trong MySQL database chạy trên môi trường K8s

### Bước 2: cài đặt ứng dụng và đóng gói dưới dạng container
> - Clone repo https://github.com/liamhubian/techmaster-obo-web
> - Sửa file application-dev.properties - sửa cấu hình ip trỏ tới ip của MySQL
    Có thể sử dụng công cụ K9s để thấy được ip của Pod chứa container MySQL.
    Thay thế địa chỉ ip được set cứng trong file application-dev.properties. Với đường dẫn tới file lã `/techmaster-obo-websrc/main/resources/application-dev.properties`
> - Sửa Dockerfile để có thể bỏ qua quá trình test khi build image.
    (quá trình test sẽ báo fail và ra lỗi khi không thể truy cập tới địa chỉ ip trong file application-dev.properties)
    Thay thế dòng lệnh `RUN mvn install` bằng `RUN mvn clean install -Dmaven.test.skip=true`
> - Build dự án thành Docker image và push lên Docker hub với các lệnh
    ```
    docker build -t vietdinhbrj/obo-web:1.1 .
    docker tag docker.io/vietdinhbrj/obo-web:1.1 vietdinhbrj/obo-web:1.1
    docker push vietdinhbrj/obo-web:1.1
    ```
> - Như vậy là đã có image của trang web obo được lưu trữ trên Docker hub

### Bước 3: triển khai ứng dụng trên môi trường Kubernetes
> - Sửa file obo-web.yaml với những cấu hình để triển khai Docker image vietdinhbrj/obo-web:1.
    ```
    containers:
    - image: vietdinhbrj/obo-web:1.1
    ```
> - Vẫn là file obo-web.yaml. Xóa cấu hình configMap. Bởi vì repo xử dụng trong bài lap này đã được gán cứng địa chỉ ip tới MySQL cho nên không cần truyền các giá trị biến môi trường qua configMap nữa.
    xóa
    ```
    volumes:
      - name: web-config
        configMap:
          name: obo-web-config
    ```
    ```
    volumeMounts:
        - name: web-config
          mountPath: /techmaster-obo-web/src/main/resources/application-dev.properties
          subPath: application-dev.properties
    ```
> - Triển khai ứng dụng bằng lệnh
    `k create -f templates/obo-web.yaml`
> - Chờ đợi và kiểm tra tình trạng của Deployment vừa xong bằng công cụ K9s

### Bước 4: truy cập tới ứng dụng
> - Publich ứng dụng qua cổng 8080
    `k port-forward deploy/obo-web --address 0.0.0.0 8080:8080`
> - Sử dụng trình duyệt web truy cập tới localhost:8080 nếu với wsl
# Clean up

Sau khi hoàn thành bài lab, học viên thực hiện xóa các tài nguyên

```bash
k delete -f template/
```
> **Note**
>
> Học viên tạo portforward tới WSL2 thực hiện xóa proxy rule
> ```command
> netsh interface portproxy delete v4tov4 listenport=8080 listenaddress=0.0.0.0
> ```

### Credit
> - Thầy Lâm, người cung cấp toàn bộ chiêu thức và hỗ trợ fix bug tối đa.
> - Anh Việt, người anh tham gia rất sâu vào quá trình fix bug để tạo ra các record chất lượng cho anh em làm sao được làm theo kiểu mì ăn liền.
> - Anh Linh, người đã chỉ cho cách pass qua bước test của Maven chứ không là em đang đánh giá là cần đi triển khai Service để publich cái MySQL ra ngoài.
> - Toàn bộ các Thầy và các bạn cùng học đã tận tình, nhiệt huyết trong suốt khóa học. Cũng thấy vui vì tới giờ chưa có rơi rót người anh em nào. Cho dù anh em cũng đã đủ bận với công việc riêng.