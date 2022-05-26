# SAMO: Streaming Architecture Mapping Optimiser

The SAMO framework provides a method of optimising the mapping of a Convolutional Neural Network model onto an FPGA platform for Streaming Architecture frameworks. Both a Simulated Annealing and Brute Force optimiser are implemented. We currently support the following frameworks:

- [FINN](https://github.com/Xilinx/finn)
- [HLS4ML](https://github.com/fastmachinelearning/hls4ml)
- [fpgaconvnet-hls](https://github.com/AlexMontgomerie/fpgaconvnet-hls) and [fpgaconvnet-model](https://github.com/AlexMontgomerie/fpgaconvnet-model)


## Installation

You can install this package using:

```
python -m pip install samo
```

## Usage

The general usage of the SAMO tool can be seen by running `python -m samo --help`.

Example platform configurations are given in the `platforms` directory and example CNN models can be generated by running `python scripts/generate_networks.py`.

### fpgaConvNet

First off, the [fpgaconvnet-model](https://github.com/AlexMontgomerie/fpgaconvnet-model) and [fpgaconvnet-hls](https://github.com/AlexMontgomerie/fpgaconvnet-hls) need to be installed.

```
python -m pip install fpgaconvnet-model fpgaconvnet-hls
```

The command-line interface can be used to optimise the model for a given platform. For example:

```
python -m samo --optimiser annealing --model models/model.onnx \
    --backend fpgaconvnet --platform platforms/platform.json \
    --output-path outputs/
```

The optimised configuration is saved under `outputs/same.json`. This can be used to generate hardware with `fpgaconvnet-hls`. 

```python
from fpgaconvnet.hls.generate.network import GenerateNetwork

# create instance of the network
gen_net = GenerateNetwork("model-name", "outputs/same.json", "models/model.onnx")

# iterate over partitions in network
for i in range(len(gen_net.partitions_generator)):
    # generate hardware and create HLS project
    gen_net.create_partition_project(i)
    # run HLS synthesis
    gen_net.generate_partition_hardware(i)
```

### FINN

In order to run the optimiser with the FINN toolflow, the first step is to download the following fork
```
git clone https://github.com/Yu-Zhewen/finn.git
cd finn
git checkout 4cc0b6fdae2f5c06f0b5bcc6fa45fba4d8b69111
```

As FINN requires docker, set SAMO_DIR to the path of SAMO in run_docker.sh, before entering the docker.
```
bash run_docker.sh
```

Within the docker, generate the FINN-ONNX through the following steps.
```
cd ../samo
cp models/${network}.onnx outputs/saved/finn/${network}.onnx
cp ../finn/notebooks/samo/config/${network}.json ../finn/notebooks/samo/config.json
jupyter nbconvert --to notebook --execute ../finn/notebooks/samo/pre_optimiser_steps.ipynb
mv ../finn/notebooks/samo/pre_optimiser_steps.nbconvert.ipynb outputs/saved/finn/${network}_pre_optimiser_steps.nbconvert.ipynb
```

To optimise the CNN model in the FINN-ONNX format, you need to do:
```
python -m samo --optimiser annealing --model outputs/saved/finn/${network}_pre_optimiser.onnx  \
    --backend finn --platform platforms/zedboard.json \
    --output-path outputs/saved/finn/${network}_post_optimiser.onnx
```

Finally, the following command is used to generate the hardware.
```
jupyter nbconvert --to notebook --execute ../finn/notebooks/samo/post_optimiser_steps.ipynb
```


### HLS4ML

This tool can be used to generate optimised designs for the [HLS4ML](https://github.com/fastmachinelearning/hls4ml) framework. SAMO tunes the `reuse-factor` for layers of the CNN model, and generates a `Resource` driven design.

To optimise a keras model for a given platform, run the following:

```
python -m samo --optimiser annealing --model models/model.keras \
    --backend hls4ml --platform platforms/zedboard.json \
    --output-path outputs/model_hls4ml.json
```

The previous command generates a configuration file (`outputs/model_hls4ml.json`), which can be used by the HLS4ML to generate hardware. To do this, you will need to use the HLS4ML API to convert this configuration file into a HLS project.

```python
import hls4ml
from tensorflow import keras

# load the configuration
with open("outputs/model_hls4ml.json", "r") as f:
    config = json.load(f)

# load the platform
with open("platforms/zedboard.json", "r") as f:
    platform = json.load(f)

# load the keras model
model = keras.models.load_model("models/model.keras")

# create the hls model
hls_model = hls4ml.converters.convert_from_keras_model(model, hls_config=config,
        output_dir="outputs/hls4ml_prj",  io_type="io_stream", fpga_part=platform["part"])

# build the HLS project
hls_model.build(csim=True, cosim=True)
```

---

Feel free to post an issue if you have any questions or problems!
