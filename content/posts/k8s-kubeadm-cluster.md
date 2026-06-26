---
title: "kubeadm으로 쿠버네티스 멀티노드 클러스터 구축하기"
date: 2026-06-26T14:42:30+09:00
draft: false
tags: ["kubernetes", "kubeadm", "containerd", "ubuntu", "infra"]
categories: ["Infra"]
summary: "Ubuntu 22.04에서 kubeadm으로 마스터1 + 워커2 정석 쿠버네티스 클러스터를 구축한 기록. 명령어 그대로 복붙 가능."
---
 
microk8s 같은 간편 도구 말고, **kubeadm으로 정석 클러스터**를 직접 구축한 기록이다.
컨테이너 런타임(containerd)부터 CNI(Flannel)까지 손으로 다 올렸다.
 
## 환경
 
| 노드 | 호스트네임 | 역할 | IP |
|------|-----------|------|-----|
| 1 | `master`  | control-plane | 2.2.3.77 |
| 2 | `worker1` | worker | 2.2.3.78 |
| 3 | `worker2` | worker | 2.2.3.79 |
 
- OS: Ubuntu 22.04 LTS / Kubernetes v1.30
- 런타임: containerd / CNI: Flannel (`10.244.0.0/16`)
- 모든 명령은 root(`sudo su -`) 기준
---
 
## 0. 사전 준비 (모든 노드)
 
호스트네임 지정:
 
```bash
hostnamectl set-hostname master    # 각 노드에서 master / worker1 / worker2
```
 
`/etc/hosts`에 노드 등록 (세 노드 모두 동일):
 
```bash
cat <<EOF >> /etc/hosts
2.2.3.77   master
2.2.3.78   worker1
2.2.3.79   worker2
EOF
```
 
> ping이 되더라도 쿠버네티스는 노드를 **이름**으로 관리하므로 hosts 등록은 필수.
 
---
 
## 1. 공통 설치 (모든 노드)
 
### swap 끄기
 
쿠버네티스는 메모리를 정확히 계산해 Pod를 배치하므로 swap을 끈다.
 
```bash
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab
```
 
### 커널 모듈 + 네트워크 설정
 
```bash
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
modprobe overlay
modprobe br_netfilter
 
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sysctl --system
```
 
### containerd 설치
 
```bash
apt update
apt install -y ca-certificates curl gnupg
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  tee /etc/apt/sources.list.d/docker.list > /dev/null
 
apt update
apt install -y containerd.io
```
 
**핵심:** `SystemdCgroup = true`로 바꿔야 kubelet과 cgroup 드라이버가 맞아 노드가 안정적이다.
 
```bash
mkdir -p /etc/containerd
containerd config default | tee /etc/containerd/config.toml > /dev/null
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
systemctl restart containerd
systemctl enable containerd
```
 
### kubeadm / kubelet / kubectl 설치
 
```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | tee /etc/apt/sources.list.d/kubernetes.list
 
apt update
apt install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
systemctl enable --now kubelet
```
 
---
 
## 2. 마스터 초기화 (master만)
 
```bash
kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=2.2.3.77
```
 
성공하면 출력 맨 아래의 `kubeadm join ...` 명령을 **복사해 둔다.**
 
kubectl 설정:
 
```bash
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```
 
CNI(Flannel) 설치 — 이걸 해야 노드가 `NotReady` → `Ready`로 바뀐다.
 
```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```
 
---
 
## 3. 워커 합류 (worker1, worker2)
 
마스터에서 받은 join 명령을 각 워커에서 실행한다. (토큰이 만료됐으면 마스터에서 재발급)
 
```bash
# 마스터에서 join 명령 재발급
kubeadm token create --print-join-command
```
 
```bash
# 각 워커에서 실행 (예시)
kubeadm join 2.2.3.77:6443 --token <토큰> \
    --discovery-token-ca-cert-hash sha256:<해시>
```
 
---
 
## 4. 확인
 
```bash
kubectl get nodes
```
 
```
NAME      STATUS   ROLES           AGE     VERSION
master    Ready    control-plane   2m31s   v1.30.14
worker1   Ready    <none>          26s     v1.30.14
worker2   Ready    <none>          23s     v1.30.14
```
 
세 노드 모두 `Ready`면 완료. (워커의 `<none>` 역할은 정상)
 
---
 
## 트러블슈팅
 
- **노드가 계속 `NotReady`** → CNI(Flannel) 미설치
- **kubelet이 안 뜸** → swap 확인 (`free -h`에서 Swap 0이어야 함)
- **노드 불안정** → containerd `SystemdCgroup = true` 확인
- **join 토큰 만료(24h)** → `kubeadm token create --print-join-command` 재발급
---
 
## 마무리
 
도커 단일 컨테이너 → docker compose(단일 호스트 다중 컨테이너) → **쿠버네티스(다중 호스트 오케스트레이션)** 로 이어지는 흐름을 직접 손으로 밟아봤다.
다음은 이 클러스터 위에 deployment / service를 올려 Pod가 워커에 분산 배치되는 걸 확인할 차례.
