Deployment file for Knative

Notes:

1. Tracing, Profiler, Stats are disabled by default

2. Use `h2c` for container ports, except frontend, as gRPC needs HTTP/2

3. Modify the Ingress IP and Port before applying

4. Check the correctness of the ksvc name