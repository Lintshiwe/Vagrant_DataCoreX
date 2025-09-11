# -*- mode: ruby -*-
# vi: set ft=ruby :

# Vagrant configuration for DataCoreX project
# This Vagrantfile will set up a VM to run the DataCoreX application from GitHub

Vagrant.configure("2") do |config|
  # Base box - using Ubuntu 20.04 LTS
  config.vm.box = "ubuntu/focal64"
  config.vm.box_check_update = false

  # Configure VM settings
  config.vm.provider "virtualbox" do |vb|
    vb.name = "DataCoreX-VM"
    vb.memory = "2048"
    vb.cpus = 2
    vb.gui = false
  end

  # Network configuration
  # Forward common ports for web applications
  config.vm.network "forwarded_port", guest: 3000, host: 3000  # React/Node.js
  config.vm.network "forwarded_port", guest: 5000, host: 5000  # Flask/Python
  config.vm.network "forwarded_port", guest: 8000, host: 8000  # Django/General
  config.vm.network "forwarded_port", guest: 8080, host: 8888  # Alternative web port (changed from 8080 to 8888)
  config.vm.network "forwarded_port", guest: 80, host: 8081    # HTTP
  
  # Create a private network with static IP to avoid DHCP conflicts
  config.vm.network "private_network", ip: "192.168.56.10"

  # Shared folder configuration
  # Sync the current directory with /vagrant in the VM
  config.vm.synced_folder ".", "/vagrant", disabled: false

  # Provisioning script to set up the environment
  config.vm.provision "shell", inline: <<-SHELL
    # Update system packages
    echo "Updating system packages..."
    apt-get update -y
    apt-get upgrade -y

    # Install essential development tools
    echo "Installing development tools..."
    apt-get install -y git curl wget vim nano htop tree unzip
    apt-get install -y build-essential software-properties-common

    # Install Node.js and npm (for JavaScript/React projects)
    echo "Installing Node.js..."
    curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
    apt-get install -y nodejs

    # Install Python and pip (for Python projects)
    echo "Installing Python..."
    apt-get install -y python3 python3-pip python3-venv python3-dev

    # Install Docker (for containerized applications)
    echo "Installing Docker..."
    curl -fsSL https://get.docker.com -o get-docker.sh
    sh get-docker.sh
    usermod -aG docker vagrant

    # Install Docker Compose
    echo "Installing Docker Compose..."
    curl -L "https://github.com/docker/compose/releases/download/v2.20.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
    chmod +x /usr/local/bin/docker-compose

    # Clone the DataCoreX repository
    echo "Cloning DataCoreX repository..."
    cd /home/vagrant
    if [ ! -d "DataCoreX" ]; then
      git clone https://github.com/Lintshiwe/DataCoreX.git
      chown -R vagrant:vagrant DataCoreX
    fi

    # Try to detect project type and set up accordingly
    cd /home/vagrant/DataCoreX
    
    # Check for package.json (Node.js project)
    if [ -f "package.json" ]; then
      echo "Detected Node.js project, installing dependencies..."
      sudo -u vagrant npm install
    fi

    # Check for requirements.txt (Python project)
    if [ -f "requirements.txt" ]; then
      echo "Detected Python project, installing dependencies..."
      sudo -u vagrant pip3 install -r requirements.txt
    fi

    # Check for Pipfile (Python project with Pipenv)
    if [ -f "Pipfile" ]; then
      echo "Detected Python project with Pipenv..."
      pip3 install pipenv
      sudo -u vagrant pipenv install
    fi

    # Check for docker-compose.yml
    if [ -f "docker-compose.yml" ] || [ -f "docker-compose.yaml" ]; then
      echo "Detected Docker Compose project..."
      # Docker is already installed above
    fi

    # Set up environment variables if .env.example exists
    if [ -f ".env.example" ]; then
      echo "Setting up environment file..."
      sudo -u vagrant cp .env.example .env
    fi

    # Create auto-start script for DataCoreX
    echo "Creating auto-start script..."
    cat > /home/vagrant/start-datacorex.sh << 'EOF'
#!/bin/bash
cd /home/vagrant/DataCoreX

echo "Starting DataCoreX application..."

# Check for different project types and start accordingly
if [ -f "package.json" ]; then
    echo "Detected Node.js project..."
    
    # Check for common start scripts
    if npm run --silent 2>/dev/null | grep -q "start"; then
        echo "Starting with npm start..."
        npm start
    elif npm run --silent 2>/dev/null | grep -q "dev"; then
        echo "Starting with npm run dev..."
        npm run dev
    elif npm run --silent 2>/dev/null | grep -q "serve"; then
        echo "Starting with npm run serve..."
        npm run serve
    elif [ -f "server.js" ]; then
        echo "Starting with node server.js..."
        node server.js
    elif [ -f "app.js" ]; then
        echo "Starting with node app.js..."
        node app.js
    elif [ -f "index.js" ]; then
        echo "Starting with node index.js..."
        node index.js
    else
        echo "No suitable start command found for Node.js project"
        echo "Available npm scripts:"
        npm run
    fi

elif [ -f "requirements.txt" ] || [ -f "app.py" ] || [ -f "main.py" ]; then
    echo "Detected Python project..."
    
    # Check for Flask application
    if [ -f "app.py" ]; then
        echo "Starting Flask app..."
        export FLASK_APP=app.py
        export FLASK_ENV=development
        python3 -m flask run --host=0.0.0.0
    elif [ -f "main.py" ]; then
        echo "Starting with python3 main.py..."
        python3 main.py
    elif [ -f "manage.py" ]; then
        echo "Detected Django project, starting server..."
        python3 manage.py runserver 0.0.0.0:8000
    elif [ -f "wsgi.py" ]; then
        echo "Starting with WSGI..."
        python3 wsgi.py
    else
        echo "Python project detected but no standard entry point found"
        echo "Available Python files:"
        find . -name "*.py" -maxdepth 2
    fi

elif [ -f "Pipfile" ]; then
    echo "Detected Python project with Pipenv..."
    if [ -f "app.py" ]; then
        echo "Starting Flask app with pipenv..."
        pipenv run python app.py
    elif [ -f "main.py" ]; then
        echo "Starting with pipenv run python main.py..."
        pipenv run python main.py
    else
        echo "Pipenv project detected but no standard entry point found"
    fi

elif [ -f "docker-compose.yml" ] || [ -f "docker-compose.yaml" ]; then
    echo "Detected Docker Compose project..."
    echo "Starting with docker-compose up..."
    docker-compose up

elif [ -f "Dockerfile" ]; then
    echo "Detected Docker project..."
    echo "Building and running Docker container..."
    docker build -t datacorex .
    docker run -p 8000:8000 -p 3000:3000 -p 5000:5000 datacorex

elif [ -f "README.md" ]; then
    echo "No automatic start method detected."
    echo "Checking README.md for startup instructions..."
    echo "========================================="
    head -50 README.md
    echo "========================================="
    echo "Please check the README.md file for specific startup instructions."

else
    echo "Unable to determine project type or start method."
    echo "Please check the project documentation for startup instructions."
    ls -la
fi
EOF

    # Make the script executable
    chmod +x /home/vagrant/start-datacorex.sh
    chown vagrant:vagrant /home/vagrant/start-datacorex.sh

    # Create systemd service for auto-starting the application
    cat > /etc/systemd/system/datacorex.service << 'EOF'
[Unit]
Description=DataCoreX Application
After=network.target

[Service]
Type=simple
User=vagrant
WorkingDirectory=/home/vagrant/DataCoreX
ExecStart=/home/vagrant/start-datacorex.sh
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
EOF

    # Enable and start the service
    systemctl daemon-reload
    systemctl enable datacorex.service

    # Display project information
    echo "========================================="
    echo "DataCoreX VM Setup Complete!"
    echo "========================================="
    echo "VM IP: $(hostname -I | awk '{print $1}')"
    echo "Project location: /home/vagrant/DataCoreX"
    echo "Auto-start script: /home/vagrant/start-datacorex.sh"
    echo ""
    echo "Available ports:"
    echo "  - 3000 (React/Node.js)"
    echo "  - 5000 (Flask/Python)"
    echo "  - 8000 (Django/General)"
    echo "  - 8080 (Alternative web)"
    echo "  - 8081 (HTTP)"
    echo ""
    echo "DataCoreX will auto-start on boot!"
    echo ""
    echo "Useful commands:"
    echo "  sudo systemctl status datacorex    # Check service status"
    echo "  sudo systemctl restart datacorex   # Restart service"
    echo "  sudo journalctl -u datacorex -f    # View logs"
    echo "  /home/vagrant/start-datacorex.sh   # Manual start"
    echo "========================================="
  SHELL

  # Additional provisioning to start the application after setup
  config.vm.provision "shell", run: "always", inline: <<-SHELL
    echo "Starting DataCoreX service..."
    systemctl start datacorex.service
    sleep 5
    
    # Show service status
    echo "DataCoreX service status:"
    systemctl is-active datacorex.service
    
    # Show recent logs
    echo "Recent application logs:"
    journalctl -u datacorex.service --no-pager -n 20
  SHELL

  # Post-up message
  config.vm.post_up_message = <<-MSG
    🚀 DataCoreX VM is ready and auto-starting!
    
    The DataCoreX application is automatically starting in the background.
    
    📍 Access your VM: vagrant ssh
    📂 Project location: /home/vagrant/DataCoreX
    
    🌐 Available forwarded ports:
    - localhost:3000 -> VM:3000 (React/Node.js)
    - localhost:5000 -> VM:5000 (Flask/Python)
    - localhost:8000 -> VM:8000 (Django/General)
    - localhost:8888 -> VM:8080 (Alternative web)
    - localhost:8081 -> VM:80 (HTTP)
    
    📋 Service Management Commands:
    - sudo systemctl status datacorex    # Check if app is running
    - sudo systemctl restart datacorex   # Restart the app
    - sudo journalctl -u datacorex -f    # View live logs
    
    ⏱️  Give it a minute to fully start, then check your browser!
    
    If the auto-start didn't work, you can manually run:
    /home/vagrant/start-datacorex.sh
  MSG
end
