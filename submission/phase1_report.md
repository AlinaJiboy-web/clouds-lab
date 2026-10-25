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

<!--answer:image_size_mb-->203<!--/answer-->

**The custom VPC and subnet you created** (names only):

<!--answer:vpc_name-->echo-net<!--/answer-->

<!--answer:subnet_name-->echo-subnet<!--/answer-->

---

## 2. Measurements

Copy these from `submission/phase1_report.json`. Cold is `cold_ms`; warm median is
`stats.median`. Numbers only, in milliseconds (e.g. `812.4`).

| Platform | Cold (ms) | Warm median (ms) |
|---|---|---|
| IaaS VM | <!--answer:vm_cold_ms-->249.4<!--/answer--> | <!--answer:vm_warm_median_ms-->212.0<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->145.5<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->635.5<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->647.0<!--/answer--> | <!--answer:function_warm_median_ms-->665.5<!--/answer--> |

Include your histogram or plot of the warm distribution below (an image, or the ASCII chart
`measure.py` prints). This is read by a human, not the grader:

<!--answer:latency_plot-->VM cold: 249.4ms, warm median: 212.0ms. Cloud Run cold: 145.5ms, warm median: 635.5ms. Cloud Function cold: 647.0ms, warm median: 665.5ms.<!--/answer-->

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

<!--answer:scaling_behaviour-->IaaS VM (e2-micro): No automatic scaling. Single Gunicorn process handles concurrency up to thread limit. Hard limit is the machine: e2-micro (1 vCPU, 1 GB RAM) set by --machine-type=e2-micro. Nothing created automatically. Cloud Run: Auto-scales from 0 upward per request. Hard limit is --max-instances=3 I passed at deploy. Cloud Function gen2: Same behaviour, runs on Cloud Run, same --max-instances=3 limit.<!--/answer-->

**Deployment effort.** What did you have to build, configure and maintain to get each of the
three running? (~100 words)

<!--answer:deployment_effort-->IaaS VM: Most effort. Created VPC echo-net, subnet echo-subnet, two firewall rules (tcp:8080, tcp:22). Provisioned e2-micro, SSHed in, authenticated Docker, started container manually. Cloud Run: Built multi-stage image, pushed to Artifact Registry, one gcloud run deploy command. No VM or network config. Cloud Function: Simplest. gcloud functions deploy from source. GCP builds and containerises automatically.<!--/answer-->

---

## 4. Why do two of your three numbers look alike?

With Cloud Run and your Function both deployed, run `gcloud run services list` and paste the
**complete output** here, unedited:

<!--answer:run_services_list-->SERVICE: echo      REGION: us-central1  LAST DEPLOYED BY: service-150736167244@gcf-admin-robot.iam.gserviceaccount.com
SERVICE: echo-bot  REGION: us-central1  LAST DEPLOYED BY: Alina.Jiboy@eurecom.fr<!--/answer-->

Now explain what that output tells you about how your Function is actually being run, why its
cold start resembles Cloud Run's so closely, and what — if anything — genuinely differs between
the two deployment paths. (~120 words)

<!--answer:convergence_explanation-->Cloud Run warm median (635.5ms) and Cloud Function warm median (665.5ms) are close because gen2 Cloud Functions run on Cloud Run infrastructure. The gcloud run services list shows both echo and echo-bot as Cloud Run services in us-central1, with the Function deployed by gcf-admin-robot. Once warm, latency is dominated by the same Flask application code path. The only difference is deployment: Cloud Run uses a pre-built image; Cloud Function deploys source code that GCP containerises.<!--/answer-->
