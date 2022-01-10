# Data-and-model-isolation-on-EKS
**利用*EKS*和*resource access manager*实现机器学习模型与数据隔离**
## 介绍
在机器学习中，有一种常见的需求：甲方提供数据，乙方提供机器学习模型。甲方要确保数据不被乙方保存副本，而乙方也要保护自己的机器学习模型不被甲方获取。这种场景借助AWS的三个服务可以轻松实现，这三个服务分别是EKS，AWS Resource Access Manager，AWS Organizations。

如下图，先使用AWS Resource Access Manage，将甲方的一个私有子网共享给乙方。并且对取消私有子网的互联网访问权限，在路由表中添加S3 endpoint，并配置权限，这个私有子网中的EC2实例仅可以访问指定指定S3存储桶。并且也在S3存储桶配置，[必须特定的s3 endpoint才能访问这个存储桶](https://aws.amazon.com/cn/premiumsupport/knowledge-center/block-s3-traffic-vpc-ip/)。然后乙方在自己的VPC内创建EKS集群，选择子网时，则选择甲方共享的子网。这样乙方可以通过在甲方共享的子网内创建机器学习推理集群，访问甲方的数据，但数据不能外传。甲方也无法获得乙方的机器学习模型。最后，再使用AWS Organizations SCP，由甲方控制乙方的权限，禁止乙方快照和挂载EBS，防止利用EKS集群复制数据。

![image](arch.png)

下面来实验下具体操作步骤。

## 第一步，使用Resource Access Manage共享子网

**首先，使用甲方账户操作。**

默认VPC的子网是不能被Resource Access Manage共享的，所以需要新创建一个VPC。

进入Resource Access Manage服务页面，选择创建新的共享资源。

![image](ram1.png)

如上图，我们选择指定的子网。

![image](ram2.png)

如上图，我们可以看到，共享出的子网，对方仅能在上面运行EC2实例，并没有其他权限。

![image](ram3.png)

如上图，输入一个要分享的AWS account ID。

![image](rtb.png)

我们现在检查下共享出子网的路由表，确定只有本地路由和S3 endpoint路由。如果没有S3 endpoint路由，请创建S3 endpoint。

## 第二步，配置S3 endpoint和存储桶权限

我们需要确保从这个s3 endpoint只能访问到指定的子网。并且也要确定存储桶只能从这个s3 endpoint来访问。

我们先来配置存储桶策略，下面存储桶策略限定了如果要访问DOC-EXAMPLE-BUCKET存储桶，则必须从名为vpce-1111111的S3 endpoint访问。我们把存储桶和s3 endpoint名字都替换为实际名称。
```
{
  "Id": "VPCe",
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "VPCe",
      "Action": "s3:*",
      "Effect": "Deny",
      "Resource": [
        "arn:aws:s3:::DOC-EXAMPLE-BUCKET",
        "arn:aws:s3:::DOC-EXAMPLE-BUCKET/*"
      ],
      "Condition": {
        "StringNotEquals": {
          "aws:SourceVpce": [
            "vpce-1111111"
          ]
        }
      },
      "Principal": "*"
    }
  ]
}
```
继续来配置S3 endpoint权限，下面的权限表示通过这个s3 endpoint，仅可以访问example-bucket存储桶，将example-bucket替换为实际名称。
```
{
  "Effect": "Allow",
  "Action": [
     "s3:ListBucket",
     "s3:GetObject",
     "s3:PutObject"
  ],
  "Resource": [
     "arn:aws:s3:::example-bucket",
     "arn:aws:s3:::example-bucket/*"
  ]
}	
```

## 第三步，创建EKS集群

**现在切换为乙方账户操作。**

我们可以通过EKS console创建EKS集群，在选择网络的时候，我们可以看到从甲方账户共享来的子网。选择这个子网，创建集群。

![image](eks.png)

由于创建的是一个不能访问互联网的私有集群，所以必须满足以下要求：

* 容器镜像必须位于或者复制到 Amazon Elastic Container Registry (Amazon ECR) 中，或复制到要拉取的 VPC 内部的注册表中。
* 节点需要端点私有访问权限才能注册到集群端点。终端节点公有访问权限是可选的。
* 您可能需要包含在私有集群的 VPC 终端节点中找到的 VPC 终端节点。
* 启动自行管理的节点时，引导参数中必须包含以下文本。此文本绕过 Amazon EKS 自检，不需要从 VPC 内访问 Amazon EKS API。将 <cluster-endpoint> 和 <cluster-certificate-authority> 替换为您的 Amazon EKS 集群中的值。 
```
--apiserver-endpoint <cluster-endpoint> --b64-cluster-ca <cluster-certificate-authority>
```
* 必须从 VPC 内创建 aws-auth ConfigMap。
  
**创建容器映像的本地副本**
  
由于私有集群没有出站 Internet 访问，因此无法从 Docker Hub 等外部源提取容器映像。相反，容器镜像必须本地复制到 Amazon ECR 或者复制到在 VPC 中可访问的备用注册表。容器镜像可以从私有 VPC 外部复制到 Amazon ECR。私有集群使用 Amazon ECR VPC 端点访问 Amazon ECR 存储库。您必须在用于创建本地副本的工作站上安装了 Docker 和 AWS CLI。

创建容器映像的本地副本

1. 创建 Amazon ECR 存储库。有关更多信息，请参阅[创建存储库](https://docs.aws.amazon.com/AmazonECR/latest/userguide/repository-create.html)。
2. 使用 docker pull 从外部注册表中提取容器映像。
3. 通过 docker tag，使用 Amazon ECR 注册表、存储库和可选镜像标签名称组合标记您的镜像。
4. 对注册表进行身份验证。有关更多信息，请参阅[注册表身份验证](https://docs.aws.amazon.com/AmazonECR/latest/userguide/Registries.html#registry_auth)。
5. 使用 docker push [将镜像推送到 Amazon ECR](https://docs.aws.amazon.com/AmazonECR/latest/userguide/docker-push-ecr-image.html)。 

以下示例展示使用标签 v1.3.1-linux-amd64 从 Docker Hub 拉取 [amazon/aws-node-termination-handler](https://hub.docker.com/r/amazon/aws-node-termination-handler) 镜像以及在 Amazon ECR 中创建本地副本。

```
aws ecr create-repository --repository-name amazon/aws-node-termination-handler
docker pull amazon/aws-node-termination-handler:v1.3.1-linux-amd64
docker tag amazon/aws-node-termination-handler <111122223333>.dkr.ecr.<region-code>.amazonaws.com/amazon/aws-node-termination-handler:v1.3.1-linux-amd64
aws ecr get-login-password --region <region-code> | docker login --username AWS --password-stdin <111122223333>.dkr.ecr.<region-code>.amazonaws.com
docker push <111122223333>.dkr.ecr.<region-code>.amazonaws.com/amazon/aws-node-termination-handler:v1.3.1-linux-amd64
```
  
**私有集群的 VPC 终端节点**

可能需要以下 VPC 终端节点，请注意，这些终端节点只有甲方账户有权限配置。
* com.amazonaws.<region>.ec2
* com.amazonaws.<region>.ecr.api
* com.amazonaws.<region>.ecr.dkr
* com.amazonaws.<region>.s3 – 用于拉取容器镜像
* com.amazonaws.<region>.logs – 适用于 CloudWatch Logs
* com.amazonaws.<region>.sts – 对服务账户使用 AWS Fargate 或 IAM 角色时
* com.amazonaws.<region>.elasticloadbalancing – 使用 Application Load Balancer 时
* com.amazonaws.<region>.autoscaling – 使用 Cluster Autoscaler 时
* com.amazonaws.<region>.appmesh-envoy-management – 使用 App Mesh 时

## 第四步，配置AWS Organizations
  
由甲方作为Organizations里的root账户创建组织，乙方作为成员加入，具体过程可以参考[这里](https://docs.aws.amazon.com/zh_cn/organizations/latest/userguide/orgs_tutorials_basic.html)
  
组织创建完成后，由甲方下发SCP策略，比如下面的策略，允许除创建EC2快照以外的所有动作。具体可以根据情况再添加禁止动作。SCP不呢添加权限，仅可以限制最大权限。
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowsAllActions",
            "Effect": "Allow",
            "Action": "*",
            "Resource": "*"
        },
        {
            "Sid": "DenyEBSsnapshot", 
            "Effect": "Deny",
            "Action": "ec2:CreateSnapshot",
            "Resource": "*"
        }
    ]
}
```
  
至此，整个POC完成。
