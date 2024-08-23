---
title: Optimize and deploy 🤗 Transformer models with Optimum-Intel and OpenVINO GenAI
authors:
- user: AlexKoff88
  guest: true
  org: Intel
- user: MrOpenVINO
  guest: true
  org: Intel
- user: katuni4ka
  guest: true
  org: Intel
- user: sandye51
  guest: true
  org: Intel
- user: raymondlo84
  guest: true
  org: Intel
- user: helenai
  guest: true
  org: Intel
- user: sayakpaul
- user: echarlaix
---

# Optimize and deploy 🤗 Transformer models with Optimum-Intel and OpenVINO GenAI

When it comes to the edge or client deployment of the Transformers models, Python is not always the most preferrable solution for this purpose. Many applications, especially in Windows, are written in C++ and require model inference API to be also in C++. Another aspect of such deployment is the application footprint, which also should be minimized to simplify software installation and update processes. [OpenVINO&trade; Toolkit](https://docs.openvino.ai/) initially emerged as a C++ AI inference solution that has bindings to popular programming languages such as Python or Java. It continues to be popular for edge and client deployment with the minimum dependencies on 3rd party software libraries.

Recently, OpenVINO introduced [Generative AI (GenAI) API](https://docs.openvino.ai/2024/learn-openvino/llm_inference_guide/genai-guide.html) to simplify integration of GenAI model inference into C++ (or Python) application with the minimum external dependencies. LLM inference is the first feature in GenAI API that is currently enabled. OpenVINO GenAI software package is supplied with [OpenVINO Tokenizers](https://docs.openvino.ai/2024/learn-openvino/llm_inference_guide/ov-tokenizers.html), a library required for text tokenization-detokenization which is lightweight and has C++ API as well. GenAI repository also contains various [use case examples](https://github.com/openvinotoolkit/openvino.genai/tree/master/samples): from naive LLM decoder-based inference to speculative decoding.

In this blog post, we will outline the LLM deployment steps that include model exportation from Transformers library to OpenVINO Intermediate Representation (IR) using [Optimum-Intel](https://huggingface.co/docs/optimum/en/intel/index), model optimization with [Neural Network Compression Framework (NNCF)](https://github.com/openvinotoolkit/nncf), and software deployment with new GenAI API. We will guide the user through all these steps and highlight changes of basic model KPIs, namely accuracy and performance.

![OpenVINO GenAI workflow diagram](/blog/assets/deploy-with-openvino/openvino_genai_workflow.png "OpenVINO GenAI workflow diagram")

## Pre-requisites

Python and C++ environments are required to run the examples below.

To install packages on the Python environment, use the following command.
```sh
pip install --upgrade --upgrade-strategy eager "optimum[openvino]"
```

The following Python packages were used to reproduce the results in this blog post.
- transformers==4.44
- openvino==24.3
- openvino-tokenizers==24.3
- optimum-intel==1.20
- lm-eval==0.4.3

For GenAI C++ libraries installation follow the instruction [here](https://docs.openvino.ai/2024/get-started/install-openvino/install-openvino-genai.html).


## Exporting model from 🤗 Transformers to OpenVINO

🤗 and Intel have a long story of collaboration and [Optimum-Intel](https://huggingface.co/docs/optimum/en/intel/index) project is a part of this story. It is designed to optimize Transformers models for inference on Intel HW. Optimum-Intel supports OpenVINO as an inference backend and its API has wrappers for various model architectures built on top of OpenVINO inference API. All of these wrappers start from `OV` prefix, for example, `OVModelForCausalLM`. Otherwise, it is similar to the API of 🤗 Transformers library.

To export 🤗 Transformers model to OpenVINO IR one can use two options: `.from_pretrained()` API method of the Optimum-Intel class in a Python script or do it with Optimum the command-line interface (CLI).
Further, we will use the recent Llama 3.1 8B decoder model as an example.
The export with the former option looks as follows:
```python
from optimum.intel import OVModelForCausalLM

MODEL_ID = "meta-llama/Meta-Llama-3.1-8B"
model = OVModelForCausalLM.from_pretrained(MODEL_ID, export=True)
model.save_pretrained("./llam-3.1-8b-ov")
```

Alternatively, the same can be done with CLI as follows:
```sh
optimum-cli export openvino -m meta-llama/Meta-Llama-3.1-8B ./llam-3.1-8b-ov
```

The `./llam-3.1-8b-ov` folder will contain `.xml` and `bin` IR model files and required configuration files that come from the source model. 🤗 tokenizer will be also converted to the format of `openvino-tokenizers` library and corresponding configuration files will be created in the same folder.

## Model Optimization

When running LLMs on the resource constrained edge and client devices, model optimization is highly recommended step. And weight-only quantization is a mainstream approach that significantly reduces latency and model footprint. Optimum-Intel provides weight-only quantization capabilities by means of NNCF which has a variety of optimization techniques designed specifically for LLMs: from data-free INT8 and INT4 weight quantization to data-aware methods such as [AWQ](https://huggingface.co/docs/transformers/main/en/quantization/awq), [GPTQ](https://huggingface.co/docs/transformers/main/en/quantization/gptq), quantization scale estimation, mixed-precision quantization.
By default, weights of the models that are larger than one billion parameters are quantized to INT8 precision which is safe in terms of accuracy. It means that the export steps described above lead to the model with 8-bit weights. However, 4-bit integer weight-only quantization allows achieving a better accuracy-performance trade-off. 

For `meta-llama/Meta-Llama-3.1-8B` model we recommend stacking AWQ, quantization scale estimation along with mixed-precision INT4/INT8 quantization of weights using a calibration dataset that reflects a deployment use case. As in the case of export, there are two options on how to apply 4-bit weight-only quantization to LLM model:
- Specify `quantization_config` parameter in the `.from_pretrained()` method. In this case `OVWeightQuantizationConfig` object should be created and set to this parameter as follows:
```pythonTransformers
from optimum.intel import OVModelForCausalLM, OVWeightQuantizationConfig

MODEL_ID = "meta-llama/Meta-Llama-3.1-8B"
quantization_config = OVWeightQuantizationConfig(awq=True, scale_estimation=True, group_size=64, dataset="c4")
model = OVModelForCausalLM.from_pretrained(MODEL_ID, export=True, quantization_config=quantization_config)
model.save_pretrained("./llam-3.1-8b-ov")
```

- Use command-line options to enable 4-bit weight-only quantization:
```sh
optimum-cli export openvino -m meta-llama/Meta-Llama-3.1-8B --weight-format int4 --awq --scale-estimation --group-size 64 --dataset wikitext2 ./llam-3.1-8b-ov
```

>**Note**: The model optimization process can take time as it and applies several methods subsequently and uses model inference over the specified dataset.

Model optimization with API is more flexible as it allows using custom datasets that can be passed as an iterable object, for example, and instance of `Dataset` object of 🤗 library or just a list of strings.

Weight quantization usually introduces some degradation of the accuracy metric. To compare optimized and source models we report Word Perplexity metric measured on the [Wikitext](https://huggingface.co/datasets/EleutherAI/wikitext_document_level) dataset with [lm-evaluation-harness](https://github.com/EleutherAI/lm-evaluation-harness.git) project which support both 🤗 Transformers and Optimum-Intel models out-of-the-box.

| Model                        | PPL PyTorch FP32 | OpenVINO INT8 | OpenVINO INT4 |
| :--------------------------- | :--------------: | :-----------: | :-----------: |
| meta-llama/Meta-Llama-3.1-8B |   7.3366         | 7.3463        | 7.8288        | 

## Deploy model with OpenVINO GenAI API

Now, we have successfully converted and optimized Llama-3.1-8B model. Let's first run it with Python API of OpenVINO GenAI as the easiest path from 🤗 Optimum-Intel. The basic concept of this API is `LLMPipeline` class. Its instance can be created directly from the folder with the converted model. It will automatically load the main model, tokenizer, detokenizer, and the default generation configuration. The simple greedy-search text generation example with the Python version of `LLMPipeline` looks as follows:
```python
import argparse
import openvino_genai

device = "CPU"  # GPU can be used as well
pipe = openvino_genai.LLMPipeline(args.model_dir, device)
config = openvino_genai.GenerationConfig()
config.max_new_tokens = 100
print(pipe.generate(args.prompt, config))
```

To run this example you need minimum dependencies to be installed into the Python enviroment as OpenVINO GenAI is designed to provide a lightweight deployment. You can install OpenVINO GenAI package to the same Python environemt or create a separate one to compare the application footprint:
```sh
pip install openvino-genai==24.3
```

Let's see how to run the same pipilene with OpenVINO GenAI C++ API. The GenAI API is designed to be intuitive and provides a seamless migration from 🤗 Transformers API. 

>**Note**: In the below example, any other available device in your environment can be specified for "device" variable. For example, if you are using an Intel CPU with integrated graphics, "GPU" is be a good option to try with. To check the available devices, you can use ov::Core::get_available_devices method (refer to [query-device-properties](https://docs.openvino.ai/2024/openvino-workflow/running-inference/inference-devices-and-modes/query-device-properties.html)).


```cpp
#include "openvino/genai/llm_pipeline.hpp"
#include <iostream>

int main(int argc, char* argv[]) {
   std::string model_path = "./llama-3.1-8b-ov";
   std::string device = "CPU"
   ov::AnyMap device_config = {}
   device_config[ov::cache_dir.name()] = "llm-cache";
   // for further device specific performance configs, refer to each device's tab under https://docs.openvino.ai/2024/openvino-workflow/running-inference/inference-devices-and-modes.html
   ov::genai::LLMPipeline pipe(model_path, device, device_config);
   std::cout << pipe.generate("What is LLM model?", ov::genai::max_new_tokens(256));
}
```

`LLMPipeline` also allows specifying custom generation options by means of `ov::genai::GenerationConfig`:
```cpp
ov::genai::GenerationConfig config;
config.max_new_tokens = 256;
std::string result = pipe.generate(prompt, config);
```

With the LLMPipieline, users can not only effortlessly leverage various decoding algorithms such as Beam Search but also construct an interactive chat scenario with a Streamer as in the below example. Moreover, one can take advantage of enhanced internal optimizations with LLMPipeline, such as reduced prompt processing time with utilization of KV cache of previous chat history with the chat methods : start_chat() and finish_chat() (refer to [using-genai-in-chat-scenario](https://docs.openvino.ai/2024/learn-openvino/llm_inference_guide/genai-guide.html#using-genai-in-chat-scenario)).

```cpp
ov::genai::GenerationConfig config;
config.max_new_tokens = 100;
config.do_sample = true;
config.top_p = 0.9;
config.top_k = 30;

auto streamer = [](std::string subword) {
    std::cout << subword << std::flush;
    return false;
};

// Since the streamer is set, the results will
// be printed each time a new token is generated.
pipe.start_chat()
for (size_t i = 0; i < questions.size(); i++) {
   std::cout << "question:\n";
   std::getline(std::cin, prompt);

   std::cout << pipe.generate(prompt) << std::endl;
}
pipe.finish_chat();

```

You can find more details in this [tutorial](https://docs.openvino.ai/2024/learn-openvino/llm_inference_guide/genai-guide.html).

To build the C++ examples above refer to this [document](https://github.com/openvinotoolkit/openvino.genai/blob/releases/2024/3/src/docs/BUILD.md).


![OpenVINO GenAI C++ chat demo](/blog/assets/deploy-with-openvino/demo.gif "OpenVINO GenAI C++ chat demo")