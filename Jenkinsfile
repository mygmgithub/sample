node {
  def registry_url = "https://index.docker.io/v1/"
  def project = 'dockergm'
  def appName = 'private-lab'
  def feSvcName = "${appName}-frontend"
  def imageTag = "docker.io/${project}/${appName}:sample-${env.BRANCH_NAME}.${env.BUILD_NUMBER}"
 
  checkout scm

  stage 'Build image'
  sh("docker build -t ${imageTag} .")
  
  stage 'Run Go tests'
  sh("docker run ${imageTag} go test")

  withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'dockerhub-dockergm', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD']]) {
	sh "docker login --password=${PASSWORD} --username=${USERNAME} ${registry_url}"
	sh("docker push ${imageTag}")
  }


  stage "Deploy Application"
  switch (env.BRANCH_NAME) {
    // Roll out to staging
    case "staging":
        // Change deployed image in staging to the one we just built
        sh("sed -i.bak 's#docker.io/dockergm/private-lab:${env.BRANCH_NAME}.99#${imageTag}#' ./k8s/${env.BRANCH_NAME}/*.yaml")
        sh("cat ./k8s/${env.BRANCH_NAME}/backend-${env.BRANCH_NAME}-deployment.yaml")
        sh("kubectl --namespace=${env.BRANCH_NAME} apply -f k8s/${env.BRANCH_NAME}/")
        sh("sleep 30")
        sh("kubectl --namespace=${env.BRANCH_NAME} get pods")
        if ( sh (script: "kubectl get pods -l app=gceme --namespace=${env.BRANCH_NAME} -o jsonpath={.items[*].status.containerStatuses[0]} | grep -qi ready:false", returnStatus: true) != 0) {
          sh("echo http://`kubectl --namespace=${env.BRANCH_NAME} get service/gceme-frontend --output=json | jq -r '.spec.externalIPs[0]'`:`kubectl --namespace=${env.BRANCH_NAME} get service/gceme-frontend --output=json | jq -r '.spec.ports[0].port'` > ${feSvcName}")
	}
	else { 
	  error("Deployment failed")
	}  
        break

    // Roll out to production
    case "master":
        // Change deployed image in staging to the one we just built
    	sh("sed -i.bak 's#docker.io/dockergm/private-lab:${env.BRANCH_NAME}.1.0.1#${imageTag}#' ./k8s/production/*.yaml")  
        sh("kubectl --namespace=production apply -f k8s/production/")
        sh("kubectl --namespace=production apply -f k8s/services/")
	sh("sleep 30")
        sh("kubectl --namespace=production get pods")  
	if ( sh (script: "kubectl get pods -l app=gceme --namespace=production -o jsonpath={.items[*].status.containerStatuses[0]} | grep -qi ready:false", returnStatus: true) != 0) {
          sh("echo http://`kubectl --namespace=production get service/gceme-frontend --output=json | jq -r '.spec.externalIPs[0]'`:`kubectl --namespace=production get service/gceme-frontend --output=json | jq -r '.spec.ports[0].port'` > ${feSvcName}")
	}
	else { 
	  error("Deployment failed")
	}  
	break

    // Roll out a dev environment
    default:
        // Create namespace if it doesn't exist
        sh("kubectl get ns ${env.BRANCH_NAME} || kubectl create ns ${env.BRANCH_NAME}")
        // Don't use public load balancing for development branches
        sh("sed -i.bak 's#LoadBalancer#ClusterIP#' ./k8s/services/frontend.yaml")
        sh("sed -i.bak 's#gcr.io/cloud-solutions-images/gceme:1.0.0#${imageTag}#' ./k8s/dev/*.yaml")
        sh("kubectl --namespace=${env.BRANCH_NAME} apply -f k8s/services/")
        sh("kubectl --namespace=${env.BRANCH_NAME} apply -f k8s/dev/")
        echo 'To access your environment run `kubectl proxy`'
        echo "Then access your service via http://localhost:8001/api/v1/proxy/namespaces/${env.BRANCH_NAME}/services/${feSvcName}:80/"
  }
}
