# GoogleCloud_Workstation_Setup_Guidance

This is a guidance on how to set up google cloud linux workstation with GPU and remote using Sunshine/Moonlight and deploy NTU Mecatron software stack on this workstation. 

## 1. Create a Google Cloud account

To create a workstation, first you have to sign up for a Gmail account and use that Gmail to create a [Google Cloud](https://cloud.google.com/?e=48754805&utm_source=google&utm_medium=cpc&utm_campaign=Cloud-SS-DR-GCP-1713664-GCP-DR-APAC-TW-zh-Google-BKWS-MIX-GenericCloud&utm_content=c-Hybrid+%7C+BKWS+-+BRO+%7C+Txt+-+Generic+Cloud-Cloud+Generic-Core+GCP-TW_en-6458750523&utm_term=google%20cloud&gclsrc=aw.ds&gad_source=1&gad_campaignid=19506976549&gclid=Cj0KCQjwnbrUBhDOARIsAKKhPpcEr2HJoKauwqwlADN-gvgZCSfG1-QNP1aUzgCUpNGQoQ443WOVBFoaAn0YEALw_wcB). 

For first time user, you will get 300 USD dollar in credits (which is a lot).

## 2. Create First Project

After getting inside the Google Cloud, you have to create a project (or it is automatially created on first time sign in). Then you have to navigate to the Cloud Shell, basically another terminal, to proceed with our setup. You can open this by clicking the shell icon on top right corner of the window, or you can just click G + S. 

When creating a workstation with GPU capability, we need to care about what type of machines we are going to use, how much persistent disk storage we are going too allocated, and the OS.

- To check the list of registered server regions and what types of machines available for each regions, please follow this [link](https://docs.cloud.google.com/compute/docs/regions-zones#available). 
- To check the list of machines, please follow this [link](https://docs.cloud.google.com/compute/docs/machine-resource).
- Google Cloud supports Ubuntu 22.04 and 24.04 and Window 10.

For this guidance, I use g2-standard-8 (8 vCPUs, 32 GB Memory), 1 x NVIDIA L4 Virtual Workstation, 100 GB persistent storage, and OS ubuntu-2204-jammy. You can adapt this to your purpose but issues arising from doing so are unknown and unchecked. 

### 2.1. Create a Workstation Instance

Firstly, you have to check what are the available computing servers. Ideally pick ones close to you physically. To do so, type this in the Cloud Shell:
```bash
gcloud compute accelerator-types list
```

If you already have in mind a specific region and you just want to see the available machines, type this: 
```bash
gcloud compute accelerator-types list --filter="zone:(your-zone-name)"
```
Example: 
```bash
gcloud compute accelerator-types list --filter="zone:(asia-southeast1-b)"
```

After finding your desired zone, config your compute/zone:
```bash
gcloud config set compute/zone your-zone-name
```

Create Workstation Instance. I name my workstation as test-workstation, but you can change if you want: 
```bash
gcloud compute instances create test-workstation     --zone=your-zone-name     --machine-type=g2-standard-8     --accelerator=type=nvidia-l4-vws,count=1     --maintenance-policy="TERMINATE"     --image-project=ubuntu-os-cloud     --image-family=ubuntu-2204-lts     --boot-disk-size=100     --boot-disk-type=pd-ssd     --network=default
```

If you create successfully, you can access the workstation using this cmd:
```bash
gcloud compute ssh test-workstation
```

#### 2.1.1 A possible issue
An issue you might face while creating a Google Cloud workstation instance:
```bash
 https://docs.cloud.google.com/compute/docs/virtual-workstation/linux-gpu

Error:

ERROR: (gcloud.compute.instances.create) Could not fetch resource:

 - Quota 'GPUS_ALL_REGIONS' exceeded.  Limit: 0.0 globally.

        metric name = compute.googleapis.com/gpus_all_regions

        limit name = GPUS-ALL-REGIONS-per-project

        limit = 0.0

        dimensions = global: global

Try your request in another zone, or view documentation on how to increase quotas: https://cloud.google.com/compute/quotas. 
```

#### 2.1.2. The cause
By default, new Google Cloud projects start with a GPU quota limit of 0.0 (both globally and regionally) to prevent accidental high-cost usage or abuse. To build your Linux virtual workstation using GPUs, you must manually request a quota increase. You have to upgrade your account from Free Trial to Full Account. Don't worry, Google Cloud will deduct your existing free credits first before touching your own money. 

#### 2.1.3. A solution
Follow these steps in the Google Cloud Console to request more quota [cite: 1.1.3]:
- Open the Google Cloud Console.
- Navigate to IAM & Admin > Quotas (or search for "Quotas" in the top search bar).
- At the top filter bar, clear existing filters if needed, and look for the Filter box. Enter GPUS_ALL_REGIONS and press Enter.(Note: Depending on the specific guide you are following, you may also need to check for specific regional GPU metrics like NVIDIA_T4_GPUS or NVIDIA_L4_GPUS depending on what kind of GPU the workstation script requests).  
- Click on the checkbox next to the matching row (GPUS_ALL_REGIONS), then click Edit Quota (or click the three dots More actions -> Edit quota on the right side).
- In the Quota changes side pane, you enter your New value (e.g., 1 or 2 depending on how many GPUs your workstation configuration requires). For this case, just 1. 
- Fill in your email and a clear, professional justification (e.g., "Requesting GPU quota to build a Linux virtual workstation for graphical development/rendering workloads"). Providing a clear use case helps get requests processed smoothly. If you are a student, please state in this justification. 
- Click Submit request.

Approval time might be varied, but I get approved within 5 mins. 








