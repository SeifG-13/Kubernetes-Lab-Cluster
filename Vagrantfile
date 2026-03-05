# -*- mode: ruby -*-
# vi: set ft=ruby :
#
# Simple 3-VM Kubernetes cluster
#
#   worker1  192.168.56.30  Worker   2GB  2CPU  (started first)
#   worker2  192.168.56.31  Worker   2GB  2CPU  (started second)
#   master1  192.168.56.20  Control plane + Ansible control node  2GB  2CPU  (last)
#
# master1 is defined LAST so workers are already running when
# master1's provisioner runs ssh-copy-id to distribute its key.

Vagrant.configure("2") do |config|
  config.vm.box = "bento/ubuntu-22.04"
  config.vm.box_check_update = false
  config.vm.boot_timeout = 300

  # Fix /vagrant permissions — VirtualBox mounts it world-writable by default,
  # which causes Ansible to refuse loading ansible.cfg from that directory.
  # dmode=755 = rwxr-xr-x on dirs, fmode=644 = rw-r--r-- on files
  config.vm.synced_folder ".", "/vagrant",
    mount_options: ["dmode=755", "fmode=644"]

  # ── Shared: /etc/hosts + enable password SSH on all VMs ────────────────────
  # WHY enable PasswordAuthentication:
  #   master1 will use ssh-copy-id to push its public key to workers.
  #   ssh-copy-id needs password auth available for the initial key exchange.
  #   After Ansible runs, we can disable it if desired.
  config.vm.provision "shell", inline: <<-SHELL
    set -e
    if ! grep -q 'k8s-hosts' /etc/hosts; then
      cat >> /etc/hosts <<'EOF'
# k8s-hosts
192.168.56.20  master1
192.168.56.30  worker1
192.168.56.31  worker2
EOF
    fi
    hostnamectl set-hostname "$(hostname)"

    # Enable SSH password authentication (needed for initial key exchange)
    sed -i 's/^#*PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config
    systemctl restart ssh
    echo "==> Common setup done on $(hostname)"
  SHELL

  # ── worker1 (defined first — boots before master1) ──────────────────────────
  config.vm.define "worker1" do |node|
    node.vm.hostname = "worker1"
    node.vm.network "private_network", ip: "192.168.56.30"
    node.vm.provider "virtualbox" do |vb|
      vb.name   = "k8s-worker1"
      vb.memory = 2048; vb.cpus = 2; vb.gui = false
      vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
      vb.customize ["modifyvm", :id, "--usb", "off"]
      vb.customize ["modifyvm", :id, "--usbehci", "off"]
    end
  end

  # ── worker2 ─────────────────────────────────────────────────────────────────
  config.vm.define "worker2" do |node|
    node.vm.hostname = "worker2"
    node.vm.network "private_network", ip: "192.168.56.31"
    node.vm.provider "virtualbox" do |vb|
      vb.name   = "k8s-worker2"
      vb.memory = 2048; vb.cpus = 2; vb.gui = false
      vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
      vb.customize ["modifyvm", :id, "--usb", "off"]
      vb.customize ["modifyvm", :id, "--usbehci", "off"]
    end
  end

  # ── master1 (defined LAST — workers are already up when this provisioner runs)
  config.vm.define "master1" do |node|
    node.vm.hostname = "master1"
    node.vm.network "private_network", ip: "192.168.56.20"
    node.vm.provider "virtualbox" do |vb|
      vb.name   = "k8s-master1"
      vb.memory = 2048; vb.cpus = 2; vb.gui = false
      vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
      vb.customize ["modifyvm", :id, "--usb", "off"]
      vb.customize ["modifyvm", :id, "--usbehci", "off"]
    end

    node.vm.provision "shell", privileged: false, inline: <<-SHELL
      set -e
      echo "==> Installing Ansible + sshpass on master1..."
      sudo apt-get update -q
      sudo apt-get install -y software-properties-common sshpass
      sudo add-apt-repository -y ppa:ansible/ansible
      sudo apt-get update -q
      sudo apt-get install -y ansible

      echo "==> Generating SSH key pair..."
      ssh-keygen -t ed25519 -f ~/.ssh/k8s_key -N "" -C "k8s-lab" <<< y || true

      echo "==> Distributing public key to workers..."
      for host in worker1 worker2; do
        sshpass -p vagrant ssh-copy-id \
          -i ~/.ssh/k8s_key.pub \
          -o StrictHostKeyChecking=no \
          vagrant@$host
        echo "  Key copied to $host"
      done

      # Also add to master1 itself (so Ansible can run localhost tasks via SSH)
      cat ~/.ssh/k8s_key.pub >> ~/.ssh/authorized_keys
      chmod 600 ~/.ssh/authorized_keys

      # SSH client config
      cat > ~/.ssh/config <<'SSHEOF'
Host master1 worker1 worker2 192.168.56.*
    User vagrant
    IdentityFile ~/.ssh/k8s_key
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null
SSHEOF
      chmod 600 ~/.ssh/config

      echo "==> Ansible version: $(ansible --version | head -1)"
      echo "==> SSH test to workers:"
      ssh -i ~/.ssh/k8s_key worker1 hostname
      ssh -i ~/.ssh/k8s_key worker2 hostname
      # Add ANSIBLE_CONFIG to .bashrc so it's always set on login
      echo 'export ANSIBLE_CONFIG=/vagrant/ansible/ansible.cfg' >> /home/vagrant/.bashrc

      echo ""
      echo "==> master1 bootstrap complete."
      echo "    Run the cluster setup:"
      echo "      export ANSIBLE_CONFIG=/vagrant/ansible/ansible.cfg"
      echo "      cd /vagrant/ansible"
      echo "      ansible-playbook -i inventory.ini site.yml"
    SHELL
  end
end
