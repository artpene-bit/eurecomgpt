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
49051070
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
| IaaS VM | <!--answer:vm_cold_ms-->247.7<!--/answer--> | <!--answer:vm_warm_median_ms-->212.3<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->1742.5<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->633.1<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->1731.1<!--/answer--> | <!--answer:function_warm_median_ms-->634.1<!--/answer--> |

Include your histogram or plot of the warm distribution below (an image, or the ASCII chart
`measure.py` prints). This is read by a human, not the grader:

<!--answer:latency_plot-->
Phase 1 — deployment measurements
------------------------------------------------------------
[OK  ] vm        cold=  247.7ms  warm_median=  212.3ms
            #############
[OK  ] cloudrun  cold= 1742.5ms  warm_median=  633.1ms
            ########################################
[OK  ] function  cold= 1731.1ms  warm_median=  634.1ms
            ########################################
------------------------------------------------------------
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
- VM: Doesn't scale at all. It's limited by the fixed CPU/RAM of the single 'e2-micro' instance I created.
- Cloud Run: Scales automatically by spinning up new containers, but it stops at the '--max-instances=3' limit I passed in the command.
- Cloud Function: Also scales automatically for each new request, limited only by Google's default project quotas.
<!--/answer-->

**Deployment effort.** What did you have to build, configure and maintain to get each of the
three running? (~100 words)

<!--answer:deployment_effort-->
- VM (IaaS) took the most effort. I had to build a custom VPC, subnets, firewall rules, fix IAM permissions, SSH in, and manually install Docker.
- Cloud Run (PaaS) was faster. I just had to write the 'Dockerfile', build the image, push it to Artifact Registry, and deploy it. No OS or network to manage.
- Cloud Function (FaaS) was the easiest. I just deployed the raw Python code. Google automatically built the container and pushed it to the registry for me.
<!--/answer-->

---

## 4. Why do two of your three numbers look alike?

With Cloud Run and your Function both deployed, run `gcloud run services list` and paste the
**complete output** here, unedited:

<!--answer:run_services_list-->
✔
SERVICE: echo
REGION: us-central1
URL: https://echo-211494103915.us-central1.run.app
LAST DEPLOYED BY: service-211494103915@gcf-admin-robot.iam.gserviceaccount.com
LAST DEPLOYED AT: 2026-11-08T21:48:34.355318Z

✔
SERVICE: echo-bot
REGION: us-central1
URL: https://echo-bot-211494103915.us-central1.run.app
LAST DEPLOYED BY: Arthur.Pene@eurecom.fr
LAST DEPLOYED AT: 2026-11-08T21:41:33.216159Z
<!--/answer-->

Now explain what that output tells you about how your Function is actually being run, why its
cold start resembles Cloud Run's so closely, and what — if anything — genuinely differs between
the two deployment paths. (~120 words)

<!--answer:convergence_explanation-->
The output shows both 'echo-bot' (my Cloud Run) and 'echo' (my Cloud Function) running as Cloud Run services. This means Cloud Functions gen 2 is just Cloud Run under the hood. Google takes my source code, automatically builds a container, and deploys it there. Since they run on the exact same underlying infrastructure, their cold starts and warm response times are almost identical. The only real difference is the developer experience: FaaS hides the Docker build process entirely, while PaaS forces me to maintain the 'Dockerfile' and manage the image registry myself.
<!--/answer-->
