# Kubernetes Federation V2 

# Multi cloud Kubernetes Federatation  AWS (KinD) + Google Kubernetes Engine (GKE) on Federation v2

1. apt-get update

2. Install GO 

	add-apt-repository ppa:gophers/archive

	apt-get update

	apt-get -y install golang-1.10-go

3. Add GOLANG path variables to .bashrc 

	vi .bashrc 

	export PATH=$PATH:/usr/lib/go-1.10/bin

	export GOROOT=/usr/lib/go-1.10

	export GOPATH=$HOME/go

	mkdir $HOME/go

	export GOBIN=$HOME/go/bin

	source .profile 

	check environment variables by doing env | grep GO 

4. Clone repo
	mkdir fed

	cd fed 

	git clone https://github.com/kubernetes-sigs/federation-v2.git

	cd $GOPATH

	git clone https://github.com/edurekakubernetes/federation.git

	mv federation/src .

5. Install docker 

	apt-get -y install docker.io

6. Start installing kubebuilder etcd kube-apiserver kube-controller and other federated binaries 

	cd /root/fed/federation-v2
	
	./scripts/download-binaries.sh
	
	You might get an error - Error downloading assets : This is acceptable. For any other error - please make sure that you are in the correct directory and the GO variables are set 

	make sure /root/fed/federation-v2/bin is created with the below content 

	root@ip-172-31-47-227:~/fed/federation-v2# ls -ltra bin
	
	total 516104
	-rwxr-xr-x  1 root root 185114550 Aug 15 20:45 kube-apiserver

	-rwxr-xr-x  1 root root 153774294 Aug 15 20:45 kube-controller-manager

	-rwxr-xr-x  1 root root  55232898 Aug 15 20:45 kubectl

	-rwxr-xr-x  1 root root  33834296 Aug 15 21:12 etcd

	-rwxr-xr-x  1 root root   8024120 Aug 15 21:12 client-gen

	-rwxr-xr-x  1 root root   7696396 Aug 15 21:12 conversion-gen

	-rwxr-xr-x  1 root root   7707745 Aug 15 21:12 deepcopy-gen

	-rwxr-xr-x  1 root root   7681057 Aug 15 21:12 defaulter-gen

	-rwxr-xr-x  1 root root   7828900 Aug 15 21:12 informer-gen

	-rwxr-xr-x  1 root root   7671465 Aug 15 21:12 lister-gen

	-rwxr-xr-x  1 root root  13668253 Aug 15 21:12 openapi-gen

	-rwxr-xr-x  1 root root  10445902 Aug 15 21:12 gen-apidocs

	-rw-r--r--  1 root root   4657856 Sep 11 21:08 vendor.tar.gz

	-rwxr-xr-x  1 root root  14863168 Sep 11 21:09 kubebuilder

	-rwxr-xr-x  1 root root  10250976 Sep 11 21:09 kubebuilder-gen


	Add the newly created bin folder to $PATH in .bashrc
	
	vi ~/.bashrc

	export PATH=$PATH:/usr/lib/go-1.10/bin:/root/fed/federation-v2/bin

7. Install KinD (Kubernetes in Docker) for quick cluster creation 

	cd /root/fed/federation-v2

	 ./scripts/download-e2e-binaries.sh

	You should get "done." as the final statement

	The KinD binary will be stored inside $GOBIN. verify - ls $GOBIN/kind

	root@ip-172-31-47-227:~/fed/federation-v2# ls $GOBIN/kind

	/root/go/bin/kind

8. Create cluster on AWS using kind 

	vi $HOME/.bashrc

	export PATH=$PATH:$GOBIN

	source $HOME/.profile

	cd /root/fed/federation-v2

	CREATE_INSECURE_REGISTRY=y ./scripts/create-clusters.sh ---Create an insecure registry and creating 2 clusters (cluster1 and cluster2)
	
	In case of any issues: Delete cluster using  - DELETE_INSECURE_REGISTRY=y ./scripts/delete-clusters.sh

	export KUBECONFIG=":/root/.kube/kind-config-1:/root/.kube/kind-config-2"

	cd ~

	root@ip-172-31-47-227:~# kubectl config get-contexts

	CURRENT   NAME       CLUSTER   AUTHINFO                  NAMESPACE

	*         cluster1   kind-1    kubernetes-kind-1-admin

        	  cluster2   kind-2    kubernetes-kind-2-admin

	kubectl config view --flatten > /root/.kube/config

	 mv .kube .kube_KIND ---preserve config from installation 

9. Install gcloud SDK on your control plane (Need a GCP account beforehand)

	export CLOUD_SDK_REPO="cloud-sdk-$(lsb_release -c -s)"

	echo "deb http://packages.cloud.google.com/apt $CLOUD_SDK_REPO main" | sudo tee -a /etc/apt/sources.list.d/google-cloud-sdk.list

	curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

	sudo apt-get update && sudo apt-get install google-cloud-sdk

	gcloud init - Enter Credentials and then enter the token in the CLI (Sensitive Data). Then give details of project and region. 

	We are choosing us-east4-b as the default region. It might vary as per your requirement

	export GKE_VERSION=1.11.3-gke.18

	export ZONE=$(gcloud config get-value compute/zone)

	gcloud container clusters create cluster3 --zone $ZONE --cluster-version $GKE_VERSION -----(This action takes sometime)

	unset KUBECONFIG

	gcloud container clusters get-credentials cluster3

	cd .kube

	kubectl config get-contexts

	kubectl config rename-context gke_edureka-devops-kube_us-east4-b_cluster3 cluster3

	kubectl create clusterrolebinding myname-cluster-admin-binding --clusterrole=cluster-admin --user=$(gcloud config get-value core/account) --context cluster

	cd ~/.kube

	cp config ../

	cp config ../.kube_KIND/config_gke

10. Merge all contexts 

	cd
	
	mv .kube .kube_gke

	mv .kube_KIND/ .kube

	export KUBECONFIG=":/root/.kube/kind-config-1:/root/.kube/kind-config-2:/root/.kube/config_gke"

	kubectl config view --flatten > /root/.kube/config

	root@ip-172-31-47-227:~/.kube# kubectl config get-contexts
		CURRENT   NAME       CLUSTER                                       AUTHINFO                                      NAMESPACE
		*         cluster1   kind-1                                        kubernetes-kind-1-admin
      			  cluster2   kind-2                                        kubernetes-kind-2-admin
          		  cluster3   gke_edureka-devops-kube_us-east4-b_cluster3   gke_edureka-devops-kube_us-east4-b_cluster3


11. Start federating 

	cd /root/fed/federation-v2
	
	kubectl config use-context cluster1

	./scripts/deploy-federation-latest.sh cluster2

	./bin/kubefed2 join cluster3 --cluster-context cluster3 --host-cluster-context cluster1 --add-to-registry --v=2

	kubectl -n federation-system describe federatedclusters

	You should be able to see all the 3 federated clusters 

	Below should be the same - 

	    Reason:                ClusterReady

	    Status:                True

    	    Type:                  Ready

12. Lets start example (Cluster1 (AWS) and Cluster3 (GCP)) 

	Enable ClusterRoleBinding and Deployments

		kubefed2 federate enable ClusterRoleBinding

		kubefed2 federate enable Deployment
	
	cd /root/fed/federation-v2/example/sample1

	Replace cluster2 (AWS) with cluster3 (GCP) or add the corresponding entries 

	sed -i 's/cluster2/cluster3/g' *

	cd /root/fed/federation-v2/example/sample1

	vi federatedclusterrolebinding-template.yaml

	Add ---- apiGroup: ""   --- below roles 

	cd /root/fed/federation-v2	

	kubectl apply -f example/sample1/federatednamespace-template.yaml -f example/sample1/federatednamespace-placement.yaml

	kubectl apply -R -f example/sample1

	cd example/sample1/

	kubectl delete -f federateddeployment-template.yaml -f federateddeployment-placement.yaml -f federateddeployment-override.yaml

	kubectl create  -f federateddeployment-template.yaml -f federateddeployment-placement.yaml -f federateddeployment-override.yaml

	In case of any issues go inside /example/sample1 and do - kubectl delete -f . to delete any recources installed

	kubectl config use-context cluster3

	kubectl get pods --all-namespaces  ---You should see test-deployment of nginx container on both cluster1 and cluster3
