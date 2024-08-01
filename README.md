# Bilú Host Development Server on Hostinger - An Opinionated Tutorial

This tutorial provides a step-by-step guide for setting up Rocky Linux 9 on a Hostinger VPS, tailored for development. It emphasises balancing security and usability based on best practices, covering essential tasks such as system updates, user management, and initial setup. Additionally, it includes the configuration of several services through containers using Podman, as of the open-source development environment manager Coder (https://coder.com/), Prometheus and Grafana.

## Prerequisites

Before starting this tutorial, ensure you have the following:

- Basic Knowledge of Linux: Understanding basic Linux commands and file system navigation is necessary.
- A Hostinger VPS Account: Sign up for a Hostinger account and have a VPS plan ready. Ensure you can access the VPS via SSH.
- Familiarity with Container Technology: Basic knowledge of container concepts and tools like Docker or Podman will be beneficial.
- Access to a Terminal and SSH Client: You will need a terminal with ssh client to connect to your VPS and run commands. Tools like PuTTY (for Windows) or a built-in terminal (for macOS/Linux) will work.
- A Domain Name (Optional): If you plan to access the server via domain name, ensure you have a domain registered and configured to point to your VPS.

## Why Rocky Linux 9?

Rocky Linux 9 is a stable, secure OS designed for server environments, offering enterprise-grade security features. Fully compatible with Red Hat Enterprise Linux, it ensures seamless transition and integration. Community-developed for transparency and trustworthiness, it provides long-term support with regular updates and patches. Free to use, it reduces costs without compromising quality. Its familiarity to RHEL administrators (like me) makes it a preferred choice.

## Why Podman?

Podman is an advanced, daemon-less container engine that offers a secure and efficient way to manage containers. Unlike Docker, Podman runs containers by default as non-root users, enhancing the security by reducing the attack surface. It is fully compatible with OCI (Open Container Initiative) standards, ensuring interoperability with other container tools and ecosystems. Its lightweight and modular architecture fits well with the needs of a versatile and secure server setup, such as the one outlined in this tutorial.

## Why Hostinger?

Hostinger's VPS plans offer a cost-effective balance of performance, ideal for projects with budget considerations (like mine).

## Why Bilú?

Bilú Gabriel was the late companion dog of the author's family, a Belgian Shepherd, it lived a long life of 14 years and 3 months. His family loved him so much and still continues to love. This server will go to production with his name on it as a tribute to him.

**Disclaimer**: This guide reflects my specific needs, knowledge, and experience. Individual user requirements may necessitate additional configurations.

## Basic server configuration

This section covers the operating system configuration steps up to the point of installing Coder.

### Adding a non-root user with sudo privileges

It's recommended to avoid using the root user for everyday tasks. Here, we create a new user (e.g., diego) and grant to it sudo privileges by adding it to the wheel group. Subsequently, we disable the root user login for increased security.

Add the administrative user:

```console
# adduser diego
# passwd diego
```

Add the newly created user to the administrative group:

```console
# usermod -aG wheel diego
```

Configure the SSH deamon to disallow the root user login:

```console
# vi /etc/ssh/sshd_config
// locate the line "PermitRootLogin yes" and change it to "PermitRootLogin no"
# systemctl restart sshd
```

For extra security measurement, remove the root ability to access the shell

```console
# usermod --shell /sbin/nologin root
```

Reboot the machine and log-in via SSH using the newly created administrative user.

### Setting the hostname

While Hostinger allows setting a hostname during VPS creation, this step ensures that the hostname is configured correctly. Use the `$ hostnamectl` command to verify the hostname after making changes.

```console
$ sudo hostnamectl --static hostname "bilu.host"
$ sudo hostnamectl --pretty hostname "Bilú Host Development Server"
```

### Updating the timezone

Update the VPS timezone to your location using the `timedatectl`. Use the `timedatectl list-timezones` command to view available options. Verify the change with the command `timedatectl`.

```console
$ sudo timedatectl set-timezone "America/Recife"
```

### Removing unecessary packages

Remove the packages that for you don't make sense to stay in the default Rocky Linux 9 installation. The example shows packages that I don't need.

```console
$ sudo dnf remove -y cockpit-ws cockpit-system cockpit-bridge
```

### Updating the system

Perform a system update to address security vulnerabilities and improve stability.

```console
$ sudo dnf update -y
```

Reboot the machine to apply the updates and log-in via SSH using the administrative user account.

### Securing the server with a Firewall

A properly configured firewall is essential for restricting unauthorized access to your server. Here, we install and enable the firewalld service. We'll configure specific firewall rules in a later section. In my case, I also removed the rules that enables communication for the Cockpit service.

Install the firewalld package:

```console
$ sudo dnf install -y firewalld
```

Enable as a systemd service and start the firewalld right away:

```console
$ sudo systemctl enable firewalld
$ sudo systemctl start firewalld
```

Remove the firewalld rules created for the removed cockpit service:

```console
$ sudo firewall-cmd --remove-service=cockpit --permanent
// reload the firewall rules
$ sudo firewall-cmd --reload
```

### Installing Podman

This is the final step before using Podman to install specific software through containers.

Install the podman package:

```console
$ sudo dnf install -y podman
```

Create a regular podman user. This is best practice to separate the podman user from the administrative user. If someone breaks into Podman, it will not be able to access root privileges.

```console
$ sudo adduser podman
$ sudo passwd podman
```

Reboot the machine and log-in via SSH using the newly created podman user.

Here, enable and start the podman and podman.socket services:

```console
$ systemctl --user enable podman
$ systemctl --user start podman
$ systemctl --user enable podman.socket
$ systemctl --user start podman.socket
```

### Installing HashiCorp Nomad

```console
// just to make sure it's installed (by default should be installed on Hostinger's default installation)
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo
sudo yum -y install nomad
```

Checkk installation with:

```console
nomad -v
```

Configure:

```console
sudo tee /etc/nomad.d/nomad.hcl <<'EOF'
name       = "bilu-host"
region     = "br"
datacenter = "bilu-dc"
data_dir   = "/opt/nomad/data"
bind_addr  = "93.127.212.100"

server {
  enabled          = true
  bootstrap_expect = 1
}

client {
  enabled = true
  servers = ["93.127.212.100"]
}

log_level = "INFO"
log_file  = "/var/log/nomad.log"
EOF
```

```console
sudo firewall-cmd --permanent --add-port=4646/tcp
sudo firewall-cmd --reload
```

```console
sudo systemctl enable nomad
sudo systemctl start nomad
```


### Installing Traefik

Since traefik needs to listen the podman.socker, it's better to install it with the podman user for a more easy set-up. This will be the only service that will share a user, all other services will have it's own user.



### Installing Coder

This is the final step, where we install Coder:

```console
$ curl -fsSL https://code-server.dev/install.sh | sudo sh
```
