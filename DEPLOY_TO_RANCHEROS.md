# DEPLOY TO RANCHEROS


## Cloud-Init

```yml
#cloud-config

runcmd:
  # Protocol 1 is insecure and must not be used.
  - echo "Protocol 2 # $(date -R)" >> /etc/ssh/sshd_config

  # Unless we really need it its best to have it disabled.
  - sed -i -E "/^#?X11Forwarding/s/^.*$/X11Forwarding no # $(date -R)/" /etc/ssh/sshd_config

  # Can be used by hackers and malware to open backdoors in the server.
  - sed -i -E "/^#?AllowTcpForwarding/s/^.*$/AllowTcpForwarding no # $(date -R)/" /etc/ssh/sshd_config

  # @link http://www.cyberciti.biz/tips/linux-unix-bsd-openssh-server-best-practices.html

  # ---> Disable root user login:
  - sed -i -E "/^#?PermitRootLogin/s/^.*$/PermitRootLogin no # $(date -R)/" /etc/ssh/sshd_config
  - sed -i -E "/^#?ChallengeResponseAuthentication/s/^.*$/ChallengeResponseAuthentication no # $(date -R)/" /etc/ssh/sshd_config
  - sed -i -E "/^#?PasswordAuthentication/s/^.*$/PasswordAuthentication no # $(date -R)/" /etc/ssh/sshd_config
  - sed -i -E "/^#?UsePAM/s/^.*$/UsePAM no # $(date -R)/" /etc/ssh/sshd_config

  # ---> Disable password based login
  - echo "AuthenticationMethods publickey # $(date -R)" >> /etc/ssh/sshd_config
  - sed -i -E "/^#?PubkeyAuthentication/s/^.*$/PubkeyAuthentication yes # $(date -R)/" /etc/ssh/sshd_config

  # ---> Limit Users ssh access
  - echo "AllowUsers rancher # $(date -R)" >> /etc/ssh/sshd_config

  # ---> Disable Empty Passwords
  - sed -i -E "/^#?PermitEmptyPasswords/s/^.*$/PermitEmptyPasswords no # $(date -R)/" /etc/ssh/sshd_config

  # ---> Change SSH Port and limit IP binding
  # changing here the default port for SSH, implies also to open it in the firewall
  - sed -i -E "/^#?Port/s/^.*$/Port 26928 # $(date -R)/" /etc/ssh/sshd_config
  # listen only in IPV6
  - sed -i -E "/^#?AddressFamily/s/^.*$/AddressFamily inet # $(date -R)/" /etc/ssh/sshd_config
  # optionally lock it down to your IPv6 address
  - sed -i -E "/^#?ListenAddress/s/^.*$/ListenAddress your.ipv6.here # $(date -R)/" /etc/ssh/sshd_config

  # ---> Thwart SSH crackers/brute force attacks
  # @TODO Install DenyHosts, Fail2Ban or similar

  # ---> Rate-limit incoming traffic at TCP port for SSH and HTTP(DDOS attacks prevention)
  # @TODO Configure the firewall with specific rules

  # ---> Use port knocking (optional)
  # @TODO Maybe with knockd https://www.cyberciti.biz/faq/debian-ubuntu-linux-iptables-knockd-port-knocking-tutorial/

  # ---> Configure idle log out timeout interval
  # set for two minutes
  - sed -i -E "/^#?ClientAliveInterval/s/^.*$/ClientAliveInterval 120 # $(date -R)/" /etc/ssh/sshd_config
  # only 1 ssh session per user is allowed
  - sed -i -E "/^#?ClientAliveCountMax/s/^.*$/ClientAliveCountMax 0 # $(date -R)/" /etc/ssh/sshd_config

  # ---> Disable .rhosts files (verification)
  - sed -i -E "/^#?IgnoreRhosts/s/^.*$/IgnoreRhosts yes # $(date -R)/" /etc/ssh/sshd_config

  # ---> Disable host-based authentication (verification)
  - sed -i -E "/^#?HostbasedAuthentication/s/^.*$/HostbasedAuthentication no # $(date -R)/" /etc/ssh/sshd_config

  # ---> Chroot OpenSSH (Lock down users to their home directories)
  # @TODO This one depends on the operational needs for each user of the system.

  # ---> Bonus tips from Mozilla
  # @link https://infosec.mozilla.org/guidelines/openssh
  # Supported HostKey algorithms by order of preference.
  - echo "HostKey /etc/ssh/ssh_host_ed25519_key" >> /etc/ssh/sshd_config
  - echo "HostKey /etc/ssh/ssh_host_rsa_key" >> /etc/ssh/sshd_config
  - echo "HostKey /etc/ssh/ssh_host_ecdsa_key" >> /etc/ssh/sshd_config
  - echo "KexAlgorithms curve25519-sha256@libssh.org,ecdh-sha2-nistp521,ecdh-sha2-nistp384,ecdh-sha2-nistp256,diffie-hellman-group-exchange-sha256" >> /etc/ssh/sshd_config
  - echo "Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com,aes256-ctr,aes192-ctr,aes128-ctr" >> /etc/ssh/sshd_config
  - echo "MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com,umac-128-etm@openssh.com,hmac-sha2-512,hmac-sha2-256,umac-128@openssh.com" >> /etc/ssh/sshd_config

  # All Diffie-Hellman moduli in use should be at least 3072-bit-long (they are used for diffie-hellman-group-exchange-sha256) as per our Key management Guidelines recommendations. See also man moduli. To deactivate short moduli in two commands:
  - awk '$5 >= 3071' /etc/ssh/moduli > /etc/ssh/moduli.tmp && mv /etc/ssh/moduli.tmp /etc/ssh/moduli

  # LogLevel VERBOSE logs user's key fingerprint on login. Needed to have a clear audit track of which key was using to log in.
  - echo "LogLevel VERBOSE" >> /etc/ssh/sshd_config

  # Log sftp level file access (read/write/etc.) that would not be easily logged otherwise.
  - echo "Subsystem sftp  /usr/lib/ssh/sftp-server -f AUTHPRIV -l INFO" >> /etc/ssh/sshd_config

  # Use kernel sandbox mechanisms where possible in unprivileged processes
  # Systrace on OpenBSD, Seccomp on Linux, seatbelt on MacOSX/Darwin, rlimit elsewhere.
  - echo "UsePrivilegeSeparation sandbox" >> /etc/ssh/sshd_config

write_files:
  - path: /etc/rc.local
    permissions: "0750"
    owner: root
    content: |
      #!/bin/sh

      set -eux

      wait-for-docker

      export traefik="traefik:1.7"
      export git="alpine/git"
      export docker_compose="docker/compose"

      for image in $traefik $git $docker_compose; do
        until docker inspect $image > /dev/null 2>&1; do
          docker pull $image
          sleep 2
        done
      done


      (
        cat <<'EOF'
      #!/bin/sh

      docker run \
        -it \
        --rm \
        --user ${UID}:${GID} \
        --volume ${PWD}:/git \
        --volume $HOME/.ssh:/root/.ssh \
        alpine/git \
        $@

      EOF
      ) > git

      sudo mv git /usr/bin
      sudo chmod +x /usr/bin/git

      (
        cat <<'EOF'
      #!/bin/sh

      docker run \
        -it \
        --rm \
        --volume ${PWD}:${PWD} \
        --volume /var/run/docker.sock:/var/run/docker.sock \
        --workdir ${PWD} \
        docker/compose \
        $@

      EOF
      ) > docker-compose

      sudo mv docker-compose /usr/bin
      sudo chmod +x /usr/bin/docker-compose

      git clone https://github.com/approov/debian-traefik-setup.git && cd debian-traefik-setup

      sudo cp -r ./traefik /opt

      (
        cat <<'EOF'
      TRAEFIK_DOCKER_DOMAIN=dev.exadra37.com
      TRAEFIK_ACME_EMAIL=ksierra37@gmail.com
      EOF
      ) > .env

      sudo mv .env /opt/traefik
      sudo chmod 660 /opt/traefik/.env

      sudo touch /opt/traefik/acme.json
      sudo chmod 600 /opt/traefik/acme.json

      docker network create traefik || true

      cd /opt/traefik

      docker-compose up -d traefik

      rm -rf /home/rancher/debian-traefik-setup
```
