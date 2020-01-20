# mongo-db-statefulset
    # Ref https://maruftuhin.com/blog/mongodb-replica-set-on-kubernetes/
# Create the Storage class

    kind: StorageClass
    apiVersion: storage.k8s.io/v1
    metadata:
      name: standard
      annotations:
        storageclass.beta.kubernetes.io/is-default-class: "true"
    provisioner: kubernetes.io/aws-ebs
    parameters:
      zones: us-east-2b
      type: gp2
      fsType: ext4
    reclaimPolicy: Retain
    allowVolumeExpansion: true
    mountOptions:
      - debug
# Check the storage class list
     kubectl get sc

# Deploy the mongodb-service

    apiVersion: v1
    kind: Service
    metadata:
      name: mongodb-service
      labels:
        name: mongo
    spec:
      ports:
      - port: 27017
        targetPort: 27017
      clusterIP: None
      selector:
        role: mongo
# Get the service list

    kubectl get svc

# Deploy the mongodb statefulset

    apiVersion: apps/v1
    kind: StatefulSet
    metadata:
      name: mongod
    spec:
      serviceName: mongodb-service
      replicas: 3
    #  podManagementPolicy: Parallel
      updateStrategy:
        type: RollingUpdate
      selector:
        matchLabels:
          role: mongo
          environment: test
          replicaset: MainRepSet
      template:
        metadata:
          labels:
            role: mongo
            environment: test
            replicaset: MainRepSet
        spec:
          terminationGracePeriodSeconds: 30
          containers:
          - name: mongod-container
            image: mongo
            command:
            - "numactl"
            - "--interleave=all"
            - "mongod"
            - "--bind_ip"
            - "0.0.0.0"
            - "--replSet"
            - "MainRepSet"
            resources:
              requests:
                cpu: 0.2
                memory: 200Mi
            ports:
            - containerPort: 27017
            readinessProbe:
              tcpSocket:
                port: 27017
              initialDelaySeconds: 10
              timeoutSeconds: 1
              periodSeconds: 5
              successThreshold: 1
            livenessProbe:
              tcpSocket:
                port: 27017
              initialDelaySeconds: 10
              timeoutSeconds: 1
              periodSeconds: 2
              failureThreshold: 3
            volumeMounts:
            - name: mongodb-persistent-storage-claim
              mountPath: /data/db
      volumeClaimTemplates:
      - metadata:
          name: mongodb-persistent-storage-claim
          annotations:
        spec:
          accessModes: [ "ReadWriteOnce" ]
          storageClassName: standard
          resources:
            requests:
              storage: 1Gi

# Get the statefulset list
     kubectl get sts

# Add Pods to Cluster
      kubectl get pods 

     kubectl exec -it mongod-0 -c mongod-container bash

     mongo

     rs.initiate({_id: "MainRepSet", version: 1, members: [
           { _id: 0, host : "mongod-0.mongodb-service.default.svc.cluster.local:27017" },
           { _id: 1, host : "mongod-1.mongodb-service.default.svc.cluster.local:27017" },
           { _id: 2, host : "mongod-2.mongodb-service.default.svc.cluster.local:27017" }
     ]});
   
    # Check the status
     rs.status();
    # Insert some data from primary db
      > use test;
      > db.testcoll.insert({a:1});
      > db.testcoll.insert({b:2});
      > db.testcoll.find();
    # Check data from secondary db
      kubectl exec -it mongod-1 -c mongod-container bash
      mongo

      > db.getMongo().setSlaveOk()
      > use test;
      > db.testcoll.find();
    
# Resizing the Persistent volume

      kubectl get pvc

      kubectl edit pvc mongodb-persistent-storage-claim-mongod-0
      
      kubectl get pv
      
      kubectl get pods
      
      # Switch the primary to secondary 
      # Run below query in Primary instance
        cfg = rs.conf()
        cfg.members[0].priority = 0.5
        cfg.members[1].priority = 1
        cfg.members[2].priority = 0.5
        rs.reconfig(cfg)
      
      # Ref https://docs.mongodb.com/manual/tutorial/force-member-to-be-primary/
      
      # Now mongod-1 become as primary then delete the mongod-0 pod
      
      # Delete the pod it will create new pod with same name and added to additional storage to the pod
      
      kubectl delete pod mongod-0 
      
      # Do same things to all pods once all pods get updated storage , update the statefulset volumetemplate 
        it won't allow to update storage so delete the statefulset without deleting the pods.
       
       kubectl delete sts --cascade=false <statefulset>
       
       Ex: kubectl delete sts --cascade=false mongod
       
       Update storage in the statefulset yaml file then apply again
       
       kubectl apply -f mongo-db-statefulset.yaml
      
