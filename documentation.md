```
$ aws configure
AWS Access Key ID [****************PL5O]: 
AWS Secret Access Key [****************BeAu]: 
Default region name [eu-west-2]: us-west-2
Default output format [json]: 
```
##### Create VPC
```
$ VPC_ID=$(aws ec2 create-vpc \
--cidr-block 172.31.0.0/16 \
--output text --query 'Vpc.VpcId'
)
```
##### Tag the VPC 
```
NAME=k8s-cluster-from-ground-up

aws ec2 create-tags \
  --resources ${VPC_ID} \
  --tags Key=Name,Value=${NAME}

```

##### Enable DNS Support and host name
```
aws ec2 modify-vpc-attribute \
--vpc-id ${VPC_ID} \
--enable-dns-support '{"Value": true}'
```
```
aws ec2 modify-vpc-attribute \
--vpc-id ${VPC_ID} \
--enable-dns-hostnames '{"Value": true}'
```

##### set default region and DHCP options
```
$ AWS_REGION=us-west-2

$ DHCP_OPTION_SET_ID=$(aws ec2 create-dhcp-options \
  --dhcp-configuration \
    "Key=domain-name,Values=$AWS_REGION.compute.internal" \
    "Key=domain-name-servers,Values=AmazonProvidedDNS" \
  --output text --query 'DhcpOptions.DhcpOptionsId')
  ```

##### Tag it
```
aws ec2 create-tags \
  --resources ${DHCP_OPTION_SET_ID} \
  --tags Key=Name,Value=${NAME}
```

##### Associate it with our VPC

```
aws ec2 associate-dhcp-options \
  --dhcp-options-id ${DHCP_OPTION_SET_ID} \
  --vpc-id ${VPC_ID}

```


##### Create the Subnet:
```
SUBNET_ID=$(aws ec2 create-subnet \
  --vpc-id ${VPC_ID} \
  --cidr-block 172.31.0.0/24 \
  --output text --query 'Subnet.SubnetId')
  ```
  ##### Tag it
  ```
aws ec2 create-tags \
  --resources ${SUBNET_ID} \
  --tags Key=Name,Value=${NAME}

```

##### Create the Internet Gateway, tag it and attach it to the VPC:
```
INTERNET_GATEWAY_ID=$(aws ec2 create-internet-gateway \
  --output text --query 'InternetGateway.InternetGatewayId')
```
```
aws ec2 create-tags \
  --resources ${INTERNET_GATEWAY_ID} \
  --tags Key=Name,Value=${NAME}
```
```
aws ec2 attach-internet-gateway \
  --internet-gateway-id ${INTERNET_GATEWAY_ID} \
  --vpc-id ${VPC_ID}
```
##### Create route tables, associate the route table to subnet, and create a route to allow external traffic to the Internet through the Internet Gateway
```
ROUTE_TABLE_ID=$(aws ec2 create-route-table \
  --vpc-id ${VPC_ID} \
  --output text --query 'RouteTable.RouteTableId')


aws ec2 create-tags \
  --resources ${ROUTE_TABLE_ID} \
  --tags Key=Name,Value=${NAME}


aws ec2 associate-route-table \
  --route-table-id ${ROUTE_TABLE_ID} \
  --subnet-id ${SUBNET_ID}

aws ec2 create-route \
  --route-table-id ${ROUTE_TABLE_ID} \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id ${INTERNET_GATEWAY_ID}
  ```

##### Configure security groups
##### Create the security group and store its ID in a variable
```
SECURITY_GROUP_ID=$(aws ec2 create-security-group \
  --group-name ${NAME} \
  --description "Kubernetes cluster security group" \
  --vpc-id ${VPC_ID} \
  --output text --query 'GroupId')

##### Create the NAME tag for the security group
aws ec2 create-tags \
  --resources ${SECURITY_GROUP_ID} \
  --tags Key=Name,Value=${NAME}

##### Create Inbound traffic for all communication within the subnet to connect on ports used by the master node(s)
aws ec2 authorize-security-group-ingress \
    --group-id ${SECURITY_GROUP_ID} \
    --ip-permissions IpProtocol=tcp,FromPort=2379,ToPort=2380,IpRanges='[{CidrIp=172.31.0.0/24}]'

##### Create Inbound traffic for all communication within the subnet to connect on ports used by the worker nodes
aws ec2 authorize-security-group-ingress \
    --group-id ${SECURITY_GROUP_ID} \
    --ip-permissions IpProtocol=tcp,FromPort=30000,ToPort=32767,IpRanges='[{CidrIp=172.31.0.0/24}]'

##### Create inbound traffic to allow connections to the Kubernetes API Server listening on port 6443
aws ec2 authorize-security-group-ingress \
  --group-id ${SECURITY_GROUP_ID} \
  --protocol tcp \
  --port 6443 \
  --cidr 0.0.0.0/0

##### Create Inbound traffic for SSH from anywhere (Do not do this in production. Limit access ONLY to IPs or CIDR that MUST connect)
aws ec2 authorize-security-group-ingress \
  --group-id ${SECURITY_GROUP_ID} \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0

##### Create ICMP ingress for all types
aws ec2 authorize-security-group-ingress \
  --group-id ${SECURITY_GROUP_ID} \
  --protocol icmp \
  --port -1 \
  --cidr 0.0.0.0/0
  ```

##### Create a network Load balancer,
```
LOAD_BALANCER_ARN=$(aws elbv2 create-load-balancer \
--name ${NAME} \
--subnets ${SUBNET_ID} \
--scheme internet-facing \
--type network \
--output text --query 'LoadBalancers[].LoadBalancerArn')
```
##### Create Target group
```
TARGET_GROUP_ARN=$(aws elbv2 create-target-group \
  --name ${NAME} \
  --protocol TCP \
  --port 6443 \
  --vpc-id ${VPC_ID} \
  --target-type ip \
  --output text --query 'TargetGroups[].TargetGroupArn')
  ```

##### Register targets: (Just like above, no real targets. You will just put the IP addresses so that, when the nodes become available, they will be used as targets.)
```
aws elbv2 register-targets \
  --target-group-arn ${TARGET_GROUP_ARN} \
  --targets Id=172.31.0.1{0,1,2}
```
##### Create listener for LB to listen and forward to target group
```
aws elbv2 create-listener \
--load-balancer-arn ${LOAD_BALANCER_ARN} \
--protocol TCP \
--port 6443 \
--default-actions Type=forward,TargetGroupArn=${TARGET_GROUP_ARN} \
--output text --query 'Listeners[].ListenerArn'
```

##### Get the Kubernetes Public address which is the loadbalancer DNSName
```
KUBERNETES_PUBLIC_ADDRESS=$(aws elbv2 describe-load-balancers \
--load-balancer-arns ${LOAD_BALANCER_ARN} \
--output text --query 'LoadBalancers[].DNSName')
```
##### address
```
$ echo $KUBERNETES_PUBLIC_ADDRESS
k8s-cluster-from-ground-up-10eba7a64017c329.elb.us-west-2.amazonaws.com
```

##### Get an image to create EC2 instances:
```
IMAGE_ID=$(aws ec2 describe-images --owners 099720109477 \
  --filters \
  'Name=root-device-type,Values=ebs' \
  'Name=architecture,Values=x86_64' \
  'Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-xenial-16.04-amd64-server-*' \
  | jq -r '.Images|sort_by(.Name)[-1]|.ImageId')
  ```

  ```
  $ echo $IMAGE_ID
ami-0688ba7eeeeefe3cd
```
##### Create SSH Pair
```
mkdir -p ssh
```
```
aws ec2 create-key-pair \
  --key-name ${NAME} \
  --output text --query 'KeyMaterial' \
  > ssh/${NAME}.id_rsa
chmod 600 ssh/${NAME}.id_rsa
```


##### Create 3 Master nodes: Note – Using t2.micro instead of t2.small as t2.micro is covered by AWS free tier
```
for i in 0 1 2; do
  instance_id=$(aws ec2 run-instances \
    --associate-public-ip-address \
    --image-id ${IMAGE_ID} \
    --count 1 \
    --key-name ${NAME} \
    --security-group-ids ${SECURITY_GROUP_ID} \
    --instance-type t2.micro \
    --private-ip-address 172.31.0.1${i} \
    --user-data "name=master-${i}" \
    --subnet-id ${SUBNET_ID} \
    --output text --query 'Instances[].InstanceId')
  aws ec2 modify-instance-attribute \
    --instance-id ${instance_id} \
    --no-source-dest-check
  aws ec2 create-tags \
    --resources ${instance_id} \
    --tags "Key=Name,Value=${NAME}-master-${i}"
done
```

##### Create 3 worker nodes

```
for i in 0 1 2; do
  instance_id=$(aws ec2 run-instances \
    --associate-public-ip-address \
    --image-id ${IMAGE_ID} \
    --count 1 \
    --key-name ${NAME} \
    --security-group-ids ${SECURITY_GROUP_ID} \
    --instance-type t2.micro \
    --private-ip-address 172.31.0.2${i} \
    --user-data "name=worker-${i}|pod-cidr=172.20.${i}.0/24" \
    --subnet-id ${SUBNET_ID} \
    --output text --query 'Instances[].InstanceId')
  aws ec2 modify-instance-attribute \
    --instance-id ${instance_id} \
    --no-source-dest-check
  aws ec2 create-tags \
    --resources ${instance_id} \
    --tags "Key=Name,Value=${NAME}-worker-${i}"
done
```


##### Self-Signed Root Certificate Authority (CA)

Here, you will provision a CA that will be used to sign additional TLS certificates.

Create a directory and cd into it:
```
mkdir ca-authority && cd ca-authority
```
##### Generate the CA configuration file, Root Certificate, and Private key:
```
{

cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF

cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "UK",
      "L": "England",
      "O": "Kubernetes",
      "OU": "DAREY.IO DEVOPS",
      "ST": "London"
    }
  ]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca

}
```
```
2022/12/26 19:56:58 [INFO] generating a new CA key and certificate from CSR
2022/12/26 19:56:58 [INFO] generate received request
2022/12/26 19:56:58 [INFO] received CSR
2022/12/26 19:56:58 [INFO] generating key: rsa-2048
2022/12/26 19:56:58 [INFO] encoded CSR
2022/12/26 19:56:58 [INFO] signed certificate with serial number 645323685689437359180652844255719425720419992167
```



##### The file defines the following:
```
CN – Common name for the authority

algo – the algorithm used for the certificates

size – algorithm size in bits

C – Country

L – Locality (city)

ST – State or province

O – Organization

OU – Organizational Unit
Output:

2021/05/16 20:18:44 [INFO] generating a new CA key and certificate from CSR
2021/05/16 20:18:44 [INFO] generate received request
2021/05/16 20:18:44 [INFO] received CSR
2021/05/16 20:18:44 [INFO] generating key: rsa-2048
2021/05/16 20:18:44 [INFO] encoded CSR
2021/05/16 20:18:44 [INFO] signed certificate with serial number 478642753175858256977534824638605235819766817855
List the directory to see the created files
```
```
ls -ltr

-rw-r--r--  1 dare  dare   232 16 May 20:18 ca-config.json
-rw-r--r--  1 dare  dare   207 16 May 20:18 ca-csr.json
-rw-r--r--  1 dare  dare  1306 16 May 20:18 ca.pem
-rw-------  1 dare  dare  1679 16 May 20:18 ca-key.pem
-rw-r--r--  1 dare  dare  1001 16 May 20:18 ca.csr
```

##### The 3 important files here are:
```
ca.pem – The Root Certificate
ca-key.pem – The Private Key
ca.csr – The Certificate Signing Request
```


##### Generate the Certificate Signing Request (CSR), Private Key and the Certificate for the Kubernetes Master Nodes.
```
{
cat > master-kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
   "hosts": [
   "127.0.0.1",
   "172.31.0.10",
   "172.31.0.11",
   "172.31.0.12",
   "ip-172-31-0-10",
   "ip-172-31-0-11",
   "ip-172-31-0-12",
   "ip-172-31-0-10.${AWS_REGION}.compute.internal",
   "ip-172-31-0-11.${AWS_REGION}.compute.internal",
   "ip-172-31-0-12.${AWS_REGION}.compute.internal",
   "${KUBERNETES_PUBLIC_ADDRESS}",
   "kubernetes",
   "kubernetes.default",
   "kubernetes.default.svc",
   "kubernetes.default.svc.cluster",
   "kubernetes.default.svc.cluster.local"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "UK",
      "L": "England",
      "O": "Kubernetes",
      "OU": "DAREY.IO DEVOPS",
      "ST": "London"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  master-kubernetes-csr.json | cfssljson -bare master-kubernetes
}


```
```
2022/12/26 20:03:01 [INFO] generate received request
2022/12/26 20:03:01 [INFO] received CSR
2022/12/26 20:03:01 [INFO] generating key: rsa-2048
2022/12/26 20:03:02 [INFO] encoded CSR
2022/12/26 20:03:02 [INFO] signed certificate with serial number 158785961958646298790666911215729371451573615371
```
##### kube-scheduler Client Certificate and Private Key
```
{

cat > kube-scheduler-csr.json <<EOF
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "UK",
      "L": "England",
      "O": "system:kube-scheduler",
      "OU": "DAREY.IO DEVOPS",
      "ST": "London"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-scheduler-csr.json | cfssljson -bare kube-scheduler

}

```
```
2022/12/26 20:04:54 [INFO] generate received request
2022/12/26 20:04:54 [INFO] received CSR
2022/12/26 20:04:54 [INFO] generating key: rsa-2048
2022/12/26 20:04:54 [INFO] encoded CSR
2022/12/26 20:04:54 [INFO] signed certificate with serial number 414209491116218536580152337309784018035836152817
2022/12/26 20:04:54 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").

```
##### kube-proxy Client Certificate and Private Key
```
{

cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "UK",
      "L": "England",
      "O": "system:node-proxier",
      "OU": "DAREY.IO DEVOPS",
      "ST": "London"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy

}
```

```
2022/12/26 20:07:51 [INFO] generate received request
2022/12/26 20:07:51 [INFO] received CSR
2022/12/26 20:07:51 [INFO] generating key: rsa-2048
2022/12/26 20:07:51 [INFO] encoded CSR
2022/12/26 20:07:51 [INFO] signed certificate with serial number 728179748060419021411780314933845375586198996264
2022/12/26 20:07:51 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
```
##### kube-controller-manager Client Certificate and Private Key
```
{
cat > kube-controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "UK",
      "L": "England",
      "O": "system:kube-controller-manager",
      "OU": "DAREY.IO DEVOPS",
      "ST": "London"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager

}
```

```
2022/12/26 20:09:00 [INFO] generate received request
2022/12/26 20:09:00 [INFO] received CSR
2022/12/26 20:09:00 [INFO] generating key: rsa-2048
2022/12/26 20:09:01 [INFO] encoded CSR
2022/12/26 20:09:01 [INFO] signed certificate with serial number 237341597231731651577204030023913031072235128484
2022/12/26 20:09:01 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
```
##### kubelet Client Certificate and Private Key


```
for i in 0 1 2; do
  instance="${NAME}-worker-${i}"
  instance_hostname="ip-172-31-0-2${i}"
  cat > ${instance}-csr.json <<EOF
{
  "CN": "system:node:${instance_hostname}",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "UK",
      "L": "England",
      "O": "system:nodes",
      "OU": "DAREY.IO DEVOPS",
      "ST": "London"
    }
  ]
}
EOF

  external_ip=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=${instance}" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')

  internal_ip=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=${instance}" \
    --output text --query 'Reservations[].Instances[].PrivateIpAddress')

  cfssl gencert \
    -ca=ca.pem \
    -ca-key=ca-key.pem \
    -config=ca-config.json \
    -hostname=${instance_hostname},${external_ip},${internal_ip} \
    -profile=kubernetes \
    ${NAME}-worker-${i}-csr.json | cfssljson -bare ${NAME}-worker-${i}
done
```
```
2022/12/26 20:11:26 [INFO] generate received request
2022/12/26 20:11:26 [INFO] received CSR
2022/12/26 20:11:26 [INFO] generating key: rsa-2048
2022/12/26 20:11:26 [INFO] encoded CSR
2022/12/26 20:11:26 [INFO] signed certificate with serial number 229962559559390919398290569377897453561046778168
2022/12/26 20:11:29 [INFO] generate received request
2022/12/26 20:11:29 [INFO] received CSR
2022/12/26 20:11:29 [INFO] generating key: rsa-2048
2022/12/26 20:11:29 [INFO] encoded CSR
2022/12/26 20:11:29 [INFO] signed certificate with serial number 177471336144391966294230044313346966724298116261
2022/12/26 20:11:32 [INFO] generate received request
2022/12/26 20:11:32 [INFO] received CSR
2022/12/26 20:11:32 [INFO] generating key: rsa-2048
2022/12/26 20:11:32 [INFO] encoded CSR
2022/12/26 20:11:32 [INFO] signed certificate with serial number 456393035792099918390845230021895018014510537782

```
##### kubernetes admin user's Client Certificate and Private Key
```
{
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "UK",
      "L": "England",
      "O": "system:masters",
      "OU": "DAREY.IO DEVOPS",
      "ST": "London"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin
}
```
```
2022/12/26 20:13:42 [INFO] generate received request
2022/12/26 20:13:42 [INFO] received CSR
2022/12/26 20:13:42 [INFO] generating key: rsa-2048
2022/12/26 20:13:43 [INFO] encoded CSR
2022/12/26 20:13:43 [INFO] signed certificate with serial number 713695575502621307308091238998277690213306449948
2022/12/26 20:13:43 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").

```
##### Generate certificate and private key for the Token Controller: a part of the Kubernetes Controller Manager kube-controller-manager responsible for generating and signing service account tokens which are used by pods or other resources to establish connectivity to the api-server. 

```

{

cat > service-account-csr.json <<EOF
{
  "CN": "service-accounts",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "UK",
      "L": "England",
      "O": "Kubernetes",
      "OU": "DAREY.IO DEVOPS",
      "ST": "London"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  service-account-csr.json | cfssljson -bare service-account
}
```

```
2022/12/26 20:16:24 [INFO] generate received request
2022/12/26 20:16:24 [INFO] received CSR
2022/12/26 20:16:24 [INFO] generating key: rsa-2048
2022/12/26 20:16:24 [INFO] encoded CSR
2022/12/26 20:16:24 [INFO] signed certificate with serial number 188384477872214889488942556768121661620534811207
2022/12/26 20:16:24 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
```
##### The next thing is to distribute these certificates securely to the nodes using scp utility
##### First the worker nodes

##### Send to each of the nodes with a loop
```
for i in 0 1 2; do
  instance="${NAME}-worker-${i}"
  external_ip=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=${instance}" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')
  scp -i ../ssh/${NAME}.id_rsa \
    ca.pem ${instance}-key.pem ${instance}.pem ubuntu@${external_ip}:~/; \
done
```
```
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '54.202.17.20' (ECDSA) to the list of known hosts.
ca.pem                                                                                                                                                                       100% 1350     7.7KB/s   00:00     
k8s-cluster-from-ground-up-worker-0-key.pem                                                                                                                                  100% 1679     9.5KB/s   00:00     
k8s-cluster-from-ground-up-worker-0.pem                                                                                                                                      100% 1513     9.3KB/s   00:00     
The authenticity of host '54.190.27.171 (54.190.27.171)' can't be established.
ECDSA key fingerprint is SHA256:g/y4nJ3Int9SQqpPT5mZDcGU3XcYKvlFYk8TxCCvRhI.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '54.190.27.171' (ECDSA) to the list of known hosts.
ca.pem                                                                                                                                                                       100% 1350     8.1KB/s   00:00     
k8s-cluster-from-ground-up-worker-1-key.pem                                                                                                                                  100% 1675    10.0KB/s   00:00     
k8s-cluster-from-ground-up-worker-1.pem                                                                                                                                      100% 1513     9.1KB/s   00:00     
The authenticity of host '34.220.174.72 (34.220.174.72)' can't be established.
ECDSA key fingerprint is SHA256:fXrI0dI+5mRfg8KDooEtkpiNmoWxyaA0LpFhe9nep84.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '34.220.174.72' (ECDSA) to the list of known hosts.
ca.pem                                                                                                                                                                       100% 1350     8.4KB/s   00:00     
k8s-cluster-from-ground-up-worker-2-key.pem                                                                                                                                  100% 1675     9.4KB/s   00:00     
k8s-cluster-from-ground-up-worker-2.pem    

```
##### Master or Controller node: – Note that only the api-server related files will be sent over to the master nodes.
```
for i in 0 1 2; do
instance="${NAME}-master-${i}" \
  external_ip=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=${instance}" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')
  scp -i ../ssh/${NAME}.id_rsa \
    ca.pem ca-key.pem service-account-key.pem service-account.pem \
    master-kubernetes.pem master-kubernetes-key.pem ubuntu@${external_ip}:~/;
done
```
```
The authenticity of host '35.88.152.252 (35.88.152.252)' can't be established.
ECDSA key fingerprint is SHA256:ndPwmbc0/ryJ4hlEUZyM3JGeMJOvFhw0fh4cwc1NwTw.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '35.88.152.252' (ECDSA) to the list of known hosts.
ca.pem                                                                                                                                                                       100% 1350     7.9KB/s   00:00    
ca-key.pem                                                                                                                                                                   100% 1679     9.5KB/s   00:00    
service-account-key.pem                                                                                                                                                      100% 1675    10.1KB/s   00:00    
service-account.pem                                                                                                                                                          100% 1440     8.4KB/s   00:00    
master-kubernetes.pem                                                                                                                                                        100% 2004    11.5KB/s   00:00    
master-kubernetes-key.pem                                                                                                                                                    100% 1679     9.3KB/s   00:00    
The authenticity of host '52.12.92.199 (52.12.92.199)' can't be established.
ECDSA key fingerprint is SHA256:9UvC3K8b7ZayzD2OggrreWr+Qp1f8L7rGm6yaQLYILE.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '52.12.92.199' (ECDSA) to the list of known hosts.
ca.pem                                                                                                                                                                       100% 1350     7.6KB/s   00:00    
ca-key.pem                                                                                                                                                                   100% 1679    10.5KB/s   00:00    
service-account-key.pem                                                                                                                                                      100% 1675    10.3KB/s   00:00    
service-account.pem                                                                                                                                                          100% 1440     9.2KB/s   00:00    
master-kubernetes.pem                                                                                                                                                        100% 2004    12.3KB/s   00:00    
master-kubernetes-key.pem                                                                                                                                                    100% 1679    10.6KB/s   00:00    
The authenticity of host '54.203.150.82 (54.203.150.82)' can't be established.
ECDSA key fingerprint is SHA256:VVQjhmbP/U6yng0WvB/Qgy+IMyBktW9Cdhndi3XTaa4.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '54.203.150.82' (ECDSA) to the list of known hosts.
ca.pem                                                                                                                                                                       100% 1350     8.0KB/s   00:00    
ca-key.pem                                                                                                                                                                   100% 1679    10.0KB/s   00:00    
service-account-key.pem                                                                                                                                                      100% 1675    10.4KB/s   00:00    
service-account.pem                                                                                                                                                          100% 1440     9.0KB/s   00:00    
master-kubernetes.pem                                                                                                                                                        100% 2004    12.4KB/s   00:00    
master-kubernetes-key.pem   

```
##### Generate the kubelet kubeconfig file - I will distribute them later
```
for i in 0 1 2; do

instance="${NAME}-worker-${i}"
instance_hostname="ip-172-31-0-2${i}"
```
 # Set the kubernetes cluster in the kubeconfig file
 ```
  kubectl config set-cluster ${NAME} \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://$KUBERNETES_API_SERVER_ADDRESS:6443 \
    --kubeconfig=${instance}.kubeconfig

# Set the cluster credentials in the kubeconfig file

  kubectl config set-credentials system:node:${instance_hostname} \
    --client-certificate=${instance}.pem \
    --client-key=${instance}-key.pem \
    --embed-certs=true \
    --kubeconfig=${instance}.kubeconfig

# Set the context in the kubeconfig file

  kubectl config set-context default \
    --cluster=${NAME} \
    --user=system:node:${instance_hostname} \
    --kubeconfig=${instance}.kubeconfig

  kubectl config use-context default --kubeconfig=${instance}.kubeconfig
done
```
```
Cluster "k8s-cluster-from-ground-up" set.
User "system:node:ip-172-31-0-20" set.
Context "default" created.
Switched to context "default".
Cluster "k8s-cluster-from-ground-up" set.
User "system:node:ip-172-31-0-21" set.
Context "default" created.
Switched to context "default".
Cluster "k8s-cluster-from-ground-up" set.
User "system:node:ip-172-31-0-22" set.
Context "default" created.
Switched to context "default".

```
##### Generate the kube-proxy kubeconfig
```
{
  kubectl config set-cluster ${NAME} \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_API_SERVER_ADDRESS}:6443 \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-credentials system:kube-proxy \
    --client-certificate=kube-proxy.pem \
    --client-key=kube-proxy-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-context default \
    --cluster=${NAME} \
    --user=system:kube-proxy \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
}

```
```
Cluster "k8s-cluster-from-ground-up" set.
User "system:kube-proxy" set.
Context "default" created.
Switched to context "default".

```
##### Generate the Kube-Controller-Manager kubeconfig
Notice that the --server is set to use 127.0.0.1. This is because, this component runs on the API-Server so there is no point routing through the Load Balancer.
```
{
  kubectl config set-cluster ${NAME} \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=kube-controller-manager.pem \
    --client-key=kube-controller-manager-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-context default \
    --cluster=${NAME} \
    --user=system:kube-controller-manager \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
}
```
```
Cluster "k8s-cluster-from-ground-up" set.
User "system:kube-controller-manager" set.
Context "default" created.
Switched to context "default".


```
##### Generating the Kube-Scheduler Kubeconfig
```
{
  kubectl config set-cluster ${NAME} \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-credentials system:kube-scheduler \
    --client-certificate=kube-scheduler.pem \
    --client-key=kube-scheduler-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-context default \
    --cluster=${NAME} \
    --user=system:kube-scheduler \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
}
```
```
Cluster "k8s-cluster-from-ground-up" set.
User "system:kube-scheduler" set.
Context "default" created.
Switched to context "default".

```
##### Finally, generate the kubeconfig file for the admin user
```
{
  kubectl config set-cluster ${NAME} \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_API_SERVER_ADDRESS}:6443 \
    --kubeconfig=admin.kubeconfig

  kubectl config set-credentials admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem \
    --embed-certs=true \
    --kubeconfig=admin.kubeconfig

  kubectl config set-context default \
    --cluster=${NAME} \
    --user=admin \
    --kubeconfig=admin.kubeconfig

  kubectl config use-context default --kubeconfig=admin.kubeconfig
}
```
```

Cluster "k8s-cluster-from-ground-up" set.
User "admin" set.
Context "default" created.
Switched to context "default".

```
##### NEXT, WE NEED TO DISTRIBUTE THE FILES TO THEIR RESPECTIVE SERVERS
##### send kubeproxy and worker-o kubeconfigs to the instance 0
```
for i in 0 ; do
  instance="${NAME}-worker-${i}"
  external_ip=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=${instance}" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')
  scp -i ../ssh/${NAME}.id_rsa \
  kube-proxy.kubeconfig  k8s-cluster-from-ground-up-worker-0.kubeconfig ubuntu@${external_ip}:~/; \
done
```
##### send kubeproxy and worker-i kubeconfigs to the worker nodes
```
for i in 1 ; do
  instance="${NAME}-worker-${i}"
  external_ip=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=${instance}" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')
  scp -i ../ssh/${NAME}.id_rsa \
  kube-proxy.kubeconfig  ${NAME}-worker-${i}.kubeconfig ubuntu@${external_ip}:~/; \
done
```
##### send kubeproxy and worker-2 kubeconfigs to the instance 2
```
for i in 2 ; do
  instance="${NAME}-worker-${i}"
  external_ip=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=${instance}" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')
  scp -i ../ssh/${NAME}.id_rsa \
  kube-proxy.kubeconfig  ${NAME}-worker-${i}.kubeconfig ubuntu@${external_ip}:~/; \
done
```

##### send kube-controller-manager.kubeconfig kube-scheduler.kubeconfig admin.kubeconfig  to 3 master nodes
```
for i in 0 1 2; do
instance="${NAME}-master-${i}" \
  external_ip=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=${instance}" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')
  scp -i ../ssh/${NAME}.id_rsa \
  admin.kubeconfig kube-controller-manager.kubeconfig kube-scheduler.kubeconfig ubuntu@${external_ip}:~/; \
done
```
```
admin.kubeconfig                                                                                                                                                                                    100% 6381    36.2KB/s   00:00     
kube-controller-manager.kubeconfig                                                                                                                                                                  100% 6441    37.8KB/s   00:00     
kube-scheduler.kubeconfig                                                                                                                                                                           100% 6387    40.0KB/s   00:00     
admin.kubeconfig                                                                                                                                                                                    100% 6381    40.0KB/s   00:00     
kube-controller-manager.kubeconfig                                                                                                                                                                  100% 6441    33.6KB/s   00:00     
kube-scheduler.kubeconfig                                                                                                                                                                           100% 6387    38.0KB/s   00:00     
admin.kubeconfig                                                                                                                                                                                    100% 6381    24.2KB/s   00:00     
kube-controller-manager.kubeconfig                                                                                                                                                                  100% 6441    34.9KB/s   00:00     
kube-scheduler.kubeconfig                                                                                                                                                                           100% 6387    30.2KB/s   00:00 

```
##### generate ETCD encryption key
```
ETCD_ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
```

##### Create an encryption-config.yaml file as documented officially by kubernetes
```
cat > encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ETCD_ENCRYPTION_KEY}
      - identity: {}
EOF
```

##### Send the encryption file to the Controller nodes using scp and a for loop
```
for i in 0 1 2; do
instance="${NAME}-master-${i}" \
  external_ip=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=${instance}" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')
  scp -i ../ssh/${NAME}.id_rsa \
  encryption-config.yaml ubuntu@${external_ip}:~/; \
done
```
##### Output was
```
> done
encryption-config.yaml                                                                                                                                               100%  240     1.0KB/s   00:00    
encryption-config.yaml                                                                                                                                               100%  240     0.3KB/s   00:00    
encryption-config.yaml                                                                                                                                               100%  240     1.5KB/s   00:00 
```
##### SSH into the controller server. We must be in ssh folder so that we can reach the private key
##### Master-0 master-1 master-2
```
master_1_ip=$(aws ec2 describe-instances \
--filters "Name=tag:Name,Values=${NAME}-master-0" \
--output text --query 'Reservations[].Instances[].PublicIpAddress')
ssh -i k8s-cluster-from-ground-up.id_rsa ubuntu@${master_1_ip}
```
```
master_2_ip=$(aws ec2 describe-instances \
--filters "Name=tag:Name,Values=${NAME}-master-1" \
--output text --query 'Reservations[].Instances[].PublicIpAddress')
ssh -i k8s-cluster-from-ground-up.id_rsa ubuntu@${master_2_ip}

```
```
master_3_ip=$(aws ec2 describe-instances \
--filters "Name=tag:Name,Values=${NAME}-master-2" \
--output text --query 'Reservations[].Instances[].PublicIpAddress')
ssh -i k8s-cluster-from-ground-up.id_rsa ubuntu@${master_3_ip}
```
systemctl status etcd.service

##### Master Node 1
```
etcd.service - etcd
   Loaded: loaded (/etc/systemd/system/etcd.service; enabled; vendor preset: enabled)
   Active: active (running) since Tue 2022-12-27 01:02:20 UTC; 13s ago
     Docs: https://github.com/coreos
 Main PID: 2783 (etcd)
    Tasks: 7
   Memory: 9.2M
      CPU: 248ms
   CGroup: /system.slice/etcd.service
           └─2783 /usr/local/bin/etcd --name master-0 --trusted-ca-file=/etc/etcd/ca.pem --peer-trusted-ca-file=/etc/etcd/ca.pem --peer-client-cert-auth --client-cert-auth --listen-peer-urls https:

Dec 27 01:02:20 ip-172-31-0-10 etcd[2783]: established a TCP streaming connection with peer ade74a4f39c39f33 (stream Message writer)
Dec 27 01:02:20 ip-172-31-0-10 etcd[2783]: established a TCP streaming connection with peer ade74a4f39c39f33 (stream MsgApp v2 writer)
Dec 27 01:02:20 ip-172-31-0-10 etcd[2783]: established a TCP streaming connection with peer ed33b44c0b153ee3 (stream MsgApp v2 writer)
Dec 27 01:02:20 ip-172-31-0-10 etcd[2783]: established a TCP streaming connection with peer ed33b44c0b153ee3 (stream Message writer)
Dec 27 01:02:20 ip-172-31-0-10 etcd[2783]: published {Name:master-0 ClientURLs:[https://172.31.0.10:2379]} to cluster 503bbccb5bc2d96d
Dec 27 01:02:20 ip-172-31-0-10 systemd[1]: Started etcd.
Dec 27 01:02:20 ip-172-31-0-10 etcd[2783]: ready to serve client requests
Dec 27 01:02:20 ip-172-31-0-10 etcd[2783]: serving client requests on 127.0.0.1:2379
Dec 27 01:02:20 ip-172-31-0-10 etcd[2783]: ready to serve client requests
Dec 27 01:02:20 ip-172-31-0-10 etcd[2783]: serving client requests on 172.31.0.10:2379
```
##### Master Node 2
```
etcd.service - etcd
   Loaded: loaded (/etc/systemd/system/etcd.service; enabled; vendor preset: enabled)
   Active: active (running) since Tue 2022-12-27 01:00:21 UTC; 50s ago
     Docs: https://github.com/coreos
 Main PID: 2727 (etcd)
    Tasks: 7
   Memory: 15.5M
      CPU: 615ms
   CGroup: /system.slice/etcd.service
           └─2727 /usr/local/bin/etcd --name master-1 --trusted-ca-file=/etc/etcd/ca.pem --peer-trusted-ca-file=/etc/etcd/ca.pem --peer-client-cert-auth --client-cert-auth --listen-peer-urls https:

Dec 27 01:00:21 ip-172-31-0-11 etcd[2727]: established a TCP streaming connection with peer 6709c481b5234095 (stream MsgApp v2 writer)
Dec 27 01:00:21 ip-172-31-0-11 etcd[2727]: established a TCP streaming connection with peer 6709c481b5234095 (stream Message writer)
Dec 27 01:00:21 ip-172-31-0-11 etcd[2727]: established a TCP streaming connection with peer ed33b44c0b153ee3 (stream Message writer)
Dec 27 01:00:21 ip-172-31-0-11 etcd[2727]: published {Name:master-1 ClientURLs:[https://172.31.0.11:2379]} to cluster 503bbccb5bc2d96d
Dec 27 01:00:21 ip-172-31-0-11 systemd[1]: Started etcd.
Dec 27 01:00:21 ip-172-31-0-11 etcd[2727]: ready to serve client requests
Dec 27 01:00:21 ip-172-31-0-11 etcd[2727]: serving client requests on 127.0.0.1:2379
Dec 27 01:00:21 ip-172-31-0-11 etcd[2727]: ready to serve client requests
Dec 27 01:00:21 ip-172-31-0-11 etcd[2727]: serving client requests on 172.31.0.11:2379
Dec 27 01:00:21 ip-172-31-0-11 etcd[2727]: established a TCP streaming connection with peer ed33b44c0b153ee3 (stream MsgApp v2 writer)
```
##### Master Node 3
```
● etcd.service - etcd
   Loaded: loaded (/etc/systemd/system/etcd.service; enabled; vendor preset: enabled)
   Active: active (running) since Tue 2022-12-27 00:50:42 UTC; 3min 6s ago
     Docs: https://github.com/coreos
 Main PID: 2879 (etcd)
    Tasks: 8
   Memory: 21.1M
      CPU: 2.059s
   CGroup: /system.slice/etcd.service
           └─2879 /usr/local/bin/etcd --name master-2 --trusted-ca-file=/etc/etcd/ca.pem --peer-trusted-ca-file=/etc/etcd/ca.pem --peer-client-cert-auth --client-cert-auth --listen-peer-urls https:

Dec 27 00:50:42 ip-172-31-0-12 etcd[2879]: established a TCP streaming connection with peer ade74a4f39c39f33 (stream MsgApp v2 writer)
Dec 27 00:50:42 ip-172-31-0-12 etcd[2879]: published {Name:master-2 ClientURLs:[https://172.31.0.12:2379]} to cluster 503bbccb5bc2d96d
Dec 27 00:50:42 ip-172-31-0-12 systemd[1]: Started etcd.
Dec 27 00:50:42 ip-172-31-0-12 etcd[2879]: ready to serve client requests
Dec 27 00:50:42 ip-172-31-0-12 etcd[2879]: serving client requests on 127.0.0.1:2379
Dec 27 00:50:42 ip-172-31-0-12 etcd[2879]: ready to serve client requests
Dec 27 00:50:42 ip-172-31-0-12 etcd[2879]: serving client requests on 172.31.0.12:2379
Dec 27 00:50:42 ip-172-31-0-12 etcd[2879]: ed33b44c0b153ee3 initialized peer connection; fast-forwarding 8 ticks (election ticks 10) with 2 active peer(s)
Dec 27 00:50:45 ip-172-31-0-12 etcd[2879]: updated the cluster version from 3.0 to 3.4
Dec 27 00:50:45 ip-172-31-0-12 etcd[2879]: enabled capabilities for version 3.4
```

##### Next thing was to configure the components for the control plane on the master/controller nodes


##### Create the Kubernetes configuration directory:

```
sudo mkdir -p /etc/kubernetes/config
```
Download the release binaries
```
wget -q --show-progress --https-only --timestamping \
"https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kube-apiserver" \
"https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kube-controller-manager" \
"https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kube-scheduler" \
"https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kubectl"
```

##### output
```
ubuntu@ip-172-31-0-10:~$ wget -q --show-progress --https-only --timestamping \
> "https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kube-apiserver" \
> "https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kube-controller-manager" \
> "https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kube-scheduler" \
> "https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kubectl"
kube-apiserver                                    100%[============================================================================================================>] 116.41M  48.0MB/s    in 2.4s     
kube-controller-manager                           100%[============================================================================================================>] 110.89M  44.2MB/s    in 2.5s     
kube-scheduler                                    100%[============================================================================================================>]  44.92M  46.6MB/s    in 1.0s     
kubectl                                           100%[============================================================================================================>]  44.29M  62.5MB/s    in 0.7

```
##### Next step is to install the downloaded binaries

```
{
chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl
sudo mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/
}
```

##### Next is to configure the API Server
```
{
sudo mkdir -p /var/lib/kubernetes/

sudo mv ca.pem ca-key.pem master-kubernetes-key.pem master-kubernetes.pem \
service-account-key.pem service-account.pem \
encryption-config.yaml /var/lib/kubernetes/
}
```
##### I will use the instance internal IP address to advertise the API Server to members of the cluster. 
```
export INTERNAL_IP=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)
```
###### Output for master node 1
```
ubuntu@ip-172-31-0-10:~$ echo $INTERNAL_IP
172.31.0.10
```







