## Online Boutique deployment file for Knative

Steps:

1. Install Knative on Cloudlab (or anywhere based on your preference)

2. Download this repo and follow the steps below to deploy online boutique
```
git clone https://github.com/ShixiongQi/spright-online-boutique-kn-deployment.git
cd spright-online-boutique-kn-deployment
python3 hack/setup.py

kubectl apply -f knative/
```

3. Install Locust load generator
```
# Install pip3
sudo apt update && sudo apt install -y python3-pip

# Download locust script
git clone https://github.com/ShixiongQi/spright-online-boutique-loadgenerator.git
cd spright-online-boutique-loadgenerator/load-generator

# Install dependencies
pip3 install -r requirements.txt

# Install Locust
pip3 install locust

# Edit .bashrc and export the path to locust
echo export PATH="$HOME/.local/bin:$PATH" >> ~/.bashrc
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

---
## How to use Locust Load Generator
1. Basic usage (make sure you have the Locust script `locustfile.py` ready)
```bash
# move to the directory that has the Locust script
locust -u 1000 -r 100 -t 2m --headless -H http://$ingressHost:$ingressPort
```
- **-u**: Peak number of concurrent Locust users. Primarily used together with `--headless` or `--autostart`. Can be changed during a test by keyboard inputs w, W (spawn 1, 10 users) and s, S (stop 1, 10 users)
- **-r**:  Spawn rate. Rate to spawn users at (users per second). Primarily used together with `--headless` or `--autostart`.
- **-t**: Stop after the specified amount of time, e.g. (300s, 20m, 3h, 1h30m, etc.). Only used together with `--headless` or `--autostart`. Defaults to run forever.

2. Advanced usage (Distributed load generation).

A single process running Locust can simulate a throughput up to 600 ~ 900 RPS (depending on CPU's performance). But if your test plan is complex or you want to run even more load, you’ll need to scale out to multiple processes, maybe even multiple machines. To do this, you start one instance of Locust in master mode using the `--master` flag and multiple worker instances using the `--worker` flag. If the workers are not on the same machine as the master you use `--master-host` to point them to the IP/hostname of the machine running the master. The master instance runs Locust’s web interface, and tells the workers when to spawn/stop Users. The workers run your Users and send back statistics to the master. The master instance doesn’t run any Users itself. **Both the master and worker machines must have a copy of the locustfile when running Locust distributed.**

I prepared several scripts to automate the distributed load generation. To start a distributed load generation, run the following command. This will create 16 Locust worker processes and can generate a throughput up to 10000 RPS.
```
./run_load_generators.sh kn $ingressHost $ingressPort
```

Reference: https://docs.locust.io/en/stable/running-distributed.html

---
## Configuration of scaling policies
```yaml
    metadata:
      annotations:
        autoscaling.knative.dev/metric: rps
        autoscaling.knative.dev/minScale: "1"
        autoscaling.knative.dev/maxScale: "1"
        autoscaling.knative.dev/target: "20"
    spec:
```
1. The metric configuration defines which metric type is watched by the Autoscaler.
- **Annotation key**: autoscaling.knative.dev/metric
- **Possible values**: "concurrency" or "rps", depending on your Autoscaler type.
- **Default**: "concurrency"

2. A **target** provide the Autoscaler with a value that it tries to maintain for the configured metric for a function.

- **Annotation key**: autoscaling.knative.dev/target
- **Possible values**: An integer (metric agnostic).

Reference: https://knative.dev/docs/serving/autoscaling/

