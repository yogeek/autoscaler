# Cluster Autoscaler on AliCloud
The cluster autoscaler on AliCloud scales worker nodes within any specified autoscaling group. It will run as a `Deployment` in your cluster. This README will go over some of the necessary steps required to get the cluster autoscaler up and running.

## Kubernetes Version
Cluster autoscaler must run on v1.9.3 or greater.

## Instance Type Support 
- **Standard Instance**x86-Architecture,suitable for common scenes such as websites or api services.
- **GPU/FPGA Instance**Heterogeneous Computing,suitable for high performance computing.
- **Bare Metal Instance**Both the elasticity of a virtual server and the high-performance and comprehensive features of a physical server.
- **Spot Instance**Spot instance are on-demand instances. They are designed to reduce your ECS costs in some cases.


## ACS Console Deployment 
doc: https://www.alibabacloud.com/help/doc-detail/89733.html

## Custom Deployment
### 1.Prepare Identity authentication
#### Use access-key-id and access-key-secret 
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cloud-config
  namespace: kube-system
data:
  # insert your base64 encoded Alicloud access id and key here, ensure there's no trailing newline:
  # such as:  echo -n "your_access_key_id" | base64
  access-key-id: "<BASE64_ACCESS_KEY_ID>"
  access-key-secret: "<BASE64_ACCESS_KEY_SECRET>"
  region-id: "<BASE64_REGION_ID>"
```
#### Use STS with RAM Role 
```yaml 
{
  "Version": "1",
  "Statement": [
    {
      "Action": [
        "ess:Describe*",
        "ess:CreateScalingRule",
        "ess:ModifyScalingGroup",
        "ess:RemoveInstances",
        "ess:ExecuteScalingRule",
        "ess:ModifyScalingRule",
        "ess:DeleteScalingRule",
        "ess:DetachInstances",
        "ecs:DescribeInstanceTypes"
      ],
      "Resource": [
        "*"
      ],
      "Effect": "Allow"
    }
  ]
}
```

### 2.ASG Setup 
* create a Scaling Group in ESS(https://essnew.console.aliyun.com) with valid configurations.
* create a Scaling Configuration for this Scaling Group with valid instanceType and User Data.In User Data,you can specific the script to initialize the environment and join this node to kubernetes cluster.If your Kubernetes cluster is hosted by ACS.you can use the attach script like this.
```shell
#!/bin/sh
# The token is generated by ACS console. https://www.alibabacloud.com/help/doc-detail/64983.htm?spm=a2c63.l28256.b99.33.46395ad54ozJFq
curl http://aliacs-k8s-cn-hangzhou.oss-cn-hangzhou.aliyuncs.com/public/pkg/run/attach/[kubernetes_cluster_version]/attach_node.sh | bash -s -- --openapi-token [token] --ess true 
```


### 3.cluster-autoscaler deployment 

#### Use access-key-id and access-key-secret 
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  labels:
    app: cluster-autoscaler
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
    spec:
      priorityClassName: system-cluster-critical
      serviceAccountName: admin
      containers:
        - image: registry.cn-hangzhou.aliyuncs.com/acs/autoscaler:v1.3.1.2
          name: cluster-autoscaler
          resources:
            limits:
              cpu: 100m
              memory: 300Mi
            requests:
              cpu: 100m
              memory: 300Mi
          command:
            - ./cluster-autoscaler
            - --v=4
            - --stderrthreshold=info
            - --cloud-provider=alicloud
            - --nodes=[min]:[max]:[ASG_ID]
          imagePullPolicy: "Always"
          env:
          - name: ACCESS_KEY_ID
            valueFrom:
              secretKeyRef:
                name: cloud-config
                key: access-key-id
          - name: ACCESS_KEY_SECRET
            valueFrom:
              secretKeyRef:
                name: cloud-config
                key: access-key-secret
          - name: REGION_ID
            valueFrom:
            secretKeyRef:
              name: cloud-config
              key: region-id
          volumeMounts:
            - name: ssl-certs
              mountPath: /etc/ssl/certs/ca-certificates.crt
              readOnly: true
          imagePullPolicy: "Always"
      volumes:
        - name: ssl-certs
          hostPath:
            path: "/etc/ssl/certs/ca-certificates.crt"
```

#### Use STS with RAM Role 

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  labels:
    app: cluster-autoscaler
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
    spec:
      priorityClassName: system-cluster-critical
      serviceAccountName: admin
      containers:
        - image: registry.cn-hangzhou.aliyuncs.com/acs/autoscaler:v1.3.1.2
          name: cluster-autoscaler
          resources:
            limits:
              cpu: 100m
              memory: 300Mi
            requests:
              cpu: 100m
              memory: 300Mi
          command:
            - ./cluster-autoscaler
            - --v=4
            - --stderrthreshold=info
            - --cloud-provider=alicloud
            - --nodes=[min]:[max]:[ASG_ID]
          imagePullPolicy: "Always"
```

### Auto-Discovery Setup
Auto Discovery is not supported in AliCloud currently.

## Common Notes and Gotchas:
- The `/etc/ssl/certs/ca-certificates.crt` should exist by default on your ecs instance.
- By default, cluster autoscaler will not terminate nodes running pods in the kube-system namespace. You can override this default behaviour by passing in the `--skip-nodes-with-system-pods=false` flag.
- By default, cluster autoscaler will wait 10 minutes between scale down operations, you can adjust this using the `--scale-down-delay` flag. E.g. `--scale-down-delay=5m` to decrease the scale down delay to 5 minutes.
- If you're running multiple ASGs, the `--expander` flag supports three options: `random`, `most-pods` and `least-waste`. `random` will expand a random ASG on scale up. `most-pods` will scale up the ASG that will scheduable the most amount of pods. `least-waste` will expand the ASG that will waste the least amount of CPU/MEM resources. In the event of a tie, cluster-autoscaler will fall back to `random`.