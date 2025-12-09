// Clean Jenkinsfile - uses ansible-ssh-key; does not require ansible-become-pass
pipeline {
  agent { label 'ansible' }

  environment {
    INVENTORY = "inventory.ini"
    PLAYBOOK  = "full-cluster-automation.yml"
  }

  options {
    timeout(time: 60, unit: 'MINUTES')
  }

  stages {
    stage('Checkout') {
      steps {
        echo "Checking out repository..."
        checkout scm
        sh 'ls -la'
      }
    }

    stage('Validate Environment') {
      steps {
        echo "Checking Ansible & Python versions on agent..."
        sh '''
          which ansible || { echo "ERROR: ansible not found on agent"; exit 2; }
          ansible --version
          python3 --version || python --version
        '''
      }
    }

    stage('Prepare SSH (known_hosts)') {
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: 'ansible-ssh-key',
                                          keyFileVariable: 'SSH_KEY_FILE',
                                          usernameVariable: 'ANSIBLE_USER')]) {
          sh '''
            echo "Setting up SSH known_hosts..."
            chmod 600 "$SSH_KEY_FILE"
            mkdir -p ~/.ssh
            touch ~/.ssh/known_hosts

            # robust host extraction: ansible_host=IP and bare IP lines
            grep -Eo 'ansible_host=[0-9]+(\\.[0-9]+){3}' ${INVENTORY} 2>/dev/null | sed 's/ansible_host=//' >> /tmp/hosts.list || true
            grep -Eo '^[0-9]+(\\.[0-9]+){3}' ${INVENTORY} 2>/dev/null >> /tmp/hosts.list || true

            sort -u /tmp/hosts.list | while read host; do
              if [ -n "$host" ]; then
                ssh-keyscan -H "$host" >> ~/.ssh/known_hosts 2>/dev/null || true
              fi
            done
            rm -f /tmp/hosts.list

            ls -la ~/.ssh
          '''
        }
      }
    }

    stage('Run Ansible Playbook') {
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: 'ansible-ssh-key', keyFileVariable: 'SSH_KEY_FILE', usernameVariable: 'ANSIBLE_USER')]) {
          sh '''
            echo "Running Ansible Playbook..."

            # ensure the private key file is readable
            chmod 600 "$SSH_KEY_FILE"

            ansible-playbook -i ${INVENTORY} ${PLAYBOOK} --private-key "$SSH_KEY_FILE" -vv

            RC=$?
            exit $RC
          '''
        }
      }
    }
  }

  post {
    success {
      echo "SUCCESS: Jenkins pipeline finished. Check master: kubectl get nodes"
    }
    failure {
      echo "FAILED: Look at console logs for more details."
    }
  }
}
