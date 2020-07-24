# vi: set ft=ruby :

PRIVATE_NET_IP = "172.28.128."

Vagrant.configure("2") do |config|

  vmservers = ["graylog", "systems"]
  last_octet = 30

  vmservers.each do |server|
    config.vm.define "#{server}" do |node|
      node.vm.box = "centos/7"
      node.vm.hostname = "#{server}"
      node.vm.network "private_network", ip: PRIVATE_NET_IP + last_octet.to_s
      node.vm.synced_folder ".", "/vagrant", type: "nfs"
      last_octet = last_octet + 1

      node.vm.provider "virtualbox" do |vbox|
        vbox.memory = 4096
        vbox.cpus = 4
      end

      # Common provision
      node.vm.provision "shell", inline: <<-SHELL

        # Set SELinux to permissive
        setenforce 0
        sed -i "s/SELINUX=enforcing/SELINUX=permissive/g" /etc/selinux/config

        # Import GPG keys
        rpm --import \
          /etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7 \
          https://download.docker.com/linux/centos/gpg \
          http://download.fedoraproject.org/pub/epel/RPM-GPG-KEY-EPEL-7 \
          https://packages.treasuredata.com/GPG-KEY-td-agent

        # Install Docker Community Edition
        yum-config-manager --add-repo \
            https://download.docker.com/linux/centos/docker-ce.repo
        yum install -y docker-ce docker-ce-cli containerd.io
        systemctl start docker
        systemctl -q enable docker
        usermod -aG docker vagrant

        # Convenience
        yum install -y vim

        # Install rsyslog
        yum install -y rsyslog
        systemctl start rsyslog
        systemctl -q enable rsyslog

        # Add rsyslog forwarding option if it does not exist
        if ! grep -q "127.0.0.1:5140" /etc/rsyslog.conf; then
          echo "*.* @127.0.0.1:5140" >> /etc/rsyslog.conf
          systemctl restart rsyslog
        fi

        # Setup TLS
        if [ ! -f /vagrant/tmp/ca_key.pem ]; then
          echo "Generating TLS certificates..."
          cd /vagrant/tmp
          openssl req -newkey rsa:4096 \
                      -x509 \
                      -sha256 \
                      -days 3650 \
                      -nodes \
                      -out ca_cert.pem \
                      -keyout ca_key.pem \
                      -subj "/C=US/ST=Local/L=Local/O=Org/OU=IT/CN=example.com" \
                      2> /dev/null
        fi

        # Install td-agent
        cp /vagrant/td-agent.repo /etc/yum.repos.d/
        yum check-update
        yum install -y td-agent
        td-agent-gem install fluent-plugin-gelf-hs gelf
        systemctl -q enable td-agent

      SHELL

      # Commmon provision: install docker-compose
      node.vm.provision "shell", path: "install-compose.sh"

      # Graylog specific provision
      if server == "graylog"
        node.vm.provision "shell", inline: <<-SHELL

          cp /vagrant/td-agent-server.conf /etc/td-agent/td-agent.conf
          systemctl restart td-agent

          # Install jq
          yum install -y epel-release
          yum install -y jq

          # Start Graylog
          cd /vagrant
          /usr/local/bin/docker-compose up -d 2> /dev/null

          # Wait 120 seconds for Graylog to come online
          SECONDS=0
          while true
          do
            GRAYLOG_STATE=$(
              docker inspect vagrant_graylog_1 \
                | jq --raw-output '.[] | .State.Health.Status')

            if [[ "$GRAYLOG_STATE" == "healthy" ]]; then
              echo "Graylog is available."
              sleep 5
              break
            elif [[ "$GRAYLOG_STATE" != "starting" ]]; then
              echo "Something is wrong with Graylog. Aborting."
              exit 1
            elif [[ $SECONDS -le 120 ]]; then
              echo "Waiting for Graylog ($SECONDS/120 seconds)"
              sleep 10
            else
              echo "Waiting on Graylog timed out. Aborting."
              exit 1
            fi
          done

          # Check for existing GELF TCP Input
          INPUTSTATE=$(
            curl -s -X GET \
                -H "Content-Type: application/json" \
                -H "X-Requested-By: cli" \
                -u admin:admin \
                "http://graylog.172.28.128.30.xip.io:8080/api/system/inputstates")

          INPUT_TYPES=$(echo $INPUTSTATE | jq --raw-output '.states | .[] | .message_input.type')

          for TYPE in $INPUT_TYPES; do
            if [[ "$TYPE" == "org.graylog2.inputs.gelf.tcp.GELFTCPInput" ]]; then
              echo "Found GELF TCP input in Graylog, aborting input installation."
              exit
            fi
          done

          # Install GELF TCP Input
          curl -i -s -X POST \
              -H "Content-Type: application/json" \
              -H "X-Requested-By: cli" \
              -u admin:admin \
              "http://graylog.172.28.128.30.xip.io:8080/api/system/inputs" \
              -d @GELFTCPInput.json

        SHELL

      elsif server == "systems"
        node.vm.provision "shell", inline: <<-SHELL

          # Install apache
          yum install -y httpd
          systemctl start httpd
          systemctl -q enable httpd

          # Configure td-agent
          cp /vagrant/td-agent.conf /etc/td-agent/td-agent.conf
          mkdir -p /var/log/containers
          chown -R td-agent:td-agent /var/log/containers
          chmod -R 755 /var/log
          systemctl restart td-agent

          # Bring up WordPress test containers
          cd /vagrant/wordpress
          /usr/local/bin/docker-compose up -d 2> /dev/null

        SHELL

      end
    end
  end
end
