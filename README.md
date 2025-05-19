# test

curl -sfL https://get.k3s.io | \
  INSTALL_K3S_EXEC="--disable traefik" sh -

# Make kubectl calls convenience alias
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
echo 'export KUBECONFIG=/etc/rancher/k3s/k3s.yaml' >> ~/.bashrc


mkdir -p ~/.kube

curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version

 Deploy MySQL Primary + 2 Replicas
2.1 Create a “db” namespace

kubectl create namespace db

2.2 Create your Helm override file

cat > ~/mysql-values.yaml << 'EOF'
architecture: replication
primary:
  persistence:
    enabled: true
    size: 8Gi
secondary:
  replicaCount: 2
  persistence:
    enabled: true
    size: 8Gi
auth:
  rootPassword: my-secret-root-pw
  database: clientlogdb
  username: logger
  password: logger-pw
EOF

2.3 Add Bitnami repo & install

helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

POD=$(kubectl -n db get pods -l app.kubernetes.io/component=primary -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n db $POD -- \
  mysql -uroot -pmy-secret-root-pw \
  -e "SHOW SLAVE HOSTS\G"

helm install mysql-cluster bitnami/mysql \
  --namespace db \
  -f ~/mysql-values.yaml

2.4 Confirm it’s running

kubectl -n db get pods
kubectl -n db get pvc

Wait until 1 “primary” + 2 “secondary” pods are Running and their PVCs are Bound.
2.5 Validate replication

kubectl exec -n db svc/mysql-cluster -it -- \
  mysql -uroot -pmy-secret-root-pw \
  -e "SHOW SLAVE HOSTS\G"

You should see two replicas listed.
3. Build & Push Your REST API Image

You can do this either on EC2 (since you installed Docker) or locally—just ensure your EC2 node can pull it.
3.1 Prepare your API directory

On EC2, for example:

mkdir -p ~/api && cd ~/api

Create server.py:

cat > server.py << 'EOF'
from flask import Flask, request, jsonify
import os, mysql.connector

app = Flask(__name__)

DB_HOST = os.getenv('DB_HOST')
DB_USER = os.getenv('DB_USER')
DB_PASS = os.getenv('DB_PASS')
DB_NAME = os.getenv('DB_NAME')

def get_db():
    return mysql.connector.connect(
        host=DB_HOST, user=DB_USER, password=DB_PASS, database=DB_NAME
    )

@app.route('/log-ip', methods=['POST'])
def log_ip():
    ip = request.remote_addr
    conn = get_db(); cur = conn.cursor()
    cur.execute("""
      CREATE TABLE IF NOT EXISTS ip_logs (
        id INT AUTO_INCREMENT PRIMARY KEY,
        ip VARCHAR(45),
        logged_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
      )
    """)
    cur.execute("INSERT INTO ip_logs (ip) VALUES (%s)", (ip,))
    conn.commit(); cur.close(); conn.close()
    return jsonify(status="ok", ip=ip), 201

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF

Create requirements.txt:

cat > requirements.txt << 'EOF'
Flask==2.2.5
mysql-connector-python==8.1.0
EOF

Create Dockerfile:

cat > Dockerfile << 'EOF'
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY server.py .
EXPOSE 5000
CMD ["python", "server.py"]
EOF

3.2 Build & Push

# log into Docker Hub if needed:
docker login

# build & tag
docker build -t YOUR_DOCKERHUB_USER/ip-logger:latest .

# push to Docker Hub
docker push YOUR_DOCKERHUB_USER/ip-logger:latest

Replace YOUR_DOCKERHUB_USER with your actual Docker Hub ID.
4. Deploy the API in k3s
4.1 Create “web” namespace

kubectl create namespace web

4.2 Create Kubernetes manifests

mkdir -p ~/k8s-manifests/web && cd ~/k8s-manifests/web

deployment.yaml:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: ip-logger
  namespace: web
spec:
  replicas: 1
  selector:
    matchLabels: { app: ip-logger }
  template:
    metadata: { labels: { app: ip-logger } }
    spec:
      containers:
      - name: api
        image: YOUR_DOCKERHUB_USER/ip-logger:latest
        env:
        - name: DB_HOST   ; value: mysql-cluster.db.svc.cluster.local
        - name: DB_USER   ; value: logger
        - name: DB_PASS   ; value: logger-pw
        - name: DB_NAME   ; value: clientlogdb
        ports: [{ containerPort: 5000 }]

service.yaml:

apiVersion: v1
kind: Service
metadata:
  name: ip-logger
  namespace: web
spec:
  type: NodePort
  selector: { app: ip-logger }
  ports:
  - port: 5000
    targetPort: 5000
    nodePort: 30080

4.3 Apply manifests

kubectl apply -f deployment.yaml
kubectl apply -f service.yaml

Check:

kubectl -n web get pods
kubectl -n web get svc

You should see your pod Running and the service exposing port 30080.
5. Final Testing

    MySQL replication (once more):

kubectl exec -n db svc/mysql-cluster -it -- \
  mysql -uroot -pmy-secret-root-pw \
  -e "SHOW SLAVE STATUS\G"

API endpoint (from your laptop or any host that can reach EC2):

curl -X POST http://<EC2_PUBLIC_IP>:30080/log-ip

You should get back:

{"status":"ok","ip":"<YOUR_CLIENT_IP>"}
