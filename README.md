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

---

## Troubleshooting

| Problem             | Fix                                                                                                          |
| ------------------- | ------------------------------------------------------------------------------------------------------------ |
| VM fails to boot    | Ensure virtualization (VT-x/AMD-V) is enabled and Hyper-V is disabled on Windows (or use VirtualBox 7.0.6+). |
| Port already in use | Edit the `config.vm.network` section to pick different host ports.                                           |
| Service not running | `vagrant ssh` → `sudo systemctl status datacorex-frontend datacorex-backend`                                 |
| Want a clean slate  | `vagrant destroy -f && vagrant up`                                                                           |

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
