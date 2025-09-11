# DataCoreX Vagrant Environment

This Vagrant configuration automatically sets up a development environment for the [DataCoreX](https://github.com/Lintshiwe/DataCoreX.git) project and **automatically starts the application** when the VM boots up.

## Prerequisites

Before using this Vagrant setup, ensure you have the following installed:

- [Vagrant](https://www.vagrantup.com/downloads) (latest version)
- [VirtualBox](https://www.virtualbox.org/wiki/Downloads) (as the VM provider)

## Quick Start

1. **Clone or download this repository**
2. **Navigate to the project directory**

   ```bash
   cd Vagrant_DataCoreX
   ```

3. **Start the VM**

   ```bash
   vagrant up
   ```

   This will:

   - Download the Ubuntu 20.04 LTS base image
   - Create and configure the virtual machine
   - Install necessary development tools
   - Clone the DataCoreX repository
   - Automatically detect and set up project dependencies
   - **🚀 Automatically start the DataCoreX application**

4. **Wait for auto-start (about 1-2 minutes)**

   The application will automatically detect the project type and start:

   - Node.js projects: `npm start`, `npm run dev`, or `node server.js`
   - Python Flask: `flask run` or `python app.py`
   - Python Django: `python manage.py runserver`
   - Docker projects: `docker-compose up` or Docker build/run

5. **Access your application**

   Check your browser at:

   - http://localhost:3000 (Node.js/React)
   - http://localhost:5000 (Flask)
   - http://localhost:8000 (Django)
   - http://localhost:8080 (Alternative)

## Auto-Start Features

### 🔄 Automatic Application Startup

The VM includes a systemd service that automatically:

- Detects the project type (Node.js, Python, Docker)
- Runs the appropriate startup command
- Restarts the application if it crashes
- Starts on every VM boot

### 📊 Service Management

```bash
# Check if the application is running
sudo systemctl status datacorex

# View live application logs
sudo journalctl -u datacorex -f

# Restart the application
sudo systemctl restart datacorex

# Stop the application
sudo systemctl stop datacorex

# Start the application
sudo systemctl start datacorex
```

### 🛠️ Manual Control

If you need to run the application manually:

```bash
# Stop the auto-service first
sudo systemctl stop datacorex

# Run manually
/home/vagrant/start-datacorex.sh
```

## VM Configuration

### System Specifications

- **OS**: Ubuntu 20.04 LTS (focal64)
- **Memory**: 2GB RAM
- **CPUs**: 2 cores
- **Disk**: Dynamic allocation

### Installed Software

- Git, curl, wget, vim, nano
- Node.js 18.x with npm
- Python 3 with pip
- Docker and Docker Compose
- Development tools (build-essential, etc.)

### Port Forwarding

The following ports are forwarded from your host machine to the VM:

| Host Port | VM Port | Purpose                    |
| --------- | ------- | -------------------------- |
| 3000      | 3000    | React/Node.js applications |
| 5000      | 5000    | Flask/Python applications  |
| 8000      | 8000    | Django/General web apps    |
| 8080      | 8080    | Alternative web port       |
| 8081      | 80      | HTTP services              |

## Usage

### Starting the VM

```bash
vagrant up
```

### Accessing the VM

```bash
vagrant ssh
```

### Stopping the VM

```bash
vagrant halt
```

### Restarting the VM

```bash
vagrant reload
```

### Destroying the VM

```bash
vagrant destroy
```

### Checking VM Status

```bash
vagrant status
```

## Project Structure

After the VM is set up, you'll find:

- **DataCoreX project**: `/home/vagrant/DataCoreX`
- **Shared folder**: `/vagrant` (synced with your host directory)

## Automatic Setup Features

The Vagrant provisioning script automatically:

1. Updates the system packages
2. Installs development tools and runtimes
3. Clones the DataCoreX repository
4. Detects project type and installs dependencies:
   - Node.js projects: runs `npm install`
   - Python projects: installs from `requirements.txt`
   - Python/Pipenv projects: sets up pipenv environment
5. Sets up environment files from `.env.example` if available

## Troubleshooting

### Common Issues

1. **VM fails to start**: Ensure VirtualBox is installed and virtualization is enabled in BIOS
2. **Port conflicts**: Check if any of the forwarded ports are already in use on your host
3. **Slow performance**: Increase VM memory in the Vagrantfile if needed
4. **Network issues**: Try `vagrant reload` to restart networking

### Useful Commands

```bash
# Reload Vagrantfile configuration
vagrant reload

# Re-run provisioning script
vagrant provision

# SSH with X11 forwarding (if needed)
vagrant ssh -- -X

# View VM logs
vagrant ssh -c "journalctl -f"
```

## Customization

You can modify the `Vagrantfile` to:

- Change VM specifications (memory, CPUs)
- Add/remove port forwarding rules
- Install additional software in the provisioning script
- Modify network configuration

## Support

If you encounter issues:

1. Check the DataCoreX repository documentation
2. Review Vagrant logs: `vagrant up --debug`
3. Ensure all prerequisites are properly installed

## License

This Vagrant configuration is provided as-is for development purposes.
