---
title: minio replication
updated: 2023-05-23 08:11:22Z
created: 2023-05-23 05:19:20Z
latitude: 35.68919750
longitude: 51.38897360
altitude: 0.0000
---

# Deploy minio helm chart

# Prerequisites
* i deployed it inside kubernetes so you would need these (you can do it however you want):
* you must have `helm` and `kubectl` and `minio Client` installed
* you need to make 2 persistent volumes if the cluster doesnt have dynamic volume provisioner
* then you need to make 2 persistent volume claims which names are important and used in values file.
* i made `my-minio1` and `my-minio2` pvc's :         
```
# pvc-1.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-minio1
  labels:
    app: runner
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi      
```
```
# pvc-2.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-minio2
  labels:
    app: runner
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```     

* after pvc's you need to write 2 `values` files for your desired deployment , heres my 2 templates:      
```
# values1.yml
mode: standalone

image:
  registry: docker.iranrepo.ir

auth:
  rootPassword: admin123

statefulset:
    replicaCount: 1
    
persistence:
  existingClaim: my-minio1

service:
  type: NodePort
  nodePorts:
    api: 30100
    console: 30101
```
```
# values2.yml
mode: standalone

image:
  registry: docker.iranrepo.ir

auth:
  rootPassword: admin123

statefulset:
    replicaCount: 1
    
persistence:
  existingClaim: my-minio2

service:
  type: NodePort
  nodePorts:
    api: 30200
    console: 30201
```      

	* user: admin , password: admin123

# deploy 2 minio pods 
* the first step is to install the helm chart with the desired values

* add the repo to helm , and install a certain version , not the latest

	```	
	helm repo add bitnami https://charts.bitnami.com/bitnami

	helm install my-minio1 bitnami/minio --version 12.4.4 --values=values1.yml

	helm install my-minio2 bitnami/minio --version 12.4.4 --values=values2.yml 
	```

* if you run `kubectl get pods` they are both pending , because you need to apply your `pvc`'s :

	```
	kubectl apply -f pvc-1.yml
	kubectl apply -f pvc-2.yml
	```

* 2 minio consoles should be accessible from `http://<your_node_ip>:30101` and `http://<your_node_ip>:30201`. if you cant access nodes directly , use `port-forward`. you only need api ports so just port-forward them:

	```
	kubectl port-forward services/my-minio2 30200:9000
	kubectl  port-forward services/my-minio1 30100:9000
	```

# using mc for replication (Enable Two-Way Server-Side Bucket Replication)

* you need to create 2 `alias`'es , the following command is for when you use `port-forward` :
	
	```
	mc alias set local1 http://127.0.0.1:30100
	<enter user and password>
	mc alias set local2 http://127.0.0.1:30200
	<enter user and password>
	```
	
* make a bucket inside local1 and local2:

	```
	mc mb local1/test
	mc mb local2/test
	```

* 	you have to enable versioning for replication for the both buckets:

	```
	mc version enable local1/test
	mc version enable local2/test
	```
	
* now we enable replication from local1 to local2:
	```
	mc replicate add local1/test \
	--remote-bucket http://admin:admin123@my-minio2:9000/test
	```
	* the main format is :
	```
	mc replicate add ALIAS/BUCKET \
	--remote-bucket http://USER:PASSWORD@MINIO2_HOST:PORT/BUCKET	
	```
	
* list the files inside both buckets :
	```
	mc ls local1/test
	mc ls local2/test
	```
	
* copy a file to local 1: 
	```
	mc cp pvc-1.yml local1/test
	```

* if you check both buckets in both aliases you should see that `pvc-1.yml` file replicated from `local1` to `local2`
* if you do the same thing with `local2` the reverse action will happen as well 
	```
	mc replicate add local2/test \
	--remote-bucket http://admin:admin123@my-minio1:9000/test
	```
	check with
	```
	mc cp pvc-2.yml local2/test
	mc ls local1/test
	```
everything should go as planned!   


# restoring minio persistentVolume

* if you delete your `persistentVolumeClaim` or dont use `persistentVolumeClaim` at all , helm will make one for you but when you uninstall the chart the pvc will be gone and on the next installation you wont have the same data.            
* if your `persistentVolume`'s `recalimPolicy` is set to reclaim , the data is still there.       
* but because it's set to the previous pvc and its gone you cant access the volume just by applying a pvc with the same name.       
* it wouldnt work because the pv's reclaimRef is set to previous pvc `uid        
* you need to delete the uid inside the pv's manifest .  
	
	```
	kubectl get pv <name_of_pv>   
	k edit pv <name_of_pv>
	```
	* if you dont know the name of the pv you can list all pv's and grep the name of your last pvc or your chart's name
* the manifest will be opened find `uid` inside `claimRef` section and delete the line, you can change the name of the pvc that can connect to the pv as well . 
* example:   
```
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 8Gi
  claimRef:
    apiVersion: v1
    kind: PersistentVolumeClaim
    name: my-minio3
    namespace: development
    resourceVersion: "29712660"
    uid: 48f3447a-7f0e-4463-aeb6-d56217f83fa3
```
to 
```
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 8Gi
  claimRef:
    apiVersion: v1
    kind: PersistentVolumeClaim
    name: my-minio4
    namespace: development
```
* now make a pvc with the same name and it will attach to this persistentVolume


as always , just saving the effort...
