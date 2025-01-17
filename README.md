# Low-rank Adaptation for Fast Text-to-Image Diffusion Fine-tuning

<!-- #region -->
<p align="center">
<img  src="contents/alpha_scale.gif">
</p>
<!-- #endregion -->

> Using LORA to fine tune on illustration dataset : $W = W_0 + \alpha \Delta W$, where $\alpha$ is the merging ratio. Above gif is scaling alpha from 0 to 1. Setting alpha to 0 is same as using the original model, and setting alpha to 1 is same as using the fully fine-tuned model.

<!-- #region -->
<p align="center">
<img  src="contents/disney_lora.jpg">
</p>
<!-- #endregion -->

> "style of sks, baby lion", with disney-style LORA model.

<!-- #region -->
<p align="center">
<img  src="contents/pop_art.jpg">
</p>
<!-- #endregion -->

> "style of sks, superman", with pop-art style LORA model.

# Web Demo

Integrated into [Huggingface Spaces 🤗](https://huggingface.co/spaces) using [Gradio](https://github.com/gradio-app/gradio). Try out the Web Demo [![Hugging Face Spaces](https://img.shields.io/badge/%F0%9F%A4%97%20Hugging%20Face-Spaces-blue)](https://huggingface.co/spaces/ysharma/Low-rank-Adaptation)

## Main Features

- Fine-tune Stable diffusion models twice as faster than dreambooth method, by Low-rank Adaptation
- Get insanely small end result, easy to share and download.
- Easy to use, compatible with diffusers
- Sometimes even better performance than full fine-tuning (but left as future work for extensive comparisons)
- Merge checkpoints by merging LORA

# UPDATES & Notes

- **Converting to CKPT format for A1111's repo consumption!** (Thanks to [jachiam](https://github.com/jachiam)'s conversion script)
- Img2Img Examples added.
- Please use large learning rate! Around 1e-4 worked well for me, but certainly not around 1e-6 which will not be able to learn anything.

# Lengthy Introduction

Thanks to the generous work of Stability AI and Huggingface, so many people have enjoyed fine-tuning stable diffusion models to fit their needs and generate higher fidelity images. **However, the fine-tuning process is very slow, and it is not easy to find a good balance between the number of steps and the quality of the results.**

Also, the final results (fully fined-tuned model) is very large. Some people instead works with textual-inversion as an alternative for this. But clearly this is suboptimal: textual inversion only creates a small word-embedding, and the final image is not as good as a fully fine-tuned model.

Well, what's the alternative? In the domain of LLM, researchers have developed Efficient fine-tuning methods. LORA, especially, tackles the very problem the community currently has: end users with Open-sourced stable-diffusion model want to try various other fine-tuned model that is created by the community, but the model is too large to download and use. LORA instead attempts to fine-tune the "residual" of the model instead of the entire model: i.e., train the $\Delta W$ instead of $W$.

$$
W' = W + \Delta W
$$

Where we can further decompose $\Delta W$ into low-rank matrices : $\Delta W = A B^T $, where $A, \in \mathbb{R}^{n \times d}, B \in \mathbb{R}^{m \times d}, d << n$.
This is the key idea of LORA. We can then fine-tune $A$ and $B$ instead of $W$. In the end, you get an insanely small model as $A$ and $B$ are much smaller than $W$.

Also, not all of the parameters need tuning: they found that often, $Q, K, V, O$ (i.e., attention layer) of the transformer model is enough to tune. (This is also the reason why the end result is so small). This repo will follow the same idea.

Enough of the lengthy introduction, let's get to the code.

# Installation

```bash
pip install git+https://github.com/cloneofsimo/lora.git
```

# Getting Started

## Fine-tuning Stable diffusion with LORA.

Basic usage is as follows: prepare sets of $A, B$ matrices in an unet model, and fine-tune them.

```python
from lora_diffusion import inject_trainable_lora, extract_lora_up_downs

...

unet = UNet2DConditionModel.from_pretrained(
    pretrained_model_name_or_path,
    subfolder="unet",
)
unet.requires_grad_(False)
unet_lora_params, train_names = inject_trainable_lora(unet)  # This will
# turn off all of the gradients of unet, except for the trainable LORA params.
optimizer = optim.Adam(
    itertools.chain(*unet_lora_params, text_encoder.parameters()), lr=1e-4
)
```

An example of this can be found in `train_lora_dreambooth.py`. Run this example with

```bash
run_lora_db.sh
```

## Loading, merging, and interpolating trained LORAs.

We've seen that people have been merging different checkpoints with different ratios, and this seems to be very useful to the community. LORA is extremely easy to merge.

By the nature of LORA, one can interpolate between different fine-tuned models by adding different $A, B$ matrices.

Currently, LORA cli has two options : merge full model with LORA, or merge LORA with LORA.

### Merging full model with LORA

```bash
$ lora_add --path_1 PATH_TO_DIFFUSER_FORMAT_MODEL --path_2 PATH_TO_LORA.PT --mode upl --alpha 1.0 --output_path OUTPUT_PATH
```

`path_1` can be both local path or huggingface model name. When adding LORA to unet, alpha is the constant as below:

$$
W' = W + \alpha \Delta W
$$

So, set alpha to 1.0 to fully add LORA. If the LORA seems to have too much effect (i.e., overfitted), set alpha to lower value. If the LORA seems to have too little effect, set alpha to higher than 1.0. You can tune these values to your needs. This value can be even slightly greater than 1.0!

**Example**

```bash
$ lora_add --path_1 stabilityai/stable-diffusion-2-base --path_2 lora_illust.pt --mode upl --alpha 1.0 --output_path merged_model
```

### Mergigng Full model with LORA and changing to original CKPT format

_TESTED WITH V2, V2.1 ONLY!_

Everything same as above, but with mode `upl-ckpt-v2` instead of `upl`.

```bash
$ lora_add --path_1 stabilityai/stable-diffusion-2-base --path_2 lora_illust.pt --mode upl-ckpt-v2 --alpha 1.2 --output_path merged_model.ckpt
```

### Merging LORA with LORA

```bash
$ lora_add --path_1 PATH_TO_LORA.PT --path_2 PATH_TO_LORA.PT --mode lpl --alpha 0.5 --output_path OUTPUT_PATH.PT
```

alpha is the ratio of the first model to the second model. i.e.,

$$
\Delta W = (\alpha A_1 + (1 - \alpha) A_2) (B_1 + (1 - \alpha) B_2)^T
$$

Set alpha to 0.5 to get the average of the two models. Set alpha close to 1.0 to get more effect of the first model, and set alpha close to 0.0 to get more effect of the second model.

**Example**

```bash
$ lora_add --path_1 lora_illust.pt --path_2 lora_pop.pt --alpha 0.3 --output_path lora_merged.pt
```

### Making Text2Img Inference with trained LORA

Checkout `scripts/run_inference.ipynb` for an example of how to make inference with LORA.

### Making Img2Img Inference with LORA

Checkout `scripts/run_img2img.ipynb` for an example of how to make inference with LORA.
