## Online Boutique deployment file for Knative

Steps:

1. Install Knative on Cloudlab

2. Download this repo and follow the steps below to deploy online boutique
```
git clone https://github.com/ShixiongQi/spright-online-boutique-kn-deployment.git
cd spright-online-boutique-kn-deployment
python3 hack/setup.py

kubectl apply -f knative/
```

3. Load testing (with Locust)
```
# Install pip3
sudo apt update && sudo apt install -y python3-pip

# Download locust script
git clone https://github.com/ShixiongQi/spright-online-boutique-loadgenerator.git
cd spright-online-boutique-loadgenerator

# Install dependencies
pip3 install -r requirements.txt

# Install Locust
pip3 install locust

# Edit .bashrc and export the path to locust
export PATH="$HOME/.local/bin:$PATH"
source ~/.bashrc

# Use locust to generate traffic
ingressHost=$(kubectl get po -l istio=ingressgateway -n istio-system -o jsonpath='{.items[0].status.hostIP}')
ingressPort=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')

locust -u 1000 -r 100 -t 2m --headless -H http://$ingressHost:$ingressPort
```

Notes:

1. Tracing, Profiler, Stats are disabled by default

2. Use `h2c` for container ports, except frontend, as gRPC needs HTTP/2

3. Modify the Ingress IP and Port before applying

4. Check the correctness of the ksvc name