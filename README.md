# Báo Cáo Phân Tích Chia Module Service & Thiết Kế Database Hệ Thống Quản Lý Bệnh Viện MediCare

## 1. Yêu Cầu 1 — Phân Tích Và Chia Module Microservice

### 1.1. Sơ đồ kiến trúc tổng thể

```
                             +-------------------------------+
                             |          API Gateway          |
                             |          (Port 8080)          |
                             +---------------+---------------+
                                             |
       +--------------------+----------------+--------------------+--------------------+
       |                    |                                     |                    |
       v                    v                                     v                    v
+--------------+     +--------------+                      +--------------+     +--------------+
|   Patient    |     |    Doctor    |                      | Medical Rec. |     |   Pharmacy   |
|   Service    |     |   Service    |                      |   Service    |     |   Service    |
| (Port 8081)  |     | (Port 8082)  |                      | (Port 8084)  |     | (Port 8085)  |
+-------+------+     +-------+------+                      +-------+------+     +-------+------+
        |                    |                                     |                    |
        v                    v                                     v                    v
+--------------+     +--------------+                      +--------------+     +--------------+
| medicare_    |     | medicare_    |                      | medicare_    |     | medicare_    |
| patient_db   |     | doctor_db    |                      | medical_     |     | pharmacy_db  |
+--------------+     +--------------+                      | record_db    |     +--------------+
                                                           +--------------+
                                     ^
                                     | (Tham chiếu ID & Gọi API/Event)
                              +------+-------+
                              | Appointment  |
                              |   Service    |
                              | (Port 8083)  |
                              +------+-------+
                                     |
                                     v
                              +--------------+
                              | medicare_    |
                              | appointment_ |
                              | db           |
                              +--------------+
```

---

### 1.2. Danh sách Microservices & Lý do phân chia

| STT | Service Name | Port | Database Riêng | Bounded Context & Trách Nhiệm Nghiệp Vụ |
| :--- | :--- | :---: | :--- | :--- |
| 1 | **patient-service** | `8081` | `medicare_patient_db` | Quản lý thông tin định danh bệnh nhân, hồ sơ hành chính, số BHYT và lịch sử tiếp nhận. |
| 2 | **doctor-service** | `8082` | `medicare_doctor_db` | Quản lý thông tin bác sĩ, chuyên khoa, phòng trực và lịch làm việc/ca trực của bác sĩ. |
| 3 | **appointment-service** | `8083` | `medicare_appointment_db` | Đặt lịch khám, quản lý trạng thái lịch khám (PENDING, CONFIRMED, COMPLETED, CANCELLED) và tránh trùng ca khám. |
| 4 | **medical-record-service** | `8084` | `medicare_medical_record_db` | Ghi nhận bệnh án, triệu chứng, chẩn đoán của bác sĩ và chi tiết các đơn thuốc sau khám. |
| 5 | **pharmacy-service** | `8085` | `medicare_pharmacy_db` | Quản lý danh mục thuốc, quản lý lô hàng tồn kho dược và xử lý xuất cấp thuốc theo đơn. |

---

### 1.3. Giải thích lý do chia tách các service

- **Patient Service (`8081`)**:
  - Dữ liệu bệnh nhân mang tính chất thông tin định danh dài hạn và tần suất đọc cao.
  - Tách riêng giúp bảo vệ thông tin cá nhân và bảo mật dữ liệu nhạy cảm theo chuẩn y tế.
  - Cho phép tích hợp với các hệ thống cổng bảo hiểm y tế quốc gia mà không ảnh hưởng tới các dịch vụ lâm sàng khác.

- **Doctor Service (`8082`)**:
  - Quản lý danh mục nhân sự y tế, chuyên khoa và ca làm việc theo tuần/tháng.
  - Phân tách độc lập giúp phòng tổ chức cán bộ/quản trị nhân sự cập nhật ca trực độc lập với các giao dịch khám chữa bệnh.
  - Tối ưu hóa việc tra cứu bác sĩ theo chuyên khoa cho bệnh nhân.

- **Appointment Service (`8083`)**:
  - Nghiệp vụ đặt lịch có tính chất giao dịch tức thời cao (concurrency cao trong khung giờ cao điểm).
  - Cần khả năng mở rộng (scale) độc lập khi số lượng người dùng đặt lịch online tăng đột biến.
  - Đảm bảo luồng đặt hẹn diễn ra mượt mà mà không làm nghẽn hệ thống bệnh án hay kho thuốc.

- **Medical Record Service (`8084`)**:
  - Bệnh án điện tử (EMR) là dữ liệu trọng yếu, yêu cầu tính toàn vẹn cao và lưu trữ lâu dài.
  - Việc tách riêng giúp tối ưu hóa dung lượng lưu trữ lớn và bảo đảm tính bảo mật nghiêm ngặt.
  - Cho phép các phòng khám/bác sĩ truy xuất lịch sử chẩn đoán nhanh chóng mà không phụ thuộc vào kho dược hay đặt lịch.

- **Pharmacy Service (`8085`)**:
  - Quản lý quy trình xuất nhập tồn kho dược phẩm với yêu cầu kiểm soát hạn sử dụng, số lô và số lượng tồn.
  - Tách riêng để nhân viên quầy dược thao tác xuất thuốc theo đơn độc lập với quy trình khám bệnh.
  - Dễ dàng mở rộng kết nối với hệ thống nhà cung cấp thuốc hoặc kiểm kê tài chính kho.

---

## 2. Yêu Cầu 2 — Thiết Kế Database Cho Từng Service (Database-per-Service)

### 2.1. Tại sao không nên chia sẻ chung một Database giữa các Microservice?
1. **Phá vỡ tính độc lập (Tight Coupling)**: Khi nhiều service cùng đọc/ghi vào một database, thay đổi schema của một bảng có thể làm sập các service khác (Break changes).
2. **Khó scale độc lập**: Không thể tối ưu phần cứng hay cấu hình database riêng cho từng nghiệp vụ (ví dụ: service đọc nhiều cần read-replica, service ghi nhiều cần storage tốc độ cao).
3. **Nguy cơ nghẽn cổ chai và Deadlock**: Giao dịch khóa bảng (table lock/row lock) từ một service có thể làm treo toàn bộ các service khác dùng chung DB.
4. **Vi phạm ranh giới Bounded Context**: Dễ dẫn đến việc viết query JOIN phức tạp xuyên module, biến kiến trúc thành Distributed Monolith.

---

### 2.2. Xử lý liên kết dữ liệu giữa các Microservices (Data Correlation)
- **Quy tắc**: Không sử dụng khóa ngoại vật lý (Foreign Key) cross-database và tuyệt đối không dùng câu lệnh `JOIN` qua lại giữa các database khác nhau.
- **Cách xử lý**:
  - **Lưu ID tham chiếu (Loose Coupling)**: Các service chỉ lưu ID định danh của đối tượng thuộc service khác (ví dụ: `patient_id`, `doctor_id`, `appointment_id`).
  - **Giao tiếp qua REST API / gRPC**: Khi cần hiển thị thông tin tổng hợp, service phía trên sẽ gọi HTTP/REST sang service quản lý dữ liệu gốc qua ID.
  - **Event-Driven / Message Broker**: Sử dụng sự kiện bất đồng bộ (Kafka/RabbitMQ) để đồng bộ trạng thái khi có thay đổi (ví dụ: Bác sĩ tạo đơn thuốc -> bắn event sang Pharmacy Service để trừ tồn kho).

---

### 2.3. Sơ đồ ERD & Script DDL chi tiết cho từng Database

#### A. Database: `medicare_patient_db` (Patient Service)

```
+----------------------------------------------------------------+
|                           PATIENTS                             |
+----------------------------------------------------------------+
| * id           : BIGINT (PK, AI)                               |
|   full_name    : VARCHAR(100) NOT NULL                         |
|   date_of_birth: DATE NOT NULL                                 |
|   gender       : ENUM('MALE','FEMALE','OTHER') NOT NULL        |
|   phone        : VARCHAR(15)                                   |
|   address      : VARCHAR(255)                                  |
|   insurance_id : VARCHAR(20)                                   |
|   created_at   : DATETIME                                      |
|   updated_at   : DATETIME                                      |
+----------------------------------------------------------------+
```

```sql
CREATE DATABASE IF NOT EXISTS medicare_patient_db;
USE medicare_patient_db;

CREATE TABLE patients (
    id            BIGINT AUTO_INCREMENT PRIMARY KEY,
    full_name     VARCHAR(100) NOT NULL,
    date_of_birth DATE NOT NULL,
    gender        ENUM('MALE','FEMALE','OTHER') NOT NULL,
    phone         VARCHAR(15),
    address       VARCHAR(255),
    insurance_id  VARCHAR(20),
    created_at    DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at    DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

---

#### B. Database: `medicare_doctor_db` (Doctor Service)

```
+------------------------------------+       +------------------------------------+
|            DEPARTMENTS             |       |              DOCTORS               |
+------------------------------------+       +------------------------------------+
| * id         : BIGINT (PK, AI)     | 1   n | * id           : BIGINT (PK, AI)   |
|   dept_code  : VARCHAR(20) NOT NULL|-------|   dept_id      : BIGINT (FK)       |
|   dept_name  : VARCHAR(100) NOTNULL|       |   full_name    : VARCHAR(100)      |
|   description: VARCHAR(255)        |       |   phone        : VARCHAR(15)       |
+------------------------------------+       |   email        : VARCHAR(100)      |
                                             |   specialization: VARCHAR(100)     |
                                             |   created_at   : DATETIME          |
                                             |   updated_at   : DATETIME          |
                                             +-----------------+------------------+
                                                               | 1
                                                               |
                                                               | n
                                             +-----------------+------------------+
                                             |          DOCTOR_SCHEDULES          |
                                             +------------------------------------+
                                             | * id           : BIGINT (PK, AI)   |
                                             |   doctor_id    : BIGINT (FK)       |
                                             |   work_date    : DATE NOT NULL     |
                                             |   shift        : ENUM('MORNING',   |
                                             |                       'AFTERNOON', |
                                             |                       'NIGHT')     |
                                             |   room_no      : VARCHAR(20)       |
                                             +------------------------------------+
```

```sql
CREATE DATABASE IF NOT EXISTS medicare_doctor_db;
USE medicare_doctor_db;

CREATE TABLE departments (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    dept_code   VARCHAR(20) NOT NULL UNIQUE,
    dept_name   VARCHAR(100) NOT NULL,
    description VARCHAR(255)
);

CREATE TABLE doctors (
    id             BIGINT AUTO_INCREMENT PRIMARY KEY,
    dept_id        BIGINT NOT NULL,
    full_name      VARCHAR(100) NOT NULL,
    phone          VARCHAR(15),
    email          VARCHAR(100),
    specialization VARCHAR(100),
    created_at     DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at     DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    CONSTRAINT fk_doctor_department FOREIGN KEY (dept_id) REFERENCES departments(id)
);

CREATE TABLE doctor_schedules (
    id         BIGINT AUTO_INCREMENT PRIMARY KEY,
    doctor_id  BIGINT NOT NULL,
    work_date  DATE NOT NULL,
    shift      ENUM('MORNING', 'AFTERNOON', 'NIGHT') NOT NULL,
    room_no    VARCHAR(20),
    CONSTRAINT fk_schedule_doctor FOREIGN KEY (doctor_id) REFERENCES doctors(id)
);
```

---

#### C. Database: `medicare_appointment_db` (Appointment Service)

```
+----------------------------------------------------------------+
|                          APPOINTMENTS                          |
+----------------------------------------------------------------+
| * id               : BIGINT (PK, AI)                           |
|   patient_id       : BIGINT NOT NULL (Ref: Patient Service)    |
|   doctor_id        : BIGINT NOT NULL (Ref: Doctor Service)     |
|   appointment_date : DATE NOT NULL                             |
|   appointment_time : TIME NOT NULL                             |
|   reason           : VARCHAR(255)                              |
|   status           : ENUM('PENDING','CONFIRMED',               |
|                           'COMPLETED','CANCELLED') NOT NULL    |
|   created_at       : DATETIME                                  |
|   updated_at       : DATETIME                                  |
+----------------------------------------------------------------+
```

```sql
CREATE DATABASE IF NOT EXISTS medicare_appointment_db;
USE medicare_appointment_db;

CREATE TABLE appointments (
    id               BIGINT AUTO_INCREMENT PRIMARY KEY,
    patient_id       BIGINT NOT NULL, -- Ref to Patient Service (no FK constraint)
    doctor_id        BIGINT NOT NULL, -- Ref to Doctor Service (no FK constraint)
    appointment_date DATE NOT NULL,
    appointment_time TIME NOT NULL,
    reason           VARCHAR(255),
    status           ENUM('PENDING', 'CONFIRMED', 'COMPLETED', 'CANCELLED') DEFAULT 'PENDING',
    created_at       DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at       DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

---

#### D. Database: `medicare_medical_record_db` (Medical Record Service)

```
+------------------------------------+       +------------------------------------+
|          MEDICAL_RECORDS           |       |            PRESCRIPTIONS           |
+------------------------------------+       +------------------------------------+
| * id            : BIGINT (PK, AI)  | 1   n | * id           : BIGINT (PK, AI)   |
|   patient_id    : BIGINT (Ref Ptn) |-------|   record_id    : BIGINT (FK)       |
|   doctor_id     : BIGINT (Ref Doc) |       |   medicine_id  : BIGINT (Ref Pharm)|
|   appointment_id: BIGINT (Ref App) |       |   medicine_name: VARCHAR(100)      |
|   diagnosis     : TEXT NOT NULL    |       |   dosage       : VARCHAR(100)      |
|   treatment_plan: TEXT             |       |   quantity     : INT NOT NULL      |
|   visit_date    : DATETIME NOT NULL|       |   instructions : VARCHAR(255)      |
|   created_at    : DATETIME         |       +------------------------------------+
+------------------------------------+
```

```sql
CREATE DATABASE IF NOT EXISTS medicare_medical_record_db;
USE medicare_medical_record_db;

CREATE TABLE medical_records (
    id             BIGINT AUTO_INCREMENT PRIMARY KEY,
    patient_id     BIGINT NOT NULL, -- Ref to Patient Service
    doctor_id      BIGINT NOT NULL, -- Ref to Doctor Service
    appointment_id BIGINT,          -- Ref to Appointment Service
    diagnosis      TEXT NOT NULL,
    treatment_plan TEXT,
    visit_date     DATETIME NOT NULL,
    created_at     DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE prescriptions (
    id            BIGINT AUTO_INCREMENT PRIMARY KEY,
    record_id     BIGINT NOT NULL,
    medicine_id   BIGINT NOT NULL,  -- Ref to Pharmacy Service
    medicine_name VARCHAR(100) NOT NULL,
    dosage        VARCHAR(100) NOT NULL,
    quantity      INT NOT NULL,
    instructions  VARCHAR(255),
    CONSTRAINT fk_prescription_record FOREIGN KEY (record_id) REFERENCES medical_records(id)
);
```

---

#### E. Database: `medicare_pharmacy_db` (Pharmacy Service)

```
+------------------------------------+       +------------------------------------+
|             MEDICINES              |       |          INVENTORY_STOCKS          |
+------------------------------------+       +------------------------------------+
| * id          : BIGINT (PK, AI)    | 1   n | * id          : BIGINT (PK, AI)    |
|   name        : VARCHAR(100) NOTNUL|-------|   medicine_id : BIGINT (FK)        |
|   category    : VARCHAR(50)        |       |   batch_number: VARCHAR(50) NOTNUL |
|   unit        : VARCHAR(20) NOTNULL|       |   quantity    : INT NOT NULL       |
|   unit_price  : DECIMAL(10,2)      |       |   expiry_date : DATE NOT NULL      |
+-----------------+------------------+       +------------------------------------+
                  | 1
                  |
                  | n
+-----------------+------------------+
|       PRESCRIPTION_DISPENSES       |
+------------------------------------+
| * id             : BIGINT (PK, AI) |
|   prescription_id: BIGINT(Ref MedR)|
|   medicine_id    : BIGINT (FK)     |
|   dispense_qty   : INT NOT NULL    |
|   dispense_date  : DATETIME        |
|   dispensed_by   : VARCHAR(100)    |
|   status         : ENUM('PENDING', |
|                         'DISPENSED'|
|                         'CANCEL')  |
+------------------------------------+
```

```sql
CREATE DATABASE IF NOT EXISTS medicare_pharmacy_db;
USE medicare_pharmacy_db;

CREATE TABLE medicines (
    id         BIGINT AUTO_INCREMENT PRIMARY KEY,
    name       VARCHAR(100) NOT NULL,
    category   VARCHAR(50),
    unit       VARCHAR(20) NOT NULL,
    unit_price DECIMAL(10, 2) NOT NULL
);

CREATE TABLE inventory_stocks (
    id           BIGINT AUTO_INCREMENT PRIMARY KEY,
    medicine_id  BIGINT NOT NULL,
    batch_number VARCHAR(50) NOT NULL,
    quantity     INT NOT NULL DEFAULT 0,
    expiry_date  DATE NOT NULL,
    CONSTRAINT fk_stock_medicine FOREIGN KEY (medicine_id) REFERENCES medicines(id)
);

CREATE TABLE prescription_dispenses (
    id              BIGINT AUTO_INCREMENT PRIMARY KEY,
    prescription_id BIGINT NOT NULL, -- Ref to Medical Record Service
    medicine_id     BIGINT NOT NULL,
    dispense_qty    INT NOT NULL,
    dispense_date   DATETIME DEFAULT CURRENT_TIMESTAMP,
    dispensed_by    VARCHAR(100),
    status          ENUM('PENDING', 'DISPENSED', 'CANCELLED') DEFAULT 'PENDING',
    CONSTRAINT fk_dispense_medicine FOREIGN KEY (medicine_id) REFERENCES medicines(id)
);
```

---

## 3. Yêu Cầu 3 — Cấu Hình `application.yml` Cho Từng Service

### 3.1. `patient-service.yml`
```yaml
server:
  port: 8081

spring:
  application:
    name: patient-service
  datasource:
    url: jdbc:mysql://localhost:3306/medicare_patient_db
    username: root
    password: root
    driver-class-name: com.mysql.cj.jdbc.Driver
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQLDialect
```

### 3.2. `doctor-service.yml`
```yaml
server:
  port: 8082

spring:
  application:
    name: doctor-service
  datasource:
    url: jdbc:mysql://localhost:3306/medicare_doctor_db
    username: root
    password: root
    driver-class-name: com.mysql.cj.jdbc.Driver
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQLDialect
```

### 3.3. `appointment-service.yml`
```yaml
server:
  port: 8083

spring:
  application:
    name: appointment-service
  datasource:
    url: jdbc:mysql://localhost:3306/medicare_appointment_db
    username: root
    password: root
    driver-class-name: com.mysql.cj.jdbc.Driver
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQLDialect
```

### 3.4. `medical-record-service.yml`
```yaml
server:
  port: 8084

spring:
  application:
    name: medical-record-service
  datasource:
    url: jdbc:mysql://localhost:3306/medicare_medical_record_db
    username: root
    password: root
    driver-class-name: com.mysql.cj.jdbc.Driver
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQLDialect
```

### 3.5. `pharmacy-service.yml`
```yaml
server:
  port: 8085

spring:
  application:
    name: pharmacy-service
  datasource:
    url: jdbc:mysql://localhost:3306/medicare_pharmacy_db
    username: root
    password: root
    driver-class-name: com.mysql.cj.jdbc.Driver
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQLDialect
```
