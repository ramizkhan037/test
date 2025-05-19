# test

curl -sfL https://get.k3s.io | \
  INSTALL_K3S_EXEC="--disable traefik" sh -

# Make kubectl calls convenience alias
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
echo 'export KUBECONFIG=/etc/rancher/k3s/k3s.yaml' >> ~/.bashrc
