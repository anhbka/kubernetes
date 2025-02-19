## Argo CD - Declarative GitOps CD for Kubernetes

### 1. ArgoCD 

ArgoCD (Argo Continuous Delivery) là một công cụ triển khai liên tục được xây dựng đặc biệt cho Kubernetes. Được phát triển như một dự án mã nguồn mở, ArgoCD đã nhanh chóng trở thành một trong những công cụ được ưa chuộng nhất trong cộng đồng DevOps và Kubernetes. Để hiểu rõ hơn về ArgoCD, chúng ta sẽ đi sâu vào ba khía cạnh chính: định nghĩa cơ bản, nguyên lý hoạt động và vai trò của nó trong quy trình DevOps hiện đại.

![alt text](https://argo-cd.readthedocs.io/en/stable/assets/argocd-ui.gif)

ArgoCD hoạt động dựa trên nguyên tắc GitOps, một phương pháp quản lý cấu hình và triển khai ứng dụng sử dụng Git làm nguồn single source of truth về trạng thái mong muốn của hệ thống. Có nghĩa là tất cả các cấu hình, manifest và định nghĩa về ứng dụng đều được lưu trữ trong một kho lưu trữ Git.

### Tại sao lại sử dụng ArgoCD?

  * Quản lý phiên bản: Áp dụng ArgoCD sẽ giúp bạn khai báo, quản lý các định nghĩa, cấu hình và môi trường của ứng dụng được triển khai theo từng phiên bản.
 
  * Tự động hóa: ArgoCD sẽ giúp bạn tự động hoá, dễ dàng kiểm tra việc triển khai và quản lý vòng đời ứng dụng.

### 2. Các chức năng chính của ArgoCD  
  
  * Continuous Deployment (CD): ArgoCD tự động triển khai ứng dụng khi có thay đổi trong mã nguồn, giúp cập nhật và triển khai nhanh chóng.
   
  * Khả năng theo dõi và quan sát (Monitoring): ArgoCD cung cấp các công cụ giám sát trạng thái ứng dụng để bạn dễ phát hiện và xử lý vấn đề phát sinh nhanh chóng.
   
  * Quản lý ứng dụng (Application Management): ArgoCD cung cấp API để bạn dễ dàng quản lý và theo dõi trạng thái ứng dụng trên giao diện thân thiện.
   
  * Xử lý lỗi và khôi phục (Error Handling and Rollback): ArgoCD tự động khôi phục lại trạng thái trước đó khi phát hiện lỗi trong quá trình triển khai đảm bảo ứng dụng luôn hoạt động ổn định.
   
  * Quản lý nhiều cluster (Multi-Cluster Management): ArgoCD hỗ trợ quản lý và triển khai ứng dụng trên nhiều cụm K8s từ một nơi duy nhất.
   
  * Hỗ trợ nhiều nguồn GIT khác nhau: ArgoCD có thể tích hợp với nhiều nguồn GIT khác nhau như GitHub, GitLab, Bitbucket và nhiều nguồn khác.
   
  * Hỗ trợ các công cụ và các tiêu chuẩn hiện đại: ArgoCD hỗ trợ tích hợp các công cụ và tiêu chuẩn hiện đại như GitOps, Helm, … và các công cụ CI/CD khác.  
  
### 3. Kiến trúc của ArgoCD  

* ArgoCD bao gồm 3 thành phần chính như hình ảnh bạn đang thấy phía trên:

  * API server
  
  * Repository Service
  
  * Application Controller
  
![test2](https://argo-cd.readthedocs.io/en/stable/assets/argocd_architecture.png)  

### 3.1 API Server

Đây là một gRPC/API server được xuất ra thành các APIs để các components khác như là CLI, WebUI và các CI/CD khác sử dụng.

API Server này chịu trách nhiệm về:

  * Quản lý và báo cáo trạng thái ứng dụng.
  
  * Thực hiện các tác vụ liên quan đến vận hành như: Sync, rollback và các tác vụ do người dùng định nghĩa ra.
  
  * Quản lý các credentials cluster và repository được lưu trữ như một Kubernetes secrets.
  
  * Cung cấp Single Sign-On.
  
  * Thực hiện phân quyền (RBAC – Role-Based Access Control).
  
  * Thực hiện lắng nghe và chuyển tiếp các sự kiện của Git webhook.
  
### 3.2 Repository Server

Đây là một internal service để local cache lại Git repository, lưu trữ những application manifests.

Repository Service sẽ tạo ra Kubernetes manifests và trả về chúng dựa trên những thông tin như repository URL, application path (đường dẫn đến file manifest), revisions (i.e., commits, tags, branches) và những template-specific settings như Helm values, parameters.

  * repository URL
  
  * revision (commit, tag, branch)
  
  * application path
  
  * template specific settings: parameters, helm values.yaml

### 3.3 Application Controller

Đây là một Kubernetes controller, nó sẽ tiếp tục theo dõi trạng thái ứng dụng, so sánh trạng thái mong muốn được lưu trữ trên Git repository và trạng thái hiện tại của ứng dụng trên các cụm K8s.

Nó phát hiện sự thay đổi (OutOfSync) của ứng dụng và có áp dụng các thay đổi này. Nó gọi các hook do người dùng định nghĩa cho các sự kiện trong vòng đời ứng dụng như PreSync, Sync và PostSync.