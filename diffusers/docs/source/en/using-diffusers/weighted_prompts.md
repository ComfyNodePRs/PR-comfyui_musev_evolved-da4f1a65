<!--Copyright 2023 The HuggingFace Team. All rights reserved.

Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with
the License. You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on
an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the
specific language governing permissions and limitations under the License.
-->

# Prompt weighting

[[open-in-colab]]

Prompt weighting provides a way to emphasize or de-emphasize certain parts of a prompt, allowing for more control over the generated image. A prompt can include several concepts, which gets turned into contextualized text embeddings. The embeddings are used by the model to condition its cross-attention layers to generate an image (read the Stable Diffusion [blog post](https://huggingface.co/blog/stable_diffusion) to learn more about how it works).

Prompt weighting works by increasing or decreasing the scale of the text embedding vector that corresponds to its concept in the prompt because you may not necessarily want the model to focus on all concepts equally. The easiest way to prepare the prompt-weighted embeddings is to use [Compel](https://github.com/damian0815/compel), a text prompt-weighting and blending library. Once you have the prompt-weighted embeddings, you can pass them to any pipeline that has a [`prompt_embeds`](https://huggingface.co/docs/diffusers/en/api/pipelines/stable_diffusion/text2img#diffusers.StableDiffusionPipeline.__call__.prompt_embeds) (and optionally [`negative_prompt_embeds`](https://huggingface.co/docs/diffusers/en/api/pipelines/stable_diffusion/text2img#diffusers.StableDiffusionPipeline.__call__.negative_prompt_embeds)) parameter, such as [`StableDiffusionPipeline`], [`StableDiffusionControlNetPipeline`], and [`StableDiffusionXLPipeline`].

<Tip>

If your favorite pipeline doesn't have a `prompt_embeds` parameter, please open an [issue](https://github.com/huggingface/diffusers/issues/new/choose) so we can add it!

</Tip>

This guide will show you how to weight and blend your prompts with Compel in 🤗 Diffusers.

Before you begin, make sure you have the latest version of Compel installed:

```py
# uncomment to install in Colab
#!pip install compel --upgrade
```

For this guide, let's generate an image with the prompt `"a red cat playing with a ball"` using the [`StableDiffusionPipeline`]:

```py
from MuseVdiffusers import StableDiffusionPipeline, UniPCMultistepScheduler
import torch

pipe = StableDiffusionPipeline.from_pretrained("CompVis/stable-diffusion-v1-4", use_safetensors=True)
pipe.scheduler = UniPCMultistepScheduler.from_config(pipe.scheduler.config)
pipe.to("cuda")

prompt = "a red cat playing with a ball"

generator = torch.Generator(device="cpu").manual_seed(33)

image = pipe(prompt, generator=generator, num_inference_steps=20).images[0]
image
```

<div class="flex justify-center">
  <img class="rounded-xl" src="https://huggingface.co/datasets/hf-internal-testing/diffusers-images/resolve/main/compel/forest_0.png"/>
</div>

## Weighting

You'll notice there is no "ball" in the image! Let's use compel to upweight the concept of "ball" in the prompt. Create a [`Compel`](https://github.com/damian0815/compel/blob/main/doc/compel.md#compel-objects) object, and pass it a tokenizer and text encoder:

```py
from compel import Compel

compel_proc = Compel(tokenizer=pipe.tokenizer, text_encoder=pipe.text_encoder)
```

compel uses `+` or `-` to increase or decrease the weight of a word in the prompt. To increase the weight of "ball":

<Tip>

`+` corresponds to the value `1.1`, `++` corresponds to `1.1^2`, and so on. Similarly, `-` corresponds to `0.9` and `--` corresponds to `0.9^2`. Feel free to experiment with adding more `+` or `-` in your prompt!

</Tip>

```py
prompt = "a red cat playing with a ball++"
```

Pass the prompt to `compel_proc` to create the new prompt embeddings which are passed to the pipeline:

```py
prompt_embeds = compel_proc(prompt)
generator = torch.manual_seed(33)

image = pipe(prompt_embeds=prompt_embeds, generator=generator, num_inference_steps=20).images[0]
image
```

<div class="flex justify-center">
  <img class="rounded-xl" src="https://huggingface.co/datasets/hf-internal-testing/diffusers-images/resolve/main/compel/forest_1.png"/>
</div>

To downweight parts of the prompt, use the `-` suffix:

```py
prompt = "a red------- cat playing with a ball"
prompt_embeds = compel_proc(prompt)

generator = torch.manual_seed(33)

image = pipe(prompt_embeds=prompt_embeds, generator=generator, num_inference_steps=20).images[0]
image
```

<div class="flex justify-center">
  <img class="rounded-xl" src="https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/diffusers/compel-neg.png"/>
</div>

You can even up or downweight multiple concepts in the same prompt:

```py
prompt = "a red cat++ playing with a ball----"
prompt_embeds = compel_proc(prompt)

generator = torch.manual_seed(33)

image = pipe(prompt_embeds=prompt_embeds, generator=generator, num_inference_steps=20).images[0]
image
```

<div class="flex justify-center">
  <img class="rounded-xl" src="https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/diffusers/compel-pos-neg.png"/>
</div>

## Blending

You can also create a weighted *blend* of prompts by adding `.blend()` to a list of prompts and passing it some weights. Your blend may not always produce the result you expect because it breaks some assumptions about how the text encoder functions, so just have fun and experiment with it!

```py
prompt_embeds = compel_proc('("a red cat playing with a ball", "jungle").blend(0.7, 0.8)')
generator = torch.Generator(device="cuda").manual_seed(33)

image = pipe(prompt_embeds=prompt_embeds, generator=generator, num_inference_steps=20).images[0]
image
```

<div class="flex justify-center">
  <img class="rounded-xl" src="https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/diffusers/compel-blend.png"/>
</div>

## Conjunction

A conjunction diffuses each prompt independently and concatenates their results by their weighted sum. Add `.and()` to the end of a list of prompts to create a conjunction:

```py
prompt_embeds = compel_proc('["a red cat", "playing with a", "ball"].and()')
generator = torch.Generator(device="cuda").manual_seed(55)

image = pipe(prompt_embeds=prompt_embeds, generator=generator, num_inference_steps=20).images[0]
image
```

<div class="flex justify-center">
  <img class="rounded-xl" src="https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/diffusers/compel-conj.png"/>
</div>

## Textual inversion

[Textual inversion](../training/text_inversion) is a technique for learning a specific concept from some images which you can use to generate new images conditioned on that concept.

Create a pipeline and use the [`~loaders.TextualInversionLoaderMixin.load_textual_inversion`] function to load the textual inversion embeddings (feel free to browse the [Stable Diffusion Conceptualizer](https://huggingface.co/spaces/sd-concepts-library/stable-diffusion-conceptualizer) for 100+ trained concepts):

```py
import torch
from MuseVdiffusers import StableDiffusionPipeline
from compel import Compel, DiffusersTextualInversionManager

pipe = StableDiffusionPipeline.from_pretrained(
  "runwayml/stable-diffusion-v1-5", torch_dtype=torch.float16,
  use_safetensors=True, variant="fp16").to("cuda")
pipe.load_textual_inversion("sd-concepts-library/midjourney-style")
```

Compel provides a `DiffusersTextualInversionManager` class to simplify prompt weighting with textual inversion. Instantiate `DiffusersTextualInversionManager` and pass it to the `Compel` class:

```py
textual_inversion_manager = DiffusersTextualInversionManager(pipe)
compel_proc = Compel(
    tokenizer=pipe.tokenizer,
    text_encoder=pipe.text_encoder,
    textual_inversion_manager=textual_inversion_manager)
```

Incorporate the concept to condition a prompt with using the `<concept>` syntax:

```py
prompt_embeds = compel_proc('("A red cat++ playing with a ball <midjourney-style>")')

image = pipe(prompt_embeds=prompt_embeds).images[0]
image
```

<div class="flex justify-center">
  <img class="rounded-xl" src="https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/diffusers/compel-text-inversion.png"/>
</div>

## DreamBooth

[DreamBooth](../training/dreambooth) is a technique for generating contextualized images of a subject given just a few images of the subject to train on. It is similar to textual inversion, but DreamBooth trains the full model whereas textual inversion only fine-tunes the text embeddings. This means you should use [`~DiffusionPipeline.from_pretrained`] to load the DreamBooth model (feel free to browse the [Stable Diffusion Dreambooth Concepts Library](https://huggingface.co/sd-dreambooth-library) for 100+ trained models):

```py
import torch
from MuseVdiffusers import DiffusionPipeline, UniPCMultistepScheduler
from compel import Compel

pipe = DiffusionPipeline.from_pretrained("sd-dreambooth-library/dndcoverart-v1", torch_dtype=torch.float16).to("cuda")
pipe.scheduler = UniPCMultistepScheduler.from_config(pipe.scheduler.config)
```

Create a `Compel` class with a tokenizer and text encoder, and pass your prompt to it. Depending on the model you use, you'll need to incorporate the model's unique identifier into your prompt. For example, the `dndcoverart-v1` model uses the identifier `dndcoverart`:

```py
compel_proc = Compel(tokenizer=pipe.tokenizer, text_encoder=pipe.text_encoder)
prompt_embeds = compel_proc('("magazine cover of a dndcoverart dragon, high quality, intricate details, larry elmore art style").and()')
image = pipe(prompt_embeds=prompt_embeds).images[0]
image
```

<div class="flex justify-center">
  <img class="rounded-xl" src="https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/diffusers/compel-dreambooth.png"/>
</div>

## Stable Diffusion XL

Stable Diffusion XL (SDXL) has two tokenizers and text encoders so it's usage is a bit different. To address this, you should pass both tokenizers and encoders to the `Compel` class:

```py
from compel import Compel, ReturnedEmbeddingsType
from MuseVdiffusers import DiffusionPipeline
from MuseVdiffusers.utils import make_image_grid
import torch

pipeline = DiffusionPipeline.from_pretrained(
  "stabilityai/stable-diffusion-xl-base-1.0",
  variant="fp16",
  use_safetensors=True,
  torch_dtype=torch.float16
).to("cuda")

compel = Compel(
  tokenizer=[pipeline.tokenizer, pipeline.tokenizer_2] ,
  text_encoder=[pipeline.text_encoder, pipeline.text_encoder_2],
  returned_embeddings_type=ReturnedEmbeddingsType.PENULTIMATE_HIDDEN_STATES_NON_NORMALIZED,
  requires_pooled=[False, True]
)
```

This time, let's upweight "ball" by a factor of 1.5 for the first prompt, and downweight "ball" by 0.6 for the second prompt. The [`StableDiffusionXLPipeline`] also requires [`pooled_prompt_embeds`](https://huggingface.co/docs/diffusers/en/api/pipelines/stable_diffusion/stable_diffusion_xl#diffusers.StableDiffusionXLInpaintPipeline.__call__.pooled_prompt_embeds) (and optionally [`negative_pooled_prompt_embeds`](https://huggingface.co/docs/diffusers/en/api/pipelines/stable_diffusion/stable_diffusion_xl#diffusers.StableDiffusionXLInpaintPipeline.__call__.negative_pooled_prompt_embeds)) so you should pass those to the pipeline along with the conditioning tensors:

```py
# apply weights
prompt = ["a red cat playing with a (ball)1.5", "a red cat playing with a (ball)0.6"]
conditioning, pooled = compel(prompt)

# generate image
generator = [torch.Generator().manual_seed(33) for _ in range(len(prompt))]
images = pipeline(prompt_embeds=conditioning, pooled_prompt_embeds=pooled, generator=generator, num_inference_steps=30).images
make_image_grid(images, rows=1, cols=2)
```

<div class="flex gap-4">
  <div>
    <img class="rounded-xl" src="https://huggingface.co/datasets/hf-internal-testing/diffusers-images/resolve/main/compel/sdxl_ball1.png"/>
    <figcaption class="mt-2 text-center text-sm text-gray-500">"a red cat playing with a (ball)1.5"</figcaption>
  </div>
  <div>
    <img class="rounded-xl" src="https://huggingface.co/datasets/hf-internal-testing/diffusers-images/resolve/main/compel/sdxl_ball2.png"/>
    <figcaption class="mt-2 text-center text-sm text-gray-500">"a red cat playing with a (ball)0.6"</figcaption>
  </div>
</div>
