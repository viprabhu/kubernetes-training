## Konfiguracja
- Nazwa hosta
    - debian
- Użytkownicy
    - nazwa: root, hasło: root
    - nazwa: administrator, hasło: administrator
- Karty sieciowe
    - 10.10.10.0/24 (Virtual NAT)
    - 192.168.56.0/24 (host-only)
- Zainstalowane pakiety
    - Server SSH
    - Podstawowe narzędzia systemowe
    
## Przygotowanie bazowej maszyny
- Usunąć cdrom jako źródło pakietów
```
nano /etc/apt/sources.list
```
- Zaktualizować system i zainstalować podstawowe narzędzia
```
apt update 
```
```
apt install sudo vim net-tools
```
- Dodać użytkownika administrator do grupy super użytkowników
```
usermod -aG sudo administrator
```
- Jeśli trzeba dodać i podnieść drugi interfejs sieciowy (enp0s8) 
```
echo auto enp0s8 >> /etc/network/interfaces
```
```
echo iface enp0s8 inet dhcp >> /etc/network/interfaces
```
```
ifup enp0s8
```
```
ip a
```
- Zainstalować Dockera https://docs.docker.com/v17.12/install/linux/docker-ce/debian
```
sudo apt-get install \
     apt-transport-https \
     ca-certificates \
     curl \
     gnupg2 \
     software-properties-common
```
```
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -
```
```
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/debian \
   $(lsb_release -cs) \
   stable"
```
```
sudo apt-get update
```
```
sudo apt-get install docker-ce
```
```
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
```
```
mkdir -p /etc/systemd/system/docker.service.d
```
```
sudo usermod -aG docker $USER
```
```
sudo apt-mark hold docker-ce
```
# Instalacja klastra - wersja uproszczona
- Sklonować maszynę bazową i nadać jej nazwę master
- Wyłączyć swap
```
swapoff -a
```
```
sed -i '/ swap / s/^/#/' /etc/fstab
```
- Zainstalować kubeadm
```
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
```
```
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
```
```
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
```
```
sudo apt-get update
```
```
sudo apt-get install -y kubelet kubeadm kubectl
```
```
sudo apt-mark hold kubelet kubeadm kubectl
```
- Skonfigurować stałe adresy ip
```
sudo nano /etc/hosts
```
```
10.10.10.10 master    
10.10.10.11 node1    
10.10.10.12 node2    
10.10.10.100 admin
```
- Wyłączyć dhcp na podstawowym interfejsie sieciowym
```
sudo nano /etc/network/interfaces
```
```
# iface enp0s3 inet dhcp
```
- Ustawić adres ip dla pierwszej maszyny (master)
```
iface enp0s3 inet static
      address 10.10.10.10
      netmask 255.255.255.0
      gateway 10.10.10.1
```
- Ustawić nazwę hosta dla pierwszej maszyny (master)
```
sudo hostnamectl set-hostname master
```
- Zrestartować maszynę
```
sudo shutdown -r now
```
- Sklonować maszynę 2 razy i zmienić adresy oraz nazwy odpowiednio dla node1 i node2
- Zainicjalizować klaster (master)
```
sudo kubeadm init
```
- Dołączyć pozostałe maszyny (node1, node2) postępując zgodnie z wyświetlaną instrukcją
- Zainstalować warstwę sieciową
```
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```
- Umożliwić logowanie użytkownika root na master
```
nano /etc/ssh/sshd_config  (ustawić PermitRootLogin yes)
```
```
/etc/init.d/ssh restart
```
- Sklonować maszynę testową i utworzyć stację administracyjną
```
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
```
```
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
```
```
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
```
```
sudo apt-get update
```
```
sudo apt-get install -y kubectl
```
```
mkdir ~/.kube
```
```
scp root@10.10.10.10:/etc/kubernetes/admin.conf ~/.kube/local
```
```
echo export KUBECONFIG=~/.kube/local >> ~/.bash_profile
```
- Skonfigurować bash completion
```
echo "source <(kubectl completion bash)" >> ~/.bashrc
```
```
echo if [ -f "$HOME/.bashrc" ]; then . "$HOME/.bashrc" fi
```
```
source .bashrc
```