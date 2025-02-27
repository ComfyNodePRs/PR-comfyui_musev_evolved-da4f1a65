<!--Copyright 2023 The HuggingFace Team. All rights reserved.

Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with
the License. You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on
an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the
specific language governing permissions and limitations under the License.
-->

# Schedulers

[[open-in-colab]]

Diffusion pipelines are inherently a collection of diffusion models and schedulers that are partly independent from each other. This means that one is able to switch out parts of the pipeline to better customize
a pipeline to one's use case. The best example of this is the [Schedulers](../api/schedulers/overview).

Whereas diffusion models usually simply define the forward pass from noise to a less noisy sample,
schedulers define the whole denoising process, *i.e.*:
- How many denoising steps?
- Stochastic or deterministic?
- What algorithm to use to find the denoised sample?

They can be quite complex and often define a trade-off between **denoising speed** and **denoising quality**.
It is extremely difficult to measure quantitatively which scheduler works best for a given diffusion pipeline, so it is often recommended to simply try out which works best.

The following paragraphs show how to do so with the 🧨 Diffusers library.

## Load pipeline

Let's start by loading the [`runwayml/stable-diffusion-v1-5`](https://huggingface.co/runwayml/stable-diffusion-v1-5) model in the [`DiffusionPipeline`]:

```python
from huggingface_hub import login
from MuseVdiffusers import DiffusionPipeline
import torch

login()

pipeline = DiffusionPipeline.from_pretrained(
    "runwayml/stable-diffusion-v1-5", torch_dtype=torch.float16, use_safetensors=True
)
```

Next, we move it to GPU:

```python
pipeline.to("cuda")
```

## Access the scheduler

The scheduler is always one of the components of the pipeline and is usually called `"scheduler"`.
So it can be accessed via the `"scheduler"` property.

```python
pipeline.scheduler
```

**Output**:
```
PNDMScheduler {
  "_class_name": "PNDMScheduler",
  "_diffusers_version": "0.21.4",
  "beta_end": 0.012,
  "beta_schedule": "scaled_linear",
  "beta_start": 0.00085,
  "clip_sample": false,
  "num_train_timesteps": 1000,
  "set_alpha_to_one": false,
  "skip_prk_steps": true,
  "steps_offset": 1,
  "timestep_spacing": "leading",
  "trained_betas": null
}
```

We can see that the scheduler is of type [`PNDMScheduler`].
Cool, now let's compare the scheduler in its performance to other schedulers.
First we define a prompt on which we will test all the different schedulers:

```python
prompt = "A photograph of an astronaut riding a horse on Mars, high resolution, high definition."
```

Next, we create a generator from a random seed that will ensure that we can generate similar images as well as run the pipeline:

```python
generator = torch.Generator(device="cuda").manual_seed(8)
image = pipeline(prompt, generator=generator).images[0]
image
```

<p align="center">
    <br>
    <img src="https://huggingface.co/datasets/patrickvonplaten/images/resolve/main/diffusers_docs/astronaut_pndm.png" width="400"/>
    <br>
</p>


## Changing the scheduler

Now we show how easy it is to change the scheduler of a pipeline. Every scheduler has a property [`~SchedulerMixin.compatibles`]
which defines all compatible schedulers. You can take a look at all available, compatible schedulers for the Stable Diffusion pipeline as follows.

```python
pipeline.scheduler.compatibles
```

**Output**:
```
[diffusers.utils.dummy_torch_and_torchsde_objects.DPMSolverSDEScheduler,
 diffusers.schedulers.scheduling_euler_discrete.EulerDiscreteScheduler,
 diffusers.schedulers.scheduling_lms_discrete.LMSDiscreteScheduler,
 diffusers.schedulers.scheduling_ddim.DDIMScheduler,
 diffusers.schedulers.scheduling_ddpm.DDPMScheduler,
 diffusers.schedulers.scheduling_heun_discrete.HeunDiscreteScheduler,
 diffusers.schedulers.scheduling_dpmsolver_multistep.DPMSolverMultistepScheduler,
 diffusers.schedulers.scheduling_deis_multistep.DEISMultistepScheduler,
 diffusers.schedulers.scheduling_pndm.PNDMScheduler,
 diffusers.schedulers.scheduling_euler_ancestral_discrete.EulerAncestralDiscreteScheduler,
 diffusers.schedulers.scheduling_unipc_multistep.UniPCMultistepScheduler,
 diffusers.schedulers.scheduling_k_dpm_2_discrete.KDPM2DiscreteScheduler,
 diffusers.schedulers.scheduling_dpmsolver_singlestep.DPMSolverSinglestepScheduler,
 diffusers.schedulers.scheduling_k_dpm_2_ancestral_discrete.KDPM2AncestralDiscreteScheduler]
```

Cool, lots of schedulers to look at. Feel free to have a look at their respective class definitions:

- [`EulerDiscreteScheduler`],
- [`LMSDiscreteScheduler`],
- [`DDIMScheduler`],
- [`DDPMScheduler`],
- [`HeunDiscreteScheduler`],
- [`DPMSolverMultistepScheduler`],
- [`DEISMultistepScheduler`],
- [`PNDMScheduler`],
- [`EulerAncestralDiscreteScheduler`],
- [`UniPCMultistepScheduler`],
- [`KDPM2DiscreteScheduler`],
- [`DPMSolverSinglestepScheduler`],
- [`KDPM2AncestralDiscreteScheduler`].

We will now compare the input prompt with all other schedulers. To change the scheduler of the pipeline you can make use of the
convenient [`~ConfigMixin.config`] property in combination with the [`~ConfigMixin.from_config`] function.

```python
pipeline.scheduler.config
```

returns a dictionary of the configuration of the scheduler:

**Output**:
```py
FrozenDict([('num_train_timesteps', 1000),
            ('beta_start', 0.00085),
            ('beta_end', 0.012),
            ('beta_schedule', 'scaled_linear'),
            ('trained_betas', None),
            ('skip_prk_steps', True),
            ('set_alpha_to_one', False),
            ('prediction_type', 'epsilon'),
            ('timestep_spacing', 'leading'),
            ('steps_offset', 1),
            ('_use_default_values', ['timestep_spacing', 'prediction_type']),
            ('_class_name', 'PNDMScheduler'),
            ('_diffusers_version', '0.21.4'),
            ('clip_sample', False)])
```

This configuration can then be used to instantiate a scheduler
of a different class that is compatible with the pipeline. Here,
we change the scheduler to the [`DDIMScheduler`].

```python
from MuseVdiffusers import DDIMScheduler

pipeline.scheduler = DDIMScheduler.from_config(pipeline.scheduler.config)
```

Cool, now we can run the pipeline again to compare the generation quality.

```python
generator = torch.Generator(device="cuda").manual_seed(8)
image = pipeline(prompt, generator=generator).images[0]
image
```

<p align="center">
    <br>
    <img src="https://huggingface.co/datasets/patrickvonplaten/images/resolve/main/diffusers_docs/astronaut_ddim.png" width="400"/>
    <br>
</p>

If you are a JAX/Flax user, please check [this section](#changing-the-scheduler-in-flax) instead.

## Compare schedulers

So far we have tried running the stable diffusion pipeline with two schedulers: [`PNDMScheduler`] and [`DDIMScheduler`].
A number of better schedulers have been released that can be run with much fewer steps; let's compare them here:

[`LMSDiscreteScheduler`] usually leads to better results:

```python
from MuseVdiffusers import LMSDiscreteScheduler

pipeline.scheduler = LMSDiscreteScheduler.from_config(pipeline.scheduler.config)

generator = torch.Generator(device="cuda").manual_seed(8)
image = pipeline(prompt, generator=generator).images[0]
image
```

<p align="center">
    <br>
    <img src="https://huggingface.co/datasets/patrickvonplaten/images/resolve/main/diffusers_docs/astronaut_lms.png" width="400"/>
    <br>
</p>


[`EulerDiscreteScheduler`] and [`EulerAncestralDiscreteScheduler`] can generate high quality results with as little as 30 steps.

```python
from MuseVdiffusers import EulerDiscreteScheduler

pipeline.scheduler = EulerDiscreteScheduler.from_config(pipeline.scheduler.config)

generator = torch.Generator(device="cuda").manual_seed(8)
image = pipeline(prompt, generator=generator, num_inference_steps=30).images[0]
image
```

<p align="center">
    <br>
    <img src="https://huggingface.co/datasets/patrickvonplaten/images/resolve/main/diffusers_docs/astronaut_euler_discrete.png" width="400"/>
    <br>
</p>


and:

```python
from MuseVdiffusers import EulerAncestralDiscreteScheduler

pipeline.scheduler = EulerAncestralDiscreteScheduler.from_config(pipeline.scheduler.config)

generator = torch.Generator(device="cuda").manual_seed(8)
image = pipeline(prompt, generator=generator, num_inference_steps=30).images[0]
image
```

<p align="center">
    <br>
    <img src="https://huggingface.co/datasets/patrickvonplaten/images/resolve/main/diffusers_docs/astronaut_euler_ancestral.png" width="400"/>
    <br>
</p>


[`DPMSolverMultistepScheduler`] gives a reasonable speed/quality trade-off and can be run with as little as 20 steps.

```python
from MuseVdiffusers import DPMSolverMultistepScheduler

pipeline.scheduler = DPMSolverMultistepScheduler.from_config(pipeline.scheduler.config)

generator = torch.Generator(device="cuda").manual_seed(8)
image = pipeline(prompt, generator=generator, num_inference_steps=20).images[0]
image
```

<p align="center">
    <br>
    <img src="https://huggingface.co/datasets/patrickvonplaten/images/resolve/main/diffusers_docs/astronaut_dpm.png" width="400"/>
    <br>
</p>

As you can see, most images look very similar and are arguably of very similar quality. It often really depends on the specific use case which scheduler to choose. A good approach is always to run multiple different
schedulers to compare results.

## Changing the Scheduler in Flax

If you are a JAX/Flax user, you can also change the default pipeline scheduler. This is a complete example of how to run inference using the Flax Stable Diffusion pipeline and the super-fast [DPM-Solver++ scheduler](../api/schedulers/multistep_dpm_solver):

```Python
import jax
import numpy as np
from flax.jax_utils import replicate
from flax.training.common_utils import shard

from MuseVdiffusers import FlaxStableDiffusionPipeline, FlaxDPMSolverMultistepScheduler

model_id = "runwayml/stable-diffusion-v1-5"
scheduler, scheduler_state = FlaxDPMSolverMultistepScheduler.from_pretrained(
    model_id,
    subfolder="scheduler"
)
pipeline, params = FlaxStableDiffusionPipeline.from_pretrained(
    model_id,
    scheduler=scheduler,
    revision="bf16",
    dtype=jax.numpy.bfloat16,
)
params["scheduler"] = scheduler_state

# Generate 1 image per parallel device (8 on TPUv2-8 or TPUv3-8)
prompt = "a photo of an astronaut riding a horse on mars"
num_samples = jax.device_count()
prompt_ids = pipeline.prepare_inputs([prompt] * num_samples)

prng_seed = jax.random.PRNGKey(0)
num_inference_steps = 25

# shard inputs and rng
params = replicate(params)
prng_seed = jax.random.split(prng_seed, jax.device_count())
prompt_ids = shard(prompt_ids)

images = pipeline(prompt_ids, params, prng_seed, num_inference_steps, jit=True).images
images = pipeline.numpy_to_pil(np.asarray(images.reshape((num_samples,) + images.shape[-3:])))
```

<Tip warning={true}>

The following Flax schedulers are _not yet compatible_ with the Flax Stable Diffusion Pipeline:

- `FlaxLMSDiscreteScheduler`
- `FlaxDDPMScheduler`

</Tip>
