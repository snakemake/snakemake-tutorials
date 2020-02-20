# GoogleMAGs

This tutorial is intended to run on Google Cloud, specifically by starting
a Google Compute Engine Instance, shelling in, installing snakemake,
and running a pipeline. Full instructions are included here.

**Important** read/write access is currently required for the bucket hn-snakemake/pig5,
along with Google Life Sciences Project permissions added for your project to the
project with this bucket to run this pipeline. Both the credentials that you
generate along with your compute engine credential will need GET access to
the bucket mentioned above. In testing, we had to use admin access for it 
to fully work.

## 1. Create an Instance

You will want to first log in to your project and [create an instance](https://console.cloud.google.com/compute/instances).
If you don't have any instances, this coincides with clicking "Create":

![img/create-an-instance.png](img/create-an-instance.png)

Specifically, these are the choices that are recommended:

 - **Region** you want to be as close to where your main operations are as possible. For California, this typically means us-west1, and then choose a,b, or c. The actual instances for a/b/c vary by project, so your choice of A isn't equal to another project's, so don't worry too much about your choice.
 - **Machine Family** I typically choose General purpose, because we just need a basic linux base.
 - You don't need to select that we are deploying a container to the instance.
 - **Machine Type** It's best to choose a smaller (but not too small) size, typically I choose n1-standard-2 (2 vCPUs, 7.5 GB memory).
 - **Boot Disk** Even for development, you always want to chose an image with long term support. E.g., for now I would choose Ubuntu 18.04 LTS (long term support). There is a minimal image that works well too. For the actual disk size, I tend to choose a larger one to be safe (100GB).
 - **API Access** you generally want to limit to only those endpoints that are needed, but if you are developing it's easier to grant access to all APIs. For this example, we would want access to Google Compute Engine, Google Life Sciences, and Google Storage.
 - I typically allow both http/https traffic in the case that I need it.

![img/instance-setup.png](img/instance-setup.png)

You don't need to enable delete protection, or to ask for a static (not ephemeral) ip address under networking. This is just a test instance and it will go away rather quickly. When you are happy with your setup, click on "Create" at the bottom.

![img/create-instance.png](img/create-instance.png)

A note that I like to share is that you can click on "equivalent rest or command line" directly below this button.
If you might want to create this programatically in the future, click there.

## 2. Shell Inside

After you create the instance, it will take a brief time to spin up, and when it's ready,
a small green dot will appear on the left side.

<!--![img/instance-ready.png](img/instance-ready.png)-->

You can then click on the "SSH" dropdown on the right side, and I like to copy
paste the command for shelling into the instance from my command line (view gcloud commnd).
If you haven't yet installed gcloud, see [these instructions](https://cloud.google.com/sdk/install).


## 3. Install dependencies

Now let's install dependencies! This means snakemake, and also downloading the
pipeline files. 

### Host Dependencies

First, here is for snakemake.

```bash
sudo apt-get update
sudo apt-get install -y git gcc
```

Let's install anaconda so we can test pipelines with conda.

```bash
$ wget https://repo.anaconda.com/archive/Anaconda3-2019.10-Linux-x86_64.sh
$ chmod +x Anaconda3-2019.10-Linux-x86_64.sh 
$ ./Anaconda3-2019.10-Linux-x86_64.sh -b -p $HOME/anaconda3
```

Add to our path (edit in your bash profile to make permanent, if desired).

```bash
export PATH=$HOME/anaconda3/bin:$PATH
which python
/home/vanessa/anaconda3/bin/python

python --version
Python 3.7.4

which pip
/home/vanessa/anaconda3/bin/pip
```

### Snakemake

If you like, you can use a virtual environment, however since this is a one-off
testing instance I'm going to install snakemake using the system python.

```bash
git clone -b add/google-cloud-pipelines https://github.com/vsoch/snakemake
cd snakemake
```

To ensure that we fetch tags (versions) from the upstream, add it as a remote,
and fetch those tags:

```bash
git remote add upstream https://github.com/snakemake/snakemake.git
git fetch upstream
```

And then install with pip. You should see a version 5.10.0 or later.

```bash
pip install -e .
```

And ensure it installed successfully and snakemake is on your path.

```bash
$ which snakemake
/home/vanessa/anaconda3/bin/snakemake
```

### Google API Python Clients

Google always suggests that you upgrade your python clients, so let's do that.
Also, these aren't provided by default with Snakemake (there are many users that want
to use Snakemake in a context outside of Google). However, we need them.

```bash
pip install --upgrade google-api-python-client
pip install --upgrade google-cloud-storage
pip install oauth2client
```

If you haven't yet, create a Google Storage Bucket in the interface.
You'll want to be sure to add pipelines service accounts to your Storage bucket users.
This step is hairy and error prone and I never really get it right the first time.

### GoogleMAGs

Let's clone the repository for Google MAGS.

```bash
cd $HOME
git clone https://github.com/WatsonLab/GoogleMAGs
cd GoogleMAGs
```

Snakemake requires GOOGLE_APPLICATION_CREDENTIALS, and since you might want to
run this is (non Google places) too, you should [download your service account](https://console.cloud.google.com/iam-admin/iam)
key and export it to the environment. From your host, you can copy to the instance as follows:

```bash
gcloud compute scp credentials.json [username]@snakemake-googlemags:/home/[username]/credentials.json
```

And note that your computer username might not correspond to your gcloud username (in the above I use the same variable).
Then you can export your credentials to the environment.

```bash
export GOOGLE_APPLICATION_CREDENTIALS="/home/[username]/credentials.json"
```

Note that I've also tested the above instead installing `python3-dev` and using [get-pip.py](https://bootstrap.pypa.io/get-pip.py)
to install pip. If you install to system Python you'd need to use sudo, otherwise you can use virtualenv.

## 4. Run Snakemake

Now let's test running Snakemake! Here we are in the root folder of the GoogleMAGs repository.
The snakefile we are targeting is `Snakefilev14` and we are going to choose the same region that our
instance is in. 

### Running from within Python

Although this is more complicated, I tested it first since I anticipated the command line would lead to troubles.
Note that this is being run after cd into the GoogleMAGs directory.

```python
from snakemake import snakemake
import os
workdir = os.getcwd()
```

Add the present working directory to the Python path:

```python
os.environ['PYTHONPATH'] = os.getcwd()
```

We want to provide the path to the snakefile:

```python
snakefile = 'Snakefilev14'
```

And now create a temporary directory for working:

```python
import tempfile
tmpdir = next(tempfile._get_candidate_names()) 
tmpdir = os.path.join(tempfile.gettempdir(), "snakemake-%s" % tmpdir) 
os.mkdir(tmpdir) 
```

Copy all our files there.

```python
import shutil

def copy(src, dst): 
    if os.path.isdir(src): 
        shutil.copytree(src, os.path.join(dst, os.path.basename(src))) 
    else: 
        shutil.copy(src, dst) 

path = os.getcwd()
for f in os.listdir(path): 
    print(f) 
    copy(os.path.join(path, f), tmpdir) 

Snakefilev6
Dockerfile
Snakefile
Snakefilev11
cluster.json
runs
Snakefilev8
envs
environment.yaml
Snakefilev7
create_local_text.sh
Snakefilev10
fileofaccessions.txt
.git
Snakefilev13
scripts
Snakefilev14
Snakefilev9
.snakemake
LICENSE
README.md
Snakefilev12
```

Let's get ready to run! These are the same parameters used in testing.

```python
cores = 3
config={}                                                                                                                        
verbose=True                                                                                                                   
success = snakemake( 
    snakefile,
    cores=cores, 
    workdir=tmpdir, 
    stats="stats.txt", 
    config=config, 
    use_conda=True,
    google_lifesciences_regions=["us-west1"],
    default_remote_prefix="hn-snakemake/pig5",
    google_lifesciences=True,
    google_lifesciences_cache=True,
    verbose=True) 
```

And then the pipeline will run and print verbose output to the screen.

### Running from the Command Line

Now that we've seen a test running from within Python, let's review the more likely 
use case (running from the command line).

```bash
snakemake  --google-lifesciences --verbose -s Snakefilev14 --default-remote-prefix hn-snakemake/pig5 --use-conda --google-lifesciences-keep-cache --google-lifesciences-region us-west1
```

And here is the same example with 400 jobs:

```bash
snakemake  --google-lifesciences --verbose -s Snakefilev14 -j 400 --default-remote-prefix hn-snakemake/pig5 --use-conda --google-lifesciences-keep-cache --google-lifesciences-region us-west1 --container-image snakemake/snakemake:v5.10.0
```


**under development**
