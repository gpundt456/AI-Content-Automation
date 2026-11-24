# AI Content Generation Automation

**A locally hosted, Telegram-controlled AI video production pipeline built with n8n, ComfyUI, and Ollama, tested on an NVIDIA RTX 3080 (10 GB VRAM) with 32 GB system RAM.** It generates a set of short, vertical AI-influencer product promotion clips and sends them directly to my phone through Telegram. The clips can be combined in an editor such as CapCut into a TikTok Shop-style video.

This example is configured for a **male influencer promoting men's cologne**, but the underlying workflow is not limited to fragrances. Changing the script library, visual prompts, and product images allows the same pipeline to create promotional content for other products.

The project focuses on **remote automation, local AI inference, multi-stage media generation, and GPU memory management**, rather than on any particular social media account or publishing platform.

> **What this repository contains:** Three exported n8n workflows covering Telegram-based control, AI content generation, and error handling. The workflows include embedded ComfyUI API graphs; the required model weights, ComfyUI custom nodes, reference media, and local file paths must be configured separately.

## How it works


A key feature of this setup is that **the AI models run locally on my computer, while Telegram acts as the remote interface**. With n8n, ComfyUI, Ollama, and the zrok connection running, I can initiate and manage generation from my phone without needing to be at the computer. Completed clips are delivered back through the same Telegram conversation.

The system can generate batches repeatedly through locally running, freely available software and model weights, **without paying per generation to a hosted image/video API**. I can receive the resulting clips on my phone ready to review or assemble. Compute is provided by my own PC, so electricity, hardware, and storage are not free. Software and open-weight models have different licenses; not every component should be described as unrestricted open source.

## What the workflow produces

- **Four short 9:16 vertical clips per selected script**, each based on one sentence of dialogue and designed to run approximately five seconds. The combined runtime is generally in the **15–20+ second range**, depending on speech timing and generation settings.
- A consistent **AI-generated male influencer** and a product represented using the supplied product image.
- Generated speech, talking-head lip synchronization, natural camera movement, and product presentation.
- Individual video files sent directly to **Telegram** as each clip finishes.
- Clips that can be combined and trimmed in **CapCut or another video editor** into a single short-form product advertisement. **Automated final stitching and publishing are not part of this repository.**

The promotional scripts in this version are generic examples, not verified claims about any specific fragrance. Product descriptions and advertising copy should be checked against the actual item before external use.

## Included workflows

The repository centers on three n8n JSON exports: `Telegram Trigger (github).json`, `Tiktok Shop Content Automation (github).json`, and `Error Trigger (github).json`. Optional screenshots, generated examples, and redistributable reference assets can be added separately.

## The three n8n workflows

### 1. Telegram Trigger — remote control

**File:** `workflows/Telegram Trigger (github).json`

This workflow listens for messages sent to my Telegram bot and serves as the control interface for my locally hosted automation.

It includes logic for:

- Starting the main content-generation workflow from a Telegram command.
- Checking whether local ComfyUI and Ollama endpoints are responding.
- Checking NVIDIA GPU memory usage with `nvidia-smi` before starting GPU-heavy work.
- Sending status messages to Telegram.
- Handling a `Stop` command using Windows process-management commands.

The Telegram Trigger uses **Execute Workflow** (subworkflow) nodes to launch the main generation workflow without putting all of the control logic in one enormous workflow. The same trigger pattern could be extended with other keywords to launch completely different automations from the same Telegram interface. The remote connection is made possible by **zrok**, which provides a public HTTPS endpoint for the locally hosted n8n webhook. Telegram delivers events to that endpoint, and n8n processes them on the computer running the AI software. The local AI services do not need to be made publicly accessible.

The separate launcher that manages the n8n/zrok setup is documented in my [n8n-zrok-launcher repository](https://github.com/gpundt456/n8n-zrok-launcher).

### 2. TikTok Shop Content Automation — video generation

**File:** `workflows/Tiktok Shop Content Automation (github).json`

This is the main production pipeline. **I maintain a local folder on my computer containing product photos of different men’s cologne bottles.** The workflow reads that directory and builds a Telegram product-selection list from the files it finds. I can pick a bottle, upload a new product photo, or let the workflow select a saved bottle at random if I do not respond before the Telegram form’s configured wait limit. This is useful for hands-off batches when I am away from the computer. The timeout is configurable in the `choose a product` Telegram node; it is not a universal two-minute default.

The main stages are:

1. **Product selection.** Read images of cologne bottles from a configured local directory, build a Telegram dropdown, accept a new upload, or fall back to a random saved bottle when the selection form times out. An optional local Qwen3-VL step can describe packaging for image editing; it is a small supporting feature, not the main driver of the workflow.
2. **Script selection.** The `parser` Code node randomly selects one of **five** cologne scripts. Each script contains **four separate sentences**, emitted as individual n8n items so subsequent generation can process one clip at a time.
3. **Influencer image generation and variation.** The `Random Scene Picker` Code node changes visual parameters such as hairstyle, clothing, color, gestures, and car-interior colors; **Z-Image Turbo** generates the initial **9:16** influencer image. This helps avoid repeatedly producing the same-looking video.
4. **Product-focused image editing.** **FLUX 3 Image Klein** edits the image to place the selected bottle in the influencer’s hand while preserving the original packaging, readable label, character identity, framing, and scene. Product realism and consistency are central design goals, not afterthoughts.
5. **Speech generation.** **Qwen3-TTS** generates speech from the selected script sentences using a configured reference-voice workflow.
6. **Motion-guided image-to-video.** The LTX workflow combines the generated frames and audio with a motion-conditioning reference to produce short talking-head video clips.
7. **Clip continuity.** Later stages reuse saved end frames as inputs for subsequent clips, helping the shots remain visually connected.
8. **File delivery and repetition.** The workflow reads generated media from local folders, sends each clip through Telegram, performs cleanup between GPU-intensive stages, and provides a way to stop or begin another batch.

The colored n8n sections and sticky notes divide the pipeline into understandable stages. I spent many hours testing different prompts, image-generation and image-editing configurations, and video models to reduce artifacts, preserve bottle appearance, keep product labels and faces consistent, improve lip synchronization, and maintain realistic movement between frames.

### 3. Error Trigger — recovery and cancellation

**File:** `workflows/Error Trigger (github).json`

This companion workflow responds to a production error with a Telegram message offering **Restart** or **Stop**. It checks local services and contains Windows commands to restart ComfyUI and Ollama when necessary.

The system also has a custom cancellation approach: sending `Stop` to the Telegram control workflow terminates the local Ollama/ComfyUI processes, causing dependent in-progress operations to fail. The Error Trigger workflow then provides a recovery/stop interface.

I developed this as a practical solution to a workflow that otherwise kept running once generation had started. **It is a deliberate, process-level workaround rather than a targeted n8n execution-cancellation API.** Its trade-off is that it affects the shared services, not just one particular generation job. It is appropriate to demonstrate the design, but should be reviewed before use on shared or production infrastructure.

## AI models and software

| Tool / model | Role in this project |
| --- | --- |
| [n8n](https://n8n.io/) | Workflow orchestration, conditional logic, file handling, Telegram integration, scheduling/continuation, error handling |
| [ComfyUI](https://github.com/comfyanonymous/ComfyUI) | Local image, audio, and video generation through node-based model graphs |
| [Ollama](https://ollama.com/) | Runs the product-analysis vision model locally |
| [Qwen3-VL 8B](https://ollama.com/library/qwen3-vl) | Analyzes product photos and produces descriptive text for image-editing prompts (`qwen3-vl:8b` in the export) |
| [Z-Image Turbo](https://huggingface.co/Tongyi-MAI/Z-Image-Turbo) | Generates the initial influencer imagery; the exported graph uses locally installed checkpoints and LoRAs |
| [FLUX 3 Image Klein](https://huggingface.co/black-forest-labs/flux-3-image-klein-comfy) | Edits generated images to incorporate the selected product |
| [Qwen3-TTS](https://github.com/QwenLM/Qwen3-TTS) | Creates spoken dialogue for the short clips using a reference-voice node |
| [LTX](https://huggingface.co/Lightricks/LTX-2.3) | Image/audio-to-video generation, lip synchronization, clip continuity, and motion guidance |
| LTX Union Control IC-LoRA | Uses a video-derived motion guide for natural body and camera movement |
| [Telegram](https://core.telegram.org/bots) | Remote control, product selection, status messages, and delivery of completed video clips |
| [zrok](https://zrok.io/) | Makes n8n's Telegram webhooks reachable through public HTTPS while n8n runs locally |

**Why LTX 2.3 instead of LTX 2.5?** I originally experimented with LTX 2.5, but in my testing it did not reliably synchronize the influencer’s mouth movements with the encoded voiceover. Switching to **LTX 2.3** produced substantially better audio-conditioned lip sync with the same overall production workflow. I kept LTX 2.3 for this reason. This is an observation from my particular setup, not a claim that LTX 2.5 cannot lip-sync in other configurations.

### Exact model filenames in the exported ComfyUI graphs

These are the model and supporting-weight filenames referenced by the embedded ComfyUI graphs in the supplied n8n export. **The files are not bundled in this repository**; install compatible weights and custom nodes, and retain the expected filenames or update the graphs accordingly.

| Stage | Exact referenced files |
| --- | --- |
| Z-Image Turbo image generation | `z-image-turbo-fp8-e4m3fn.safetensors`, `RealisticSnapshot-Zimage-Turbov5.safetensors`, `intorealism_zitV30.safetensors`, `ae.safetensors`, `Qwen3-4B-UD-Q8_K_XL.gguf` |
| FLUX 3 Klein image editing | `flux3-image-klein-int8-convrot.safetensors`, `Flux3Kleinvae.safetensors`, `qwen3vl_4b_fp8_scaled.safetensors` |
| LTX 2.3 core | `ltx-2.3-22b-dev-Q4_K_M.gguf`, `ltx-2.3-22b-dev_audio_vae.safetensors`, `ltx-2.3-22b-dev_video_vae.safetensors`, `ltx-2.3-22b-dev_embeddings_connectors.safetensors`, `ltx-2.3_text_projection_bf16.safetensors` |
| LTX 2.3 motion, talking head, and enhancement | `LTX-2.3-22b-AV-LoRA-talking-head-v1.safetensors`, `ltx-2.3-22b-ic-lora-union-control-ref0.5.safetensors`, `ltx-2.3-22b-distilled-lora-384-1.1.safetensors`, `ltx-2.3-spatial-upscaler-x2-1.1.safetensors` |
| LTX supporting models | `gemma-3-12b-it-Q8_0.gguf`, `yolox_l.torchscript.pt`, `dw-ll_ucoco_384_bs5.torchscript.pt` |
| Local vision inference via Ollama | `qwen3-vl:8b` (Ollama model identifier, not a ComfyUI file) |
| Voice synthesis | Qwen3-TTS via custom ComfyUI nodes; the graph loads a voice reference such as `Video Project 5.mp3`. Its Qwen3-TTS weights are managed by the installed custom node and are not specified as a named checkpoint file in the exported graph. |

Exact model weights and custom-node packages must be obtained from their official or authorized sources, following the individual model licenses. Model filenames alone are not guarantees of version compatibility.

### Union Control: motion, not visual content

The reference video is used **only to guide realistic influencer-style handheld camera movement and body movement**. It is **not** a reference for the person's identity, clothing, product, background, or visual appearance. Those aspects come from the independently generated images, product input, and other prompts earlier in the pipeline.

The `random reference video` Code node currently selects from one numeric identifier (`1023`), and the LTX graphs load that as `1023.mp4` from ComfyUI's input directory. The code can be expanded to choose among multiple motion-reference clips.

## GPU memory management

One of the most difficult parts of building this system was getting several resource-intensive models to work reliably on a consumer GPU. I developed and tested the pipeline on an **NVIDIA RTX 3080 (10 GB VRAM) with 32 GB of system RAM**.

Early versions encountered GPU memory allocation failures when multiple models remained loaded. After considerable trial and error, I organized the workflow so that **one major GPU-intensive generation task runs at a time**, with ComfyUI memory-cleanup operations between stages. The exported workflow contains repeated `clean sys and vram` nodes that run cleanup graphs, including model-memory release and VRAM cleanup operations.

The controller also checks GPU usage before starting parts of the pipeline. This sequencing made it possible to combine image generation, image editing, speech synthesis, and image-to-video generation without needing a separate paid inference service for every stage.

This is **resource management through workflow design**, not a guarantee that every model combination will fit on a 10 GB GPU. Installed models, quantization, resolution, frame counts, other running applications, and custom-node versions all affect memory requirements.

## Adapting the workflow to other products

Although this version is designed around **men's cologne**, the architecture is reusable. The product itself is not hard-coded into the orchestration system.

For a different product, the main changes are:

1. **Upload a different product image** through the Telegram form or place it in the configured product-image folder.
2. **Update the scripts** in the `parser` Code node. Its current structure has five alternative scripts, each producing four sentence items. This output format can stay the same for any product category.
3. **Change the influencer or scene prompts** in the image-generation stage if a car selfie is no longer the right presentation style.
4. **Adjust the image-editing prompts** so the new product is shown naturally. For example, a watch, headphones, or skincare package requires different instructions than holding a cologne bottle with its label facing the camera.
5. **Review the video prompts** for product-specific gestures and remove any language that assumes the item is a fragrance bottle.

The Telegram interface, product-selection logic, ComfyUI orchestration, audio generation, motion conditioning, intermediate-file handling, VRAM cleanup, and clip delivery can generally remain in place.

In other words, this is **a reusable local AI promotional-video pipeline with one product-specific configuration**, not a separate automation that must be built from scratch for every product. Actual changes depend on the item's shape, marketing claims, and desired presentation. The file-based product library and the reusable scene/script logic make it relatively easy to retarget without restructuring the automation.

## Why the outputs vary

Repeatedly uploading an identical video is not a useful content strategy and may run into platform duplicate-content rules. Rather than reproduce the same visual every time, the `Random Scene Picker` generates different combinations of appearance, wardrobe, gestures, car interiors, and random seeds. The `parser` also selects among five different scripts. This introduces meaningful variation into batches, although it does **not** guarantee that every final video is unique or compliant with a platform’s rules.

## Setup and reproduction

> **This is a portfolio project and an exported snapshot of a customized local setup, not a one-click installer.** The n8n files are useful for examining the architecture, but running the entire system requires model downloads, supporting custom nodes, local media, credentials, and path configuration.

### 1. Set up n8n, ComfyUI, Ollama, and Telegram

- Install a supported, self-hosted version of **n8n**.
- Install **ComfyUI** and the model files required by the embedded generation graphs.
- Install **Ollama**, start its local service, and make the vision model available:

  ```bash
  ollama pull qwen3-vl:8b
  ```

- Create a Telegram bot with **BotFather**, obtain your bot credentials, and configure them in n8n.
- Set up a stable public HTTPS webhook endpoint for Telegram. My local arrangement uses **zrok** and the separate [n8n-zrok-launcher](https://github.com/gpundt456/n8n-zrok-launcher).

The Telegram endpoint should expose only the webhook functionality required for the automation. Do not publicly expose an unauthenticated n8n editor, ComfyUI server, or Ollama API.

### 2. Install the n8n ComfyUI community node

The ComfyUI integration **does not come with a default n8n installation**. The exports use node type `n8n-nodes-comfyui.comfyui`, provided by the community package:

**[`n8n-nodes-comfyui`](https://github.com/timoof/n8n-comfyui)**

On a self-hosted n8n instance, install it using the **Community Nodes** settings, then configure its ComfyUI connection credentials. Installation availability and behavior vary by n8n version and deployment method.

The workflow also uses **Execute Command** nodes for Windows process management and `nvidia-smi`. These nodes can be disabled by default in newer n8n installations, and enabling them grants substantial access to the host operating system. They should be used only in a trusted, suitably restricted self-hosted environment.

**n8n version warning:** n8n 3.x uses Docker-based self-hosting rather than a direct npm/npx installation. Docker also changes what `localhost` and filesystem paths refer to. A Docker-based reproduction must explicitly configure host-service access, local volume mounts, and command-execution behavior. The original workflow was developed for a local Windows installation; **n8n 3.x/Docker compatibility has not been validated here**.

### 3. Install the required ComfyUI models and custom nodes

Install model families compatible with the specific embedded API graphs: Z-Image Turbo and its selected checkpoints/LoRAs, FLUX 3 Image Klein, Qwen3-TTS, LTX-2.3 and its VAEs/adapters/LoRAs/upscaler, and the supporting vision/text/pose components.

Some of the embedded graphs use community ComfyUI nodes, including nodes for audio, video loading, pose processing, GGUF model loading, custom image editing, and memory cleanup. **ComfyUI-Manager can help identify missing custom nodes** after loading the corresponding graphs. Package versions and model filenames must match the API graphs used by the installed setup.

### 4. Provide reference assets

- **Motion reference:** Place a video named `1023.mp4` inside your ComfyUI `input` directory, or change the `random reference video` Code node and the associated video-loading expressions to match your filename. The video's purpose is movement guidance, not visual-content copying. Include a sample in GitHub only if you own it or have redistribution rights.
- **Voice reference:** The current Qwen3-TTS graphs load a file called `Video Project 5.mp3` from ComfyUI input. Replace it with an audio sample you are permitted to use, and update the Qwen3-TTS nodes to reference that file. The workflow's voice-reference audio is not supplied by the JSON export.
- **Product images:** Add at least one usable product photo to the configured local product-images folder or supply one through the Telegram form.

### 5. Import the three workflows and reconnect them

Import the three JSON files into n8n, then:

1. Recreate/select the **Telegram** credentials and the **ComfyUI API** credentials.
2. Replace all hard-coded Telegram chat IDs with your own intended chat ID or a suitable expression.
3. In **Telegram Trigger**, reselect the imported main content workflow in its **Execute Workflow** node; the exported workflow ID belongs to the original installation.
4. In the **main content workflow**, reselect its own workflow in the continuation/recursion **Execute Workflow** node if you want repeated batches.
5. **Required:** Open `Tiktok Shop Content Automation` in n8n, go to its **Workflow Settings**, and select the imported **Error Trigger** workflow in the **Error Workflow** field. Merely importing the Error Trigger JSON is not enough; this association must be configured before automatic error notifications/recovery will run.
6. In **Error Trigger**, review its restart/Execute Workflow target. The uploaded export retains a target named `bedtime`, which must be replaced with the correct imported workflow before relying on restart.
7. Configure the local service ports and test the message, stop, and recovery paths separately before enabling automatic repeat generation.

### 6. Update local file paths

The exported workflow references machine-specific directories for product photos, cached product descriptions, ComfyUI input/output images, audio, and final clips. These paths are **examples of my original local layout, not portable defaults**.

Update each `Read/Write Files from Disk` node to point to directories accessible from **the n8n process**. Separately ensure that ComfyUI's `input` and `output` directories contain the files referenced by its API graphs. On Docker, a Windows host path is not automatically visible inside the n8n container; use appropriate mounts and container paths.

Avoid changing output filenames or n8n expressions blindly: some later nodes look up a particular generated file using seed-based prefixes and extensions. Change paths while keeping those filename relationships intact.

### 7. Test a single batch

Start the local services, confirm Telegram webhooks are reaching n8n, and check that ComfyUI and Ollama are responsive. Select a product, run one batch, and confirm that the four generated clips arrive in Telegram. Only then enable the continuation behavior for repeated batches.

## Challenges and design decisions

- **Remote control of a local machine:** Telegram messages reach a locally hosted automation through a public HTTPS tunnel, removing the need to interact directly with the computer running the models.
- **Multiple AI systems in one workflow:** n8n coordinates a local vision model, image generation/editing, voice synthesis, motion-guided video generation, file movement, and Telegram delivery.
- **Limited GPU memory:** Sequential execution and repeated cleanup nodes avoid unnecessarily keeping several large models loaded at once.
- **Audio-driven lip sync:** When LTX 2.5 did not reliably lip-sync to an encoded voiceover in my tests, I used LTX 2.3, which performed better in this pipeline. The choice came from hands-on compatibility testing, not simply preferring a newer model version.
- **Consistent clips:** Generated frames, saved end frames, and an external motion guide allow related clips to be generated as separate stages.
- **Fault handling:** A separate Error Trigger workflow sends Telegram recovery controls; a process-level stop workaround trades precision for a simple, explainable way to interrupt a hung generation path.
- **Variation without rebuilding:** The `Random Scene Picker` changes scene/appearance combinations, while scripts and seeds introduce further variety. This helps avoid identical-looking clips while preserving the overall workflow structure.
- **Quality testing and product fidelity:** I spent countless hours iterating on image generation, product-insertion edits, video-model behavior, and prompts to keep the cologne bottle believable and recognizable across shots.

## Project authorship and AI assistance

**I independently conceptualized the architecture and the complete workflow structure**, including how the Telegram control workflow, subworkflow execution, local model stages, file-handling steps, GPU cleanup, and error handling work together. The overall automation design and the decisions about how to connect and sequence its components are my own.

I used AI as a coding and prompting assistant for **JavaScript Code nodes, Windows command/Execute Command nodes, image-generation and image-editing prompts, and image-to-video prompts**. I integrated, tested, debugged, and refined the resulting components through hands-on use. AI assistance supported implementation details rather than designing the overall workflow architecture.

## Limitations

- This repository includes workflow definitions, **not downloadable model checkpoints or a preconfigured ComfyUI installation**.
- Generated clips require local hardware and processing time. “No paid generation API” does **not** mean zero running cost.
- The current export is **Windows-oriented**, particularly the PowerShell/command-prompt process controls.
- Paths, workflow IDs, Telegram IDs, credentials, reference media, and local ports must be reviewed before running on another system.
- Motion control, lip sync, product lettering, visual continuity, and output duration may vary between generations.
- The workflow generates **individual clips**; it does not automatically combine them into one final video or post them to a platform.
- Repeated generation depends on the configured continuation branch and the local services remaining available.
- Public or commercial use of model weights, reference videos, voices, product imagery, and generated advertisements is subject to their respective licenses, permissions, and applicable rules. **Open weights and open-source software are not necessarily the same license category.**

## Responsible use and repository hygiene

Before publishing the exported JSON files, remove or replace any private chat IDs, personal directory names, credential references, webhook identifiers, execution traces, and pinned data. Recreate the credentials after import rather than publishing credentials or bot tokens.

Only share motion-reference videos, voice samples, and product imagery that you have permission to redistribute. AI-generated product promotions should avoid unverified product claims or misleading representations of a real person's endorsement.

## Related project

- **[n8n-zrok-launcher](https://github.com/gpundt456/n8n-zrok-launcher)** — the separate local launcher/webhook setup that enables Telegram messages to reach n8n on my computer.

---

*Built as a hands-on automation project to explore local AI generation, remote orchestration, GPU resource management, and multi-step content production.*
