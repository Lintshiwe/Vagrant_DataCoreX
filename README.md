# DataCoreX Vagrant Environment

This repository contains a Vagrant configuration that provisions an Ubuntu 22.04
virtual machine, clones the
[DataCoreX](https://github.com/Lintshiwe/DataCoreX) project, installs every
required dependency, and starts both the React frontend and Spring Boot backend
automatically with systemd services.

---

## Prerequisites

- [VirtualBox](https://www.virtualbox.org/wiki/Downloads) 6.1 or newer with
  virtualization enabled in BIOS/UEFI
- [Vagrant](https://www.vagrantup.com/downloads) 2.3 or newer
- At least **4 GB** of free RAM and **20 GB** of disk space

---

## Quick Start

```bash
git clone https://github.com/Lintshiwe/Vagrant_DataCoreX.git
cd Vagrant_DataCoreX
vagrant up
```

> **Private repo?** Export a GitHub personal access token with `repo` scope before running
> `vagrant up` so the VM can clone DataCoreX:
>
> ```bash
> # Windows PowerShell           # Windows CMD             # macOS/Linux
> $env:GITHUB_TOKEN="token"     set GITHUB_TOKEN=token    export GITHUB_TOKEN=token
> GITHUB_TOKEN=token vagrant up  # one-off invocation from any shell
> ```

> **Need public URLs?** Export `DATACOREX_NGROK_TOKEN=<your-ngrok-token>` (or `NGROK_AUTHTOKEN`) before `vagrant up`.
> Provisioning installs ngrok, starts tunnels for ports 3000/8081, prints the URLs, and writes them to `ngrok_urls.txt` in the project root.

The first run can take 10–15 minutes because the VM will:

1. Install Node.js 18, Temurin JDK 21, Git, build tools, etc.
2. Clone the upstream DataCoreX repository (or reset it if it already exists).
3. Install frontend dependencies (`npm ci`).
4. Prime backend dependencies (`./mvnw dependency:go-offline`).
5. Register and start two systemd services:
   - `datacorex-frontend` → `npm start` on port **3000**
   - `datacorex-backend` → `./mvnw spring-boot:run` on port **8081**

Once provisioning completes you can open:

- Frontend UI: <http://localhost:3000>
- Backend API: <http://localhost:8081>

You can double-check that each endpoint is live from the host machine:

```bash
curl -I http://localhost:3000    # should return HTTP/1.1 200 OK
curl -I http://localhost:8081    # should return HTTP/1.1 200
```

SSH access remains available with `vagrant ssh`.

---

## Managing Services

The services restart automatically if they crash and they start on every boot.
Useful commands inside the VM (`vagrant ssh`):

```bash
sudo systemctl status datacorex-frontend
sudo systemctl status datacorex-backend
sudo journalctl -u datacorex-frontend -f   # Tail frontend logs
sudo journalctl -u datacorex-backend -f    # Tail backend logs
sudo systemctl restart datacorex-backend   # Restart Spring Boot
```

---

## Vagrant Lifecycle Commands

```bash
vagrant up        # Create and provision the VM
vagrant halt      # Gracefully shut it down
vagrant reload    # Restart and re-apply networking config
vagrant destroy   # Remove the VM entirely
vagrant provision # Re-run the provisioning script
```

_Tip_: provisioning is idempotent—running `vagrant provision` keeps the cloned
DataCoreX workspace aligned with the latest `origin/main` branch.

---

## Customising the VM

- Adjust memory/CPU in the `config.vm.provider` block of the Vagrantfile.
- Add additional forwarded ports if you expose more services.
- Insert extra provisioning logic in the shell script section (e.g. seed data,
  install monitoring tools, etc.).
- Need LAN access? Export `DATACOREX_ENABLE_BRIDGE=true` before `vagrant up` to attach a bridged adapter so other devices on your network can reach the VM directly. Optionally set `DATACOREX_BRIDGE="<adapter name>"` and/or `DATACOREX_BRIDGE_IP="<static IP>"` for finer control.
- Need public URLs? Provide `DATACOREX_NGROK_TOKEN=<token>` (or `NGROK_AUTHTOKEN=<token>`) so ngrok launches automatically and records the tunnel URLs.

### Reaching the VM from other devices

1. **Same LAN (bridged adapter)**

   - On the host (before `vagrant up`):

     ```bash
       # Prompt for interface selection and use DHCP
       DATACOREX_ENABLE_BRIDGE=true vagrant up

       # Specify the adapter explicitly (Wi-Fi example)
       DATACOREX_ENABLE_BRIDGE=true DATACOREX_BRIDGE="Intel(R) Wi-Fi 6" vagrant up

       # Assign a static IP on your LAN (must be free and in the same subnet)
       DATACOREX_ENABLE_BRIDGE=true \
       DATACOREX_BRIDGE="Intel(R) Wi-Fi 6" \
       DATACOREX_BRIDGE_IP="192.168.1.240" \
       vagrant up
     ```

   - If you used DHCP, note the VM's assigned IP from the boot logs (`vagrant ssh -c "ip addr show"`).
   - Ensure your router/firewall allows inbound TCP 3000/8081 to that IP, then browse to `http://<vm-ip>:3000` or `:8081` from any device on the same network.

1. **Public internet (temporary tunnel)**

   - Set `DATACOREX_NGROK_TOKEN=<token>` before `vagrant up` (or `vagrant provision`). Provisioning installs ngrok, registers a systemd service, and writes the resulting URLs to `ngrok_urls.txt`.
   - To refresh or inspect later, run `vagrant ssh -c "curl -s http://127.0.0.1:4040/api/tunnels"`.
   - Prefer a different tunnel provider? Disable the service (`sudo systemctl stop ngrok-datacorex.service`) and run your own tool, e.g. [Cloudflare Tunnel](https://www.cloudflare.com/products/tunnel/) or `ssh -R`.

1. **Public internet (production)**
   - Deploy the frontend/backend to a public cloud VM or managed service (Azure App Service, AWS EC2/Elastic Beanstalk, etc.).
   - Configure TLS, environment variables, database connectivity, and firewalls according to your environment’s security policies.

---

## Troubleshooting

| Problem                  | Fix                                                                                                          |
| ------------------------ | ------------------------------------------------------------------------------------------------------------ |
| VM fails to boot         | Ensure virtualization (VT-x/AMD-V) is enabled and Hyper-V is disabled on Windows (or use VirtualBox 7.0.6+). |
| Port already in use      | Edit the `config.vm.network` section to pick different host ports.                                           |
| Service not running      | `vagrant ssh` → `sudo systemctl status datacorex-frontend datacorex-backend`                                 |
| No response on 3000/8081 | `curl -I http://localhost:3000` / `curl -I http://localhost:8081`; check firewall if bridged.                |
| Clone fails with 403     | Provide `GITHUB_TOKEN`/`GIT_TOKEN` with `repo` access so the VM can reach the private DataCoreX repo.        |
| Want a clean slate       | `vagrant destroy -f && vagrant up`                                                                           |

For deeper logs check `vagrant up --debug` or inspect
`/var/log/cloud-init.log` inside the VM.

---

## What Gets Installed

- Ubuntu 22.04 (bento box)
- Node.js 18 LTS & npm
- Temurin JDK 21
- Git, build-essential, unzip, curl, ca-certificates
- DataCoreX source code at `/home/vagrant/DataCoreX`
- systemd services `datacorex-frontend` and `datacorex-backend`

---

## License

This repository only contains infrastructure scaffolding; refer to the
DataCoreX project for application licensing. Use this configuration for local
development and testing purposes.
