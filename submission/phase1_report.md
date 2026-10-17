# Phase 1 — Comparison Report

Copy this file to `submission/phase1_report.md`, fill in every answer slot, and commit it:

```bash
cp phase-1-echo-bot/report_template.md submission/phase1_report.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or images freely.

Numbers must **match `submission/phase1_report.json`** exactly — that file is produced by
`measure.py` and is what your figures are checked against. Copy them across; do not retype from
memory.

---

## 1. Setup facts

**Your image size**, in MB, from `gcloud artifacts docker images list` (number only, e.g. `142`):

<!--answer:image_size_mb-->
78
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
| IaaS VM | <!--answer:vm_cold_ms-->
145.7
<!--/answer--> | <!--answer:vm_warm_median_ms-->
n/a
<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->
145.5
<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->
635.5
<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->
647.0
<!--/answer--> | <!--answer:function_warm_median_ms-->
665.5
<!--/answer--> |

Include your histogram or plot of the warm distribution below (an image, or the ASCII chart
`measure.py` prints). This is read by a human, not the grader:

<!--answer:latency_plot-->
Cloud Run warm latency (20 requests): median 635.5 ms
Cloud Function warm latency (20 requests): median 665.5 ms
IaaS VM: warm measurement failed (cold only: 145.7 ms)
<!--/answer-->

---

## 3. Per-platform analysis

**Scaling behaviour.** Imagine requests arriving one at a time, then ten at once, then a
hundred. For each of the three platforms, answer concretely:

- Does anything get created automatically to absorb the load — and if so, *what* exactly?
  (One platform here creates nothing at all.)
- What is the hard limit, and where does it come from — a flag you passed yourself, or the
  size of the machine you picked?

(~80 words. Your `--machine-type=e2-micro` and `--max-instances=3` are both concrete answers
to the second question.)

<!--answer:scaling_behaviour-->
IaaS VM (e2-micro): No automatic scaling occurs. The VM runs a single Gunicorn process with 8 threads; concurrency is absorbed by those threads until exhausted. The hard limit is the machine itself — e2-micro (1 vCPU, 1 GB RAM) chosen with --machine-type=e2-micro at creation. Nothing is created automatically; traffic beyond thread capacity queues or drops.

Cloud Run: Automatically scales container instances from 0 upward as load grows. New instances start without any manual action. The hard limit is --max-instances=3, a flag I passed at deploy time.

Cloud Function gen2: Identical behaviour — it runs on Cloud Run infrastructure and auto-scales up to --max-instances=3, the same flag I passed at deploy time.
<!--/answer-->

**Deployment effort.** What did you have to build, configure and maintain to get each of the
three running? (~100 words)

<!--answer:deployment_effort-->
IaaS VM required the most effort: created a custom VPC (echo-net), subnet (echo-subnet, 10.0.0.0/24, us-central1), and two firewall rules (echo-allow-http on tcp:8080, echo-allow-ssh on tcp:22). Provisioned an e2-micro VM with a startup script, then had to SSH in manually, authenticate Docker with sudo gcloud auth configure-docker, and start the container by hand. Ongoing maintenance means manually restarting if the container crashes.

Cloud Run: Built a multi-stage Docker image, pushed it to Artifact Registry, then one gcloud run deploy command handled the rest. No VM or network config needed.

Cloud Function gen2: Simplest of all — gcloud functions deploy from source. GCP builds and containerises the code automatically; no Dockerfile, no VM, no network setup required.
<!--/answer-->

---

## 4. Why do two of your three numbers look alike?

With Cloud Run and your Function both deployed, run `gcloud run services list` and paste the
**complete output** here, unedited:

<!--answer:run_services_list-->
✓
SERVICE: echo
REGION: us-central1
URL: https://echo-150736167244.us-central1.run.app
LAST DEPLOYED BY: service-150736167244@gcf-admin-robot.iam.gserviceaccount.com
LAST DEPLOYED AT: 2026-10-17T13:11:59.433742Z

✓
SERVICE: echo-bot
REGION: us-central1
URL: https://echo-bot-150736167244.us-central1.run.app
LAST DEPLOYED BY: Alina.Jiboy@eurecom.fr
LAST DEPLOYED AT: 2026-10-17T13:04:53.320765Z
<!--/answer-->

Now explain what that output tells you about how your Function is actually being run, why its
cold start resembles Cloud Run's so closely, and what — if anything — genuinely differs between
the two deployment paths. (~120 words)

<!--answer:convergence_explanation-->
The two similar numbers are the warm-request medians: Cloud Run at 635.5 ms and Cloud Function gen2 at 665.5 ms. They converge because Cloud Functions gen2 is built directly on Cloud Run infrastructure — the gcloud run services list output confirms this: both echo-bot (the explicit Cloud Run service) and echo (the Cloud Function) appear as Cloud Run services in us-central1, the Function deployed by the gcf-admin-robot service account. Both run the identical Flask echo-bot image through Gunicorn. Once a container instance is warm, request latency is dominated by the application's own execution — parsing the query, building the JSON response, and returning it — which follows the exact same code path in both cases. The infrastructure layer is transparent, so the two warm medians naturally land within 30 ms of each other.
<!--/answer-->
