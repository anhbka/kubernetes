### Configure Quality of Service for Pods

Quality of Service (QoS) trong Kubernetes là một cơ chế để quản lý và phân bổ tài nguyên cho các Pod. Kubernetes sử dụng QoS để xác định cách thức mà các Pod nên được ưu tiên trong việc phân bổ tài nguyên, đặc biệt trong các tình huống tài nguyên bị hạn chế. Có ba loại QoS chính trong Kubernetes:

### 1. Guaranteed QoS

  * Đặc Điểm: Pods được phân loại là Guaranteed khi họ có cấu hình tài nguyên (CPU và RAM) cho cả yêu cầu và giới hạn. Yêu cầu và giới hạn phải giống nhau cho mỗi tài nguyên.

  * Cách Định Nghĩa

```
apiVersion: v1
kind: Pod
metadata:
  name: guaranteed-pod
spec:
  containers:
  - name: mycontainer
    image: myimage
    resources:
      requests:
        memory: "256Mi"
        cpu: "500m"
      limits:
        memory: "256Mi"
        cpu: "500m"  
```  
  * Ưu Điểm: Các Pod Guaranteed sẽ không bị kill trong trường hợp tài nguyên bị thiếu, vì chúng được ưu tiên cao nhất.
  
### 2. Burstable QoS  
  
  * Đặc Điểm: Pods được phân loại là Burstable khi chúng có yêu cầu tài nguyên, nhưng giới hạn tài nguyên có thể lớn hơn yêu cầu. Điều này cho phép các Pod này sử dụng nhiều tài nguyên hơn khi có sẵn.

  * Cách Định Nghĩa:
  
```
apiVersion: v1
kind: Pod
metadata:
  name: burstable-pod
spec:
  containers:
  - name: mycontainer
    image: myimage
    resources:
      requests:
        memory: "128Mi"
        cpu: "250m"
      limits:
        memory: "512Mi"
        cpu: "1"
```		
  * Ưu Điểm: Các Pod Burstable có thể sử dụng nhiều tài nguyên hơn khi có sẵn, nhưng sẽ bị giới hạn khi tài nguyên bị thiếu.
  
### 3. Best-Effort QoS  
  
  * Đặc Điểm: Pods được phân loại là Best-Effort khi không có yêu cầu tài nguyên nào được chỉ định. Điều này có nghĩa là chúng có thể sử dụng tài nguyên khi có sẵn, nhưng không có bảo đảm gì.
  * Cách Định Nghĩa:  
  
```
apiVersion: v1
kind: Pod
metadata:
  name: besteffort-pod
spec:
  containers:
  - name: mycontainer
    image: myimage
    resources: {}  # Không có yêu cầu hay giới hạn
```	
  * Ưu Điểm: Các Pod Best-Effort sẽ bị ưu tiên thấp nhất và có thể bị kill bất kỳ lúc nào nếu tài nguyên không đủ. 