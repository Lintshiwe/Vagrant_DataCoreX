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

    # Optional bridged networking for LAN access (set DATACOREX_ENABLE_BRIDGE=true)
    if ENV.fetch("DATACOREX_ENABLE_BRIDGE", "false").downcase == "true"
        bridge_iface = ENV["DATACOREX_BRIDGE"]&.strip
        bridge_ip = ENV["DATACOREX_BRIDGE_IP"]&.strip

        network_options = {}
        network_options[:bridge] = bridge_iface unless bridge_iface.nil? || bridge_iface.empty?
        network_options[:ip] = bridge_ip unless bridge_ip.nil? || bridge_ip.empty?

        if network_options.empty?
            config.vm.network "public_network"
        else
            config.vm.network "public_network", **network_options
        end
    end

    config.vm.synced_folder ".", "/vagrant", disabled: false

        config.vm.provision "shell",
                privileged: true,
                env: {
                    "GITHUB_TOKEN" => ENV['GITHUB_TOKEN'] || "",
                    "GIT_TOKEN" => ENV['GIT_TOKEN'] || "",
                    "DATACOREX_NGROK_TOKEN" => ENV['DATACOREX_NGROK_TOKEN'] || ENV['NGROK_AUTHTOKEN'] || ""
                },
                inline: <<-'SHELL'
set -euo pipefail

export DEBIAN_FRONTEND=noninteractive

# Stop running services during reprovision to avoid file locks
systemctl stop datacorex-frontend.service 2>/dev/null || true
systemctl stop datacorex-backend.service 2>/dev/null || true
systemctl stop ngrok-datacorex.service 2>/dev/null || true

# Base tooling
apt-get update
apt-get install -y curl git ca-certificates gnupg lsb-release build-essential unzip maven

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

REPO_URL="https://github.com/Lintshiwe/DataCoreX.git"
AUTH_HEADER=""
if [ -n "${GITHUB_TOKEN:-}" ]; then
    AUTH_HEADER="Authorization: Bearer ${GITHUB_TOKEN}"
elif [ -n "${GIT_TOKEN:-}" ]; then
    AUTH_HEADER="Authorization: Bearer ${GIT_TOKEN}"
fi

mkdir -p DataCoreX
if [ ! -d DataCoreX/.git ]; then
    rm -rf DataCoreX
    if [ -n "$AUTH_HEADER" ]; then
        if ! git -c http.extraHeader="$AUTH_HEADER" clone "$REPO_URL" DataCoreX; then
            printf '%s\n' "ERROR: Unable to clone DataCoreX. Ensure GITHUB_TOKEN or GIT_TOKEN is set with repo access." >&2
            exit 1
        fi
    else
        if ! git clone "$REPO_URL" DataCoreX; then
            cat <<'ERR' >&2
ERROR: Unable to clone DataCoreX. If the repository is private, export GITHUB_TOKEN (or GIT_TOKEN) before running `vagrant up`.
ERR
            exit 1
        fi
    fi
else
    cd DataCoreX
    git remote set-url origin "$REPO_URL"
    if [ -n "$AUTH_HEADER" ]; then
        if ! git -c http.extraHeader="$AUTH_HEADER" fetch origin; then
            printf '%s\n' "ERROR: Unable to fetch DataCoreX. Check your GITHUB_TOKEN or GIT_TOKEN." >&2
            exit 1
        fi
    else
        if ! git fetch origin; then
            printf '%s\n' "ERROR: Unable to fetch DataCoreX. Provide GITHUB_TOKEN or GIT_TOKEN if the repo is private." >&2
            exit 1
        fi
    fi
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
if [ -f package-lock.json ]; then
    npm ci
else
    npm install
fi
cd ..

cd datacorex-backend
if [ -f mvnw ]; then
    chmod +x mvnw
    ./mvnw -q dependency:go-offline || ./mvnw dependency:go-offline
else
    mvn -q dependency:go-offline || mvn dependency:go-offline
fi
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
ExecStart=/usr/bin/env bash -lc 'if [ -f ./mvnw ]; then chmod +x ./mvnw; ./mvnw spring-boot:run; else mvn spring-boot:run; fi'
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

# Optional ngrok tunnel setup for external access
if [ -n "${DATACOREX_NGROK_TOKEN:-}" ]; then
    if ! command -v ngrok >/dev/null 2>&1; then
        curl -fsSL https://bin.equinox.io/c/bNyj1mQVY4c/ngrok-v3-stable-linux-amd64.tgz -o /tmp/ngrok.tgz
        tar -C /usr/local/bin -xzf /tmp/ngrok.tgz
        rm -f /tmp/ngrok.tgz
    fi

    sudo -u vagrant bash <<EOF
set -euo pipefail
CONFIG_DIR="\$HOME/.config/ngrok"
mkdir -p "\$CONFIG_DIR"
cat > "\$CONFIG_DIR/ngrok.yml" <<NGCFG
version: "3"
agent:
  authtoken: "${DATACOREX_NGROK_TOKEN}"
  log: stdout
  log_level: info
tunnels:
  frontend:
    proto: http
    addr: 3000
  backend:
    proto: http
    addr: 8081
NGCFG
EOF

    cat <<'EOF' >/etc/systemd/system/ngrok-datacorex.service
[Unit]
Description=ngrok tunnels for DataCoreX
After=network-online.target datacorex-backend.service datacorex-frontend.service
Wants=network-online.target

[Service]
Type=simple
User=vagrant
WorkingDirectory=/home/vagrant
Environment=HOME=/home/vagrant
ExecStart=/usr/local/bin/ngrok start --all --config /home/vagrant/.config/ngrok/ngrok.yml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

    systemctl daemon-reload
    systemctl enable ngrok-datacorex.service
    systemctl restart ngrok-datacorex.service

    sudo -u vagrant python3 - <<'PY'
import json
import time
from pathlib import Path
from urllib.request import urlopen

urls_file = Path('/vagrant/ngrok_urls.txt')

def fetch_urls():
    with urlopen('http://127.0.0.1:4040/api/tunnels', timeout=1.0) as resp:
        data = json.load(resp)
    tunnels = {t['name']: t.get('public_url') for t in data.get('tunnels', [])}
    frontend = tunnels.get('frontend')
    backend = tunnels.get('backend')
    if frontend and backend:
        return frontend, backend
    raise RuntimeError('tunnels not ready')

for _ in range(30):
    try:
        frontend_url, backend_url = fetch_urls()
        message = (
            'Ngrok frontend URL: {front}\n'
            'Ngrok backend URL: {back}\n'
        ).format(front=frontend_url, back=backend_url)
        print(message, end='')
        urls_file.write_text(message)
        break
    except Exception:
        time.sleep(1)
else:
    print('WARNING: Unable to retrieve ngrok tunnel URLs within timeout.')
PY

    systemctl --no-pager status ngrok-datacorex.service || true
else
    echo "Skipping ngrok tunnel setup (DATACOREX_NGROK_TOKEN not provided)."
fi

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
Ngrok URLs (if configured): see ngrok_urls.txt or run vagrant ssh -c "curl -s http://127.0.0.1:4040/api/tunnels"
    MSG
end
