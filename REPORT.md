# Báo cáo kỹ thuật: Kiến trúc Container vs Virtual Machine

**Sinh viên:** Nguyễn Thế Kiên
**Bối cảnh:** Phân tích lý do doanh nghiệp chuyển từ mô hình VPS (máy ảo) sang Docker (container).

---

## 1. Tổng quan kiến trúc

| Virtual Machine (VPS) | Container (Docker) |
|---|---|
| Hardware → Host OS → **Hypervisor** → Guest OS (mỗi VM một kernel riêng) → Libs → App | Hardware → Host OS (**dùng chung 1 Linux kernel**) → **Container Engine** → Libs → App |

- **VM là ảo hóa phần cứng.** Hypervisor (KVM, VMware ESXi, Hyper-V) giả lập CPU, RAM, ổ đĩa, card mạng cho từng máy ảo. Mỗi VM phải cài và chạy một hệ điều hành đầy đủ.
- **Container là ảo hóa hệ điều hành.** Không có Guest OS. Container chỉ là một nhóm process chạy trực tiếp trên kernel của máy host. Process đó bị kernel "rào" lại bằng Namespaces và bị giới hạn tài nguyên bằng cgroups.

## 2. Bảng so sánh VM và Container theo 5 tiêu chí

| Tiêu chí | Virtual Machine | Container | Ý nghĩa với doanh nghiệp |
|---|---|---|---|
| **Dung lượng Image** | Vài GB đến hàng chục GB, vì chứa cả Guest OS | Vài MB đến vài trăm MB, chỉ chứa app và thư viện. Ví dụ `alpine` khoảng 8 MB, image Spring Boot khoảng 200–300 MB | Tốn ít ổ đĩa hơn, kéo/đẩy image qua mạng nhanh hơn |
| **Thời gian khởi động** | Vài chục giây đến vài phút, vì phải boot cả OS | Dưới 1 giây đến vài giây, vì chỉ start một process | Deploy, rollback và tự phục hồi gần như tức thì |
| **Hao phí OS (overhead)** | Cao. Mỗi VM tốn sẵn RAM/CPU cho kernel và dịch vụ hệ thống của Guest OS | Rất thấp. Dùng chung kernel host, không có OS trung gian | Cùng một server chạy được nhiều ứng dụng hơn, giảm chi phí hạ tầng |
| **Mức độ cô lập** | Rất mạnh. Mỗi VM có kernel riêng, ranh giới do phần cứng ảo hóa đảm bảo | Ở mức process. Nếu kernel có lỗ hổng thì có thể ảnh hưởng mọi container | VM phù hợp với môi trường nhiều khách hàng không tin nhau. Container phù hợp với các service nội bộ của cùng một hệ thống |
| **Khả năng mở rộng** | Chậm. Phải tạo VM mới, cài OS và cấu hình (tính bằng phút) | Nhanh. `docker compose up --scale app=5` hoặc Kubernetes tạo thêm bản sao trong vài giây | Dễ scale theo tải, hợp với kiến trúc microservices và CI/CD |

**Kết luận:** Doanh nghiệp chuyển từ VPS sang Docker vì 3 lý do chính:
- Tiết kiệm tài nguyên.
- Triển khai nhanh và đồng nhất: build một lần, chạy ở mọi nơi, hết cảnh "máy em chạy được".
- Dễ tự động hóa bằng pipeline CI/CD.

VM không bị thay thế hoàn toàn. Thực tế thường kết hợp cả hai: thuê VPS, rồi chạy Docker bên trong VPS đó.

## 3. Cơ chế cô lập: Namespaces và Control Groups (cgroups)

Container không phải "máy ảo nhẹ". Nó được ghép từ hai tính năng có sẵn trong Linux kernel.

### 3.1. Namespaces: giới hạn những gì process **nhìn thấy**

Namespace tạo cho mỗi container một góc nhìn riêng về tài nguyên hệ thống:

| Namespace | Cô lập cái gì | Ví dụ trong Docker |
|---|---|---|
| **PID** | Danh sách process | Trong container, app là `PID 1` và không thấy process nào của host hay container khác |
| **NET** | Card mạng, IP, bảng định tuyến, cổng | Mỗi container có `eth0` và IP riêng (ví dụ `172.18.0.2`). Hai container cùng lắng nghe cổng 8080 mà không xung đột |
| **MNT** | Mount point, cây thư mục | Container thấy `/` là filesystem của image, không thấy `/home` của host trừ khi được mount volume |
| UTS | Hostname | `hostname` bên trong là ID của container |
| IPC | Shared memory, message queue | Process ở các container khác nhau không giao tiếp IPC trực tiếp với nhau |
| USER | UID/GID | `root` trong container có thể được map thành user thường ở host (rootless) |

### 3.2. cgroups: giới hạn những gì process **được dùng**

Namespace chỉ che tầm nhìn chứ không ngăn được việc chiếm tài nguyên. cgroups làm việc này: kernel đặt hạn mức và đo mức sử dụng tài nguyên của từng nhóm process.

- **CPU:** `docker run --cpus=1.5` giới hạn container dùng tối đa 1.5 core.
- **RAM:** `docker run --memory=512m` đặt trần 512 MB. Vượt trần thì kernel OOM-kill container, các container khác không bị ảnh hưởng.
- Ngoài ra cgroups còn giới hạn I/O ổ đĩa (`--device-read-bps`) và số process (`--pids-limit`).

**Tóm lại: Container = Process + Namespaces (cô lập) + cgroups (giới hạn) + Union filesystem (image nhiều layer).**

## 4. Thông số hệ thống từ lệnh `docker info`

Lệnh đã chạy bằng user thường, không cần `sudo`:

```bash
docker version
docker info
docker info | grep -i cgroup
```

| Thông số | Giá trị trên máy |
|---|---|
| Server Version | `<điền từ docker info>` |
| Cgroup Driver | `<điền, ví dụ: cgroupfs>` |
| Cgroup Version | `<điền, ví dụ: 2>` |
| Containers (Running / Paused / Stopped) | `<tổng> (<running> / <paused> / <stopped>)` |
| Images | `<điền>` |
| Operating System / Kernel | `<điền, ví dụ: Docker Desktop / 6.x linuxkit>` |

**Nhận xét:**
- **Cgroup Driver** là thành phần Docker dùng để tạo và quản lý cgroups cho container:
  - `cgroupfs`: Docker tự ghi trực tiếp vào `/sys/fs/cgroup`.
  - `systemd`: giao việc quản lý cgroups cho systemd. Đây là lựa chọn khuyến nghị trên Ubuntu Server và Kubernetes để tránh có hai trình quản lý cgroups cùng lúc.
- **Docker Desktop trên macOS** chạy Docker Engine bên trong một VM Linux nhỏ (LinuxKit), vì macOS không có Namespaces và cgroups. Dòng Kernel Version trong `docker info` vì vậy là kernel Linux, không phải kernel macOS. Điều này cho thấy container luôn cần một kernel Linux bên dưới.
- Số Containers và Images phản ánh các bài thực hành trước: `student-api`, `student-mysql`, các image build từ Dockerfile single-stage và multi-stage.

