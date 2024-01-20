## Talkinghead

<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-refresh-toc -->
**Table of Contents**

- [Talkinghead](#talkinghead)
    - [Introduction](#introduction)
    - [Live mode](#live-mode)
        - [Configuration](#configuration)
        - [Emotion templates](#emotion-templates)
        - [Animator configuration](#animator-configuration)
        - [Postprocessor configuration](#postprocessor-configuration)
        - [Postprocessor example: HDR, scifi hologram](#postprocessor-example-hdr-scifi-hologram)
        - [Postprocessor example: cheap video camera, amber monochrome computer monitor](#postprocessor-example-cheap-video-camera-amber-monochrome-computer-monitor)
        - [Postprocessor example: HDR, cheap video camera, 1980s VHS tape](#postprocessor-example-hdr-cheap-video-camera-1980s-vhs-tape)
        - [Complete example: animator and postprocessor settings](#complete-example-animator-and-postprocessor-settings)
    - [Manual poser](#manual-poser)
    - [Troubleshooting](#troubleshooting)
        - [Low framerate](#low-framerate)
        - [Low VRAM - what to do?](#low-vram---what-to-do)
        - [Missing model at startup](#missing-model-at-startup)
    - [Creating a character](#creating-a-character)
        - [Tips for Stable Diffusion](#tips-for-stable-diffusion)
    - [Acknowledgements](#acknowledgements)

<!-- markdown-toc end -->

### Introduction

This module renders a **live, AI-based custom anime avatar for your AI character**.

In contrast to VTubing software, `talkinghead` is an **AI-based** character animation technology, which produces animation from just **one static 2D image**. This makes creating new characters accessible and cost-effective. All you need is Stable Diffusion and an image editor to get started! Additionally, you can experiment with your character's appearance in an agile way, animating each revision of your design.

The animator is built on top of a deep learning model, so optimal performance requires a fast GPU. The model can vary the character's expression, and pose some joints by up to 15 degrees. This allows producing parametric animation on the fly, just like from a traditional 2D or 3D model - but from a small generative AI. Modern GPUs have enough compute to do this in realtime.

You only need to provide **one** expression for your character. The model automatically generates the rest of the 28, and seamlessly animates between them. The expressions are based on *emotion templates*, which are essentially just morph settings. To make it convenient to edit the templates, we provide a GUI editor (the manual poser), where you can see how the resulting expression looks on your character.

As with any AI technology, there are limitations. The AI-generated animation frames may not look perfect, and in particular the model does not support characters wearing large hats or props. For details (and many example outputs), refer to the [tech report](https://web.archive.org/web/20220606125507/https://pkhungurn.github.io/talking-head-anime-3/full.html) by the model's original author.

Still images do not do the system justice; the realtime animation is a large part of its appeal. Preferences vary here. If you have the hardware, try it, you might like it. Especially, if you like to make new characters, or to tweak your character design often, this is the animator for you. On the other hand, if you prefer still images, and focus on one particular design, you may get more aesthetically pleasing results by inpainting static expression sprites in Stable Diffusion.

Currently, `talkinghead` is focused on providing 1-on-1 interactions with your AI character, so support for group chats and visual novel mode are not included, nor planned. However, as a community-driven project, we appreciate any feedback or especially code or documentation contributions towards the growth and development of this extension.


### Live mode

To activate the live mode:

- Configure your *SillyTavern-extras* installation so that it loads the `talkinghead` module. This makes the backend available.
- Ensure that your character has a `SillyTavern/public/characters/yourcharacternamehere/talkinghead.png`. This is the input image for the animator.
  - You can upload one in the *SillyTavern* settings, in *Extensions ⊳ Character Expressions*.
- To enable **talkinghead mode** in *Character Expressions*, check the checkbox *Extensions ⊳ Character Expressions ⊳ Image Type - talkinghead (extras)*.

CUDA (*SillyTavern-extras* option `--talkinghead-gpu`) is very highly recommended. As of late 2023, a recent GPU is also recommended. For example, on a laptop with an RTX 3070 Ti mobile GPU, and the `separable_half` THA3 poser model (fastest and smallest; default when running on GPU), you can expect ≈40-50 FPS render performance. VRAM usage in this case is about 520 MB. CPU mode exists, but is very slow, about ≈2 FPS on an i7-12700H.

We rate-limit the output to 25 FPS (maximum) to avoid DoSing the SillyTavern GUI, and attempt to reach a constant 25 FPS. If the renderer runs faster, the average GPU usage will be lower, because the animation engine only generates as many frames as are actually consumed. If the renderer runs slower, the latest available frame will be re-sent as many times as needed, to isolate the client side from any render hiccups.

To customize which model variant of the THA3 poser to use, and where to install the models from, see the `--talkinghead-model=...` and `--talkinghead-models=...` options, respectively.

If the directory `talkinghead/tha3/models/` (under the top level of *SillyTavern-extras*) does not exist, the model files are automatically downloaded from HuggingFace and installed there.

#### Configuration

The live mode is configured per-character, via files **at the client end**:

- `SillyTavern/public/characters/yourcharacternamehere/talkinghead.png`: required. The input image for the animator.
  - The `talkinghead` extension does not use or even see the other `.png` files. They are used by *Character Expressions* when *talkinghead mode* is disabled.
- `SillyTavern/public/characters/yourcharacternamehere/_animator.json`: optional. Animator and postprocessor settings.
  - If a character does not have this file, default settings are used.
- `SillyTavern/public/characters/yourcharacternamehere/_emotions.json`: optional. Custom emotion templates.
  - If a character does not have this file, default settings are used. Most of the time, there is no need to customize the emotion templates per-character.
  - At the client end, only this one file is needed (or even supported) to customize the emotion templates.

#### Emotion templates

Emotion templates use the same format as the factory settings in `SillyTavern-extras/talkinghead/emotions/_defaults.json`. The manual poser app included with `talkinghead` is a GUI editor for these templates.

The batch export of the manual poser produces a set of static expression images (and corresponding emotion templates), but also an `_emotions.json`, in your chosen output folder. You can use this file at the client end as `SillyTavern/public/characters/yourcharacternamehere/_emotions.json`. This is convenient if you have customized your emotion templates, and wish to share one of your characters with other users, making it automatically use your version of the templates.

Emotion template lookup order is:

- The set of custom templates sent by the ST client, read from `SillyTavern/public/characters/yourcharacternamehere/_emotions.json` if it exists.
- Server defaults, from the individual files `SillyTavern-extras/talkinghead/emotions/emotionnamehere.json`.
  - These are customizable. You can e.g. overwrite `curiosity.json` to change the default template for the *"curiosity"* emotion.
  - **IMPORTANT**: *However, updating SillyTavern-extras from git may overwrite your changes to the server-side default emotion templates.*
- Factory settings, from `SillyTavern-extras/talkinghead/emotions/_defaults.json`.
  - **IMPORTANT**: Never overwrite or remove this file.

Any emotion that is missing from a particular level in the lookup order falls through to be looked up at the next level.

If you want to edit the emotion templates manually (without using the GUI) for some reason, the following may be useful sources of information:

- `posedict_keys` in [`talkinghead/tha3/app/util.py`](tha3/app/util.py) lists the morphs available in THA3.
- [`talkinghead/tha3/poser/modes/pose_parameters.py`](tha3/poser/modes/pose_parameters.py) contains some more detail.
  - *"Arity 2"* means `posedict_keys` has separate left/right morphs.
- The GUI panel implementations in [`talkinghead/tha3/app/manual_poser.py`](tha3/app/manual_poser.py).

Any morph that is not mentioned for a particular emotion defaults to zero. Thus only those morphs that have nonzero values need to be mentioned.


#### Animator configuration

*The available settings keys and examples are kept up-to-date on a best-effort basis, but there is a risk of this documentation being out of date. When in doubt, refer to the actual source code, which comes with extensive docstrings and comments. The final authoritative source is the implementation itself.*

The file `SillyTavern/public/characters/yourcharacternamehere/_animator.json` contains the animator and postprocessor settings. For any setting not specified in the file, the default value is used.

The idea is that this allows giving some personality to different characters; for example, they may sway by different amounts, the breathing cycle duration may be different, and importantly, the postprocessor settings may be different - which allows e.g. making a specific character into a scifi hologram, while others render normally.

Here is a complete example of `_animator.json`, showing the default values:

```
{"target_fps": 25,
 "pose_interpolator_step": 0.1,
 "blink_interval_min": 2.0,
 "blink_interval_max": 5.0,
 "blink_probability": 0.03,
 "blink_confusion_duration": 10.0,
 "talking_fps": 12,
 "talking_morph": "mouth_aaa_index",
 "sway_morphs": ["head_x_index", "head_y_index", "neck_z_index", "body_y_index", "body_z_index"],
 "sway_interval_min": 5.0,
 "sway_interval_max": 10.0,
 "sway_macro_strength": 0.6,
 "sway_micro_strength": 0.02,
 "breathing_cycle_duration": 4.0,
 "postprocessor_chain": []}
```

where:

- `target_fps`: Desired output frames per second. Note this only affects smoothness of the output (provided that the hardware is fast enough). The speed at which the animation evolves is based on wall time. Snapshots are rendered at the target FPS, or if the hardware is slower, then as often as hardware allows. *Recommendation*: For smooth animation, make the FPS lower than what your hardware could produce, so that some compute remains untapped, available to smooth over the occasional hiccup from other running programs.
- `pose_interpolator_step`: A value such that `0 < step <= 1`. Applied at each frame at a reference of 25 FPS (to standardize the meaning of the setting), with automatic internal FPS-correction to the actual output FPS. Note that the animation is nonlinear. The step controls how much of the *remaining distance* to the current target pose is covered in 1/25 seconds.
- `blink_interval_min`: seconds. After blinking, lower limit for random minimum time until next blink is allowed.
- `blink_interval_max`: seconds. After blinking, upper limit for random minimum time until next blink is allowed.
- `blink_probability`: Applied at each frame at a reference of 25 FPS, with automatic internal FPS-correction to the actual output FPS. This is the probability of initiating a blink in each 1/25 second interval.
- `blink_confusion_duration`: seconds. Upon entering the `"confusion"` emotion, the character may blink quickly in succession, temporarily disregarding the blink interval settings. This sets how long that state lasts.
- `talking_fps`: How often to re-randomize the mouth during the talking animation. The default value is based on the fact that early 2000s anime used ~12 FPS as the fastest actual framerate of new cels, not counting camera panning effects and such.
- `talking_morph`: Which mouth-open morph to use for talking. For available values, see `posedict_keys` in [`talkinghead/tha3/app/util.py`](tha3/app/util.py).
- `sway_morphs`: Which morphs participate in the sway (fidgeting) animation. This setting is mainly useful for disabling some or all of them, e.g. for a robot character. For available values, see `posedict_keys` in [`talkinghead/tha3/app/util.py`](tha3/app/util.py).
- `sway_interval_min`: seconds. Lower limit for random time interval until randomizing a new target pose for the sway animation.
- `sway_interval_max`: seconds. Upper limit for random time interval until randomizing a new target pose for the sway animation.
- `sway_macro_strength`: A value such that `0 < strength <= 1`. In the sway target pose, this sets the maximum absolute deviation from the target pose specified by the current emotion, but also the maximum deviation from the center position. The setting is applied to each sway morph separately. The emotion pose itself may use higher values for the morphs; in such cases, sway will only occur toward the center. For details, see `compute_sway_target_pose` in [`talkinghead/tha3/app/app.py`](tha3/app/app.py).
- `sway_micro_strength`: A value such that `0 < strength <= 1`. This is the maximum absolute value of random noise added to the sway target pose at each frame. To this, no limiting is applied, other than a clamp of the final randomized value of each sway morph to the valid range [-1, 1]. A small amount of random jitter makes the character look less robotic.
- `breathing_cycle_duration`: seconds. The duration of a full cycle of the breathing animation.
- `postprocessor_chain`: Pixel-space glitch artistry settings. The default is empty (no postprocessing); see below for examples of what can be done with this. For details, see [`talkinghead/tha3/app/postprocessor.py`](tha3/app/postprocessor.py).

#### Postprocessor configuration

*The available settings keys and examples are kept up-to-date on a best-effort basis, but there is a risk of this documentation being out of date. When in doubt, refer to the actual source code, which comes with extensive docstrings and comments. The final authoritative source is the implementation itself.*

The postprocessor configuration is part of `_animator.json`, stored under the key `"postprocessor_chain"`.

Postprocessing requires some additional compute, depending on the filters used and their settings. When `talkinghead` runs on the GPU, also the postprocessor filters run on the GPU. In gaming technology terms, they are essentially fragment shaders, implemented in PyTorch.

The filters in the postprocessor chain are applied to the image in the order in which they appear in the list. That is, the filters themselves support rendering in any order. However, for best results, it is useful to keep in mind the process a real physical signal would travel through:

*Light* ⊳ *Camera* ⊳ *Transport* ⊳ *Display*

and set the order for the filters based on that. However, this does not mean that there is just one correct ordering. Some filters are *general-use*, and may make sense at several points in the chain, depending on what you wish to simulate. Feel free to improvise, but make sure to understand why your filter chain makes sense.

The following postprocessing filters are available. Options for each filter are documented in the docstrings in [`talkinghead/tha3/app/postprocessor.py`](tha3/app/postprocessor.py).

**Light**:

- `bloom`: Bloom effect (fake HDR). Popular in early 2000s anime. Makes bright parts of the image bleed light into their surroundings, enhancing perceived contrast. Only makes sense when the talkinghead is rendered on a relatively dark background (such as the cyberpunk bedroom in the ST default backgrounds).

**Camera**:

- `chromatic_aberration`: Simulates the two types of chromatic aberration in a camera lens, axial (index of refraction varying w.r.t. wavelength) and transverse (focal distance varying w.r.t. wavelength).
- `vignetting`: Simulates vignetting, i.e. less light hitting the corners of a film frame or CCD sensor, causing the corners to be slightly darker than the center.

**Transport**:

Currently, we provide some filters that simulate a lo-fi analog video look.

- `analog_lowres`: Simulates a low-resolution analog video signal by blurring the image.
- `analog_badhsync`: Simulates bad horizontal synchronization (hsync) of an analog video signal, causing a wavy effect that causes the outline of the character to ripple.
- `analog_vhsglitches`: Simulates a damaged 1980s VHS tape. In each frame, causes random lines to glitch with VHS noise.
- `analog_vhstracking`: Simulates a 1980s VHS tape with bad tracking. The image floats up and down, and a band of VHS noise appears at the bottom.

**Display**:

- `translucency`: Makes the character translucent, as if a scifi hologram.
- `banding`: Simulates the look of a CRT display as it looks when filmed on video without syncing. Brighter and darker bands travel through the image.
- `scanlines`: Simulates CRT TV like scanlines. Optionally dynamic (flipping the dimmed field at each frame).
  - From my experiments with the Phosphor deinterlacer in VLC, which implements the same effect, dynamic mode for `scanlines` would look *absolutely magical* when synchronized with display refresh, closely reproducing the look of an actual CRT TV. However, that is not possible here. Thus, it looks best at low but reasonable FPS, and a very high display refresh rate, so that small timing variations will not make much of a difference in how long a given field is actually displayed on the physical monitor.
  - If the timing is too uneven, the illusion breaks. In that case, consider using the static mode (`"dynamic": false`).

**General use**:

- `alphanoise`: Adds noise to the alpha channel (translucency).
- `desaturate`: A desaturation filter with bells and whistles. Beside converting the image to grayscale, can optionally pass through colors that match the hue of a given RGB color (e.g. keep red things, while desaturating the rest), and tint the final result (e.g. for an amber monochrome computer monitor look).

The `alphanoise` filter could represent the display of a lo-fi scifi hologram, as well as noise in an analog video tape (which in this scheme belongs to "transport").

The `desaturate` filter could represent either a black and white video camera, or a monochrome display.

#### Postprocessor example: HDR, scifi hologram

The bloom works best on a dark background. We use `alphanoise` to add an imperfection to the simulated display device, causing individual pixels to dynamically vary in their alpha. The `banding` and `scanlines` filters complete the look of how holograms are often depicted in scifi video games and movies. The `"dynamic": true` makes the dimmed field (top or bottom) flip each frame, like on a CRT television.

```
"postprocessor_chain": [["bloom", {}],
                        ["translucency", {"alpha": 0.9}],
                        ["alphanoise", {"magnitude": 0.1, "sigma": 0.0}],
                        ["banding", {}],
                        ["scanlines", {"dynamic": true}]
                       ]
```

#### Postprocessor example: cheap video camera, amber monochrome computer monitor

We first simulate a cheap video camera with low-quality optics via the `chromatic_aberration` and `vignetting` filters.

We then use `desaturate` with the tint option to produce the amber monochrome look.

The `banding` and `scanlines` filters suit this look, so we apply them here, too. They could be left out to simulate a higher-quality display device. Setting `"dynamic": false` makes the scanlines stay stationary.

```
"postprocessor_chain": [["chromatic_aberration", {}],
                        ["vignetting", {}],
                        ["desaturate", {"tint_rgb": [1.0, 0.5, 0.2]}],
                        ["banding", {}],
                        ["scanlines", {"dynamic": false}]
                       ]
```

#### Postprocessor example: HDR, cheap video camera, 1980s VHS tape

After capturing the light with a cheap video camera (just like in the previous example), we simulate the effects of transporting the signal on a 1980s VHS tape. First, we blur the image with `analog_lowres`. Then we apply `alphanoise` with a nonzero `sigma` to make the noise blobs larger than a single pixel, and a rather high `magnitude`. This simulates the brightness noise on a VHS tape. Then we make the image ripple horizontally with `analog_badhsync`, and finally add a bad VHS tracking effect to complete the look.

Then we again render the output on a simulated CRT TV, as appropriate for the 1980s time period.

```
"postprocessor_chain": [["bloom", {}],
                        ["chromatic_aberration", {}],
                        ["vignetting", {}],
                        ["analog_lowres", {}],
                        ["alphanoise", {"magnitude": 0.2, "sigma": 2.0}],
                        ["analog_badhsync", {}],
                        ["analog_vhstracking", {}],
                        ["banding", {}],
                        ["scanlines", {"dynamic": true}]
                       ]
```

#### Complete example: animator and postprocessor settings

This example uses the default values for the animator (to give a template that is easy to tune), but sets up the postprocessor as in the "scifi hologram" example above.

To use this, save this **at the client end** as `SillyTavern/public/characters/yourcharacternamehere/_animator.json` (i.e. as `_animator.json`, in the same folder where the sprites for your character are), and make `talkinghead` reload your character (for example, by toggling `talkinghead` off and back on in the SillyTavern settings).

```json
{"target_fps": 25,
 "pose_interpolator_step": 0.1,
 "blink_interval_min": 2.0,
 "blink_interval_max": 5.0,
 "blink_probability": 0.03,
 "blink_confusion_duration": 10.0,
 "talking_fps": 12,
 "talking_morph": "mouth_aaa_index",
 "sway_morphs": ["head_x_index", "head_y_index", "neck_z_index", "body_y_index", "body_z_index"],
 "sway_interval_min": 5.0,
 "sway_interval_max": 10.0,
 "sway_macro_strength": 0.6,
 "sway_micro_strength": 0.02,
 "breathing_cycle_duration": 4.0,
 "postprocessor_chain": [["bloom", {}],
                         ["translucency", {"alpha": 0.9}],
                         ["alphanoise", {"magnitude": 0.1, "sigma": 0.0}],
                         ["banding", {}],
                         ["scanlines", {"dynamic": true}]
                        ]
}
```


### Manual poser

This is a standalone wxPython app that you can run locally on the machine where you installed *SillyTavern-extras*. It is based on the original manual poser app in the THA3 tech demo, but this version has some important new convenience features and usability improvements.

It uses the same models as the live mode. If the directory `talkinghead/tha3/models/` (under the top level of *SillyTavern-extras*) does not exist, the model files are automatically downloaded from HuggingFace and installed there.

With this app, you can:

- **Graphically edit the emotion templates** used by the live mode.
  - They are JSON files, found in `talkinghead/emotions/` under your *SillyTavern-extras* folder.
    - The GUI also has a dropdown to quickload any preset.
  - **NEVER** delete or modify `_defaults.json`. That file stores the factory settings, and the app will not run without it.
  - For blunder recovery: to reset an emotion back to its factory setting, see the `--factory-reset=EMOTION` option, which will use the factory settings to overwrite the corresponding emotion preset JSON. To reset **all** emotion presets to factory settings, see `--factory-reset-all`. Careful, these operations **cannot** be undone!
    - Currently, these options do **NOT** regenerate the example images also provided in `talkinghead/emotions/`.
- **Batch-generate the 28 static expression sprites** for a character.
  - Input is the same single static image format as used by the live mode.
  - You can then use the generated images as the static expression sprites for your AI character. No need to run the live mode.

To run the manual poser:

- Open a terminal in your `talkinghead` subdirectory
- `conda activate extras`
- `python -m tha3.app.manual_poser`.
  - For systems with `bash`, a convenience wrapper `./start_manual_poser.sh` is included.

Run the poser with the `--help` option for a description of its command-line options. The command-line options of the manual poser are **completely independent** from the options of *SillyTavern-extras* itself.

Currently, you can choose the device to run on (GPU or CPU), and which THA3 model to use. By default, the manual poser uses GPU and the `separable_float` model.

GPU mode gives the best response, but CPU mode (~2 FPS) is useful at least for batch-exporting static sprites when your VRAM is already full of AI.

To load a PNG image or emotion JSON, you can either use the buttons, their hotkeys, or **drag'n'drop a PNG or JSON** file from your favorite file manager into the source image pane.


### Troubleshooting

#### Low framerate

The poser is a deep-learning model. Each animation frame requires an inference pass. This requires lots of compute.

Thus, if you have a CUDA-capable GPU, enable GPU support by using the `--talkinghead-gpu` setting of *SillyTavern-extras*.

CPU mode is very slow, and without a redesign of the AI model (or distillation, like in the newer [THA4 paper](https://arxiv.org/abs/2311.17409)), there is not much that can be done. It is already running as fast as PyTorch can go, and the performance impact of everything except the posing engine is almost negligible.

#### Low VRAM - what to do?

Observe that the `--talkinghead-gpu` setting is independent of the CUDA device setting of the rest of *SillyTavern-extras*.

So in a low-VRAM environment such as a gaming laptop, you can run just `talkinghead` on the GPU (VRAM usage about 520 MB) to get acceptable animation performance, while running all other extras modules on the CPU. The `classify` or `summarize` AI modules do not require realtime performance, whereas `talkinghead` does.

#### Missing model at startup

The `separable_float` variant of the THA3 models was previously included in the *SillyTavern-extras* repository. However, `talkinghead` was recently (December 2023) changed to download these models from HuggingFace if necessary, so a local copy of the model is no longer provided in the repository.

Therefore, if you updated your *SillyTavern-extras* installation from *git*, it is likely that *git* deleted your local copy of that particular model, leading to an error message like:

```
FileNotFoundError: Model file /home/xxx/SillyTavern-extras/talkinghead/tha3/models/separable_float/eyebrow_decomposer.pt not found, please check the path.
```

The solution is to remove (or rename) your `SillyTavern-extras/talkinghead/tha3/models` directory, and restart *SillyTavern-extras*. If the model directory does not exist, `talkinghead` will download the models at the first run.

The models are actually shared between the live mode and the manual poser, so it doesn't matter which one you run first.


### Creating a character

To create an AI avatar that `talkinghead` understands:

- The image must be of size 512x512, in PNG format.
- **The image must have an alpha channel**.
  - Any pixel with nonzero alpha is part of the character.
  - If the edges of the silhouette look like a cheap photoshop job (especially when ST renders the character on a different background), check them manually for background bleed.
- Using any method you prefer, create a front view of your character within [these specifications](Character_Card_Guide.png).
  - In practice, you can create an image of the character in the correct pose first, and align it as a separate step.
  - If you use Stable Diffusion, see separate section below.
  - **IMPORTANT**: *The character's eyes and mouth must be open*, so that the model sees what they look like when open.
    - See [the THA3 example character](tha3/images/example.png).
    - If that's easier to produce, an open-mouth smile also works.
- To add an alpha channel to an image that has the character otherwise fine, but on a background:
  - In Stable Diffusion, you can try the [rembg](https://github.com/AUTOMATIC1111/stable-diffusion-webui-rembg) extension for Automatic1111 to get a rough first approximation.
  - Also, you can try the *Fuzzy Select* (magic wand) tool in traditional image editors such as GIMP or Photoshop.
  - Manual pixel-per-pixel editing of edges is recommended for best results. Takes about 20 minutes per character.
    - If you rendered the character on a light background, use a dark background layer when editing the edges, and vice versa.
    - This makes it much easier to see which pixels have background bleed and need to be erased.
- Finally, align the character on the canvas to conform to the placement the THA3 posing engine expects.
  - We recommend using [the THA3 example character](tha3/images/example.png) as an alignment template.
  - **IMPORTANT**: Export the final edited image, *without any background layer*, as a PNG with an alpha channel.
- Load up the result into *SillyTavern* as a `talkinghead.png`, and see how well it performs.

#### Tips for Stable Diffusion

It is possible to create a suitable character render with Stable Diffusion. We assume that you already have a local installation of the [Automatic1111](https://github.com/AUTOMATIC1111/stable-diffusion-webui-rembg) webui.

- Don't initially worry about the alpha channel. You can add that after you have generated the image.
- Try the various VTuber checkpoints floating around the Internet.
  - These are trained on talking anime heads in particular, so it's much easier getting a pose that works as input for THA3.
  - Many human-focused SD checkpoints render best quality at 512x768 (portrait). You can always crop the image later.
- I've had good results with `meina-pro-mistoon-hll3`.
  - It can produce good quality anime art (that looks like it came from an actual anime), and it knows how to pose a talking head.
  - It's capable of NSFW so be careful. Use the negative prompt appropriately.
  - As the VAE, the standard `vae-ft-mse-840000-ema-pruned.ckpt` is fine.
  - Settings: *512x768, 20 steps, DPM++ 2M Karras, CFG scale 7*.
  - Optionally, you can use the [Dynamic Thresholding (CFG Scale Fix)](https://github.com/mcmonkeyprojects/sd-dynamic-thresholding) extension for Automatic1111 to render the image at CFG 15 (to increase the chances of SD following the prompt correctly), but make the result look like as if it was rendered at CFG 7.
    - Recommended settings: *Half Cosine Up, minimum CFG scale 3, mimic CFG scale 7*, all else at default values.
- Expect to render **upwards of a hundred** *txt2img* gens to get **one** result good enough for further refinement. (At least you can produce and triage them quickly.)
- **Make it easy for yourself to find and fix the edges.**
  - If your character's outline consists mainly of dark colors, ask for a light background, and vice versa.
- As always with SD, some unexpected words may generate undesirable elements that are impossible to get rid of.
  - For example, I wanted an AI character wearing a *"futuristic track suit"*, but SD interpreted the *"futuristic"* to mean that the character should be posed on a background containing unrelated scifi tech greebles, or worse, that the result should look something like the female lead of [*Saikano* (2002)](https://en.wikipedia.org/wiki/Saikano). Removing that word solved it, but did change the outfit style, too.

**Prompt**:

```
(front view, symmetry:1.2), ...character description here..., standing, arms at sides, open mouth, smiling,
simple white background, single-color white background, (illustration, 2d, cg, masterpiece:1.2)
```

The `front view` and `symmetry`, appropriately weighted and placed at the beginning, greatly increase the chances of actually getting a direct front view.

**Negative prompt**:

```
(three quarters view, detailed background:1.2), full body shot, (blurry, sketch, 3d, photo:1.2),
...character-specific negatives here..., negative_hand-neg, verybadimagenegative_v1.3
```

As usual, the negative embeddings can be found on [Civitai](https://civitai.com/) ([negative_hand-neg](https://civitai.com/models/56519), [verybadimagenegative_v1.3](https://civitai.com/models/11772))

Then just test it, and equip the negative prompt with NSFW terms if needed.

The camera angle terms in the prompt may need some experimentation. Above, we put `full body shot` in the negative prompt, because in SD 1.5, at least with many anime models, full body shots often get a garbled face. However, a full body shot can actually be useful here, because it has the legs available so you can crop them at whatever point they need to be cropped to align the character's face with the template.

One possible solution is to ask for a `full body shot`, and *txt2img* for a good pose and composition only, no matter the face. Then *img2img* the result, using the [ADetailer](https://github.com/Bing-su/adetailer) extension for Automatic1111 (0.75 denoise, with [ControlNet inpaint](https://stable-diffusion-art.com/controlnet/#ControlNet_Inpainting) enabled) to fix the face.

**ADetailer notes**

- Some versions of ADetailer may fail to render anything into the final output image if the main denoise is set to 0, no matter the ADetailer denoise setting.
  - To work around this, use a small value for the main denoise (0.05) to force it to render, without changing the rest of the image too much.
- When inpainting, **the inpaint mask must cover the whole area that contains the features to be detected**. Otherwise ADetailer will start to process correctly, but since the inpaint mask doesn't cover the area to be edited, it can't write there in the final output image.
  - This makes sense in hindsight: when inpainting, the area to be edited must be masked. It doesn't matter how the inpainted image data is produced.


### Acknowledgements

This software incorporates the [THA3](https://github.com/pkhungurn/talking-head-anime-3-demo) AI-based anime posing engine developed by Pramook Khungurn. The THA3 code is used under the MIT license, and the THA3 AI models are used under the Creative Commons Attribution 4.0 International license. The THA3 example character is used under the Creative Commons Attribution-NonCommercial 4.0 International license. The trained models are currently mirrored [on HuggingFace](https://huggingface.co/OktayAlpk/talking-head-anime-3).

In this software, the manual poser code has been mostly rewritten, and the live mode code is original to `talkinghead`.