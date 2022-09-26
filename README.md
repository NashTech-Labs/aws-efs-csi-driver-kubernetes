## Setting up Amazon EFS CSI in EKS cluster for dynamic provisioning of EFS

## Create an IAM policy and role

1. Create an IAM policy that allows the CSI driver's service account to make calls to AWS APIs on your behalf.

    a) Download the IAM policy document from GitHub. 

        curl -o iam-policy-example.json https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/master/docs/iam-policy-example.json
        
    b) Create the policy using aws cli as below:
    
       aws iam create-policy --policy-name AmazonEKS_EFS_CSI_Driver_Policy --policy-document file://iam-policy-example.json
    
2. Create an IAM role and attach the IAM policy to it. Annotate the Kubernetes service account with the IAM role ARN and the IAM role with the Kubernetes service account name.

    a) Determine your cluster's OIDC provider URL. Replace my-cluster with your cluster name.
    
        aws eks describe-cluster --name my-cluster --query "cluster.identity.oidc.issuer" --output text
        
    The example output is as follows:
    https://oidc.eks.region-code.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE

    b) Create the IAM role, granting the Kubernetes service account the AssumeRoleWithWebIdentity action.
        i. Copy the following contents to a file named trust-policy.json. Replace 111122223333 with your account ID. Replace EXAMPLED539D4633E53DE1B71EXAMPLE and region-code with the values returned in the previous step:
        
        	{					
				"Version": "2012-10-17",
				"Statement": [
					{
					"Effect": "Allow",
					"Principal": {
						"Federated": "arn:aws:iam::111122223333:oidc-provider/oidc.eks.region-code.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE"
					},
					"Action": "sts:AssumeRoleWithWebIdentity",
					"Condition": {
							"StringEquals": {
						"oidc.eks.region-code.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE:sub": "system:serviceaccount:kube-system:efs-csi-controller-sa"
						}
					}
					}
				]
			}	
    ii. Create the role. You can change AmazonEKS_EFS_CSI_DriverRole to a different name, but if you do, make sure to change it in later steps too.
    
        aws iam create-role --role-name AmazonEKS_EFS_CSI_DriverRole --assume-role-policy-document file://"trust-policy.json"
        
    c) Attach the IAM policy to the role. Replace 111122223333 with your account ID. before running the following command.
    
        aws iam attach-role-policy --policy-arn arn:aws:iam::111122223333:policy/AmazonEKS_EFS_CSI_Driver_Policy --role-name AmazonEKS_EFS_CSI_DriverRole

3. Update the service account for controller in helm charts for
aws-efs-csi-dr.

        name: efs-csi-controller-sa
	    annotations: 
		 eks.amazonaws.com/role-arn: arn:aws:iam::111122223333:role/AmazonEKS_EFS_CSI_DriverRole_Nexus

4. Deploy the helm chart:

        helm install aws-efs-csi-driver aws-efs-csi-driver/ -f aws-efs-csi-driver/values.yaml --namespace kube-system
        
5. Storage class "efs-sc" will be created. Now create the persistent volume claim (PVC) and the pod which consumes PV. Example pod:

        ---
        apiVersion: v1
        kind: PersistentVolumeClaim
        metadata:
            name: efs-claim
        spec:
            accessModes:
                - ReadWriteMany
            storageClassName: efs-sc
            resources:
                requests:
                    storage: 5Gi
        ---
        apiVersion: v1
        kind: Pod
        metadata:
            name: efs-app
        spec:
            containers:
                - name: app
                  image: centos
                  command: ["/bin/sh"]
                  args: ["-c", "while true; do echo $(date -u) >> /data/out; sleep 5; done"]
                  volumeMounts:
                    - name: persistent-storage
                      mountPath: /data
            volumes:
                - name: persistent-storage
                  persistentVolumeClaim:
                        claimName: efs-claim
						
## References:

1) https://aws.amazon.com/blogs/containers/introducing-efs-csi-dynamic-provisioning/
2) https://docs.aws.amazon.com/eks/latest/userguide/efs-csi.html#efs-create-iam-resources