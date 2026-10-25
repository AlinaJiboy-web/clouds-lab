# Phase 1 — Comparison Report

## 1. Setup facts

Image size (MB):
<!--answer:image_size_mb-->
203
<!--answer:image_size_mb-->

VPC name:
<!--answer:vpc_name-->
echo-net
<!--answer:vpc_name-->

Subnet name:
<!--answer:subnet_name-->
echo-subnet
<!--answer:subnet_name-->

## 2. Measurements

| Platform      | Cold (ms) | Warm median (ms) |
|---------------|-----------|-----------------|
| IaaS VM       | 249.4     | 212.0           |
| Cloud Run     | 145.5     | 635.5           |
| Cloud Function| 647.0     | 665.5           |

<!--answer:vm_cold_ms-->
249.4
<!--answer:vm_cold_ms-->

<!--answer:vm_warm_median_ms-->
212.0
<!--answer:vm_warm_median_ms-->

<!--answer:cloudrun_cold_ms-->
145.5
<!--answer:cloudrun_cold_ms-->

<!--answer:cloudrun_warm_median_ms-->
635.5
<!--answer:cloudrun_warm_median_ms-->

<!--answer:function_cold_ms-->
647.0
<!--answer:function_cold_ms-->

<!--answer:function_warm_median_ms-->
665.5
<!--answer:function_warm_median_ms-->

<!--answer:warm_distribution-->
Cloud Run warm latency (20 requests): median 635.5 ms
Cloud Function warm latency (20 requests): median 665.5 ms
IaaS VM warm latency (20 requests): median 212.0 ms, cold: 249.4 ms
<!--answer:warm_distribution-->

## 3. Per-platform analysis

<!--answer:scaling-->
IaaS VM (e2-micro): No automatic scaling occurs. The VM runs a single Gunicorn process; concurrency is absorbed by its threads. The hard limit is the machine itself — e2-micro (1 vCPU, 1 GB RAM) chosen with --machine-type=e2-micro. Nothing is created automatically; traffic beyond thread capacity queues or drops.

Cloud Run: Automatically scales container instances from 0 upward as load grows. New instances start without manual action. The hard limit is --max-instances=3, a flag I passed at deploy time.

Cloud Function gen2: Identical behaviour — it runs on Cloud Run infrastructure and auto-scales up to --max-instances=3, the same flag passed at deploy time.
<!--answer:scaling-->

<!--answer:deployment_effort-->
IaaS VM required the most effort: created a custom VPC (echo-net), subnet (echo-subnet), and two firewall rules (tcp:8080 and tcp:22). Provisioned an e2-micro VM, SSHed in manually, authenticated Docker, and started the container by hand.

Cloud Run: Built a multi-stage Docker image, pushed to Artifact Registry, then one gcloud run deploy command handled the rest. No VM or network config needed.

Cloud Function gen2: Simplest — gcloud functions deploy from source. GCP builds and containerises automatically; no Dockerfile, no VM, no network setup required.
<!--answer:deployment_effort-->

## 4. Why do two of your three numbers look alike?

<!--answer:gcloud_run_services_list-->
SERVICE: echo    REGION: us-central1  URL: https://echo-150736167244.us-central1.run.app  LAST DEPLOYED BY: service-150736167244@gcf-admin-robot.iam.gserviceaccount.com
SERVICE: echo-bot  REGION: us-central1  URL: https://echo-bot-150736167244.us-central1.run.app  LAST DEPLOYED BY: Alina.Jiboy@eurecom.fr
<!--answer:gcloud_run_services_list-->

<!--answer:why_similar-->
Cloud Run warm median (635.5 ms) and Cloud Function gen2 warm median (665.5 ms) are close because Cloud Functions gen2 runs on Cloud Run infrastructure. The gcloud run services list output confirms this: both echo-bot (explicit Cloud Run service) and echo (the Cloud Function) appear as Cloud Run services in us-central1, with the Function deployed by the gcf-admin-robot service account. Once a container instance is warm, latency is dominated by the application's own execution — parsing the query, building the JSON response — which follows the same code path in both cases. The infrastructure layer is transparent, so the two warm medians land within 30 ms of each other. The only genuine difference is the deployment path: Cloud Run deploys a pre-built image, Cloud Function deploys source code that GCP containerises automatically.
<!--answer:why_similar-->
