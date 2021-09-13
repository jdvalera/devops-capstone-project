## DESCRIPTION

Create a DevOps infrastructure for an e-commerce application to run on high-availability mode.

### Background of the problem statement:

A popular payment application, EasyPay where users add money to their wallet accounts, faces an issue in its payment success rate. The timeout that occurs with
The connectivity of the database has been the reason for the issue.
While troubleshooting, it is found that the database server has several downtime instances at irregular intervals. This situation compels the company to create their own infrastructure that runs in high-availability mode.
Given that online shopping experiences continue to evolve as per customer expectations, the developers are driven to make their app more reliable, fast, and secure for improving the performance of the current system.

### Implementation requirements:

1. Create the cluster (EC2 instances with load balancer and elastic IP in case of AWS)
2. Automate the provisioning of an EC2 instance using Ansible or Chef Puppet
3. Install Docker and Kubernetes on the cluster
4. Implement the network policies at the database pod to allow ingress traffic from the front-end application pod
5. Create a new user with permissions to create, list, get, update, and delete pods
6. Configure application on the pod
7. Take snapshot of ETCD database
8. Set criteria such that if the memory of CPU goes beyond 50%, environments automatically get scaled up and configured

### The following tools must be used:

1. EC2
2. Kubernetes
3. Docker
4. Ansible or Chef or Puppet

---

## Automate, Provision, and Install Kubernetes in EC2 Instances Using Ansible (1-3)

1. Build the Dockerfile in `/ansible-ubuntu`
2. Go into the `/ansible-ubuntu` directory and run the docker container from the current directory
3. Make sure the configuration and variables are correct for the desired setup
4. Create a `cred.yml` file with AWS credentials (You can use ansible-vault for some extra security)
5. Run `ansible-playbook setup.yml`
6. Wait until the EC2 instances are created and running

## Implement Network Policies and Configure Application On The Pod

1. The yaml files in the folder can all be applied which will setup a frontend, a redis backend, and a PHP page for load testing
2. Most of the yaml files contain services inside them which will set up the network policies
3. A node port will be open which will be used by the AWS Load Balancer for routing traffic into the pods
4. `metrics.yaml` will be used to track cpu usage for horizontal pod scaling
5. ssh into the master instance
6. Use GIT to clone this repository
7. Apply each yaml file using `kubectl apply -f <yaml-file>`

## Autoscaling of Deployment When CPU > 50%

1. Using php-apache deployment to test autoscaling feature
2. The deployment will start with 3 pods and will scale up as the CPU goes over 50% utilization
3. The following command will be used in a separate terminal to increase the cpu usage

```
kubectl run -i --tty load-generator --rm --image=busybox --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"
```

4. After running the load increasing container check the deployments and the hpa metrics using:

```
kubectl get hpa
kubectl get deployment php-apache
```

5. Inspect that the number of pods have increased after the CPU usage goes over 50%

## Taking an ETCD Snapshot

1. Install jq `sudo apt install jq`
2. Install etcd-client using `sudo apt install etcd-client`
3. Use `kubectl get pods -n kube-system` to find the etcd pod
4. Use the following command to get information used to create the snapshot. In my case, my etcd pod is **etcd-ip-172-31-6-73**

```
kubectl get pods etcd-ip-172-31-6-73 -n kube-system \
-o=jsonpath='{.spec.containers[0].command}' | jq
```

Sample Output:

```
[
  "etcd",
  "--advertise-client-urls=https://172.31.6.73:2379",
  "--cert-file=/etc/kubernetes/pki/etcd/server.crt",
  "--client-cert-auth=true",
  "--data-dir=/var/lib/etcd",
  "--initial-advertise-peer-urls=https://172.31.6.73:2380",
  "--initial-cluster=ip-172-31-6-73=https://172.31.6.73:2380",
  "--key-file=/etc/kubernetes/pki/etcd/server.key",
  "--listen-client-urls=https://127.0.0.1:2379,https://172.31.6.73:2379",
  "--listen-metrics-urls=http://127.0.0.1:2381",
  "--listen-peer-urls=https://172.31.6.73:2380",
  "--name=ip-172-31-6-73",
  "--peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt",
  "--peer-client-cert-auth=true",
  "--peer-key-file=/etc/kubernetes/pki/etcd/peer.key",
  "--peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt",
  "--snapshot-count=10000",
  "--trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt"
]
```

4. Use information from the previous command to make a snapshot using the following command:

```
sudo ETCDCTL_API=3 etcdctl snapshot save /tmp/etcdBackup.db \
--endpoints=https://172.31.6.73:2379 \
--cacert=/etc/kubernetes/pki/etcd/ca.crt \
--cert=/etc/kubernetes/pki/etcd/server.crt \
--key=/etc/kubernetes/pki/etcd/server.key
```

5. Verify that the snapshot was created using:

```
ETCDCTL_API=3 \
etcdctl --write-out=table snapshot status /tmp/etcdBackup.db
```

Sample verification output:

```
ubuntu@ip-172-31-6-73:~$ ETCDCTL_API=3 \
> etcdctl --write-out=table snapshot status /tmp/etcdBackup.db
+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| 6be0ec04 |     4484 |       1173 |     3.6 MB |
+----------+----------+------------+------------+
```

## Create a New User With Permissions to Create, List, Get, Update, and Delete Pods

1. Use the following commands to create a new user jean

```
sudo adduser jean --disabled-password
sudo usermod -aG sudo jean
sudo visudo
```

2. Disalow passwords when using sudo for the new user by adding `jean ALL=(ALL) NOPASSWD:ALL` to sudoers.d using visudo
3. Use following commands to allow the new user to ssh into the instance. Get the authorized key information from currunt ubuntu user and copy it into the new user's .ssh/authorized_keys

```
sudo su - jean
mkdir .ssh
chmod 700 .ssh
touch .ssh/authorized_keys
chmod 600 .ssh/authorized_keys
cat >> .ssh/authorized_keys
```

4. Try logging into the instance as the new user with the same .pem file `ssh -i ansible.pem jean@54.215.243.118`
5. Use the following commands to generate certs and keys needed for kubectl access

```
sudo openssl genrsa -out jean.key 2048

sudo openssl req -new -key jean.key \
  -out jean.csr \
  -subj "/CN=jean"

sudo openssl x509 -req -in jean.csr \
  -CA /etc/kubernetes/pki/ca.crt \
  -CAkey /etc/kubernetes/pki/ca.key \
  -CAcreateserial \
  -out jean.crt -days 500

sudo mkdir .certs
sudo mv jean.crt jean.key .certs
sudo chown -R jean: /home/jean/
```

6. Log into the instance with ubuntu user to set the kubectl credentials using:

```
kubectl config set-credentials jean \
>   --client-certificate=/home/jean/.certs/jean.crt \
>   --client-key=/home/jean/.certs/jean.key
```

7. Add a config file for the new user `nano .kube/config`
8. Use the `certificate-authority-data` and `server` found in the ubuntu user's `.kube/config` file. The following file should look similar to:

```
apiVersion: v1
clusters:
  - cluster:
      certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUMvakNDQWVhZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRJeE1Ea3hNakl6TURReU5Gb1hEVE14TURreE1ESXpNRFF5TkZvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBTVhqCjJPZWtVbUxaWWdJdk56akovZTlQNFhBUGNCZlFNM0xvUmVNZXZYZ04yQ05LTUtqTGVNME9ueHpYSEdkTFZQbDMKV01Nak1HTHQyRVlXcDlqd2c4UUNqOUU1NzZ2M2FoaGdFc2dCckJYbUtreE40dzJ0OUZwZFp0RXQyZzliK1ErMwo2Si9VNnBLL0xsR1A3OEMwQTFkVllLOHhrd1BaLzZBdnF6Z2ozY2lmM0FCd2U4R2g0L0cwT2NQcWlYUzBRUjJICkdkSXEzZnh6Zm43SVNLck90TzhOWlhqcVp6TkhHenVybkxWSUtaN3RValdkeXM3SnNEN0YxS1ZsbFQ5Zk5zYUEKUzUrRC8vMGl4SkdCdlBxK2hPU2JNU0xPbFF5V1FzVExRWHpvRFpPQWJWZ0xRelNYbUNxM0ttYldHWk5wSW9ieQpPbjZMZnJmNkY1T1VwUG8wMGtrQ0F3RUFBYU5aTUZjd0RnWURWUjBQQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0hRWURWUjBPQkJZRUZBRVg5eVkvby9TUnNMcVdxckQxbkZ6K1luYjNNQlVHQTFVZEVRUU8KTUF5Q0NtdDFZbVZ5Ym1WMFpYTXdEUVlKS29aSWh2Y05BUUVMQlFBRGdnRUJBRTdiOEppbTFYL2tBTEI1dkl3RApWTVpHV0dKU3ZjMEp2YnNCVnFQUTFsLzdJQ0ZUK1k0SldVQ1FrNmZmVWNIZVFGOXpJcHRramZZNlJBckdNbG1lCnozdDRIcmtLY3dVRWJsZHk1a1ZoUk9pTHc1WFoycG84Q2RCQ1RPQzVhckh4YmlWOU53eUltNnFadFliMXIybUgKcWtIMW4wL1BGT2krZ1c3WHMxK0l3K001VVBhV082YUc4QkFpUU4vNnlXUUE0OXduek0zY0RnVm5WOS9PMzg4QQpGajhadlJBOGJDb2VNMStiT2loaFEyeFcrcitZYzdtanVUYThVMTZPOFgwWXppcDFDUEVvVkxXRjdHeDNqeWUvCnM4SnJHTGdSak9aVHpza2NiMGZUWlE4WXVIZ1Vubi9Hc0c1R3BuLzhmVkVaUGRnTFpISE1JQnZ1bVJSSnBEa1AKR09JPQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
      server: https://172.31.6.73:6443
    name: kubernetes
contexts:
  - context:
      cluster: kubernetes
      user: jean
    name: jean-context
current-context: jean-context
kind: Config
preferences: {}
users:
  - name: jean
    user:
      client-certificate: /home/jean/.certs/jean.crt
      client-key: /home/jean/.certs/jean.key
```

9. Create a role binding yaml file to give the new user permission to use kubectl. The following file gives the user admin access to the cluster.

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jean
subjects:
- kind: User
  name: jean
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

10. Apply the role binding yaml file using the ubuntu user

```
kubectl apply -f kube-permissions.yaml
clusterrolebinding.rbac.authorization.k8s.io/jean created
```

11. Log into the master instance using the new user and verify that `kubectl` commands work

### Resources

- [Horizontal Pod Autoscaler Walkthrough](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)
- [Backup and Restore Kubernetes Etcd on the Same Control Plane Node](https://ystatit.medium.com/backup-and-restore-kubernetes-etcd-on-the-same-control-plane-node-20f4e78803cb)
- [Deploying Multi-Node Kubernetes on AWS Using Ansible Automation](https://medium.com/geekculture/deploying-multi-node-kubernetes-cluster-on-aws-using-ansible-automation-7d8a5eb25c53)
- [Create a guestbook with Redis and PHP](https://cloud.google.com/kubernetes-engine/docs/tutorials/guestbook)
- [Users and RBAC authorizations in Kubernetes](https://www.adaltas.com/en/2019/08/07/users-rbac-kubernetes/)
