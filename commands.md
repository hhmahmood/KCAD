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
    $ kubectl run nginx --image=nginx # Depreciated (deployment). You can use  --replicas=2 --port=80
    $ kubectl run nginx --image=nginx --restart=Never  # (pod)
    $ kubectl run busybox --image=busybox --restart=OnFailure  # Depreciated (job)
    $ kubectl create job busybox --image=busybox -- /bin/sh -c 'echo hello;sleep 30;echo world' # (job) this job will execute the command
      # you can edit the job file, add completions (run new once the previous is finished successfully) or parallelism (run all pods in parallel) under spec. 
    $ kubectl create cronjob busybox --image=busybox --schedule="*/1 * * * *" -- /bin/sh -c 'date; echo Hello from the Kubernetes cluster'
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
    ```
- Bash Commands
    ```bash
    $ echo -e "var1=val1\n# this is a comment\n\nvar2=val2\n#anothercomment" > config.env # create a config file
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

    
 
