packer {
  required_version = ">= 1.10.0"
  required_plugins {
    qemu = {
      source  = "hashicorp/qemu"
      version = ">= 1.1.0"
    }
  }
}

#######################################
# Globals / tunables
#######################################
variable "ovmf_code"   { default = "/usr/share/edk2/ovmf/OVMF_CODE.cc.fd" }
# If you truly reuse one VARS file, leave this. (Best practice is per-VM VARS files.)
variable "ovmf_vars"   { default = "/var/tmp/efivars.fd" }

# Your upstream infra & slirp settings (unchanged topology)
variable "lan_cidr"    { default = "172.16.100.0/24" }
variable "dns_ip"      { default = "172.16.100.10" }
variable "smb_ip"      { default = "172.16.100.11" }
variable "slirp_hostname" { default = "vendor-nat" } # replaces invalid 'host='

# Socket backhaul the second VM dials into (same as original)
variable "socket_host" { default = "localhost" }
variable "socket_port" { default = 58000 }

# VNC/monitor bindings (kept as you had them)
variable "vnc_bind"    { default = "0.0.0.0" }

# Communicator (so Packer/Ansible can SSH during the build)
variable "ssh_user"    { default = "ubuntu" }
variable "ssh_password" { default = "" }  # set if you use password auth
variable "ssh_private_key" { default = "" } # set if you use key auth, e.g. "~/.ssh/id_rsa"

locals {
  # choose password or key automatically
  use_key_auth = length(var.ssh_private_key) > 0
}

#######################################
# VM1 specifics (SLIRP owner)
#######################################
variable "vm1_disk" { default = "/var/tmp/vm.qcow2" }
variable "vm1_name" { default = "packer-vendor-vm1" }
variable "vm1_mac"  { default = "52:53:00:45:01:01" }
variable "vm1_ip"   { default = "172.16.100.22" }
variable "vm1_ssh_host_port" { default = 2223 }
variable "vm1_vnc_index"     { default = 2 }      # -> :02
variable "vm1_monitor_port"  { default = 55555 }

#######################################
# VM2 specifics (joins via socket)
#######################################
variable "vm2_disk" { default = "/var/tmp/vm2" }  # your original path
variable "vm2_name" { default = "packer-vendor-vm2" }
variable "vm2_mac"  { default = "52:54:00:49:02:01" }
variable "vm2_ip"   { default = "172.16.100.23" }
variable "vm2_ssh_host_port" { default = 2224 }
variable "vm2_vnc_index"     { default = 3 }      # -> :03
variable "vm2_monitor_port"  { default = 55558 }

# A tiny bit of guard-rail
variable "memory_mb" { default = 8192 }
variable "cpus"      { default = 2 }

validation "ports_distinct" {
  condition     = var.vm1_ssh_host_port != var.vm2_ssh_host_port
  error_message = "vm1_ssh_host_port and vm2_ssh_host_port must be different."
}

#######################################
# Source: VM1 (owns SLIRP and both hostfwds)
#######################################
source "qemu" "vm1" {
  # boot existing qcow2
  disk_image     = true
  iso_url        = var.vm1_disk
  format         = "qcow2"

  # firmware
  efi_boot          = true
  efi_firmware_code = var.ovmf_code
  efi_firmware_vars = var.ovmf_vars

  # hardware
  machine_type  = "q35"
  accelerator   = "kvm"
  cpus          = var.cpus
  memory        = var.memory_mb
  headless      = true
  disk_interface= "virtio"
  disk_cache    = "writeback"
  disk_discard  = "ignore"

  # SSH to VM1 via the hostfwd you define below
  communicator           = "ssh"
  ssh_username           = var.ssh_user
  ssh_host               = "127.0.0.1"
  ssh_port               = var.vm1_ssh_host_port
  ssh_timeout            = "30m"
  ssh_skip_nat_mapping   = true
  ssh_password           = local.use_key_auth ? null : var.ssh_password
  ssh_private_key_file   = local.use_key_auth ? var.ssh_private_key : null

  qemu_binary = "/usr/libexec/qemu-kvm"
  qemuargs = [
    # keep your -name and CPU model
    ["-name", var.vm1_name],
    ["-cpu",  "host"],

    # === network topology: SLIRP + hub + VM1 NIC + socket listener ===
    # (unchanged behavior; both hostfwds live on VM1's SLIRP)
    ["-netdev", "user,id=lan,restrict=on,ipv6=off,net=${var.lan_cidr},smbserver=${var.smb_ip},dns=${var.dns_ip},hostname=${var.slirp_hostname},hostfwd=tcp:127.0.0.1:${var.vm1_ssh_host_port}-${var.vm1_ip}:22,hostfwd=tcp:127.0.0.1:${var.vm2_ssh_host_port}-${var.vm2_ip}:22"],
    ["-netdev", "hubport,id=port-lan,hubid=0,netdev=lan"],
    ["-netdev", "hubport,id=port-nic0,hubid=0"],
    ["-device", "virtio-net,netdev=port-nic0,mac=${var.vm1_mac}"],

    ["-netdev", "socket,id=remote0,listen=${var.socket_host}:${var.socket_port}"],
    ["-netdev", "hubport,id=port-remote,hubid=0,netdev=remote0"],

    # === visibility as you had it ===
    ["-vnc",     "${var.vnc_bind}:${var.vm1_vnc_index}"],
    ["-monitor", "telnet:127.0.0.1:${var.vm1_monitor_port},server,nowait"]
  ]

  vm_name          = var.vm1_name
  output_directory = "output/${var.vm1_name}"
}

#######################################
# Source: VM2 (joins via socket; gets its own VNC/monitor)
#######################################
source "qemu" "vm2" {
  disk_image     = true
  iso_url        = var.vm2_disk
  format         = "qcow2"

  efi_boot          = true
  efi_firmware_code = var.ovmf_code
  efi_firmware_vars = var.ovmf_vars

  machine_type  = "q35"
  accelerator   = "kvm"
  cpus          = var.cpus
  memory        = var.memory_mb
  headless      = true
  disk_interface= "virtio"
  disk_cache    = "writeback"
  disk_discard  = "ignore"

  # SSH to VM2 via VM1's SLIRP forward (still 127.0.0.1:<vm2_ssh_host_port>)
  communicator           = "ssh"
  ssh_username           = var.ssh_user
  ssh_host               = "127.0.0.1"
  ssh_port               = var.vm2_ssh_host_port
  ssh_timeout            = "30m"
  ssh_skip_nat_mapping   = true
  ssh_password           = local.use_key_auth ? null : var.ssh_password
  ssh_private_key_file   = local.use_key_auth ? var.ssh_private_key : null

  qemu_binary = "/usr/libexec/qemu-kvm"
  qemuargs = [
    ["-name", var.vm2_name],
    ["-cpu",  "host"],

    # join the same L2 via the socket you already expose on VM1
    ["-netdev", "socket,id=nic0,connect=${var.socket_host}:${var.socket_port}"],
    ["-device", "virtio-net,netdev=nic0,mac=${var.vm2_mac}"],

    ["-vnc",     "${var.vnc_bind}:${var.vm2_vnc_index}"],
    ["-monitor", "telnet:127.0.0.1:${var.vm2_monitor_port},server,nowait"]
  ]

  vm_name          = var.vm2_name
  output_directory = "output/${var.vm2_name}"
}

#######################################
# Build
#######################################
build {
  name    = "vendor-two-vms"
  sources = ["source.qemu.vm1", "source.qemu.vm2"]

  # Example Ansible hooks (uncomment if you want Packer to run plays right away):
  # provisioner "ansible" {
  #   playbook_file = "playbooks/vm1.yml"
  #   use_proxy     = false
  #   only          = ["qemu.vm1"]
  # }
  # provisioner "ansible" {
  #   playbook_file = "playbooks/vm2.yml"
  #   use_proxy     = false
  #   only          = ["qemu.vm2"]
  # }
}
