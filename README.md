# PyTritonStudies

Some scripts used to study with PyTriton to test performance on AMD GPUs etc.

### On HPCFund
More information can be found [here](https://amdresearch.github.io/hpcfund/access.html#login-node)
```sh
ssh username@hpcfund.amd.edu
```
To allocate resources for GPU testing, run e.g.:
```sh
salloc -N 1 -n 1 -t 1:00:00
```
More information [here](https://amdresearch.github.io/hpcfund/jobs.html#batch-job-submission)


### Preparing the environment

```sh
# install pytriton
## taken from https://triton-inference-server.github.io/pytriton/0.1.4/installation/
CONDA_VERSION=latest
TARGET_MACHINE=x86_64
curl "https://repo.anaconda.com/miniconda/Miniconda3-${CONDA_VERSION}-Linux-${TARGET_MACHINE}.sh" --output miniconda.sh

sh miniconda.sh -b -p ~/.cache/conda
rm miniconda.sh
conda create -y -n pytriton python=3.8 numpy~=1.21 pyzmq~=23.0
conda activate pytriton
pip install nvidia-pytriton
conda deactivate
```

Then install the needed ML modules (ROCm version) inside the pytriton conda environment:
```sh  
conda activate pytriton
# install tensorflow
pip install tensorflow-rocm
# install pytorch
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/rocm5.7
# install onnx runtime
pip install https://download.onnxruntime.ai/onnxruntime_training-1.16.3%2Brocm55-cp38-cp38-manylinux_2_17_x86_64.manylinux2014_x86_64.whl
conda deactivate
```

Remaining issue: 
- there seems to be some incompatiblity between TF and PyTorch on ROCm. `Import` would crash if both are imported in the same script.
- Recent versions of ONNX would crash on ROCm [issue here](https://github.com/microsoft/onnxruntime/issues/20203). The version above is the latest that works on ROCm 5.7 and ONNX 1.16.3.

### Get Perf Client
(This step is not needed if not using the Perf Analyzer)

Get the Perf Client from the Triton Inference Server image:
```sh
APPTAINER_CACHEDIR=/work1/yfeng/$USER/.cache apptainer pull triton_23.01.sdk.sif docker://nvcr.io/nvidia/tritonserver:23.01-py3-sdk
```
The version from singularity runs fine. The one from pip installed would crash with 
```log
perf_client: error while loading shared libraries: libb64.so.0d: cannot open shared object file: No such file or directory
```

### Clone code and models
```sh
cd /work1/yfeng/$USER
# clone code
git clone git@github.com:yongbinfeng/PyTritonStudies.git
# clone models
git clone git@github.com:fastmachinelearning/sonic-models.git
```
In case the ssh key is not set up, use the https link instead, or create the ssh key and add to ssh agent (instructions [here](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent))

Sometimes the add key needs to be added to to the ssh-agent again:
```sh
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```

### Running direct inference

```sh
conda activate pytriton
cd PyTritonStudies/directinference/
python test_tf.py
```
and similarly for other ones.

One could open a new terminal, log in to the same node, and monitor the GPU usage with
```sh
watch -n 1 rocm-smi
```

### Launching servers and clients
```sh
conda activate pytriton
# start the server
cd PyTritonStudies/server
python server_pnet_onnx.py
```
The server should be launched with the output ending with
```log
I0430 14:34:48.115255 3379381 grpc_server.cc:2450] Started GRPCInferenceService at 0.0.0.0:9001
I0430 14:34:48.115401 3379381 http_server.cc:3555] Started HTTPService at 0.0.0.0:9000
I0430 14:34:48.156420 3379381 http_server.cc:185] Started Metrics Service at 0.0.0.0:9002
```
Then in another terminal, on the same node, check the server is launched properly:
```sh
curl localhost:9002/metrics
```
Change `localhost` to the node name if not on the same node. The output should be similar to
```log
# HELP nv_inference_request_success Number of successful inference requests, all batch sizes
# TYPE nv_inference_request_success counter
nv_inference_request_success{model="pnet_onnx",version="1"} 1000
# HELP nv_inference_request_failure Number of failed inference requests, all batch sizes
# TYPE nv_inference_request_failure counter
nv_inference_request_failure{model="pnet_onnx",version="1"} 0
# HELP nv_inference_count Number of inferences performed (does not include cached requests)
# TYPE nv_inference_count counter
...
```
Run the client (in the pytriton environment):
```sh
conda activate pytriton
cd PyTritonStudies/client
python client.py
```
Similar client and server scripts can be built for other models as well.

### Running Perf Analyzer
Keep the server running, and start the container in another terminal:
```sh
apptainer run triton_23.01.sdk.sif
```
and inside the container run:
```sh
perf_analyzer -m pnet_onnx --percentile=95 -u localhost:9001 -i grpc --async -p 9001 --concurrency-range 8:8 -b 100 --shape INPUT_1:2,50 --shape INPUT_2:20,50 --shape INPUT_3:1,50 --shape INPUT_4:2,4 --shape INPUT_5:11,4 --shape INPUT_6:1,4
```
The output should be similar to
```log
*** Measurement Settings ***
  Batch size: 100
  Service Kind: Triton
  Using "time_windows" mode for stabilization
  Measurement window: 9001 msec
  Latency limit: 0 msec
  Concurrency limit: 8 concurrent requests
  Using asynchronous calls for inference
  Stabilizing using p95 latency

Request concurrency: 8
  Client: 
    Request count: 1260
    Throughput: 3888.33 infer/sec
    p50 latency: 205425 usec
    p90 latency: 209508 usec
    p95 latency: 210364 usec
    p99 latency: 212468 usec
    Avg gRPC time: 205733 usec ((un)marshal request/response 34 usec + response wait 205699 usec)
  Server: 
    Inference count: 126000
    Execution count: 315
    Successful request count: 1260
    Avg request latency: 205157 usec (overhead 16 usec + queue 102258 usec + compute input 105 usec + compute infer 102691 usec + compute output 86 usec)

Inferences/Second vs. Client p95 Batch Latency
Concurrency: 8, throughput: 3888.33 infer/sec, latency 210364 usec
```

# Play with TritonServer WITHOUT python interface

*Still under development. Might crash or not be fully functioning*

### Build PyTriton on AlmaLinux 9

This might conflict with the PyTriton environment installed with `pip` before. The recommendation is to start from a new python environment, e.g., create a new python environment with `conda`.

```sh
git clone -b testCpp git@github.com:yongbinfeng/pytriton.git
## Compile 
make install-dev
make dist
```
More information on pytriton build can be found [here](https://triton-inference-server.github.io/pytriton/0.5.2/guides/building/)

The build would need `docker`. GPU support with CUDA libraries under `/usr/local/cuda/` might also be needed. If the build runs successfully, it should generate a python wheel under `dist`. Install the wheel with
```
pip install dist/nvidia_pytriton-*-py3-none-*.whl
```

All the relevant libs and header files should be around the directory:
```sh
PyTritonBase=`python -c "import pytriton;print(pytriton.__path__[0])"`
echo $PyTritonBase
ll $PyTritonBase
```
especially the binary exectuable `tritonserver` in `$PyTritonBase/tritonserver/bin` and the backend modules in `$PyTritonBase/tritonserver/backends/python`.

### Testing the identity cpp backend with TritonServer executable

The identity backend is a simple backend that returns the input tensor as the output tensor. It is included in the PyTriton build, saved in the wheel, and located under `$PyTritonBase/tritonserver/backends/python/identity`.

Copy it to the model directory
```sh
cp $PyTritonBase/tritonserver/backends/python/identity $Path_to_PyTritonStudies/models_cpp/custom_zero_1_float32/1/libtriton_identity.so
```

Then launch the server using the binary executable `tritonserver` installed with PyTriton:
```sh
$PyTritonBase/tritonserver/bin/tritonserver --model-repository $Path_to_PyTritonStudies/models_cpp
```

The expected output should be similar to
```log
I0521 05:35:06.139750 574190 grpc_server.cc:2450] Started GRPCInferenceService at 0.0.0.0:8001
I0521 05:35:06.139868 574190 http_server.cc:3555] Started HTTPService at 0.0.0.0:8000
I0521 05:35:06.180917 574190 http_server.cc:185] Started Metrics Service at 0.0.0.0:8002
```

The client script for testing is under `client/client_identity.py`, which requires client container to run it.
```sh
docker pull nvcr.io/nvidia/tritonserver:23.04-py3-sdk
docker run -v$PWD:$PWD --net=host nvcr.io/nvidia/tritonserver:23.04-py3-sdk python $PWD/client/client_identity.py
```

### Testing Python backend with TritonServer executable

*Need to update the script*

Launch the server:
```sh
PYTHONPATH=/work1/yfeng/yfeng/newtests/PyTritonStudies/server:/home1/yfeng/work/.cache/conda/envs/pytriton/lib/python38.zip:/home1/yfeng/work/.cache/conda/envs/pytriton/lib/python3.8:/home1/yfeng/work/.cache/conda/envs/pytriton/lib/python3.8/lib-dynload:/home1/yfeng/work/.cache/conda/envs/pytriton/lib/python3.8/site-packages /home1/yfeng/work/.cache/conda/envs/pytriton/lib/python3.8/site-packages/pytriton/tritonserver/bin/tritonserver --log-verbose 10000 --http-port 9004 --grpc-port 9005 --metrics-port 9006 --backend-config python,shm-region-prefix-name=pytrtion3285102-96eaafbb --backend-config python,shm-default-byte-size=4194304 --backend-config python,shm-growth-byte-size=1048576 --model-repository /work1/yfeng/yfeng/newtests/PyTritonStudies/models_python  --backend-directory /home1/yfeng/work/.cache/conda/envs/pytriton/lib/python3.8/site-packages/pytriton/tritonserver/backends --cache-directory /home1/yfeng/work/.cache/conda/envs/pytriton/lib/python3.8/site-packages/pytriton/tritonserver/caches
```

Launch the client (under `pytriton` environment since it needs the client modules)
```sh
python client/client_add_sub.py
```