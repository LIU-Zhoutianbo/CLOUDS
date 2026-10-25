# Phase 1 — Comparison Report

## 1. Setup facts

**Your image size**, in MB, from `gcloud artifacts docker images list` (number only, e.g. `142`):

<!--answer:image_size_mb-->
49.99
<!--/answer-->

**The custom VPC and subnet you created** (names only):

<!--answer:vpc_name-->
echo-net
<!--/answer-->

<!--answer:subnet_name-->
echo-subnet
<!--/answer-->

---

## 2. Measurements

Copy these from `submission/phase1_report.json`. Cold is `cold_ms`; warm median is
`stats.median`. Numbers only, in milliseconds (e.g. `812.4`).

| Platform | Cold (ms) | Warm median (ms) |
|---|---|---|
| IaaS VM | <!--answer:vm_cold_ms-->8.4<!--/answer--> | <!--answer:vm_warm_median_ms-->2.5<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->1605.7<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->68.0<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->1864.1<!--/answer--> | <!--answer:function_warm_median_ms-->71.8<!--/answer--> |

<!--answer:latency_plot-->
Warm-request latency range plot (milliseconds; dots indicate medians):

```text
VM            2.1 ms ├─●────────┤ 3.7 ms     median 2.5 ms
Cloud Run    65.8 ms ├────●─────┤ 70.9 ms    median 68.0 ms
Cloud Function 67.1 ms ├────●─────────┤ 76.6 ms median 71.8 ms
```

The VM has the lowest and narrowest warm-latency range. Cloud Run and Cloud Function have much higher warm latency and wider ranges. The values shown are the observed minimum, median and maximum from the warm measurements in `submission/phase1_report.json`.
<!--/answer-->

---

## 3. Per-platform analysis

**Scaling behaviour.** Imagine requests arriving one at a time, then ten at once, then
a hundred. For each of the three platforms, answer concretely:

- Does anything get created automatically to absorb the load — and if so, *what* exactly?
  (One platform here creates nothing at all.)
- What is the hard limit, and where does it come from — a flag you passed yourself, or the
  size of the machine you picked?

(~80 words. Your `--machine-type=e2-micro` and `--max-instances=3` are both concrete answers
to the second question.)

<!--answer:scaling_behaviour-->
The IaaS VM does not create extra machines automatically: all requests share the single `e2-micro` VM, whose CPU and memory constrain throughput. Cloud Run can automatically start additional container instances as concurrent demand rises; the configured `--max-instances=3` caps the number of instances if that limit was applied to the deployment. The second-generation Cloud Function can also scale out by creating more function instances. Its scaling ceiling depends on its deployment configuration and platform limits. More simultaneous requests can therefore be handled by adding instances on the managed platforms, unlike the single VM.
<!--/answer-->

**Deployment effort.** What did you have to build, configure and maintain to get each of the three running? (~100 words)

<!--answer:deployment_effort-->
For the VM, I provisioned a Compute Engine `e2-micro` instance, configured the custom VPC and subnet, handled network/firewall access, installed or ran the application and its dependencies, and kept the service running and reachable. This gives direct control but leaves OS, process, patching and availability responsibilities to me. For Cloud Run, I built a container image, uploaded it to Artifact Registry and deployed the image as a managed service, configuring its access and scaling settings. For Cloud Functions, I deployed the function source through the Functions interface and configured its trigger and runtime. Google manages more of the infrastructure for both managed services, reducing server maintenance but adding platform-specific deployment configuration.
<!--/answer-->

---

## 4. Why do two of your three numbers look alike?

With Cloud Run and your Function both deployed, run `gcloud run services list` and paste the
**complete output** here, unedited:

<!--answer:run_services_list-->
```text
   SERVICE   REGION       URL                                                LAST DEPLOYED BY                                              LAST DEPLOYED AT
✔  echo      us-central1  https://echo-371522130732.us-central1.run.app      service-371522130732@gcf-admin-robot.iam.gserviceaccount.com  2026-10-25T10:04:44.371938Z
✔  echo-bot  us-central1  https://echo-bot-371522130732.us-central1.run.app  371522130732-compute@developer.gserviceaccount.com            2026-10-25T09:34:47.334976Z
```
<!--/answer-->

Now explain what that output tells you about how your Function is actually being run, why its
cold start resembles Cloud Run's so closely, and what — if anything — genuinely differs between
the two deployment paths. (~120 words)

<!--answer:convergence_explanation-->
The service list shows two Cloud Run services. The `echo` service was last deployed by a Google Cloud Functions service agent (`gcf-admin-robot`), while `echo-bot` was deployed by the Compute Engine default service account. This is consistent with the second-generation Cloud Function being backed by Cloud Run infrastructure: its function deployment is represented by a Cloud Run service. Both managed deployments therefore incur runtime provisioning and container or instance initialization, helping explain why their cold-start measurements are in the same order of magnitude (1,605.7 ms for Cloud Run and 1,864.1 ms for the Function). The deployment paths are still different: Cloud Run deploys a container image and exposes service configuration, whereas Cloud Functions deploys function source and manages the function-specific runtime and trigger. Similar latency does not mean the two deployments are identical.
<!--/answer-->
