pipeline {
  agent {
    docker {
      image 'python:3.11-slim'
      args '-u root:root'
    }
  }

  environment {
    INVENTORY = "inventory.ini"
    PLAYBOOK  = "full-cluster-automation.yml"
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Install Ansible') {
      steps {
        sh '''
          python -m pip install --upgrade pip
          pip install ansible==8.5.0 || pip install ansible
          ansible --version
        '''
      }
    }

    stage('Prepare SSH (known_hosts)') {
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: 'ansible-ssh-key',
                                          keyFileVariable: 'SSH_KEY_FILE',
                                          usernameVariable: 'ANSIBLE_USER')]) {
          sh '''
            chmod 600 "$SSH_KEY_FILE"
            mkdir -p ~/.ssh
            touch ~/.ssh/known_hosts
            # Collect hosts from inventory and add to known_hosts
            awk '/ansible_host=/{match($0,/ansible_host=([^ ]+)/,a); if(a[1]) print a[1]} /^[0-9]/ {print $1}' ${INVENTORY} | sort -u | while read host; do
              if [ -n "$host" ]; then ssh-keyscan -H "$host" >> ~/.ssh/known_hosts 2>/dev/null || true; fi
            done
            ls -la ~/.ssh
          '''
        }
      }
    }

    stage('Run Ansible Playbook') {
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: 'ansible-ssh-key',
                                          keyFileVariable: 'SSH_KEY_FILE',
                                          usernameVariable: 'ANSIBLE_USER'),
                         string(credentialsId: 'ansible-vault-pass', variable: 'VAULT_PASS'),
                         string(credentialsId: 'ansible-become-pass', variable: 'BECOME_PASS')]) {
          sh '''
            # vault option
            VAULT_OPT=""
            if [ -n "$VAULT_PASS" ]; then
              VAULT_FILE=$(mktemp)
              printf "%s" "$VAULT_PASS" > "$VAULT_FILE"
              chmod 600 "$VAULT_FILE"
              VAULT_OPT="--vault-password-file $VAULT_FILE"
            fi

            # become option (if needed)
            BECOME_OPT=""
            if [ -n "$BECOME_PASS" ]; then
              BECOME_OPT="--extra-vars ansible_become_pass=${BECOME_PASS}"
            fi

            # run playbook using Jenkins-provided private key
            ansible-playbook -i ${INVENTORY} ${PLAYBOOK} --private-key "$SSH_KEY_FILE" ${VAULT_OPT} ${BECOME_OPT} -vv
            RC=$?

            # cleanup vault file
            if [ -n "$VAULT_FILE" ]; then shred -u "$VAULT_FILE" || rm -f "$VAULT_FILE"; fi
            exit $RC
          '''
        }
      }
    }
  }

  post {
    success {
      echo 'Pipeline succeeded. Please check master: kubectl get nodes'
    }
    failure {
      echo 'Pipeline failed. Check console output for errors.'
    }
  }
}
