# Building the StorRAG Appliance

In order to build the StorRAG Appliance we will need two models, a Vector Database, a Model Service and an AI Application.

Download models
Deploy the Vector Database
Build the Model Service
Deploy the Model Service
Build the AI Application
Deploy the AI Application
Interact with the AI Application

## Download models

If you are just getting started, we recommend using granite-3.3-8b-instruct. This is a well performant mid-sized model with an apache-2.0 license that has been quanitzed and served into the GGUF format.

The recommended model can be downloaded using the code snippet below:

cd ../../../models
curl -sLO https://huggingface.co/ibm-granite/granite-3.3-8b-instruct-GGUF/resolve/main/granite-3.3-8b-instruct-Q4_K_M.gguf
cd ../recipes/natural_language_processing/rag

In addition to the LLM, RAG applications also require an embedding model to convert documents between natural language and vector representations. For this demo we will use `BAAI/bge-base-en-v1.5` it is a fairly standard model for this use case and has an MIT license.

The code snippet below can be used to pull a copy of the BAAI/bge-base-en-v1.5 embedding model and store it in your models/ directory.

from huggingface_hub import snapshot_download
snapshot_download(repo_id="BAAI/bge-base-en-v1.5",
                cache_dir="models/",
                local_files_only=False)

## Deploy the Vector Database

To deploy the Vector Database service locally, simply use the existing ChromaDB or Milvus image. The Vector Database is ephemeral and will need to be re-populated each time the container restarts. When implementing RAG in production, you will want a long running and backed up Vector Database.

### ChromaDB

podman pull chromadb/chroma
podman run --rm -it -p 8000:8000 chroma
Milvus
podman pull milvusdb/milvus:master-20240426-bed6363f
podman run -it \
        --name milvus-standalone \
        --security-opt seccomp:unconfined \
        -e ETCD_USE_EMBED=true \
        -e ETCD_CONFIG_PATH=/milvus/configs/embedEtcd.yaml \
        -e COMMON_STORAGETYPE=local \
        -v $(pwd)/volumes/milvus:/var/lib/milvus \
        -v $(pwd)/embedEtcd.yaml:/milvus/configs/embedEtcd.yaml \
        -p 19530:19530 \
        -p 9091:9091 \
        -p 2379:2379 \
        --health-cmd="curl -f http://localhost:9091/healthz" \
        --health-interval=30s \
        --health-start-period=90s \
        --health-timeout=20s \
        --health-retries=3 \
        milvusdb/milvus:master-20240426-bed6363f \
        milvus run standalone  1> /dev/null

Note: For running the Milvus instance, make sure you have the $(pwd)/volumes/milvus directory and $(pwd)/embedEtcd.yaml file as shown in this repository. These are required by the database for its operations.

### Build the Model Service

The complete instructions for building and deploying the Model Service can be found in the the llamacpp_python model-service document.

The Model Service can be built with the following code snippet:

cd model_servers/llamacpp_python
podman build -t llamacppserver -f ./base/Containerfile .
Deploy the Model Service
The complete instructions for building and deploying the Model Service can be found in the the llamacpp_python model-service document.

The local Model Service relies on a volume mount to the localhost to access the model files. You can start your local Model Service using the following Podman command:

podman run --rm -it \
        -p 8001:8001 \
        -v Local/path/to/locallm/models:/locallm/models \
        -e MODEL_PATH=models/<model-filename> \
        -e HOST=0.0.0.0 \
        -e PORT=8001 \
        llamacppserver

### Build the Application

Now that the Model Service is running we want to build and deploy our AI Application. Use the provided Containerfile to build the AI Application image in the rag-langchain/ directory.

cd rag
make APP_IMAGE=rag build

### Deploy the Application

Make sure the Model Service and the Vector Database are up and running before starting this container image. When starting the AI Application container image we need to direct it to the correct MODEL_ENDPOINT. This could be any appropriately hosted Model Service (running locally or in the cloud) using an OpenAI compatible API. In our case the Model Service is running inside the Podman machine so we need to provide it with the appropriate address 10.88.0.1. The same goes for the Vector Database. Make sure the VECTORDB_HOST is correctly set to 10.88.0.1 for communication within the Podman virtual machine.

There also needs to be a volume mount into the models/ directory so that the application can access the embedding model as well as a volume mount into the data/ directory where it can pull documents from to populate the Vector Database.

The following Podman command can be used to run your AI Application:

podman run --rm -it -p 8501:8501 \
-e MODEL_ENDPOINT=http://10.88.0.1:8001 \
-e VECTORDB_HOST=10.88.0.1 \
-v Local/path/to/locallm/models/:/rag/models \
rag   

### Interact with the Application

Everything should now be up an running with the rag application available at `http://localhost:8501`. By using this recipe and getting this starting point established, users should now have an easier time customizing and building their own LLM enabled RAG applications.

Embed the AI Application in a Bootable Container Image

To build a bootable container image that includes this sample RAG chatbot workload as a service that starts when a system is booted, cd into this folder and run:

make BOOTC_IMAGE=quay.io/your/rag-bootc:latest bootc
Substituting the bootc/Containerfile FROM command is simple using the Makefile FROM option.

make FROM=registry.redhat.io/rhel9/rhel-bootc:9.4 BOOTC_IMAGE=quay.io/your/rag-bootc:latest bootc
The magic happens when you have a bootc enabled system running. If you do, and you'd like to update the operating system to the OS you just built with the RAG chatbot application, it's as simple as ssh-ing into the bootc system and running:

bootc switch quay.io/your/rag-bootc:latest
Upon a reboot, you'll see that the RAG chatbot service is running on the system.

### Check on the service with

ssh user@bootc-system-ip
sudo systemctl status rag

### Creating bootable disk images

You can convert a bootc image to a bootable disk image using the quay.io/centos-bootc/bootc-image-builder container image.

This container image allows you to build and deploy multiple disk image types from bootc container images.

Default image types can be set via the DISK_TYPE Makefile variable.

make bootc-image-builder DISK_TYPE=ami

### Makefile variables

There are several Makefile variables defined which can be used to override defaults for a variety of make targets.
