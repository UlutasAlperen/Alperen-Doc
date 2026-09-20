---
title: "kubernetes-scaling-horizontal"
weight: 12
---
# Horizontal Pod Autoscaling (HPA)

A [Horizontal Pod Autoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) can automatically scale the number of Pods in a Deployment based on observed CPU utilization or other custom metrics. It's very common in a Kubernetes environment to have a low number of pods in a deployment, and then scale up the number of pods automatically as CPU usage increases.

## Assignment

First, delete the `replicas: 1` line from the `synergychat-testcpu` deployment. This will allow our new autoscaler to have full control over the number of pods.

Create a new file called `testcpu-hpa.yaml`. Add the following YAML to it:

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: testcpu-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: x
  minReplicas: x
  maxReplicas: x
  targetCPUUtilizationPercentage: x
```

Set the following values:

- `name`: The name of the `synergychat-testcpu` deployment
- `minReplicas`: `1`
- `maxReplicas`: `4`
- `targetCPUUtilizationPercentage`: `50`

This hpa will monitor the CPU usage of the pods in the `synergychat-testcpu` deployment. Its goal is to scale up or down the number of pods in the deployment so that the average CPU usage of all pods is around 50%. As CPU usage increases, it will add more pods. As CPU usage decreases, it will remove pods. You can find the algorithm it uses [here](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#algorithm-details) if you're interested.

Apply the hpa, then run the following commands every few seconds to watch as the number of pods scales up:

```bash
kubectl get pods
kubectl top pods
```

An `hpa` is just another resource, so you can also use `kubectl get hpa` to see the current state of the autoscaler.

If `kubectl get hpa` shows `<unknown>` for CPU after a reboot, make sure your local cluster and metrics server are running again before debugging the YAML.

# HPA - Web

Now that you've seen how an application that chews through CPU will quickly scale up from a single pod to multiple pods, let's see what happens with an application that doesn't have much going on in terms of compute resources.

## Assignment

Delete the line "replicas: 3" from the `web` deployment. This will allow our new autoscaler to have full control over the number of pods.

Copy your `testcpu-hpa.yaml` file and call it `web-hpa.yaml`. Update the following values:

- `name: web-hpa`
- Target the "web" deployment
- Keep the scaling values the same

Apply the hpa, then use the following commands to see if any scaling happens:

```bash
kubectl get pods
kubectl top pods
```

```bash
NAME                                   READY   STATUS    RESTARTS      AGE
synergychat-api-74f6575bc8-hstcq       1/1     Running   1 (14h ago)   16h
synergychat-testcpu-69b77d7596-2gc5x   1/1     Running   0             8m33s
synergychat-testcpu-69b77d7596-9pqlq   1/1     Running   0             8m33s
synergychat-testcpu-69b77d7596-ntkr6   1/1     Running   0             9m51s
synergychat-testcpu-69b77d7596-wflhx   1/1     Running   0             10m
synergychat-testram-64bc6dbdd6-hzwdv   1/1     Running   0             3h56m
synergychat-web-5466578854-nsgmv       1/1     Running   0             119s
NAME                                   CPU(cores)   MEMORY(bytes)   
synergychat-api-74f6575bc8-hstcq       1m           5Mi             
synergychat-testcpu-69b77d7596-2gc5x   50m          9Mi             
synergychat-testcpu-69b77d7596-9pqlq   50m          9Mi             
synergychat-testcpu-69b77d7596-ntkr6   50m          9Mi             
synergychat-testcpu-69b77d7596-wflhx   50m          10Mi            
synergychat-testram-64bc6dbdd6-hzwdv   1m           7Mi             
synergychat-web-5466578854-nsgmv       1m           2Mi           
```

he reason the `web` deployment stabilizes at **1** pod is due to the way the Horizontal Pod Autoscaler (HPA) interprets resource usage versus your configuration.

When you create an HPA, you typically define a `minReplicas` value. In the previous exercises, we set that minimum to 1. The HPA's job is to look at the current resource consumption (like CPU utilization) and compare it against the target percentage you specified.

Here is why it scales down to 1:

1. **Low Resource Demand**: Unlike the `testcpu` application, which was specifically designed to hog CPU cycles, the `web` application is currently idle or handling very little traffic. It isn't "chewing through" resources.
2. **Target Thresholds**: Since the CPU usage is likely near 0%, it is well below the target percentage (e.g., 50%) you set in the HPA.
3. **Efficiency**: Kubernetes aims to be efficient. If one pod can easily handle the current load without exceeding the CPU target, the HPA will terminate the extra, unnecessary pods to save resources.
4. **Replicas Override**: By removing the `replicas: 3` line from the deployment, you handed over the "steering wheel" to the HPA. It saw that the load was low and immediately moved to the `minReplicas` floor you defined in the HPA manifest.

In a real-world scenario, this is exactly what you want: your cluster should shrink when nobody is using your site to save you money, and grow only when the traffic spikes!
for more [storage](kubernetes-storage/)
