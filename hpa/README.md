# Make Kubernetes horizontal pod autoscaling with yaml  file
  The Horizontal Pod Autoscaler (HPA) is a powerful feature in Kubernetes that automatically scales the number of pods in a deployment or replica set based on observed CPU utilization (or other select metrics). This dynamic scaling helps maintain optimal performance and resource utilization, ensuring your applications can handle varying loads efficiently.


# Prerequisites
1.	Kubernetes Cluster: Ensure you have a running Kubernetes cluster.
2.	Metrics Server: Install and configure the Kubernetes Metrics Server.  It is a necessary component for HPA.


----------------------------------------------------------

# Step 1: Install Metrics Server ( if it is already install skip this step)
 
  Install Metrics Server CMD : 

  $ kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
# OR DOWNLOAD 
 $ curl https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# components.yaml file 
          apiVersion: v1
          kind: ServiceAccount
          metadata:
            labels:
              k8s-app: metrics-server
            name: metrics-server
            namespace: kube-system
          ---
          apiVersion: rbac.authorization.k8s.io/v1
          kind: ClusterRole
          metadata:
            labels:
              k8s-app: metrics-server
              rbac.authorization.k8s.io/aggregate-to-admin: "true"
              rbac.authorization.k8s.io/aggregate-to-edit: "true"
              rbac.authorization.k8s.io/aggregate-to-view: "true"
            name: system:aggregated-metrics-reader
          rules:
          - apiGroups:
            - metrics.k8s.io
            resources:
            - pods
            - nodes
            verbs:
            - get
            - list
            - watch
          ---
          apiVersion: rbac.authorization.k8s.io/v1
          kind: ClusterRole
          metadata:
            labels:
              k8s-app: metrics-server
            name: system:metrics-server
          rules:
          - apiGroups:
            - ""
            resources:
            - nodes/metrics
            verbs:
            - get
          - apiGroups:
            - ""
            resources:
            - pods
            - nodes
            verbs:
            - get
            - list
            - watch
          ---
          apiVersion: rbac.authorization.k8s.io/v1
          kind: RoleBinding
          metadata:
            labels:
              k8s-app: metrics-server
            name: metrics-server-auth-reader
            namespace: kube-system
          roleRef:
            apiGroup: rbac.authorization.k8s.io
            kind: Role
            name: extension-apiserver-authentication-reader
          subjects:
          - kind: ServiceAccount
            name: metrics-server
            namespace: kube-system
          ---
          apiVersion: rbac.authorization.k8s.io/v1
          kind: ClusterRoleBinding
          metadata:
            labels:
              k8s-app: metrics-server
            name: metrics-server:system:auth-delegator
          roleRef:
            apiGroup: rbac.authorization.k8s.io
            kind: ClusterRole
            name: system:auth-delegator
          subjects:
          - kind: ServiceAccount
            name: metrics-server
            namespace: kube-system
          ---
          apiVersion: rbac.authorization.k8s.io/v1
          kind: ClusterRoleBinding
          metadata:
            labels:
              k8s-app: metrics-server
            name: system:metrics-server
          roleRef:
            apiGroup: rbac.authorization.k8s.io
            kind: ClusterRole
            name: system:metrics-server
          subjects:
          - kind: ServiceAccount
            name: metrics-server
            namespace: kube-system
          ---
          apiVersion: v1
          kind: Service
          metadata:
            labels:
              k8s-app: metrics-server
            name: metrics-server
            namespace: kube-system
          spec:
            ports:
            - name: https
              port: 443
              protocol: TCP
              targetPort: https
            selector:
              k8s-app: metrics-server
          ---
          apiVersion: apps/v1
          kind: Deployment
          metadata:
            labels:
              k8s-app: metrics-server
            name: metrics-server
            namespace: kube-system
          spec:
            selector:
              matchLabels:
                k8s-app: metrics-server
            strategy:
              rollingUpdate:
                maxUnavailable: 0
            template:
              metadata:
                labels:
                  k8s-app: metrics-server
              spec:
                containers:
                - args:
                  - --cert-dir=/tmp
                  - --secure-port=10250
                  - --kubelet-insecure-tls
                  - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
                  - --kubelet-use-node-status-port
                  - --metric-resolution=15s
                  image: registry.k8s.io/metrics-server/metrics-server:v0.7.2
                  imagePullPolicy: IfNotPresent
                  livenessProbe:
                    failureThreshold: 3
                    httpGet:
                      path: /livez
                      port: https
                      scheme: HTTPS
                    periodSeconds: 10
                  name: metrics-server
                  ports:
                  - containerPort: 10250
                    name: https
                    protocol: TCP
                  readinessProbe:
                    failureThreshold: 3
                    httpGet:
                      path: /readyz
                      port: https
                      scheme: HTTPS
                    initialDelaySeconds: 20
                    periodSeconds: 10
                  resources:
                    requests:
                      cpu: 100m
                      memory: 200Mi
                  securityContext:
                    allowPrivilegeEscalation: false
                    capabilities:
                      drop:
                      - ALL
                    readOnlyRootFilesystem: true
                    runAsNonRoot: true
                    runAsUser: 1000
                    seccompProfile:
                      type: RuntimeDefault
                  volumeMounts:
                  - mountPath: /tmp
                    name: tmp-dir
                nodeSelector:
                  kubernetes.io/os: linux
                priorityClassName: system-cluster-critical
                serviceAccountName: metrics-server
                volumes:
                - emptyDir: {}
                  name: tmp-dir
          ---
          apiVersion: apiregistration.k8s.io/v1
          kind: APIService
          metadata:
            labels:
              k8s-app: metrics-server
            name: v1beta1.metrics.k8s.io
          spec:
            group: metrics.k8s.io
            groupPriorityMinimum: 100
            insecureSkipTLSVerify: true
            service:
              name: metrics-server
              namespace: kube-system
            version: v1beta1
            versionPriority: 100



# Verify the Metrics Server is running:
  $ kubectl apply -f Metrics-server.yaml

  $ kubectl get deployment metrics-server -n kube-system


-------------------------------------------------

# Step 2: Deploy an Application
# Create a Deployment YAML file (nginx-deployment.yaml):

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: nginx-deployment
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: nginx
      template:
        metadata:
          labels:
            app: nginx
        spec:
          containers:
          - name: nginx
            image: nginx
            ports:
            - containerPort: 80
            resources:
              requests:
                cpu: "100m"
                memory: "128Mi"
              limits:
                cpu: "250m"
                memory: "256Mi"

# Apply the Deployment: Verify the Deployment:
 $ kubectl apply -f nginx-deployment.yaml

 $ kubectl get deployment nginx-deployment

 $ kubectl expose deployment nginx-deployment --port=80 --target-port=80

# Step 3: Create the HPA
  
      ........ Version 1
      apiVersion: autoscaling/v1
      kind: HorizontalPodAutoscaler
      metadata:
        name: nginx-hpa
      spec:
        scaleTargetRef:
          apiVersion: apps/v1
          kind: Deployment
          name: nginx-deployment
        minReplicas: 1
        maxReplicas: 10
        targetCPUUtilizationPercentage: 50

       ............
# Autoscaling version 2 is the new and better approach which gives you more accessibility on pods and lets you assign different policy on the HPA.
   
              Version 2

              apiVersion: autoscaling/v2
              kind: HorizontalPodAutoscaler
              metadata:
                name: nginx-hpa
              spec:
                scaleTargetRef:
                  apiVersion: apps/v1
                  kind: Deployment
                  name: nginx-deployment
                minReplicas: 1
                maxReplicas: 10
                metrics:
                - type: Resource
                  resource:
                    name: cpu
                    target:
                      type: Utilization
                      averageUtilization: 50
                ................
                  apiVersion: autoscaling/v2
              kind: HorizontalPodAutoscaler
              metadata:
                name: nginx-hpa
              spec:
                scaleTargetRef:
                  apiVersion: apps/v1
                  kind: Deployment
                  name: nginx-deployment
                minReplicas: 1
                maxReplicas: 10
                metrics:
                - type: Resource
                  resource:
                    name: memory
                    target:
                      type: AverageValue
                      averageValue: 500Mi



                ..............

                  apiVersion: autoscaling/v2
              kind: HorizontalPodAutoscaler
              metadata:
                name: nginx-hpa
              spec:
                scaleTargetRef:
                  apiVersion: apps/v1
                  kind: Deployment
                  name: nginx-deployment
                minReplicas: 1
                maxReplicas: 10
                metrics:
                - type: Resource
                  resource:
                    name: cpu
                    target:
                      type: Utilization
                      averageUtilization: 50
                - type: Pods
                  pods:
                    metric:
                      name: packets-per-second
                    target:
                      type: AverageValue
                      averageValue: 1k
                - type: Object
                  object:
                    metric:
                      name: requests-per-second
                    describedObject:
                      apiVersion: networking.k8s.io/v1
                      kind: Ingress
                      name: main-route
                    target:
                      type: Value
                      value: 10k

                
              <!-- #   type: Object
              # object:
              #   metric:
              #     name: http_requests
              #     selector: {matchLabels: {verb: GET}}



              #  type: External
              #   external:
              #     metric:
              #       name: queue_messages_ready
              #       selector:
              #         matchLabels:
              #           queue: "worker_tasks"
              #     target:
              #       type: AverageValue
              #       averageValue: 30 -->



# Apply / verify HPA
 $ kubectl apply -f hpa.yaml

 $ kubectl get  hpa.yaml

        
    
#  But the most important feature is behavior V2

On the autoscaling/v2 you can still see min and max replicas keys and they have same behavior. But the most important feature is behavior. Behavior is divided into two scale down and scale up section which lets you define policies for both up and down scale.


# create a hpa-behavior.yaml file 

          apiVersion: autoscaling/v2
          kind: HorizontalPodAutoscaler
          metadata:
            name: nginx-hpahpa
          spec:
            scaleTargetRef:           # Refers to the target workload to scale (a Deployment named  nginx-deployment ).
              apiVersion: apps/v1
              kind: Deployment
              name: nginx-deployment
            minReplicas: 1
            maxReplicas: 4
            behavior:                   # behavior: Defines how scaling should occur:
              scaleDown:

              # Ensures the HPA waits 300 seconds (5 minutes) before scaling down after detecting conditions.
                stabilizationWindowSeconds: 300       
                policies:
                  - type: Percent                # Scale down by up to 90% of the pods every 15 seconds.
                    value: 90
                    periodSeconds: 15
              scaleUp:
                stabilizationWindowSeconds: 0
                policies:
                  - type: Percent
                    value: 50
                selectPolicy: Min
              
  
  
  # selectPolicy: Min under scaleUp
      This ensures the scaling policy chooses the smallest increase in pod replicas when multiple policies could apply.





  