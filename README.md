<!---
Copyright 2022 The HuggingFace Team. All rights reserved.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->

![](https://github.com/huggingface/optimum-habana/blob/main/readme_logo.png)


# Optimum Habana

🤗 Optimum Habana is the interface between the 🤗 Transformers and Diffusers libraries and [Habana's Gaudi processor (HPU)](https://docs.habana.ai/en/latest/index.html).
It provides a set of tools enabling easy model loading, training and inference on single- and multi-HPU settings for different downstream tasks.
The list of officially validated models and tasks is available [here](https://github.com/huggingface/optimum-habana#validated-models). Users can try other models and tasks with only few changes.


## What is a Habana Processing Unit (HPU)?

HPUs offer fast model training and inference as well as a great price-performance ratio.
Check out [this blog post about BERT pre-training](https://huggingface.co/blog/pretraining-bert) and [this article benchmarking Habana Gaudi2 versus Nvidia A100 GPUs](https://huggingface.co/blog/habana-gaudi-2-benchmark) for concrete examples.
If you are not familiar with HPUs and would like to know more about them, we recommend you take a look at [our conceptual guide](https://huggingface.co/docs/optimum/habana/concept_guides/hpu).


## Install the library and get example scripts

### Option 1: Use the latest stable release

To install the latest stable release of this package
>```bash
>pip install --upgrade-strategy eager optimum[habana]
>```

The `--upgrade-strategy eager` option is needed to ensure `optimum-habana` is upgraded to the latest stable release.

To use the example associated with the latest stable release, run:
> ```
> git clone https://github.com/huggingface/optimum-habana
> cd optimum-habana && git checkout v1.11.1
> ```
> with `v1.11.1` the version number of this release.

### Option 2: Use the latest main branch under development

Optimum Habana is a fast-moving project, and you may want to install it from source and get the latest scripts :

```bash
pip install git+https://github.com/huggingface/optimum-habana.git
git clone https://github.com/huggingface/optimum-habana
```

## Install dependencies

To use DeepSpeed on HPUs, you also need to run the following command:
>```bash
>pip install git+https://github.com/HabanaAI/DeepSpeed.git@1.15.0
>```

To install the requirements for every example:
>```bash
>cd <example-folder>
>pip install -r requirements.txt
>```


## How to use it?

### Quick Start

🤗 Optimum Habana was designed with one goal in mind: **to make training and inference straightforward for any 🤗 Transformers and 🤗 Diffusers user while leveraging the complete power of Gaudi processors**.

#### Transformers Interface

There are two main classes one needs to know:
- [GaudiTrainer](https://huggingface.co/docs/optimum/habana/package_reference/trainer): the trainer class that takes care of compiling and distributing the model to run on HPUs, and performing training and evaluation.
- [GaudiConfig](https://huggingface.co/docs/optimum/habana/package_reference/gaudi_config): the class that enables to configure Habana Mixed Precision and to decide whether optimized operators and optimizers should be used or not.

The [GaudiTrainer](https://huggingface.co/docs/optimum/habana/package_reference/trainer) is very similar to the [🤗 Transformers Trainer](https://huggingface.co/docs/transformers/main_classes/trainer), and adapting a script using the Trainer to make it work with Gaudi will mostly consist in simply swapping the `Trainer` class for the `GaudiTrainer` one.
That's how most of the [example scripts](https://github.com/huggingface/optimum-habana/tree/main/examples) were adapted from their [original counterparts](https://github.com/huggingface/transformers/tree/main/examples/pytorch).

Here is an example:
```diff
- from transformers import Trainer, TrainingArguments
+ from optimum.habana import GaudiConfig, GaudiTrainer, GaudiTrainingArguments

- training_args = TrainingArguments(
+ training_args = GaudiTrainingArguments(
  # training arguments...
+ use_habana=True,
+ use_lazy_mode=True,  # whether to use lazy or eager mode
+ gaudi_config_name=path_to_gaudi_config,
)

# A lot of code here

# Initialize our Trainer
- trainer = Trainer(
+ trainer = GaudiTrainer(
    model=model,
    args=training_args,  # Original training arguments.
    train_dataset=train_dataset if training_args.do_train else None,
    eval_dataset=eval_dataset if training_args.do_eval else None,
    compute_metrics=compute_metrics,
    tokenizer=tokenizer,
    data_collator=data_collator,
)
```

where `gaudi_config_name` is the name of a model from the [Hub](https://huggingface.co/Habana) (Gaudi configurations are stored in model repositories) or a path to a local Gaudi configuration file (you can see [here](https://huggingface.co/docs/optimum/habana/package_reference/gaudi_config) how to write your own).


#### Diffusers Interface

You can generate images from prompts using Stable Diffusion on Gaudi using the [`GaudiStableDiffusionPipeline`](https://huggingface.co/docs/optimum/habana/package_reference/stable_diffusion_pipeline) class and the [`GaudiDDIMScheduler`] which have been both optimized for HPUs. Here is how to use them and the differences with the 🤗 Diffusers library:

```diff
- from diffusers import DDIMScheduler, StableDiffusionPipeline
+ from optimum.habana.diffusers import GaudiDDIMScheduler, GaudiStableDiffusionPipeline


model_name = "runwayml/stable-diffusion-v1-5"

- scheduler = DDIMScheduler.from_pretrained(model_name, subfolder="scheduler")
+ scheduler = GaudiDDIMScheduler.from_pretrained(model_name, subfolder="scheduler")

- pipeline = StableDiffusionPipeline.from_pretrained(
+ pipeline = GaudiStableDiffusionPipeline.from_pretrained(
    model_name,
    scheduler=scheduler,
+   use_habana=True,
+   use_hpu_graphs=True,
+   gaudi_config="Habana/stable-diffusion",
)

outputs = generator(
    ["An image of a squirrel in Picasso style"],
    num_images_per_prompt=16,
+   batch_size=4,
)
```


### Documentation

Check out [the documentation of Optimum Habana](https://huggingface.co/docs/optimum/habana/index) for more advanced usage.


## Validated Models

The following model architectures, tasks and device distributions have been validated for 🤗 Optimum Habana:

> In the tables below, :heavy_check_mark: means single-card, multi-card and DeepSpeed have all been validated.

- Transformers:
<div align="center">

| Architecture | Training | Inference | <center>Tasks</center> |
|--------------|:--------:|:---------:|:-----------------------|
| BERT         | :heavy_check_mark: | :heavy_check_mark: | <li>[text classification](https://github.com/huggingface/optimum-habana/tree/main/examples/text-classification)</li><li>[question answering](https://github.com/huggingface/optimum-habana/tree/main/examples/question-answering)</li><li>[language modeling](https://github.com/huggingface/optimum-habana/tree/main/examples/language-modeling)</li> |
| RoBERTa | :heavy_check_mark: | :heavy_check_mark: | <li>[question answering](https://github.com/huggingface/optimum-habana/tree/main/examples/question-answering)</li><li>[language modeling](https://github.com/huggingface/optimum-habana/tree/main/examples/language-modeling)</li> |
| ALBERT | :heavy_check_mark: | :heavy_check_mark: | <li>[question answering](https://github.com/huggingface/optimum-habana/tree/main/examples/question-answering)</li><li>[language modeling](https://github.com/huggingface/optimum-habana/tree/main/examples/language-modeling)</li> |
| DistilBERT |:heavy_check_mark: | :heavy_check_mark: | <li>[question answering](https://github.com/huggingface/optimum-habana/tree/main/examples/question-answering)</li><li>[language modeling](https://github.com/huggingface/optimum-habana/tree/main/examples/language-modeling)</li> |
| GPT2             | :heavy_check_mark: | :heavy_check_mark: | <li>[language modeling](https://github.com/huggingface/optimum-habana/tree/main/examples/language-modeling)</li><li>[text generation](https://github.com/huggingface/optimum-habana/tree/main/examples/text-generation)</li> |
| BLOOM(Z) |   | <div style="text-align:left"><li>DeepSpeed</li></div> | <li>[text generation](https://github.com/huggingface/optimum-habana/tree/main/examples/text-generation)</li> |
| StarCoder |   | <div style="text-align:left"><li>Single card</li></div> | <li>[text generation](https://github.com/huggingface/optimum-habana/tree/main/examples/text-generation)</li> |
| GPT-J | <div style="text-align:left"><li>DeepSpeed</li></div> | <div style="text-align:left"><li>Single card</li><li>DeepSpeed</li></div> | <li>[language modeling](https://github.com/huggingface/optimum-habana/tree/main/examples/language-modeling)</li><li>[text generation](https://github.com/huggingface/optimum-habana/tree/main/examples/text-generation)</li> |
| GPT-NeoX | <div style="text-align:left"><li>DeepSpeed</li></div> | <div style="text-align:left"><li>DeepSpeed</li></div> | <li>[language modeling](https://github.com/huggingface/optimum-habana/tree/main/examples/language-modeling)</li><li>[text generation](https://github.com/huggingface/optimum-habana/tree/main/examples/text-generation)</li> |
| OPT |   | <div style="text-align:left"><li>DeepSpeed</li></div> | <li>[text generation](https://github.com/huggingface/optimum-habana/tree/main/examples/text-generation)</li> |
| Llama 2 / CodeLlama / Llama 3 | :heavy_check_mark: | :heavy_check_mark: | <li>[language modeling](https://github.com/huggingface/optimum-habana/tree/main/examples/language-modeling)</li><li>[text generation](https://github.com/huggingface/optimum-habana/tree/main/examples/text-generation)</li><li>[question answering](https://github.com/huggingface/optimum-habana/tree/main/examples/question-answering)</li> |
| StableLM |   | <div style="text-align:left"><li>Single card</li></div> | <li>[text generation](https://github.com/huggingface/optimum-habana/tree/main/examples/text-generation)</li> |
| Falcon | <div style="text-align:left"><li>LoRA</li></div> | :heavy_check_mark: | <li>[language modeling](https://github.com/huggingface/optimum-habana/tree/main/examples/language-modeling)</li><li>[text generation](https://github.com/huggingface/optimum-habana/tree/main/examples/text-generation)</li> |
| CodeGen |   | <div style="text-align:left"><li>Single card</li></div> | <li>[text generation](https://github.com/huggingface/optimum-habana/tree/main/examples/text-generation)</li> |
| MPT |   | <div style="text-align:left"><li>Single card</li></div> | <li>[text generation](https://github.com/huggingface/optimum-habana/tree/main/examples/text-generation)</li> |
| Mistral |   | <div style="text-align:left"><li>Single card</li></div> | <li>[text generation](https://github.com/huggingface/optimum-habana/tree/main/examples/text-generation)</li> |
| Phi | :heavy_check_mark:  | <div style="text-align:left"><li>Single card</li></div> | <li>[language modeling](https://github.com/huggingface/optimum-habana/tree/main/examples/language-modeling)</li><li>[text generation](https://github.com/huggingface/optimum-habana/tree/main/examples/text-generation)</li> |
| Mixtral |   | <div style="text-align:left"><li>Single card</li></div> | <li>[text generation](https://github.com/huggingface/optimum-habana/tree/main/examples/text-generation)</li> |
| Persimmon |   | <div style="text-align:left"><li>Single card</li></div> | <li>[text generation](https://github.com/huggingface/optimum-habana/tree/main/examples/text-generation)</li> |
| Qwen2 | <div style="text-align:left"><li>Single card</li></div> | <div style="text-align:left"><li>Single card</li></div> | <li>[language modeling](https://github.com/huggingface/optimum-habana/tree/main/examples/language-modeling)</li><li>[text generation](https://github.com/huggingface/optimum-habana/tree/main/examples/text-generation)</li> |
| Gemma | :heavy_check_mark:  | <div style="text-align:left"><li>Single card</li></div> | <li>[language modeling](https://github.com/huggingface/optimum-habana/tree/main/examples/language-modeling)</li><li>[text generation](https://github.com/huggingface/optimum-habana/tree/main/examples/text-generation)</li> |
| T5 / Flan T5 | :heavy_check_mark: | :heavy_check_mark: | <li>[summarization](https://github.com/huggingface/optimum-habana/tree/main/examples/summarization)</li><li>[translation](https://github.com/huggingface/optimum-habana/tree/main/examples/translation)</li><li>[question answering](https://github.com/huggingface/optimum-habana/tree/main/examples/question-answering#fine-tuning-t5-on-squad20)</li> |
| BART |   | <div style="text-align:left"><li>Single card</li></div> | <li>[summarization](https://github.com/huggingface/optimum-habana/tree/main/examples/summarization)</li><li>[translation](https://github.com/huggingface/optimum-habana/tree/main/examples/translation)</li><li>[question answering](https://github.com/huggingface/optimum-habana/tree/main/examples/question-answering#fine-tuning-t5-on-squad20)</li> |
| ViT | :heavy_check_mark: | :heavy_check_mark: | <li>[image classification](https://github.com/huggingface/optimum-habana/tree/main/examples/image-classification)</li> |
| Swin | :heavy_check_mark: | :heavy_check_mark: | <li>[image classification](https://github.com/huggingface/optimum-habana/tree/main/examples/image-classification)</li> |
| Wav2Vec2 | :heavy_check_mark: | :heavy_check_mark: | <li>[audio classification](https://github.com/huggingface/optimum-habana/tree/main/examples/audio-classification)</li><li>[speech recognition](https://github.com/huggingface/optimum-habana/tree/main/examples/speech-recognition)</li> |
| Whisper | :heavy_check_mark: | :heavy_check_mark: | <li>[speech recognition](https://github.com/huggingface/optimum-habana/tree/main/examples/speech-recognition)</li> |
| SpeechT5 |   | <div style="text-align:left"><li>Single card</li></div> | <li>[text to speech](https://github.com/huggingface/optimum-habana/tree/main/examples/text-to-speech)</li> |
| CLIP | :heavy_check_mark: | :heavy_check_mark: | <li>[contrastive image-text training](https://github.com/huggingface/optimum-habana/tree/main/examples/contrastive-image-text)</li> |
| BridgeTower | :heavy_check_mark: | :heavy_check_mark: | <li>[contrastive image-text training](https://github.com/huggingface/optimum-habana/tree/main/examples/contrastive-image-text)</li> |
| ESMFold |   | <div style="text-align:left"><li>Single card</li></div> | <li>[protein folding](https://github.com/huggingface/optimum-habana/tree/main/examples/protein-folding)</li> |
| Blip |   | <div style="text-align:left"><li>Single card</li></div> | <li>[visual question answering](https://github.com/huggingface/optimum-habana/tree/main/examples/visual-question-answering)</li><li>[image to text](https://github.com/huggingface/optimum-habana/tree/main/examples/image-to-text)</li> |
| OWLViT |   | <div style="text-align:left"><li>Single card</li></div> | <li>[zero shot object detection](https://github.com/huggingface/optimum-habana/tree/main/examples/zero-shot-object-detection)</li> |

</div>

- Diffusers:

<div align="center">

| Architecture     | Training | Inference            | Tasks |
|------------------|:--------:|:--------------------:|:------|
| Stable Diffusion | <li>[textual inversion](https://github.com/huggingface/optimum-habana/tree/main/examples/stable-diffusion/training#textual-inversion)</li><li>[ControlNet](https://github.com/huggingface/optimum-habana/tree/main/examples/stable-diffusion/training#controlnet-training)</li> | <li>Single card</li> | <li>[text-to-image generation](https://github.com/huggingface/optimum-habana/tree/main/examples/stable-diffusion)</li> |
| Stable Diffusion XL | <li>[fine-tuning](https://github.com/huggingface/optimum-habana/tree/main/examples/stable-diffusion/training#fine-tuning-for-stable-diffusion-xl)</li> | <li>Single card</li> | <li>[text-to-image generation](https://github.com/huggingface/optimum-habana/tree/main/examples/stable-diffusion)</li> |
| LDM3D            |          | <li>Single card</li> | <li>[text-to-image generation](https://github.com/huggingface/optimum-habana/tree/main/examples/stable-diffusion)</li> |

</div>

- TRL:

<div align="center">

| Architecture     | Training | Inference            | Tasks                                                                                          |
|------------------|:--------:|:--------------------:|:-----------------------------------------------------------------------------------------------|
| Llama 2          | :heavy_check_mark: |           | <li>[DPO Pipeline](https://github.com/huggingface/optimum-habana/tree/main/examples/trl)</li>  |
| Llama 2          | :heavy_check_mark: |           | <li>[PPO Pipeline](https://github.com/huggingface/optimum-habana/tree/main/examples/trl)</li>  |
| Stable Diffusion | :heavy_check_mark: |           | <li>[DDPO Pipeline](https://github.com/huggingface/optimum-habana/tree/main/examples/trl)</li> |

</div>

Other models and tasks supported by the 🤗 Transformers and 🤗 Diffusers library may also work. You can refer to this [section](https://github.com/huggingface/optimum-habana#how-to-use-it) for using them with 🤗 Optimum Habana. Besides, [this page](https://github.com/huggingface/optimum-habana/tree/main/examples) explains how to modify any [example](https://github.com/huggingface/transformers/tree/main/examples/pytorch) from the 🤗 Transformers library to make it work with 🤗 Optimum Habana.

If you find any issues while using those, please open an issue or a pull request.

After training your model, feel free to submit it to the Intel [leaderboard](https://huggingface.co/spaces/Intel/powered_by_intel_llm_leaderboard) which is designed to evaluate, score, and rank open-source LLMs that have been pre-trained or fine-tuned on Intel Hardwares. Models submitted to the leaderboard will be evaluated on the Intel Developer Cloud. The evaluation platform consists of Gaudi Accelerators and Xeon CPUs running benchmarks from the Eleuther AI Language Model Evaluation Harness.

## Gaudi Setup

Please refer to Habana Gaudi's official [installation guide](https://docs.habana.ai/en/latest/Installation_Guide/index.html).

> Tests should be run in a Docker container based on Habana Docker images.
>
> The current version has been validated for SynapseAI 1.15.


## Development

Check the [contributor guide](https://github.com/huggingface/optimum/blob/main/CONTRIBUTING.md) for instructions.
