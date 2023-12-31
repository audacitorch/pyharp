# PyHARP

PyHARP is a Python library designed to embed Gradio apps for audio processing in a Digital Audio Workstation (DAW) through the [HARP](https://github.com/audacitorch/HARP) plugin. PyHARP creates hosted, asynchronous, remote processing (HARP) endpoints in DAWs, facilitating the integration of deep learning audio models into DAW environments through Gradio.

PyHARP makes use of [Gradio](https://www.gradio.app) to provide a web endpoint for your python processing code, and [ARA](https://blog.landr.com/ara2-plugins/) to route audio from the DAW to the Gradio endpoint. 

## What is HARP? 

HARP is designed for processing audio in a DAW with deep learning models that are too large to run in real-time and/or on a user's CPU, or otherwise require a large offline context for processing. There are many examples of these kinds of models, like [MusicGen](https://huggingface.co/spaces/facebook/MusicGen) or [VampNet](https://huggingface.co/spaces/descript/vampnet). 

HARP makes use of the [ARA](https://blog.landr.com/ara2-plugins/) framework to access all of the audio data in a track asynchronously, allowing the audio to be processed offline by a remote server, like a [Gradio application](https://www.gradio.app/demos) hosted in a [Hugging Face Space](https://huggingface.co/spaces).

## How is HARP different? 

Other solutions for realtime processing with neural networks in DAWs (e.g. [NeuTone](https://neutone.space/)), require highly optimized models that can run on a local CPU and rely on the model code to be [traced/scripted into a JIT representation](), which can be a challenge for the model developer.

PyHARP, on the other hand, relies on [Gradio](), which allows for the use of any Python function as a processing endpoint. PyHARP doesn't require for your deep learning code to be optimized or JIT compiled. PyHARP lets you process audio in a DAW using your deep learning library of choice. [Tensorflow]()? [PyTorch]()? [Jax]()? [Librosa]()? You pick. It doesn't even have to be deep learning code! Any arbitrary Python function can be used as a HARP processing endpoint, as long as it can be wrapped in a Gradio interface.

- Are you a deep learning researcher working on a new model and you'd like to give it a spin in a DAW and share it with your producer friends? Try HARP!
- Want to wrap your favorite librosa routine so you always have it handy when you're mixing? Try HARP! 
- Have a cool and quick audio processing idea and don't have the time to write C++ code for a plugin? Try HARP!
- Want to make an audio research demo that you'd like to put into an audio production workflow? Try HARP!

# Getting Started

## Download HARP
To use pyHARP apps in your DAW, you'll need the [HARP](https://github.com/audacitorch/harp) plugin. HARP is available as an ARA VST3 plugin, and can be used by any DAW host that supports the ARA framework. 

You can download HARP from the HARP [releases](https://github.com/audacitorch/HARP/releases) page. 

## Installing PyHARP
```
git clone https://github.com/audacitorch/pyharp.git
cd pyharp
pip install -e .
```

## Examples
There are examples of how to build HARP endpoints with PyHARP in the `examples/` directory. 
To run an example, you'll need to install that example's dependencies. 

For example, to run the pitch shifter example: 

```bash
cd examples/pitch_shifter
pip install -r requirements.txt
```

Here are some of the examples available:
- [Pitch Shifter](examples/pitch_shifter/)
- [Harmonic/Percussive Source Separation](examples/harmonic_percussive/)

We also provide a [template](examples/template/) to start building a new endpoint.

# Tutorial: Making a HARP Pitch Shifter.

To make a HARP app, you'll need to write some audio processing code and wrap it using Gradio and PyHARP.

## Write your processing code

Create a function that defines the processing logic for your audio model. This could be a source separation model, a text-to-music generation model, a music inpainting system, a librosa processing routine, etc. 

**NOTE** This function should be a Gradio-compatible processing function, and should thus take the values of some input widgets as arguments. To work with HARP, the function should accept exactly ONE audio input argument + any number of other sliders, texboxes, etc. Additionally, the function should output exactly one audio file. 

For our tutorial, we'll make a pitch shifter. We'll use the [audiotools](https://github.com/descriptinc/audiotools) library from Descript (installation instructions can be found [here](https://github.com/descriptinc/audiotools#installation)).
Our function will take two arguments:
- `input_audio_path`: the audio filepath to be processed
- `pitch_shift`: the amount of pitch shift to apply to the audio
and return the following:
- `output_audio_path`: the filepath of the processed audio

```python
import torch
import torchaudio
from pyharp import save_and_return_filepath
# Define the process function
@torch.inference_mode()
def process_fn(input_audio_path, pitch_shift_amount):
    from audiotools import AudioSignal
    
    sig = AudioSignal(input_audio_path)

    ps = torchaudio.transforms.PitchShift(
        sig.sample_rate, 
        n_steps=pitch_shift_amount, 
        bins_per_octave=12, 
        n_fft=512
    ) 
    sig.audio_data = ps(sig.audio_data)

    output_audio_path = save_and_return_filepath(sig)

    return output_audio_path
```

## Create a Model Card

A model card helps users identify your model and keep track of what it does. 
```python
from pyharp import ModelCard
# Create a ModelCard
card = ModelCard(
    name="Pitch Shifter",
    description="A pitch shifting example for HARP.",
    author="Hugo Flores Garcia",
    tags=["example", "pitch shift"]
)
```

## Create a Gradio Interface

Now, we'll create a [Gradio](https://www.gradio.app) interface for our processing function, connecting the input and output widgets to the function, and making our processing code accessible via a Gradio // HARP endpoint. 

To achieve this, we'll create a list of Gradio input widgets, as well as an audio output widget, then use the `build_endpoint` function from PyHARP to create a Gradio interface for our processing function.

**NOTE**: make sure that the order of your inputs matches the order of the defined arguments in your processing function. 

**NOTE**: all of the `gr.Audio` widgets MUST have `type="filepath"` in order to work with HARP.


```python
from pyharp import build_endpoint
import gradio as gr
# Build the endpoint
with gr.Blocks() as demo:

    # Define your Gradio interface
    inputs = [
        gr.Audio(
            label="Audio Input", 
            type="filepath"
        ), # make sure to have an audio input with type="filepath"!
        gr.Slider(
            minimum=-24, 
            maximum=24, 
            step=1, 
            value=7, 
            label="Pitch Shift (semitones)"
        ),
    ]
    
    # make an output audio widget
    output = gr.Audio(label="Audio Output", type="filepath")

    # Build the endpoint
    widgets = build_endpoint(inputs, output, process_fn, card)

demo.queue()  # see the NOTE below
demo.launch(share=True)
```
**NOTE**: in order for HARP users to be able to cancel an ongoing processing job, you'll need to enable the queueing system in Gradio. This is done by calling `demo.queue()`.


Documentation for Gradio widgets can be found [here](https://www.gradio.app/docs/components). Currently, HARP supports the following widgets:
- [Audio](https://www.gradio.app/docs/audio)
- [Textbox](https://www.gradio.app/docs/textbox)
- [Slider](https://www.gradio.app/docs/slider)

## Run the app

Now, we can run our app and test it out. 

```bash
python examples/pitch_shifter/app.py
```

This will create a local Gradio endpoint on `http://localhost:<PORT>`, as well as a forwarded gradio endpoint, with a format like `https://<RANDOM_ID>.gradio.live/`. You can copy that link and enter it in your HARP plugin to use your app in your DAW. 


Note that automatically generated Gradio endpoints are only available for 72 hours. If you'd like to keep your endpoint alive and share it with other users, you can easily create a [HuggingFace Space](https://huggingface.co/docs/hub/spaces-overview) to host your HARP app indefinitely, or alternatively, host your gradio app using other hosting services.  


## 

1. Create the space in HuggingFace Spaces

2. Clone the initialized repo locally
```bash
git clone https://huggingface.co/spaces/<YOUR_USERNAME>/<YOUR_SPACE_NAME>
```

3. Add your files to the repo and commit/push
```bash
git add .
git commit -m "first commit"
git push -u origin main
```

Here are a few tips and best-practices when dealing with HuggingFace Spaces:
- Spaces operate based off of whatever is in the `main` branch
- A `requirements.txt` specifying all dependencies must be included for a Space to work properly
- A `.gitignore` file should be added to maintain orderliness of the repo (_e.g._, to ignore `src`/`_outputs` directories)
- An [access token](https://huggingface.co/docs/hub/security-tokens) may be required to push commits to HuggingFace Spaces
- A `README.md` file with metadata will be created automatically when a Space is initialized


## Utilizing pre-trained models
If you want to build an endpoint that utilizes a pre-trained model, we recommend the following:
- Load the model outside of `process_fn`, so that it is not continually re-initialized for each audio file that is processed
- If model weights must be stored locally (_i.e._, they are not easily accessible through the internet or python packages), save them to your repo using [Git Large File Storage](https://git-lfs.com/)
