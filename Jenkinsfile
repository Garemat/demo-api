node {
  stage('Checkout') {
    checkout scm
  }

  stage('Build') {
    withCredentials([
        usernamePassword(credentialsId: 'docker-credentials',
                         usernameVariable: 'USERNAME',
                         passwordVariable: 'PASSWORD')]) {
      sh 'docker image build -t ${USERNAME}/demo-api:latest .'
    }
  }

  stage('Scan') {
    withCredentials([
        string(credentialsId: 'microscanner-token',
               variable: 'MICROSCANNER_TOKEN'),
        usernamePassword(credentialsId: 'docker-credentials',
                         usernameVariable: 'USERNAME',
                         passwordVariable: 'PASSWORD')]) {
      sh 'wget -q https://github.com/lukebond/microscanner-wrapper/raw/master/scan.sh -O /usr/local/bin/scan.sh && chmod +x /usr/local/bin/scan.sh'
      sh 'MICROSCANNER_OPTIONS=--continue-on-failure /usr/local/bin/scan.sh ${USERNAME}/demo-api:latest'
    }
  }

  stage('Sign') {
    withCredentials([
        usernamePassword(credentialsId: 'docker-credentials',
                         usernameVariable: 'USERNAME',
                         passwordVariable: 'PASSWORD')]) {
      sh '''
				GPG_EMAIL=luke@control-plane.io
				DIGEST=$(docker image inspect ${USERNAME}/demo-api:latest | jq -r .[0].Id)
				echo "${DIGEST}" | gpg2 -u ${GPG_EMAIL} --armor --sign --output=signature.gpg
				RESOURCE_URL=https://docker.io/${USERNAME}/demo-api@${DIGEST}
				GPG_SIGNATURE=$(cat signature.gpg | base64 -w 0)
				GPG_KEY_ID=$(gpg --with-colons --with-fingerprint --force-v4-certs --list-keys ${GPG_EMAIL} | grep '^fpr' | awk -F: '{print $10}')
				cat > image-signing-occurence.json <<EOF
				{
				  "resourceUrl": "${RESOURCE_URL}",
				  "noteName": "projects/image-signing/notes/production",
				  "attestation": {
				    "pgpSignedAttestation": {
				       "signature": "${GPG_SIGNATURE}",
				       "contentType": "application/text",
				       "pgpKeyId": "${GPG_KEY_ID}"
				    }
				  }
				}
				EOF
				curl -i -X POST \
				  "http://127.0.0.1:8081/v1alpha1/projects/image-signing/occurrences" \
				  -d @image-signing-occurence.json
      '''
    }
  }

  stage('Push') {
    withCredentials([
        usernamePassword(credentialsId: 'docker-credentials',
                         usernameVariable: 'USERNAME',
                         passwordVariable: 'PASSWORD')]) {
      sh 'docker login -p "${PASSWORD}" -u "${USERNAME}"'
      sh 'docker image push ${USERNAME}/demo-api:latest'
    }
  }

  stage('Deploy') {
    withCredentials([
        file(credentialsId: 'kube-config',
             variable: 'KUBECONFIG')]) {
      //sh 'gpg2 --output - --verify signature.gpg'
      sh 'kubectl apply -f deployment.yaml'
    }
  }
}
