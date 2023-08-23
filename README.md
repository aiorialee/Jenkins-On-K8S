## Jenkins for Kubernetes
**Author: JingLi <lijingakaaioria@gmail.com>**
### Jenkins Deployment
Deploy目录中的配置文件，用于将Jenkins部署于Kubernetes集群之上，它依赖于：
- Kubernetes集群上部署有基于nfs-csi的存储服务，且创建了名称为nfs-csi的StorageClass资源
- Kubernetes集群上部署有Ingress Nginx Controller

### Deploy or Remove
```bash
kubectl apply -f deploy/  # deploy
kubectl delete -f deploy/  # remove
```

### 设定Jenkins能够解析外部域名（Options）

修改coredns的configmap类似如下内容, 以确保Cluster内部能解析Jenkins Service的名称jenkins.jingli.com和jenkins-jnlp.jingli.com，且自动将其Reslove为相应Service的ClusterIP

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
           lameduck 5s
        }
        ready
        rewrite stop {
            name regex (jenkins.*)\.jingli\.com  {1}.jenkins.svc.cluster.local 
            answer (jenkins.*)\.jenkins\.svc\.cluster\.local {1}.jingli.com
        }
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf {
           max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
```

### pipeline example

First: Test for running agent pods
```
// Author: "Jingli <lijingakaaioria@gmail.com>"
// Site: www.jingli.com
pipeline {
    agent {
        kubernetes {
            inheritFrom 'jenkins-slave'
        }
    }
    stages {
        stage('Testing...') {
            steps {
                sh 'java -version'
            }
        }
    }
}
```

Second: Test for maven env
```
pipeline {
    agent {
        kubernetes {
            inheritFrom 'maven-3.8'
        }
    }
    stages {
        stage('Build...') {
            steps {
                container('maven') {
                    sh 'mvn -version'
                }
            }
        }
    }
}
```

Third: Test for docker in docker env
```
pipeline {
    agent {
        kubernetes {
            inheritFrom 'maven-and-docker'
        }
    }
    stages {
        stage('maven version') {
            steps {
                container('maven') {
                    sh 'mvn -version'
                }
            }
        }
        stage('docker info') {
            steps {
                container('docker') {
                    sh 'docker info'
                }
            }
        }
    }
}
```

