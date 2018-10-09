## Helm使用教程全解析

### 项目中如何使用

-  针对每个项目形成一个chart，最后形成一个Chart Package

比如下面针对hello-svc这个基于tomcat的项目，先生成一个chart的结构

- 创建chart及部署

    [root@k8s-master ~]# helm create hello-svc
    Creating hello-svc

按照我们自己的需求修改模板中的deployment.yaml,service.yaml和values.yaml文件


    [root@k8s-master templates]# cat deployment.yaml 
    apiVersion: extensions/v1beta1
    kind: Deployment
    metadata:
      name: tomcatjmx
    spec:
      replicas: {{.Values.replicas}}
      template:
    metadata:
      labels:
    tomcat-app: "tomcatjmx"
    version: "1"
    spec:
      containers:
      - name: tomcatjmx
    image: tomcat:{{.Values.images.dockerTag}}
    ports:
    - containerPort: {{.Values.images.Port}}
      name: tomcatport
    - containerPort: 35135
      name: jmx






    [root@k8s-master templates]# cat service.yaml 
    apiVersion: v1
    kind: Service
    metadata:
      name: {{.Values.service.name}} 
      labels:
    tomcat-app: tomcatjmx
    spec:
      ports:
      - port: {{.Values.service.Port}} 
    protocol: TCP
    targetPort: 8080
    name: http
      - name: jmx
    protocol: TCP
    port: 35135
    targetPort: {{.Values.service.targetPort}}
      type: NodePort
      selector:
    tomcat-app: tomcatjmx



    [root@k8s-master hello-svc]# cat values.yaml 
    # Default values for hello-svc.
    # This is a YAML-formatted file.
    # Declare variables to be passed into your templates.
    
    replicas: 1
    
    images:
      dockerTag: jmxv4 
      Port: 8080
    
    service:
      name: tomcatjmxsvc
      Port: 80
      targetPort: 35135


相应的NOTES.txt也进行调整直到验证没有问题，验证完成通过install安装


    #helm install --dry-run --debug ./




    [root@k8s-master hello-svc]# helm install ./
    NAME:   kindly-worm
    LAST DEPLOYED: Sat Feb 24 14:45:58 2018
    NAMESPACE: default
    STATUS: DEPLOYED
    
    RESOURCES:
    ==> v1/Service
    NAME  CLUSTER-IP EXTERNAL-IP  PORT(S)   AGE
    tomcatjmxsvc  10.254.25.181  <nodes>  80:32733/TCP,35135:30714/TCP  1s
    
    ==> v1beta1/Deployment
    NAME   DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
    tomcatjmx  111   0  1s
    
    
    NOTES:
    1. Get the application URL by running these commands:
      export POD_NAME=$(kubectl get pods -l "app=hello-svc,release=kindly-worm" -o jsonpath="{.items[0].metadata.name}")
      echo "Visit http://127.0.0.1:8080 to use your application"
      kubectl port-forward $POD_NAME 8080:80
    
    [root@k8s-master hello-svc]# helm list
    NAME   REVISIONUPDATED STATUS  CHART  NAMESPACE
    kindly-worm1   Sat Feb 24 14:45:58 2018DEPLOYEDhello-svc-0.1.0default


- 形成一个chart Package

打包形成一个tgz文件，估计是每个项目一个chart,对应一个tgz

    helm package ./


- Chart Package的集中管理和存放

上面我们是从本地的目录结构中的chart去进行部署，如果要集中管理chart,就需要涉及到repository的问题，因为helm repository都是指到外面的地址，接下来我们可以通过minio建立一个企业私有的存放仓库。

Minio提供对象存储服务。它的应用场景被设定在了非结构化的数据的存储之上了。众所周知，非结构化对象诸如图像/音频/视频/log文件/系统备份/镜像文件…等等保存起来管理总是不那么方便，size变化很大，类型很多，再有云端的结合会使得情况更加复杂，minio就是解决此种场景的一个解决方案。Minio号称其能很好的适应非结构化的数据，支持AWS的S3，非结构化的文件从数KB到5TB都能很好的支持。

Minio的使用比较简单，只有两个文件，服务端minio,客户访问端mc,比较简单。

在项目中，我们可以直接找一台虚拟机作为Minio Server,提供服务，当然minio也支持作为Pod部署。


