# Coresets On Bacalhau 


[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/bacalhau-project/examples/blob/main/Coreset/BIDS/index.ipynb)
[![Open In Binder](https://mybinder.org/badge.svg)](https://mybinder.org/v2/gh/bacalhau-project/examples/HEAD?labpath=miscellaneous/Coreset/index.ipynb)

## **Introduction**

[Coreset ](https://arxiv.org/abs/2011.09384)is a data subsetting method, Since the uncompressed datasets involved can get very big when compressed it becomes much harder to train them as training time increases with the dataset size, to reduce the training time to save costs we use the coreset method the coreset method can also be applied to other datasets

Coresets similar functionality as same as the whole dataset

![](https://i.imgur.com/AQDLMXn.png)

In this case, we use the coreset method which can lead to a fast speed in solving the k-means problem among the big data with high accuracy in the meantime.

We construct a small coreset for arbitrary shapes of numerical data with a decent time cost. The implementation was mainly based on the coreset construction algorithm that was proposed by Braverman et al. (SODA 2021).


## **Running Locally**

Clone the repo which contains the code



```bash
git clone https://github.com/js-ts/Coreset
```


Downloading the dataset

Open Street Map, which is a public repository that aims to generate and distribute accessible geographic data for the whole world. Basically, it supplies detailed position information, including the longitude and latitude of the places around the world. 

 The dataset is a osm.pbf (compressed format for .osm file), the file can be downloaded from [Geofabrik Download Server](https://download.geofabrik.de/) 



```bash
wget https://download.geofabrik.de/europe/liechtenstein-latest.osm.pbf -o liechtenstein-latest.osm.pbf
```


Installing the Linux dependencies



```bash
sudo apt-get -y update \
sudo apt-get -y install osmium-tool \
sudo apt update \
sudo apt-get -y install libpq-dev gdal-bin libgdal-dev libxml2-dev libxslt-dev
```

Installing Python Dependencies



```bash
pip3 install -r Coreset/requirements.txt
```

Running coreset locally

Convert from compressed pbf format to geojson format


```bash
osmium export liechtenstein-latest.osm.pbf -o liechtenstein-latest.geojson
```

 Running the python script to generate the coreset


```bash
python Coreset/python/coreset.py -f liechtenstein-latest.geojson
```

Building the docker container

In this step you will create a  `Dockerfile` to create your Docker deployment. The `Dockerfile` is a text document that contains the commands used to assemble the image.

First, create the `Dockerfile`.

Next, add your desired configuration to the `Dockerfile`. These commands specify how the image will be built, and what extra requirements will be included.

Dockerfile


```
FROM python:3.8

RUN apt-get -y update && apt-get -y install osmium-tool && apt update && apt-get -y install libpq-dev gdal-bin libgdal-dev libxml2-dev libxslt-dev

ADD Coreset Coreset

ADD monaco-latest.geojson .

RUN cd Coreset && pip3 install -r requirements.txt
```


We will use the `python:3.8` image, and we will choose the src directory in the container as our work directory, we run the same commands for installing dependencies that we used locally, but we also add files and directories which are present on our local machine, we also run a test command, in the end, to check whether the script works

To Build the docker container run the docker build command


```
docker build -t <hub-user>/<repo-name>:<tag> .
```


Please replace

&lt;hub-user> with your docker hub username, If you don’t have a docker hub account [Follow these instructions to create docker account](https://docs.docker.com/docker-id/), and use the username of the account you created

&lt;repo-name> This is the name of the container, you can name it anything you want

&lt;tag> This is not required but you can use the latest tag

After you have build the container, the next step is to test it locally and then push it docker hub

Now you can push this repository to the registry designated by its name or tag.


```
 docker push <hub-user>/<repo-name>:<tag>
```


After the repo image has been pushed to docker hub, we can now use the container for running on bacalhau


## Running on Bacalhau

COMMAND


```
bacalhau docker run \
-v QmXuatKaWL24CwrBPC9PzmLW8NGjgvBVJfk6ZGCWUGZgCu:/input \
jsace/coreset \
-- /bin/bash -c 'osmium export input/liechtenstein-latest.osm.pbf -o liechtenstein-latest.geojson;
python Coreset/python/coreset.py -f input/liechtenstein-latest.geojson -o outputs'
```


Backend: Docker backend here for running the job

Input dataset: Upload the .osm.pbf file while you want to use as a dataset to IPFS, use this CID here 

we mount it to the folder inside the container so it can be used by the script

Image: custom docker Image (it has osmium, python and the requirements for the script installed )

Command:

Convert the osm.pbf dataset to geojson (the dataset is stored in the input volume folder)


```
osmium export input/.osm.pbf -o liechtenstein-latest.geojson
```


Run the script ‘-f’ path of the output geojson file from the above step


```
python Coreset/python/coreset.py -f liechtenstein-latest.geojson -o outputs
```


We get the output in stdout

Additional parameters: -k amount of initialized centers (default=5)

-n: size of coreset (default=50)

-o the folder where you want to store you outputs

Insalling bacalhau


```bash
curl -sL https://get.bacalhau.org/install.sh | bash
```


```bash
bacalhau docker run \
--id-only \
--wait \
--timeout 3600 \
--wait-timeout-secs 3600 \
-v QmXuatKaWL24CwrBPC9PzmLW8NGjgvBVJfk6ZGCWUGZgCu:/input \
jsace/coreset
-- /bin/bash -c 'osmium export input/liechtenstein-latest.osm.pbf -o liechtenstein-latest.geojson; python Coreset/python/coreset.py -f liechtenstein-latest.geojson -o outputs'
```


```python
%env JOB_ID={job_id}
```


Running the commands will output a UUID (like `54506541-4eb9-45f4-a0b1-ea0aecd34b3e`). This is the ID of the job that was created. You can check the status of the job with the following command:



```bash
bacalhau list --id-filter ${JOB_ID} --wide
```


Where it says "`Completed`", that means the job is done, and we can get the results.

To find out more information about your job, run the following command:


```bash
bacalhau describe ${JOB_ID}
```

Since there is no error we can’t see any error instead we see the state of our job to be complete, that means 
we can download the results!
we create a temporary directory to save our results

To Download the results of your job, run 

---

the following command:


```bash
rm -rf results && mkdir -p results
bacalhau get $JOB_ID --output-dir results
```

After the download has finished you should 
see the following contents in results directory


```bash
ls results/
```

### VIEWING THE OUTPUT CSV FILES


```bash
cat results/combined_results/outputs/centers.csv | head -n 10
```


```bash
cat results/combined_results/outputs/coreset-values-liechtenstein-latest.csv | head -n 10
```


```bash
cat results/combined_results/outputs/coreset-weights-liechtenstein-latest.csv | head -n 10
```


Sources

[1] [http://proceedings.mlr.press/v97/braverman19a/braverman19a.pdf](http://proceedings.mlr.press/v97/braverman19a/braverman19a.pdf)

[2][https://aaltodoc.aalto.fi/bitstream/handle/123456789/108293/master_Wu_Xiaobo_2021.pdf?sequence=2](https://aaltodoc.aalto.fi/bitstream/handle/123456789/108293/master_Wu_Xiaobo_2021.pdf?sequence=2)

