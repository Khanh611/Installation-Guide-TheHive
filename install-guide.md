# Hướng dẫn cài đặt TheHive trên ubuntu 20.04

## Mục lục


1. [Ubuntu](#ubuntu)
1. [Cài đặt Java Virtual Machine](#java)
1. [Cài đặt Elasticsearch](#Elasticsearch)
1. [Cài đặt TheHive](#thehive)
1. [Khởi động đầu tiên](#first-start)
1. [Cập nhật](#update)


## Ubuntu <a name="ubuntu"></a>


Máy có các phần mềm sau :

- Java runtime environment 1.8+ (JRE)
- Elasticsearch 5.x

Đảm bảo hệ thống được cập nhật:

```
sudo apt-get update
sudo apt-get upgrade
```

## Cài đặt Java Virtual Machine <a name="java"></a>

Bạn có thể cài đặt Oracle Java hoặc OpenJDK. Khuyến khích nên dùng bản sau:

``` sudo apt-get install openjdk-11-jre-headless ```

## Cài đặt Elasticsearch <a name="Elasticsearch"></a>

- Cài đặt gói Elasticsearch do Elastic cung cấp

```
# Cài đặt PGP key
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-key D88E42B4

# Nếu lệnh trên không cài được thì có thể dùng lệnh sau để thay thế
# wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -

# Cấu hình Debian repository
echo "deb https://artifacts.elastic.co/packages/5.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-5.x.list

# Cài đặt https support for apt
sudo apt install apt-transport-https

# Cài đặt Elasticsearch
sudo apt update && sudo apt install elasticsearch
```

- Cấu hình

Nếu Elasticsearch và TheHive chạy trên cùng một máy chủ, hãy chỉnh sửa file ``/etc/elasticsearch/elasticsearch.yml`` và đặt tham số cho ``network.host`` là ``127.0.0.1``. TheHive sử dụng các tập lệnh động để cập nhật từng phần. Do đó, chúng phải được kích hoạt bằng cách sử dụng ``script.inline: true``

The cluster name cũng phải được đặt. Kích thước hàng đợi Threadpool phải được đặt với giá trị cao (``100000``). Kích thước mặc định sẽ khiến hàng đợi dễ bị quá tải.

Chỉnh sửa ``/etc/elasticsearch/elasticsearch.yml`` và thêm các dòng sau:

```
network.host: 127.0.0.1
script.inline: true
cluster.name: hive
thread_pool.index.queue_size: 100000
thread_pool.search.queue_size: 100000
thread_pool.bulk.queue_size: 100000
```
![image](https://user-images.githubusercontent.com/65669673/92312597-74da3800-efec-11ea-9de6-f4a9d1eb15a0.png)

- Bắt đầu dịch vụ

Bây giờ Elasticsearch đã được định cấu hình, hãy khởi động nó như một dịch vụ và kiểm tra xem nó có đang chạy hay không:

```
sudo systemctl enable elasticsearch.service
sudo systemctl start elasticsearch.service
sudo systemctl status elasticsearch.service
```

![image](https://user-images.githubusercontent.com/65669673/92312634-d8646580-efec-11ea-808c-24d81fe39659.png)

## Cài đặt TheHive <a name="thehive"></a>

Tải TheHive về máy từ [Bintray](https://dl.bintray.com/thehive-project/binary/). Phiên bản mới nhất có tên là [thehive-latest.zip](https://dl.bintray.com/thehive-project/binary/thehive-latest.zip).

Tải xuống và giải nén gói tin đã chọn. Tệp TheHive có thể được cài đặt ở bất cứ đâu bạn muốn. Trong hướng dẫn này, TheHive được cài đặt ở ``/opt``.

```
cd /opt
wget https://dl.bintray.com/thehive-project/binary/thehive-latest.zip
unzip thehive-latest.zip
ln -s thehive-x.x.x thehive
```
***Chú ý:*** thay ``x.x.x`` bằng tên phiên bản

![image](https://user-images.githubusercontent.com/65669673/92312707-896b0000-efed-11ea-82ca-222762954af9.png)

## Khởi động đầu tiên <a name="first-start"></a>

Nên sử dụng tài khoản người dùng chuyên dụng, không có đặc quyền để khởi động TheHive. Nếu vậy, hãy đảm bảo rằng tài khoản đã chọn có thể tạo tệp trong ``/opt/thehive/logs``.

- Thêm user thehive

```
sudo useradd -s /bin/bash -m thehive
```
- Cấp quyền sudo và đăng nhập vào thehive
```
echo "thehive ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/thehive
sudo su - thehive
```

Tham số bắt buộc duy nhất để khởi động TheHive là khóa của máy chủ (``play.http.secret.key``). Khóa này được sử dụng để xác thực cookie có chứa dữ liệu. Nếu TheHive chạy ở chế độ cluster, tất cả các phiên bản phải dùng chung một khóa. Có thể tạo cấu hình tối thiểu bằng các lệnh sau (đã tạo một người dùng riêng cho TheHive, có tên ``thehive``):

```
sudo mkdir /etc/thehive
(cat << _EOF_
# Secret key
# ~~~~~
# The secret key is used to secure cryptographics functions.
# If you deploy your application to several instances be sure to use the same key!
play.http.secret.key="$(cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 64 | head -n 1)"
_EOF_
) | sudo tee -a /etc/thehive/application.conf
```
![image](https://user-images.githubusercontent.com/65669673/92312765-29c12480-efee-11ea-84f8-e5cdb2db6faa.png)

Khởi động ứng dụng dưới dạng dịch vụ, hãy sử dụng các lệnh sau:

```
sudo cp /opt/thehive/package/thehive.service /usr/lib/systemd/system
sudo chown -R thehive:thehive /opt/thehive
sudo chgrp thehive /etc/thehive/application.conf
sudo chmod 640 /etc/thehive/application.conf
sudo systemctl enable thehive
sudo service thehive start
```
![image](https://user-images.githubusercontent.com/65669673/92312818-c2f03b00-efee-11ea-86a6-c922d2961320.png)

Thêm ``search.uri`` vào file ``/etc/thehive/application.conf``

![image](https://user-images.githubusercontent.com/65669673/92312852-20848780-efef-11ea-9e8e-78ca1fc73e83.png)

Bây giờ có thể bắt đầu TheHive. Để làm như vậy, hãy thay đổi thư mục hiện tại thành thư mục cài đặt TheHive ( ``/opt/thehive``), sau đó thực hiện:

```
bin/thehive -Dconfig.file=/etc/thehive/application.conf
```

Xin lưu ý rằng dịch vụ có thể mất một thời gian để bắt đầu. Sau khi nó được khởi động, mở trình duyệt lên và kết nối tới ``http://127.0.0.1:9000/``

![image](https://user-images.githubusercontent.com/65669673/92313088-8eca4980-eff1-11ea-8fe3-679709b462a9.png)

Lần đầu tiên kết nối, sẽ phải tạo lược đồ cơ sở dữ liệu. Nhấp vào "Update Database" để tạo lược đồ DB

![image](https://user-images.githubusercontent.com/65669673/92313101-bcaf8e00-eff1-11ea-9367-726f37377a66.png)

Sau đó, sẽ được chuyển đến trang tạo tài khoản của quản trị viên

![image](https://user-images.githubusercontent.com/65669673/92313106-dc46b680-eff1-11ea-855c-cc160b10ca05.png)

Sau khi tạo, sẽ được chuyển hướng đến trang đăng nhập

![image](https://user-images.githubusercontent.com/65669673/92313124-06987400-eff2-11ea-945f-0f948a4f830f.png)

Sau khi đăng nhập thành công

![image](https://user-images.githubusercontent.com/65669673/92313132-39db0300-eff2-11ea-8cef-a4ac88fbd7bb.png)

**Cảnh báo**: ở giai đoạn này, nếu đã lỡ tạo tài khoản quản trị, sẽ không thể thực hiện được lại được trừ khi xóa index của TheHive khỏi Elasticsearch. Trong trường hợp mắc lỗi, trước tiên hãy tìm index hiện tại của TheHive bằng cách chạy lệnh sau trên máy chủ lưu trữ Elasticsearch DB được TheHive sử dụng

```
curl http://127.0.0.1:9200/_cat/indices?v
```

Các index mà TheHive sử dụng luôn bắt đầu bằng ``the_hive_`` theo sau là 1 số

![image](https://user-images.githubusercontent.com/65669673/92313171-b40b8780-eff2-11ea-93ff-82d7fe4fef33.png)

Index được sử dụng bởi TheHive là ``the_hive_15``. Để xóa nó, hãy chạy lệnh sau

```
curl -X DELETE http://127.0.0.1:9200/the_hive_15
```

Sau đó tải lại trang hoặc khởi động lại TheHive

![image](https://user-images.githubusercontent.com/65669673/92313187-fa60e680-eff2-11ea-8766-bef22299c069.png)

## Cập nhật <a name="update"></a>

Để cập nhật TheHive chỉ cần dừng dịch vụ, tải xuống gói mới nhất, xây dựng lại liên kết ``/opt/thehive`` và khởi động lại dịch vụ

```
service thehive stop
cd /opt
wget https://dl.bintray.com/thehive-project/binary/thehive-latest.zip
unzip thehive-latest.zip
rm /opt/thehive && ln -s thehive-x.x.x thehive
chown -R thehive:thehive /opt/thehive /opt/thehive-x.x.x
service thehive start
```

***Chú ý:*** thay ``x.x.x`` bằng tên phiên bản.
