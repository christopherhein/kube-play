# AWS Cluster Setup

## Create VPC
```
aws ec2 create-vpc --cidr-block 10.4.0.0/16
# export VPC_ID=$(aws ec2 create-vpc --cidr-block 10.4.0.0/16 | jq -r ".Vpc.VpcId")
{
    "Vpc": {
        "VpcId": "vpc-fbf5a09f",
        "State": "pending",
        "CidrBlock": "10.4.0.0/16",
        "DhcpOptionsId": "dopt-a59944c1",
        "Tags": [],
        "InstanceTenancy": "default",
        "IsDefault": false,
        "Ipv6CidrBlockAssociationSet": []
    }
}
```

## Configure VPC
```
export VPC_ID=vpc-fbf5a09f
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-support
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames
```

## Setup Subnets
```
aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.4.1.0/24
# export SUBNET_1_ID=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.4.1.0/24 | jq -r ".Subnet.SubnetId")
{
    "Subnet": {
        "SubnetId": "subnet-962226ce",
        "State": "pending",
        "VpcId": "vpc-fbf5a09f",
        "CidrBlock": "10.4.1.0/24",
        "Ipv6CidrBlockAssociationSet": [],
        "AssignIpv6AddressOnCreation": false,
        "AvailableIpAddressCount": 251,
        "AvailabilityZone": "us-west-1a",
        "DefaultForAz": false,
        "MapPublicIpOnLaunch": false
    }
}
aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.4.2.0/24
# export SUBNET_2_ID=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.4.2.0/24 | jq -r ".Subnet.SubnetId")
{
    "Subnet": {
        "SubnetId": "subnet-892226d1",
        "State": "pending",
        "VpcId": "vpc-fbf5a09f",
        "CidrBlock": "10.4.2.0/24",
        "Ipv6CidrBlockAssociationSet": [],
        "AssignIpv6AddressOnCreation": false,
        "AvailableIpAddressCount": 251,
        "AvailabilityZone": "us-west-1a",
        "DefaultForAz": false,
        "MapPublicIpOnLaunch": false
    }
}
```

## Create IGW
```
aws ec2 create-internet-gateway
# export IGW_ID=$(aws ec2 create-internet-gateway | jq -r ".InternetGateway.InternetGatewayId")
{
    "InternetGateway": {
        "InternetGatewayId": "igw-220bf246",
        "Attachments": [],
        "Tags": []
    }
}
```

## Configure IGW
```
export IGW_ID=igw-220bf246
aws ec2 attach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID
```

## Create Route Table
```
aws ec2 create-route-table --vpc-id $VPC_ID
# export RTB_ID=$(aws ec2 create-route-table --vpc-id $VPC_ID | jq -r ".RouteTable.RouteTableId")
{
    "RouteTable": {
        "RouteTableId": "rtb-f1332b95",
        "VpcId": "vpc-fbf5a09f",
        "Routes": [
            {
                "DestinationCidrBlock": "10.4.0.0/16",
                "GatewayId": "local",
                "State": "active",
                "Origin": "CreateRouteTable"
            }
        ],
        "Associations": [],
        "Tags": [],
        "PropagatingVgws": []
    }
}
```

## Associate Route Table
```
export RTB_ID=rtb-f1332b95
export SUBNET_1_ID=subnet-962226ce
export SUBNET_2_ID=subnet-892226d1
aws ec2 associate-route-table --route-table-id $RTB_ID --subnet-id $SUBNET_1_ID
aws ec2 associate-route-table --route-table-id $RTB_ID --subnet-id $SUBNET_2_ID
```

## Create Security Groups
```
aws ec2 create-security-group --vpc-id $VPC_ID --group-name workers --description k8s-workers
# export SG_1_ID=$(aws ec2 create-security-group --vpc-id $VPC_ID --group-name workers --description k8s-workers | jq -r ".GroupId")
{
    "GroupId": "sg-5eb20b38"
}
aws ec2 create-security-group --vpc-id $VPC_ID --group-name master --description k8s-master
# export SG_2_ID=$(aws ec2 create-security-group --vpc-id $VPC_ID --group-name master --description k8s-master | jq -r ".GroupId")
{
    "GroupId": "sg-a1bf06c7"
}
aws ec2 create-security-group --vpc-id $VPC_ID --group-name bastion --description k8s-bastion
# export SG_3_ID=$(aws ec2 create-security-group --vpc-id $VPC_ID --group-name bastion --description k8s-bastion | jq -r ".GroupId")
{
    "GroupId": "sg-f09f2696"
}
```

```
export SG_1_ID=sg-5eb20b38
export SG_2_ID=sg-a1bf06c7
export SG_3_ID=sg-f09f2696
aws ec2 authorize-security-group-ingress --group-id $SG_1_ID --port 0-65535 --protocol tcp --source-group $SG_2_ID
aws ec2 authorize-security-group-ingress --group-id $SG_2_ID --port 0-65535 --protocol tcp --source-group $SG_1_ID
aws ec2 authorize-security-group-ingress --group-id $SG_1_ID --port 0-65535 --protocol tcp --source-group $SG_3_ID
aws ec2 authorize-security-group-ingress --group-id $SG_2_ID --port 0-65535 --protocol tcp --source-group $SG_3_ID
aws ec2 authorize-security-group-ingress --group-id $SG_3_ID --port 22 --protocol tcp --cidr 0.0.0.0/0
```

# NOT USING
> ## Create EC2 Role
> ```
> cat > ec2-role.json <<EOF
> {
>   "Version": "2012-10-17",
>   "Statement": [
>     {
>       "Action": [
>         "sts:AssumeRole"
>       ],
>       "Effect": "Allow",
>       "Resource": "*"
>     }
>   ]
> }
> EOF
> aws iam create-policy --policy-name kubernetes-worker-role --policy-document file://ec2-role.json
> {
>     "Policy": {
>         "PolicyName": "kubernetes-worker-role",
>         "PolicyId": "ANPAILMYZIK3JYLKGWONU",
>         "Arn": "arn:aws:iam::915347744415:policy/kubernetes-worker-role",
>         "Path": "/",
>         "DefaultVersionId": "v1",
>         "AttachmentCount": 0,
>         "IsAttachable": true,
>         "CreateDate": "2017-09-06T21:16:38.480Z",
>         "UpdateDate": "2017-09-06T21:16:38.480Z"
>     }
> }
>
> cat > ec2-sts-role.json <<EOF
> {
>   "Version": "2012-10-17",
>   "Statement": [
>     {
>       "Sid": "",
>       "Effect": "Allow",
>       "Principal": {
>         "Service": "ec2.amazonaws.com"
>       },
>       "Action": "sts:AssumeRole"
>     },
>     {
>       "Sid": "",
>       "Effect": "Allow",
>       "Principal": {
>         "AWS": "arn:aws:iam::123456789012:role/kubernetes-worker-role"
>       },
>       "Action": "sts:AssumeRole"
>     }
>   ]
> }
> EOF
> aws iam create-role --role-name kubernetes-worker-trust-role --assume-role-policy-document file://ec2-sts-role.json
> ```

## Create Instances
Refer to https://kubernetes.io/docs/admin/cluster-large/#size-of-master-and-master-components
for sizing
```
export MASTER_SIZE=m3.medium
export WORKER_SIZE=m3.large
export BASTION_SIZE=m3.medium
export KEY_NAME=k8sthw
export IMAGE_ID=ami-969ab1f6
aws ec2 run-instances --image-id $IMAGE_ID --count 3 --instance-type $MASTER_SIZE --key-name $KEY_NAME --security-group-ids $SG_2_ID --subnet-id $SUBNET_1_ID --associate-public-ip-address
aws ec2 run-instances --image-id $IMAGE_ID --count 3 --instance-type $WORKER_SIZE --key-name $KEY_NAME --security-group-ids $SG_1_ID --subnet-id $SUBNET_2_ID --associate-public-ip-address
aws ec2 run-instances --image-id $IMAGE_ID --count 1 --instance-type $BASTION_SIZE --key-name $KEY_NAME --security-group-ids $SG_3_ID --subnet-id $SUBNET_1_ID --associate-public-ip-address
```

## Create PKI Infra

```
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
```

```
cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "San Francisco",
      "O": "Kubernetes",
      "OU": "CA",
      "ST": "California"
    }
  ]
}
EOF
```

```
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

```
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "San Francisco",
      "O": "system:masters",
      "OU": "K8s The Hard Way",
      "ST": "California"
    }
  ]
}
EOF
```

```
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin
```

```
for instance in worker-1 worker-2 worker-3; do
cat > ${instance}-csr.json <<EOF
{
  "CN": "system:node:${instance}",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "San Francisco",
      "O": "system:nodes",
      "OU": "K8s The Hard Way",
      "ST": "California"
    }
  ]
}
EOF

EXTERNAL_IP=$(aws ec2 describe-instances --filter "Name=tag:Name,Values=${instance}" | jq -r ".Reservations[0].Instances[0].NetworkInterfaces[0].Association.PublicIp")
INTERNAL_IP=$(aws ec2 describe-instances --filter "Name=tag:Name,Values=${instance}" | jq -r ".Reservations[0].Instances[0].NetworkInterfaces[0].PrivateIpAddress")

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=${instance},${EXTERNAL_IP},${INTERNAL_IP} \
  -profile=kubernetes \
  ${instance}-csr.json | cfssljson -bare ${instance}
done
```

```
cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "San Francisco",
      "O": "system:node-proxier",
      "OU": "K8s The Hard Way",
      "ST": "California"
    }
  ]
}
EOF
```

```
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy
```

```
export KUBERNETES_PUBLIC_ADDRESS=$(aws ec2 describe-instances --filter "Name=tag:Name,Values=master-1,master-2,master-3" | jq -r ".Reservations[0].Instances[] | .NetworkInterfaces[] | .Association.PublicIp" | paste -sd "," -)

cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "San Francisco",
      "O": "Kubernetes",
      "OU": "K8s The Hard Way",
      "ST": "California"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=10.32.0.1,10.240.0.10,10.240.0.11,10.240.0.12,${KUBERNETES_PUBLIC_ADDRESS},127.0.0.1,kubernetes.default \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes
```

```
for instance in worker-1 worker-2 worker-3; do
  DNS=$(aws ec2 describe-instances --filter "Name=tag:Name,Values=${instance}" | jq ".Reservations[0].Instances[0].NetworkInterfaces[0].Association.PublicDnsName" -r)
  scp -i ~/.ssh/$KEY_NAME.pem ca.pem ${instance}-key.pem ${instance}.pem ubuntu@$DNS:~/
done
```
