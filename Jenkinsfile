// Jenkinsfile — agent-labeled (requires Jenkins agent with Ansible installed)
pipeline {
  agent { label 'ansible' } // change label if your Jenkins node label is different

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

            # Add hosts from inventory into known_hosts to avoid interactive SSH prompt
            awk '/ansible_host=/{match($0,/ansible_host=([^ ]+)/,a); if(a[1]) print a[1]} /^[0-9]/ {print $1}' ${INVENTORY} \
              | sort -u | while read host; do
              if [ -n "$host" ]; then
                ssh-keyscan -H "$host" >> ~/.ssh/known_hosts 2>/dev/null || true
              fi
            done

            ls -la ~/.ssh
          '''
        }
      }
    }

    stage('Run Ansible Playbook') {
      steps {
        withCredentials([
          sshUserPrivateKey(credentialsId: 'ansible-ssh-key', keyFileVariable: 'SSH_KEY_FILE', usernameVariable: 'ANSIBLE_USER'),
          string(credentialsId: 'ansible-vault-pass', variable: 'VAULT_PASS'),
          string(credentialsId: 'ansible-become-pass', variable: 'BECOME_PASS')
        ]) {
          sh '''
            echo "Running Ansible Playbook..."

            VAULT_OPT=""
            if [ -n "$VAULT_PASS" ]; then
              VAULT_FILE=$(mktemp)
              printf "%s" "$VAULT_PASS" > "$VAULT_FILE"
              chmod 600 "$VAULT_FILE"
              VAULT_OPT="--vault-password-file $VAULT_FILE"
            fi

            BECOME_OPT=""
            if [ -n "$BECOME_PASS" ]; then
              BECOME_OPT="--extra-vars ansible_become_pass=${BECOME_PASS}"
            fi

            ansible-playbook -i ${INVENTORY} ${PLAYBOOK} \
              --private-key "$SSH_KEY_FILE" ${VAULT_OPT} ${BECOME_OPT} -vv

            RC=$?

            [ -n "$VAULT_FILE" ] && shred -u "$VAULT_FILE" || true

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
