# -*- mode: ruby -*-
# vi: set ft=ruby :

# Vagrant configuration that provisions the DataCoreX application automatically

Vagrant.configure("2") do |config|
    config.vm.box = "bento/ubuntu-22.04"
    config.vm.box_check_update = false
    config.vm.boot_timeout = 600
    config.ssh.insert_key = false

    config.vm.provider "virtualbox" do |vb|
        vb.name = "DataCoreX-VM"
        vb.memory = 4096
        vb.cpus = 2
        vb.gui = false
    end

    # Expose the React development server and Spring Boot API to the host
    config.vm.network "forwarded_port", guest: 3000, host: 3000, auto_correct: true
    config.vm.network "forwarded_port", guest: 8081, host: 8081, auto_correct: true

    config.vm.synced_folder ".", "/vagrant", disabled: false

    config.vm.provision "shell",
        privileged: true,
        shell: "bash",
        keep_color: true,
        inline: <<-'SHELL'
set -euo pipefail

export DEBIAN_FRONTEND=noninteractive

# Base tooling
apt-get update
apt-get install -y curl git ca-certificates gnupg lsb-release build-essential unzip

# Node.js 18 LTS
if ! command -v node >/dev/null 2>&1; then
    curl -fsSL https://deb.nodesource.com/setup_18.x | bash -
    apt-get install -y nodejs
fi

# Java 21 (Temurin)
if ! command -v java >/dev/null 2>&1 || ! java -version 2>&1 | grep -q "21."; then
    mkdir -p /etc/apt/keyrings
    curl -fsSL https://packages.adoptium.net/artifactory/api/gpg/key/public | gpg --dearmor -o /etc/apt/keyrings/adoptium.gpg
    echo "deb [signed-by=/etc/apt/keyrings/adoptium.gpg] https://packages.adoptium.net/artifactory/deb $(lsb_release -cs) main" > /etc/apt/sources.list.d/adoptium.list
    apt-get update
    apt-get install -y temurin-21-jdk
fi

# Fetch or refresh the DataCoreX repository
sudo -u vagrant bash <<'EOF'
set -euo pipefail
cd /home/vagrant
if [ ! -d DataCoreX ]; then
    git clone https://github.com/Lintshiwe/DataCoreX.git
else
    cd DataCoreX
    git fetch origin
    git reset --hard origin/main
fi
EOF

# Install project dependencies
sudo -u vagrant bash <<'EOF'
set -euo pipefail
cd /home/vagrant/DataCoreX

if [ -f .env.example ] && [ ! -f .env ]; then
    cp .env.example .env
fi

cd datacorex-frontend
npm ci
cd ..

cd datacorex-backend
chmod +x mvnw
./mvnw -q dependency:go-offline
EOF

# Backend service (Spring Boot)
cat <<'EOF' >/etc/systemd/system/datacorex-backend.service
[Unit]
Description=DataCoreX Backend (Spring Boot)
After=network.target
StartLimitIntervalSec=0

[Service]
Type=simple
User=vagrant
WorkingDirectory=/home/vagrant/DataCoreX/datacorex-backend
Environment=SPRING_PROFILES_ACTIVE=default
ExecStart=/usr/bin/env bash -lc './mvnw spring-boot:run'
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
EOF

# Frontend service (React dev server)
cat <<'EOF' >/etc/systemd/system/datacorex-frontend.service
[Unit]
Description=DataCoreX Frontend (React)
After=network.target
StartLimitIntervalSec=0

[Service]
Type=simple
User=vagrant
WorkingDirectory=/home/vagrant/DataCoreX/datacorex-frontend
Environment=BROWSER=none
Environment=HOST=0.0.0.0
Environment=PORT=3000
ExecStart=/usr/bin/env bash -lc 'npm start'
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable datacorex-backend.service
systemctl enable datacorex-frontend.service
systemctl restart datacorex-backend.service
systemctl restart datacorex-frontend.service

systemctl --no-pager status datacorex-backend.service || true
systemctl --no-pager status datacorex-frontend.service || true
SHELL

    config.vm.post_up_message = <<-MSG
DataCoreX VM is ready.
Frontend UI: http://localhost:3000
Backend API: http://localhost:8081
SSH access: vagrant ssh
Check services: sudo systemctl status datacorex-backend datacorex-frontend
Tail logs: sudo journalctl -u datacorex-backend -f
    MSG
end
