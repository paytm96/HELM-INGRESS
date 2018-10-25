# HELM-INGRESS
POC for deploying UI app on Kubernetes using Helm and ingress on HTTPS

Deploying UI application on Kubernetes using Helm on HTTPS using NGINX-INGRESS-CONTROLLER on Local VM

ASSUMPTIONS
	- Docker, Kubernetes and Helm installed on VM
	- Kubernetes cluster up and running on local VM
	- Self signed certificates created (Eg. tls.crt and tls.key)
	
Steps:
	Building UI application using DockerFile on nginx server
		- Create folder to keep UI related components together
			- mkdir ui-app
				- copy web directory of ui app running on nginx in ui-app
				- copy tls.crt and tls.key in ui-app folder
				- create mime.types (required for nginx) with below contents in ui-app folder					
					types {
					  text/html                             html htm shtml;
					  text/css                              css;
					}
					
				- create nginx.conf file and add below contents in this file to enable http and https port with certs
					worker_processes 1;

					events { worker_connections 1024; }

					http {
						include    mime.types;
						sendfile on;
						server {
							root /usr/share/nginx/html/;
							index index.html;
							server_name localhost;
							listen 80;
							listen 443 ssl;
							ssl_certificate /etc/nginx/certs/tls.crt;
							ssl_certificate_key /etc/nginx/certs/tls.key;

						}
					}

				- create Dockerfile with below contents
					#
					# BudgetTracker Dockerfile for UI
					#
					FROM nginx:latest

					USER root

					# Remove the default Nginx configuration file
					RUN rm -v /etc/nginx/nginx.conf

					# Copy a configuration file from the current directory
					ADD nginx.conf /etc/nginx/

					# Create folder for keeping the certificates
					RUN mkdir /etc/nginx/certs
					ADD tls.key /etc/nginx/certs/
					ADD tls.crt /etc/nginx/certs/
					RUN chmod 777 -R /etc/nginx/certs/
					ADD web /usr/share/nginx/html/
					RUN chmod 777 -R /usr/share/nginx
					ADD web /var/www/html/

					# Append "daemon off;" to the beginning of the configuration
					RUN echo "daemon off;" >> /etc/nginx/nginx.conf

					# Expose ports
					EXPOSE 443 80

					# Set the default command to execute
					# when creating a new container
					CMD service nginx start

		- Build docker image of UI app
			- Go to /ui-app directory created in above Steps
			- execute below docker command to build UI app docker image
				- docker build -t ui-helm-poc .
				
		- Host a local docker registry on VM
			- Use below command to host a docker registry 
			- docker run -d -p 5000:5000 --name registry registry:2
			
		- Push the UI app docker image to locally hosted registry
			- First tag the ui app docker image using below command
				- docker image tag ui-helm-poc localhost:5000/ui-helm-poc
			- Now push docker image to local registry using below command
				- docker push localhost:5000/ui-helm-poc
				
	- Create Kubernetes secret using tls.crt and tls.key
		- Use below command to create Kubernetes secret
		- kubectl create secret tls helmpochost.com-tls --key /ui-app/tls.key --cert /ui-app/tls.crt 
	
	- Install nginx-ingress controller in Kubernetes cluster
		- Create folder nginx-ingress-controller
		- Create values.yaml file inside this fole using below contents
			## Ref: https://github.com/kubernetes/ingress/blob/master/controllers/nginx/configuration.md
			##
			controller:
			  name: controller
			  image:
				repository: quay.io/kubernetes-ingress-controller/nginx-ingress-controller
				tag: "0.19.0"
				pullPolicy: IfNotPresent

			  config: {}
			  # Will add custom header to Nginx https://github.com/kubernetes/ingress-nginx/tree/master/docs/examples/customization/custom-headers
			  headers: {}

			  # Required for use with CNI based kubernetes installations (such as ones set up by kubeadm),
			  # since CNI and hostport don't mix yet. Can be deprecated once https://github.com/kubernetes/kubernetes/issues/23920
			  # is merged
			  hostNetwork: false

			  # Optionally change this to ClusterFirstWithHostNet in case you have 'hostNetwork: true'.
			  # By default, while using host network, name resolution uses the host's DNS. If you wish nginx-controller
			  # to keep resolving names inside the k8s network, use ClusterFirstWithHostNet.
			  dnsPolicy: ClusterFirst

			  ## Use host ports 80 and 443
			  daemonset:
				useHostPort: false

				hostPorts:
				  http: 80
				  https: 443

			  ## Required only if defaultBackend.enabled = false
			  ## Must be <namespace>/<service_name>
			  ##
			  defaultBackendService: ""

			  ## Election ID to use for status update
			  ##
			  electionID: ingress-controller-leader

			  ## Name of the ingress class to route through this controller
			  ##
			  ingressClass: nginx

			  # labels to add to the pod container metadata
			  podLabels: {}
			  #  key: value

			  ## Allows customization of the external service
			  ## the ingress will be bound to via DNS
			  publishService:
				enabled: false
				## Allows overriding of the publish service to bind to
				## Must be <namespace>/<service_name>
				##
				pathOverride: ""

			  ## Limit the scope of the controller
			  ##
			  scope:
				enabled: false
				namespace: ""   # defaults to .Release.Namespace

			  ## Additional command line arguments to pass to nginx-ingress-controller
			  ## E.g. to specify the default SSL certificate you can use
			  ## extraArgs:
			  ##   default-ssl-certificate: "<namespace>/<secret_name>"
			  extraArgs:
				default-ssl-certificate: default/helmpochost.com-tls

			  ## Additional environment variables to set
			  extraEnvs: []
			  # extraEnvs:
			  #   - name: FOO
			  #     valueFrom:
			  #       secretKeyRef:
			  #         key: FOO
			  #         name: secret-resource

			  ## DaemonSet or Deployment
			  ##
			  kind: Deployment

			  # The update strategy to apply to the Deployment or DaemonSet
			  ##
			  updateStrategy: {}
			  #  rollingUpdate:
			  #    maxUnavailable: 1
			  #  type: RollingUpdate

			  # minReadySeconds to avoid killing pods before we are ready
			  ##
			  minReadySeconds: 0


			  ## Node tolerations for server scheduling to nodes with taints
			  ## Ref: https://kubernetes.io/docs/concepts/configuration/assign-pod-node/
			  ##
			  tolerations: []
			  #  - key: "key"
			  #    operator: "Equal|Exists"
			  #    value: "value"
			  #    effect: "NoSchedule|PreferNoSchedule|NoExecute(1.6 only)"

			  affinity: {}

			  ## Node labels for controller pod assignment
			  ## Ref: https://kubernetes.io/docs/user-guide/node-selection/
			  ##
			  nodeSelector: {}

			  ## Liveness and readiness probe values
			  ## Ref: https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes
			  ##
			  livenessProbe:
				failureThreshold: 3
				initialDelaySeconds: 10
				periodSeconds: 10
				successThreshold: 1
				timeoutSeconds: 1
				port: 10254
			  readinessProbe:
				failureThreshold: 3
				initialDelaySeconds: 10
				periodSeconds: 10
				successThreshold: 1
				timeoutSeconds: 1
				port: 10254

			  ## Annotations to be added to controller pods
			  ##
			  podAnnotations: {}

			  replicaCount: 1

			  minAvailable: 1

			  resources: {}
			  #  limits:
			  #    cpu: 100m
			  #    memory: 64Mi
			  #  requests:
			  #    cpu: 100m
			  #    memory: 64Mi

			  autoscaling:
				enabled: false
				minReplicas: 1
				maxReplicas: 11
				targetCPUUtilizationPercentage: 50
				targetMemoryUtilizationPercentage: 50

			  ## Override NGINX template
			  customTemplate:
				configMapName: ""
				configMapKey: ""

			  service:
				annotations: {}
				labels: {}
				clusterIP: ""

				## List of IP addresses at which the controller services are available
				## Ref: https://kubernetes.io/docs/user-guide/services/#external-ips
				##
				externalIPs: []

				loadBalancerIP: ""
				loadBalancerSourceRanges: []

				enableHttp: true
				enableHttps: true

				## Set external traffic policy to: "Local" to preserve source IP on
				## providers supporting it
				## Ref: https://kubernetes.io/docs/tutorials/services/source-ip/#source-ip-for-services-with-typeloadbalancer
				externalTrafficPolicy: ""

				healthCheckNodePort: 0

				targetPorts:
				  http: http
				  https: https

				type: NodePort

				# type: NodePort
				# nodePorts:
				#   http: 32080
				#   https: 32443
				nodePorts:
				  http: 32080
				  https: 32443

			  extraContainers: []
			  ## Additional containers to be added to the controller pod.
			  ## See https://github.com/lemonldap-ng-controller/lemonldap-ng-controller as example.
			  #  - name: my-sidecar
			  #    image: nginx:latest
			  #  - name: lemonldap-ng-controller
			  #    image: lemonldapng/lemonldap-ng-controller:0.2.0
			  #    args:
			  #      - /lemonldap-ng-controller
			  #      - --alsologtostderr
			  #      - --configmap=$(POD_NAMESPACE)/lemonldap-ng-configuration
			  #    env:
			  #      - name: POD_NAME
			  #        valueFrom:
			  #          fieldRef:
			  #            fieldPath: metadata.name
			  #      - name: POD_NAMESPACE
			  #        valueFrom:
			  #          fieldRef:
			  #            fieldPath: metadata.namespace
			  #    volumeMounts:
			  #    - name: copy-portal-skins
			  #      mountPath: /srv/var/lib/lemonldap-ng/portal/skins

			  extraVolumeMounts: []
			  ## Additional volumeMounts to the controller main container.
			  #  - name: copy-portal-skins
			  #   mountPath: /var/lib/lemonldap-ng/portal/skins

			  extraVolumes: []
			  ## Additional volumes to the controller pod.
			  #  - name: copy-portal-skins
			  #    emptyDir: {}

			  extraInitContainers: []
			  ## Containers, which are run before the app containers are started.
			  # - name: init-myservice
			  #   image: busybox
			  #   command: ['sh', '-c', 'until nslookup myservice; do echo waiting for myservice; sleep 2; done;']


			  stats:
				enabled: false

				service:
				  annotations: {}
				  clusterIP: ""

				  ## List of IP addresses at which the stats service is available
				  ## Ref: https://kubernetes.io/docs/user-guide/services/#external-ips
				  ##
				  externalIPs: []

				  loadBalancerIP: ""
				  loadBalancerSourceRanges: []
				  servicePort: 18080
				  type: ClusterIP

			  ## If controller.stats.enabled = true and controller.metrics.enabled = true, Prometheus metrics will be exported
			  ##
			  metrics:
				enabled: false

				service:
				  annotations: {}
				  # prometheus.io/scrape: "true"
				  # prometheus.io/port: "10254"

				  clusterIP: ""

				  ## List of IP addresses at which the stats-exporter service is available
				  ## Ref: https://kubernetes.io/docs/user-guide/services/#external-ips
				  ##
				  externalIPs: []

				  loadBalancerIP: ""
				  loadBalancerSourceRanges: []
				  servicePort: 9913
				  type: ClusterIP

			  lifecycle: {}

			  priorityClassName: ""

			## Rollback limit
			##
			revisionHistoryLimit: 10

			## Default 404 backend
			##
			defaultBackend:

			  ## If false, controller.defaultBackendService must be provided
			  ##
			  enabled: true

			  name: default-backend
			  image:
				repository: k8s.gcr.io/defaultbackend
				tag: "1.4"
				pullPolicy: IfNotPresent

			  extraArgs: {}

			  port: 8080

			  ## Node tolerations for server scheduling to nodes with taints
			  ## Ref: https://kubernetes.io/docs/concepts/configuration/assign-pod-node/
			  ##
			  tolerations: []
			  #  - key: "key"
			  #    operator: "Equal|Exists"
			  #    value: "value"
			  #    effect: "NoSchedule|PreferNoSchedule|NoExecute(1.6 only)"

			  affinity: {}

			  # labels to add to the pod container metadata
			  podLabels: {}
			  #  key: value

			  ## Node labels for default backend pod assignment
			  ## Ref: https://kubernetes.io/docs/user-guide/node-selection/
			  ##
			  nodeSelector: {}

			  ## Annotations to be added to default backend pods
			  ##
			  podAnnotations: {}

			  replicaCount: 1

			  minAvailable: 1

			  resources: {}
			  # limits:
			  #   cpu: 10m
			  #   memory: 20Mi
			  # requests:
			  #   cpu: 10m
			  #   memory: 20Mi

			  service:
				annotations: {}
				clusterIP: ""

				## List of IP addresses at which the default backend service is available
				## Ref: https://kubernetes.io/docs/user-guide/services/#external-ips
				##
				externalIPs: []

				loadBalancerIP: ""
				loadBalancerSourceRanges: []
				servicePort: 80
				type: ClusterIP

			  priorityClassName: ""

			## Enable RBAC as per https://github.com/kubernetes/ingress/tree/master/examples/rbac/nginx and https://github.com/kubernetes/ingress/issue
			s/266
			rbac:
			  create: true

			# If true, create & use Pod Security Policy resources
			# https://kubernetes.io/docs/concepts/policy/pod-security-policy/
			podSecurityPolicy:
			  enabled: false

			serviceAccount:
			  create: true
			  name:

			## Optional array of imagePullSecrets containing private registry credentials
			## Ref: https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/
			imagePullSecrets: []
			# - name: secretName

			# TCP service key:value pairs
			# Ref: https://github.com/kubernetes/contrib/tree/master/ingress/controllers/nginx/examples/tcp
			##
			tcp: {}
			#  8080: "default/example-tcp-svc:9000"

			# UDP service key:value pairs
			# Ref: https://github.com/kubernetes/contrib/tree/master/ingress/controllers/nginx/examples/udp
			##
			udp: {}
			#  53: "kube-system/kube-dns:53"
			
		- Deploy Nginx-Ingress-Controller using above values.yaml in Kubernetes ClusterFirst
			- Execute below command to deploy nginx ingress controlelr
				- helm install --name nginx-controller --values=nginx-ingress-controller/values.yaml stable/nginx-ingress

	- Create Helm chart to package the UI app and deploy in kubernetes cluster using ingress
		- Create helm chart for ui app using below command
			- helm create ui-chart
			
		- Replace values.yaml file inside ui-chart folder with below contents
			# Default values for ui-chart.
			# This is a YAML-formatted file.
			# Declare variables to be passed into your templates.

			replicaCount: 1

			image:
			  repository: localhost:5000/ui-helm-poc
			  tag: latest
			  pullPolicy: Always

			nameOverride: ""
			fullnameOverride: ""

			service:
			  type: NodePort
			  nodePort: 30051
			  port: 443
			  httpNodePort: 30055
			  httpPort: 80

			ingress:
			  enabled: true
			  annotations:
				 kubernetes.io/ingress.class: nginx
				 kubernetes.io/tls-acme: "true"
			  path: /
			  hosts:
				- helmpochost.com
			  tls:
				- secretName: helmpochost.com-tls
				  hosts:
					- helmpochost.com

			resources: {}
			  # We usually recommend not to specify default resources and to leave this as a conscious
			  # choice for the user. This also increases chances charts run on environments with little
			  # resources, such as Minikube. If you do want to specify resources, uncomment the following
			  # lines, adjust them as necessary, and remove the curly braces after 'resources:'.
			  # limits:
			  #  cpu: 100m
			  #  memory: 128Mi
			  # requests:
			  #  cpu: 100m
			  #  memory: 128Mi

			nodeSelector: {}

			tolerations: []

			affinity: {}

		- Replace deployments.yaml inside templates folder with below contents
			apiVersion: apps/v1beta2
			kind: Deployment
			metadata:
			  name: {{ include "ui-chart.fullname" . }}
			  labels:
				app.kubernetes.io/name: {{ include "ui-chart.name" . }}
				helm.sh/chart: {{ include "ui-chart.chart" . }}
				app.kubernetes.io/instance: {{ .Release.Name }}
				app.kubernetes.io/managed-by: {{ .Release.Service }}
			spec:
			  replicas: {{ .Values.replicaCount }}
			  selector:
				matchLabels:
				  app.kubernetes.io/name: {{ include "ui-chart.name" . }}
				  app.kubernetes.io/instance: {{ .Release.Name }}
			  template:
				metadata:
				  labels:
					app.kubernetes.io/name: {{ include "ui-chart.name" . }}
					app.kubernetes.io/instance: {{ .Release.Name }}
				spec:
				  containers:
					- name: {{ .Chart.Name }}
					  image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
					  imagePullPolicy: {{ .Values.image.pullPolicy }}
					  ports:
						- name: https
						  containerPort: 443
						  protocol: TCP
						- name: http
						  containerPort: 80
						  protocol: TCP

					  resources:
			{{ toYaml .Values.resources | indent 12 }}
				{{- with .Values.nodeSelector }}
				  nodeSelector:
			{{ toYaml . | indent 8 }}
				{{- end }}
				{{- with .Values.affinity }}
				  affinity:
			{{ toYaml . | indent 8 }}
				{{- end }}
				{{- with .Values.tolerations }}
				  tolerations:
			{{ toYaml . | indent 8 }}
				{{- end }}

		- Replace ingress.yaml inside templates folder with below contents
			apiVersion: extensions/v1beta1
			kind: Ingress
			metadata:
			  name: ingresspoc
			  labels:
				app.kubernetes.io/name: {{ include "ui-chart.name" . }}
				helm.sh/chart: {{ include "ui-chart.chart" . }}
				app.kubernetes.io/instance: {{ .Release.Name }}
				app.kubernetes.io/managed-by: {{ .Release.Service }}
			  annotations:
				kubernetes.io/ingress.class: "nginx"
				nginx.ingress.kubernetes.io/secure-backends: "true"
			spec:
			  tls:
				- hosts:
				  - helmpochost.com
				  secretName: helmpochost.com-tls
			  rules:
				- host: helmpochost.com
				  http:
					paths:
					  - path: /
						backend:
						  serviceName: {{ include "ui-chart.fullname" . }}
						  servicePort: 443

		- Replace service.yaml inside templates folder with below contents
			apiVersion: v1
			kind: Service
			metadata:
			  name: {{ include "ui-chart.fullname" . }}
			  labels:
				app.kubernetes.io/name: {{ include "ui-chart.name" . }}
				helm.sh/chart: {{ include "ui-chart.chart" . }}
				app.kubernetes.io/instance: {{ .Release.Name }}
				app.kubernetes.io/managed-by: {{ .Release.Service }}
			spec:
			  type: {{ .Values.service.type }}
			  ports:
				- port: {{ .Values.service.port }}
				  targetPort: https
				  nodePort: {{ .Values.service.nodePort }}
				  protocol: TCP
				  name: https
				- port: {{ .Values.service.httpPort }}
				  targetPort: http
				  nodePort: {{ .Values.service.httpNodePort }}
				  protocol: TCP
				  name: http
			  selector:
				app.kubernetes.io/name: {{ include "ui-chart.name" . }}
				app.kubernetes.io/instance: {{ .Release.Name }}

	- Deploying UI app on Kubernetes using helm
		- Use below command to deploy UI app helm chart
			- helm install --name helmchart-poc ui-chart
			
	Your UI Application is not deployed in Kubernetes with HTTP and HTTPS supporting
		- Edit /etc/hosts file on your local VM
		- add VM IP with helmpochost.com as below	
			9.12.23.14 helmpochost.com
			
		Execute curl commands to verify if you can access your UI app using https and http
		- Curl for http	
			- curl http://helmpochost.com:<NodePort_port mentioned in ui pod>
		
		- Curl for https
			- curl https://helmpochost.com:<NodePort_port mentioned in nginx ingress controller pod> --cacert tls.crt
		
