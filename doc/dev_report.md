# Framework Development Report

This report is more engineering-oriented, focusing on the technical details of the project. For a more general overview, please refer to [the full paper on MIG 2024](https://dl.acm.org/doi/10.1145/3677388.3696321).

It is structured as follows:
- A summary about what generative models and tools are used in the project.
- A detailed description of each module, including the model choice, the challenges faced, and the solutions explored (both successful and unsuccessful).

## Model Choice Summary

### Story & Screenplay

[AutoGen](https://microsoft.github.io/autogen/0.2/docs/Getting-Started/)

### Image

**Image Generation**: [Playground v2 - 1024px Aesthetic](https://huggingface.co/playgroundai/playground-v2-1024px-aesthetic)  
**Character**: [CC2D](https://assetstore.unity.com/packages/2d/characters/cc2d-essential-bundle-187410)  
**Object Segmentation**: [SegFormer (b5-sized)](https://huggingface.co/nvidia/segformer-b5-finetuned-ade-640-640)  
**Depth Estimation**: [ZoeDepth](https://github.com/isl-org/ZoeDepth?tab=readme-ov-file)  
**Image Captioning**: [vit-gpt2-image-captioning](https://huggingface.co/nlpconnect/vit-gpt2-image-captioning)  

### Sound

**Background Music**: [MusicGen](https://github.com/facebookresearch/audiocraft/blob/main/docs/MUSICGEN.md)  
**Sound Effects**: [FreeSound](https://freesound.org/) + [AudioGen](https://github.com/facebookresearch/audiocraft/blob/main/docs/AUDIOGEN.md)  
**Voice Fingerprint Generation**: [AudioGen](https://github.com/facebookresearch/audiocraft/blob/main/docs/AUDIOGEN.md) + [XTTS](https://github.com/coqui-ai/TTS?tab=readme-ov-file)  
**Text-to-Speech**: [ElevenLabs](https://github.com/elevenlabs/elevenlabs-python) (paid) / [XTTS](https://github.com/coqui-ai/TTS?tab=readme-ov-file)

### Final Composition

[Unity3D](https://unity.com)

## Character Module

While [CC2D](https://assetstore.unity.com/packages/2d/characters/cc2d-essential-bundle-187410) offers configurable 2D characters, its rigging system differs from those in 3D characters. Thus we are unable to utilize rich 3D motion dataset or generative models. A self-designed motion projection algorithm may help but we have not explored that yet.

There exists another bottleneck, where we find it hard to generate plausible 2D garments:
- We tried generating garment parts using ControlNet, where garment part outlines are given as input. But the quality is not satisfying, as there lacks consistency between different parts, and the generation may not fully comply with the given outlines.
- We also tried generating 2D characters as a whole and then segmenting them into parts. But the quality is not satisfying because in the animation, we need overlaps between different parts. However, no overlaps are generated in the segmentation process.

## Compostion Module

We chose Unity in this study because it integrates a wide range of digital storytelling tools such as character animation, lighting, particle system, and camera control (we did not integrate all of them). However, we did not find an elegant way to fully automate the composition process, since Unity requires asset indexing before using them in the scene. We may explore other game engines such as Blender in the future.

## Sound Module

### Final implementation

Sound Module takes audio descriptions from Story Generation Module as input, leverages OpenAI ChatGPT to process descriptions into model prompts, and finally uses various generation models to create audio files.

It handles three generation tasks in parallel:

- Background Music: Directly feed music descriptions to MusicGen.

- Foreground / Background SFX: Inspired by [WavJourney](https://arxiv.org/abs/2307.14335), we firstly decompose the original sound description into several simpler sound clips. Such decompositions are handled by LLM (OpenAI GPT4-turbo), to which the original sound description and a pre-defined sound clip composition toolset are provided. After the decomposed sound descriptions and composition commands are created by LLM. Sound descriptions are fed to [AudioGen](https://github.com/facebookresearch/audiocraft/blob/main/docs/AUDIOGEN.md) to get sound clips. Finally the clips are composited using the generated commands.

- Speech: The original character description is firstly simplified by LLM. Then we feed the simplified description to AudioGen. The generated audio is treated as a voice fingerprint. Next, the voice fingerprint is sent to XTTS as a reference wav file, which XTTS uses to produce an example speech audio. The example audio is used as the reference of all later generations.


Compared to WavJourney, we improve the generation quality in the following aspects:

- Using text description, We generate character voice styles from scratch, rather than to retrieve from pre-defined voice presets.

- We introduce a new set of compositional toolkits for background sound, enabling looping sound effect generations with longer durations.

### Exploration

#### Background Music

We only tried MusicGen, which is also the choice of WavJourney.

Music generation cannot be longer than 30 seconds, which results from the capability of MusicGen.

#### Sound effects

We tried [AudioLDM](https://arxiv.org/abs/2301.12503) v1 and v2 before. AudioGen is slightly better than both. So we finally chose it, which aligns with the switch of WavJourney.

Artifacts may occur in sound generation due to AudioGen's limitation. We tried a detector which invokes sound re-generations when the generated sound is detected as bad. But the detector has a limited quality improvement. So we disable such a feature by default.

Meta's AudioBox has better quality but it is not available through APIs by March 2024.

#### Speech

Hoping to find Text-to-Speech models that can be conditioned on style referecne and emotion, we tried the following models:

- Bark: Quality is good but it can only choose offical voice presets by default. We tried community-implement voice preset generator. But hallucinations emerge for unknown reasons.

- [StyleTTS](https://styletts2.github.io/): Has the best alignment with style reference wav file. However, it uses a too simple phoneme module that hinders its overall performance.

- [XTTS](https://huggingface.co/coqui/XTTS-v2): Balance between style reference and generation quality. Also, it is way faster than Bark. But it can not provide emotional conditioning by text prompts.

- [ElevenLabs](https://github.com/elevenlabs/elevenlabs-python): It offers emotional conditioning by text prompts. And the quality is on par with XTTS. We choose it as the default TTS model.


Possible solutions to explore:

- Use AudioGen to generate emotional-conditioned voice fingerprints, but we cannot guarantee the consistency of voice style within characters.

- Use commercial TTS services with emotional conditioning, then use voice conversion models to transfer the default voice to our customed voice: Microsoft Azure has good solutions in emotional TTS. But we have not found a good voice conversion model. We tried [FreeVC](https://olawod.github.io/FreeVC-demo/) and [KNNVC](https://bshall.github.io/knn-vc/). Neither of them are satisfying. We may try [YourTTS](https://edresson.github.io/YourTTS/) in the future.

- One-stop commercial solution: We can upload sound file references to Azure and train a voice preset online. We have not priorized this yet because we think training voice presets can be time-consuming, which may not be an ideal choice.

- Meta's AudioBox: It offers a text-prompt-based voice style conversion capabiltiy. But the performance is not good.

- [Meta's 2022 emotional conversion paper](https://speechbot.github.io/emotion/): we have not explored that.