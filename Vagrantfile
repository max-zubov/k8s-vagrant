# Setup kubernetes cluster


Vagrant.configure("2") do |config|
    
    [
        { 
            :hostname => 'master', 
            :ip => '192.168.33.1', 
            :box => 'bento/centos-8.2',
            :cpus => "2",
            :ram => "2048",
            # only one master when initializing the cluster using kubeadm
            :count => 1             
        },
        { 
            :hostname => 'minion', 
            :ip => '192.168.33.2',
            :box => 'bento/centos-8.2',
            :cpus => "2",
            :ram => "2048", 
            :count => 1             
        }
    ].each do |node|
      (1..node[:count]).each do |i|    
        config.vm.define "#{node[:hostname]}-#{i}" do |nodeconfig|
            nodeconfig.vm.box = node[:box]
            nodeconfig.vm.hostname = "#{node[:hostname]}-#{i}"
            nodeconfig.vm.network :private_network, ip: "#{node[:ip]}#{i}"
            nodeconfig.ssh.insert_key = false
            nodeconfig.vm.provider :virtualbox do |vb|
                vb.name = "#{node[:hostname]}-#{i}"
                vb.cpus = node[:cpus] ? node[:cpus].to_s : "2"
                vb.memory = node[:ram] ? node[:ram].to_s : "2048"
                vb.gui = false
            end
            nodeconfig.vm.provision "shell", inline: <<-'SCRIPT'
                setenforce 0
                sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
                modprobe br_netfilter
                { echo 'net.bridge.bridge-nf-call-ip6tables = 1'; \
                  echo 'net.bridge.bridge-nf-call-iptables = 1'; \
                } > /etc/sysctl.d/k8s.conf
                sysctl --system
                dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
                dnf -y install https://download.docker.com/linux/centos/7/x86_64/stable/Packages/containerd.io-1.2.6-3.3.el7.x86_64.rpm
                dnf -y  install docker-ce
                mkdir -p /etc/docker
                echo '{  "exec-opts": ["native.cgroupdriver=systemd"], "log-driver": "journald", "storage-driver": "overlay2" }' \
                      > /etc/docker/daemon.json
                systemctl enable docker
                systemctl start docker
                { echo '[kubernetes]'; \
                  echo 'name=Kubernetes'; \
                  echo 'baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64'; \
                  echo 'enabled=1'; \
                  echo 'gpgcheck=1'; \
                  echo 'repo_gpgcheck=1'; \
                  echo 'gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg'; \
                } > /etc/yum.repos.d/kubernetes.repo
                dnf -y  install kubeadm
                swapoff -a
                sed -i '/swap/ s/^/#/' /etc/fstab
                echo "KUBELET_EXTRA_ARGS=--node-ip=$(hostname -I | awk '{print $2}') --cgroup-driver=systemd" > /etc/sysconfig/kubelet
                systemctl enable kubelet
                systemctl start kubelet
                if [[ $(hostname -s) == 'master-1' ]]; then
                    kubeadm init --apiserver-advertise-address=$(hostname -I | awk '{print $2}') --pod-network-cidr=10.244.0.0/16 | \
                    sed '/^kubeadm join/ {/\\$/N;s/\\\n//}' | tee /vagrant/.init_out
                    mkdir /home/vagrant/.kube
                    cp -i /etc/kubernetes/admin.conf /home/vagrant/.kube/config
                    chown vagrant:vagrant /home/vagrant/.kube/config
                    su vagrant -c 'kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml'
                    dnf -y install bash-completion
                    su vagrant -c 'source <(kubectl completion bash)'
                    su vagrant -c 'echo "source <(kubectl completion bash)" >>~/.bashrc'
                else
                    grep 'kubeadm join' /vagrant/.init_out | bash
                fi                
            SCRIPT
        end
      end
    end
end
