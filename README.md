# mongo-db-statefulset
# Ref https://maruftuhin.com/blog/mongodb-replica-set-on-kubernetes/

# MongoDB will use this key to communicate internal cluster.
    openssl rand -base64 741 > key.txt
    kubectl create secret generic shared-bootstrap-data --from-file=internal-auth-mongodb-keyfile=key.txt

# Deploy the mongodb using below yaml file

    apiVersion: apps/v1
    kind: StatefulSet
    metadata:
      name: mongod
    spec:
      serviceName: mongodb-service
      replicas: 3
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
          containers:
          - name: mongod-container
            image: mongo:3.4
            command:
            - "numactl"
            - "--interleave=all"
            - "mongod"
            - "--bind_ip"
            - "0.0.0.0"
            - "--replSet"
            - "MainRepSet"
            - "--auth"
            - "--clusterAuthMode"
            - "keyFile"
            - "--keyFile"
            - "/etc/secrets-volume/internal-auth-mongodb-keyfile"
            - "--setParameter"
            - "authenticationMechanisms=SCRAM-SHA-1"
            resources:
              requests:
                cpu: 0.2
                memory: 200Mi
            ports:
            - containerPort: 27017
            volumeMounts:
            - name: secrets-volume
              readOnly: true
              mountPath: /etc/secrets-volume
            - name: mongodb-persistent-storage-claim
              mountPath: /data/db
          volumes:
          - name: secrets-volume
            secret:
              secretName: shared-bootstrap-data
              defaultMode: 256
      volumeClaimTemplates:
      - metadata:
          name: mongodb-persistent-storage-claim
          annotations:
            volume.beta.kubernetes.io/storage-class: "gp2"
        spec:
          accessModes: [ "ReadWriteOnce" ]
          resources:
            requests:
              storage: 1Gi



# Resizing the Persistent volume

      kubectl get pvc

      kubectl edit pvc mongodb-persistent-storage-claim-mongod-1
      
      kubectl get pv
      
      kubectl get pods
      
      # Delete the pod it will create new pod with same name and added to additional storage to the pod
      
      kubectl delete pod mongod-1 
      
      # Do same things to all pods once all pods get updated storage , update the statefulset volumetemplate 
        it won't allow to update storage so delete the statefulset without deleting the pods.
       
       kubectl delete sts --cascade=false <statefulset>
       
       Ex: kubectl delete sts --cascade=false mongod
       
       Update storage in the statefulset yaml file then apply again
       
       kubectl apply -f mongo-db-statefulset.yaml
      
