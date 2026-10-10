Ephemeral volumes
- Configmap as volume
- emptyDir -> at pod level -> sidecar containers
- hostPath -> underlying host level -> daemonset(a pod replica runs on each and every node) -> collect host level logs and metrics

volumes:
- name: <some-name>
  source-of-volume
  
  volumeMounts:
  - name:
	mountPath:
	
Static provisioning
Dynamic provisioning

EBS and EFS

Static provisioning
===================
PV -> physical representation of the underlying disk, it is equivalant k8 resource to the disk. we can work with pv to manage the disk
PVC -> claiming the disk from PV
SC -> part of dynamic provisioning. it creates disk and pv on behalf of us..

1. install EBS drivers
2. attach EBS IAM role to the instances
3. create disk
4. create pv and pvc
5. mount volume in pod


1. install EBS drivers
2. attach EBS IAM role to the instances
3. create storage class
4. create pvc and pod

if EBS volume should be in same az as server

aws eks update-kubeconfig --region us-east-1 --name roboshop

kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.58"

EBS vs EFS
===========
1. EBS is like HD EFS is like drive following NFS protocol
2. EBS should be in same AZ but EFS can be anywhere in network
3. EBS is faster than EFS
4. EBS is suitable for OS and DB, EFS is suitable to store some files
5. EBS is fixed size, EFS can grow automatically
6. EBS we can select fs type, EFS is set to NFS
7. EBS SG is not required, EFS SG is mandatory

EFS static provisioning
=======
1. Install drivers
2. Add IAM role
3. Create filesystem
4. Make sure filesystem SG allows 2049(NFS port)
5. Create PV, PVC and pod

kubectl kustomize \
    "github.com/kubernetes-sigs/aws-efs-csi-driver/deploy/kubernetes/overlays/stable/ecr/?ref=release-3.0" > private-ecr-driver.yaml