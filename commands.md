- Set Alias
    ```bash
    $ alias k=kubectl
    ```
- Explain Resources for manually additions in yaml file
    ```bash
    $ kubectl explain deploy # deployment
    $ kubectl explain deploy.spec # go one level inside
    $ kubectl explain deploy.spec --recursive # object details
    ```
- Create resources
    ```bash
    $ kubectl create deployment nginx --image=nginx --dry-run -o yaml > nginx-deploy.yaml # (deployment) add options in template
    $ kubectl run nginx --image=nginx --restart=Never  # (pod)
    $ kubectl run busybox --image=busybox --restart=Never --dry-run -o yaml --command -- env # run command in pod
    $ kubectl run nginx --image=nginx --restart=Never --env=v1=val1 --dry-run -o yaml # create pod with env
    $ kubectl run busybox --image=busybox --restart=Never --dry-run --labels=app=v1 -o yaml # create label in POD
    $ kubectl create job busybox --image=busybox -- /bin/sh -c 'echo hello;sleep 30;echo world' # (job) this job will execute the command
      # you can edit the job file, add completions (run new once the previous is finished successfully) or parallelism (run all pods in parallel) under spec. 
    $ kubectl create cronjob busybox --image=busybox --schedule="*/1 * * * *" -- /bin/sh -c 'date; echo Hello from the Kubernetes cluster'
    $ kubectl run busybox --rm -it --image=busybox --restart=Never -- /bin/sh # create temporary pod and go into that
    # Create a Service named redis-service of type ClusterIP to expose pod redis on port 6379 (This will automatically use the pod's labels as selectors)
    $ kubectl expose pod redis --port=6379 --name redis-service --dry-run -o yaml 
    $ kubectl create service clusterip redis --tcp=6379:6379 --dry-run -o yaml # (This will not use the pods labels as selectors, instead it will assume selectors as app=redis. You cannot pass in selectors as an option. So it does not work very well if your pod has a different label set. So generate the file and modify the selectors before creating the service)
    #####################
    #Create a Service named nginx of type NodePort to expose pod nginx's port 80 on port 30080 on the nodes:
    $ kubectl expose pod nginx --port=80 --name nginx-service --dry-run -o yaml

    # (This will automatically use the pod's labels as selectors, but you cannot specify the node port. You have to generate a definition file and then add the node port in manually before creating the service with the pod.)
    #####################
    ```

- Updates
    ```bash
    $ kubectl set image deploy nginx nginx=nginx:1.9.1 # (deployment image update)
    ```
- Rollout
    ```bash
    $ kubectl rollout status deploy nginx
    $ kubectl rollout history deploy nginx
    $ kubectl rollout undo deploy nginx 
    $ kubectl rollout pause deploy nginx
    $ kubectl rollout resume deploy nginx
    ```
- Scale
    ```bash
    $ kubectl scale deploy nginx --replicas=5
    ```
- Autoscale
    ```bash
    $ kubectl autoscale deploy nginx --min=5 --max=10 --cpu-percent=80 # Autoscale with the number of pods between 5 and 10, target CPU utilization at 80%
    ```
- Delete
    ```bash
    $ kubectl delete hpa nginx #horizontal pod autoscaler
    ```
- Jobs
    ```bash
    $ kubectl get jobs -w # wait untill the job is completed
    ```
- Multicontainers
    ```bash
    $ kubectl exec -it busybox -c busybox2 -- /bin/sh # go into a specific container
    ```
- ConfigMap
    ```bash
    $ kubectl create configmap config --from-literal=foo=lala --from-literal=foo2=lolo
    # Create a new configmap named my-config with specified keys instead of file basenames on disk
    $ kubectl create configmap my-config --from-file=key1=/path/to/bar/file1.txt --from-file=key2=/path/to/bar/file2.txt
    $ kubectl create cm configmap3 --from-env-file=config.env # create configMap from .env file
    
    #######################################
    #load env into pod from ConfigMap
    #######################################
    spec:
      containers:
      - image: nginx
        envFrom: # different than previous one, that was 'env'
        - configMapRef: # different from the previous one, was 'configMapKeyRef'
            name: anotherone # the name of the config map
            
   ########################################
   # load/mount configMap as a volume in pod
   ########################################
   spec:
      volumes: # add a volumes list
      - name: myvolume # just a name, you'll reference this in the pods
        configMap:
          name: cmvolume # name of your configmap
      containers:
      - image: nginx
        volumeMounts: # your volume mounts are listed here
        - name: myvolume # the name that you specified in pod.spec.volumes.name
          mountPath: /etc/lala # the path inside your container
   
   ####################################

    ```
- Security Context
    ```bash
    # run as POD as a user
    spec:
      securityContext: # insert this line
        runAsUser: 101 # UID for the user
      containers:
    ###################################
    # capabilities on the container
    ###################################
    spec:
      containers:
      - image: nginx
        securityContext: # insert this line
          capabilities: # and this
            add: ["NET_ADMIN", "SYS_TIME"] # this as well
    
    ####################################
    # mount secret as a volume to the POD
    ####################################
    spec:
      volumes: # specify the volumes
      - name: foo # this name will be used for reference inside the container
        secret: # we want a secret
          secretName: mysecret2 # name of the secret - this must already exist on pod creation
      containers:
      - image: nginx
        volumeMounts: # our volume mounts
        - name: foo # name on pod.spec.volumes
          mountPath: /etc/foo #our mount path
    ####################################
    # mount variable from secret to POD env
    ####################################
    spec:
      containers:
      - image: nginx
        imagePullPolicy: IfNotPresent
        name: nginx
        resources: {}
        env: # our env variables
        - name: USERNAME # asked name
          valueFrom:
            secretKeyRef: # secret reference
              name: mysecret2 # our secret's name
              key: username # the key of the data in the 
              
    ####################################
    # use serviceaccount in the pod, ServiceAccount is same like the roles in AWS
    ####################################
    spec:
      serviceAccountName: myuser # we use pod.spec.serviceAccountName
      containers:
 
    ```

- Bash Commands
    ```bash
    $ echo -e "var1=val1\n# this is a comment\n\nvar2=val2\n#anothercomment" > config.env # create a config file
    $ wget -O- IP:PORT # for example clusterIP of POD is 10.99.234.79 and the port on which this POD is exposed is 80. 
    $ kubectl exec -it nginx -- env # display all env variables set in the nginx pod
    $ echo YWRtaW4K | base64 -d # decode value
    
    
    ```

- Readiness/Liveness probing

    ```yaml
    # Both readiness and liveness can be tested by 1) ececuting command 2) http test 3) tcp test 
    
    spec:      
      containers:
        livenessProbe: # check whether the pod is still live or not
          initialDelaySeconds: 5 # probing starts after 5 seconds
          periodSeconds: 10 # prob after every 10 seconds
          failureThreshold: 8
          exec: # check by executing command
            command: 
            - ls # any command
            - /app/is_live 
            
    spec:
      containers:
        ports:
          - containerPort: 80 # Note: Readiness probes runs on the container during its whole lifecycle. Since nginx exposes 80, containerPort: 80 is not required for readiness to work.
        readinessProbe: # declare the readiness probe
          httpGet: # add this line
            path: / # prob on this path and port 80
            port: 80 # 
            
    spec:
      containers:
        readinessProbe:
          tcpSocket: # tcp test on port 3306
            port: 3306 
    ```
- Volumes
    ```yaml
    #########################################################          
    # create volume empty directory and mount to container on path /etc/foo 
    #########################################################
    spec:
      containers:
      - args:
        - /bin/sh
        - -c
        - sleep 3600
        image: busybox
        imagePullPolicy: IfNotPresent
        name: busybox
        resources: {}
        volumeMounts: #
        - name: myvolume # emptyDir is mounted here
          mountPath: /etc/foo #      
      volumes: #
        - name: myvolume # use this name above
          emptyDir: {} #
   ######################################################### 
    # create volume on host (single node) and mount to the container path /opt, volume will be deleted as we delete the POD
   ######################################################### 
    spec:
      containers:
        - image: alpine
          name: alpine
          command: ["/bin/sh","-c"]
          args: ["shuf -i 0-100 -n 1 >> /opt/number.out;"]
          volumeMounts:
            - mountPath: /opt
              name: data-volume
      volumes: # this volume is created on the host at path /data. whatever is written onto the file number.out will be stored on this path
      - name: data-volume
        hostPath:
          path: /data
          type: Directory
   #########################################################       
    # create persistent volume (cluster wide) hosted on /etc/foo path - Administrator create storage pool (PV) and all applications share it
   #########################################################  
    kind: PersistentVolume
    apiVersion: v1
    metadata:
      name: myvolume
    spec:
      storageClassName: normal
      capacity:
        storage: 10Gi
      accessModes:
        - ReadWriteOnce
        - ReadWriteMany
      hostPath:
        path: /etc/foo
   #########################################################      
    # create persistent volume claim with specified storage - user create PVC. 
   ######################################################### 
    kind: PersistentVolumeClaim
    apiVersion: v1
    metadata:
      name: mypvc
    spec:
      storageClassName: normal
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 4Gi #claim the storage
    #########################################################          
    # mount the persistent volume claim to the container - One PVC can only be attached with single PV. 1-1 relation
    # when PVC is created it looks for the volumn, if there is no volume available, it will remain in the pending state
    # Once the volume gets available/create, PVC automatically attached to the PV
    #########################################################
    spec:
      containers:
      - args:
        - /bin/sh
        - -c
        - sleep 3600
        image: busybox
        imagePullPolicy: IfNotPresent
        name: busybox
        resources: {}
        volumeMounts: #
        - name: myvolume #
          mountPath: /etc/foo # node local directory
      
      volumes: #
      - name: myvolume #
        persistentVolumeClaim: #
          claimName: mypvc #
     #########################################################
    ```
- Sidecar Pattern
    ```bash
    apiVersion: v1
    kind: Pod
    metadata:
      name: pod-with-sidecar
    spec:
      # Create a volume called 'shared-logs' that the
      # app and sidecar share.
      volumes:
      - name: shared-logs 
        emptyDir: {}

      # In the sidecar pattern, there is a main application
      # container and a sidecar container.
      containers:

      # Main application container
      - name: app-container
        # Simple application: write the current date
        # to the log file every five seconds
        image: alpine # alpine is a simple Linux OS image
        command: ["/bin/sh"]
        args: ["-c", "while true; do date >> /var/log/app.txt; sleep 5;done"]

        # Mount the pod's shared log file into the app 
        # container. The app writes logs here.
        volumeMounts:
        - name: shared-logs
          mountPath: /var/log

      # Sidecar container
      - name: sidecar-container
        # Simple sidecar: display log files using nginx.
        # In reality, this sidecar would be a custom image
        # that uploads logs to a third-party or storage service.
        image: nginx:1.7.9
        ports:
          - containerPort: 80

        # Mount the pod's shared log file into the sidecar
        # container. In this case, nginx will serve the files
        # in this directory.
        volumeMounts:
        - name: shared-logs
          mountPath: /usr/share/nginx/html # nginx-specific mount path
    ```
- Adapter Pattern
    ```bash
    apiVersion: v1
    kind: Pod
    metadata:
      name: pod-with-adapter
    spec:
      # Create a volume called 'shared-logs' that the
      # app and adapter share.
      volumes:
      - name: shared-logs 
        emptyDir: {}

      containers:

      # Main application container
      - name: app-container
        # This application writes system usage information (`top`) to a status 
        # file every five seconds.
        image: alpine
        command: ["/bin/sh"]
        args: ["-c", "while true; do date > /var/log/top.txt && top -n 1 -b >> /var/log/top.txt; sleep 5;done"]

        # Mount the pod's shared log file into the app 
        # container. The app writes logs here.
        volumeMounts:
        - name: shared-logs
          mountPath: /var/log

      # Adapter container
      - name: adapter-container
        # This sidecar container takes the output format of the application
        # (the current date and system usage information), simplifies
        # and reformats it for the monitoring service to come and collect.

        # In this example, our monitoring service requires status files
        # to have the date, then memory usage, then CPU percentage each 
        # on a new line.

        # Our adapter container will inspect the contents of the app's top file,
        # reformat it, and write the correctly formatted output to the status file.
        image: alpine
        command: ["/bin/sh"]

        # A long command doing a simple thing: read the `top.txt` file that the
        # application wrote to and adapt it to fit the status file format.
        # Get the date from the first line, write to `status.txt` output file.
        # Get the first memory usage number, write to `status.txt`.
        # Get the first CPU usage percentage, write to `status.txt`.

        args: ["-c", "while true; do (cat /var/log/top.txt | head -1 > /var/log/status.txt) && (cat /var/log/top.txt | head -2 | tail -1 | grep
     -o -E '\\d+\\w' | head -1 >> /var/log/status.txt) && (cat /var/log/top.txt | head -3 | tail -1 | grep
    -o -E '\\d+%' | head -1 >> /var/log/status.txt); sleep 5; done"]


        # Mount the pod's shared log file into the adapter
        # container.
        volumeMounts:
        - name: shared-logs
          mountPath: /var/log
    ```
- Services & Networking
    ```bash
    $ kubectl run nginx --image=nginx --restart=Never --port=80 --expose # create pod and expose it on POD 80.
    $ kubectl expose deploy foo --port=6262 --target-port=8080 # expose deployment ports, --port is service port and --target-port port is container port
    ```
- Network Policy
    ```bash
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: test-network-policy
      namespace: default
    spec:
      podSelector: 
        matchLabels:
          role: db
      policyTypes:
      - Ingress
      - Egress
      ingress: #allow traffic from 172.17.0.0/16
      - from:
        - ipBlock:
            cidr: 172.17.0.0/16
            except:
            - 172.17.1.0/24
        - namespaceSelector:
            matchLabels:
              project: myproject
        - podSelector: # allow trafic from this pod (frontend) on port 6379
            matchLabels:
              role: frontend
        ports: 
        - protocol: TCP
          port: 6379
      egress:
      - to:
        - ipBlock:
            cidr: 10.0.0.0/24
        ports:
        - protocol: TCP
          port: 5978
    ```
    
 
