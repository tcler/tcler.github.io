---
layout: post
title: "huggingface gated repo"
---

## what's wrong
huggingface model download fail: Cannot access gated repo for url ...

```
$ https_proxy=squid.xxxxxx.com:8080 ilab model download --hf-token $MY_HF_TOKEN  --repository  mistralai/Mixtral-8x7B-Instruct-v0.1
INFO 2025-06-18 20:41:14,165 instructlab.model.download:77: Downloading model from Hugging Face:
    Model: mistralai/Mixtral-8x7B-Instruct-v0.1@main
    Destination: /home/jiyin/.cache/instructlab/models
Downloading failed with the following exception: 
Downloading model failed with the following error: 
Downloading model failed with the following Hugging Face Hub error:
    
Downloading safetensors model failed with the following Hugging Face Hub error:
    403 Client Error. (Request ID: Root=1-6852b3ec-7f5e44cb45b811cb35df3c8f;4a7d0270-bf0c-4a60-a9da-7a9be958c794)

Cannot access gated repo for url https://huggingface.co/mistralai/Mixtral-8x7B-Instruct-v0.1/resolve/41bd4c9e7e4fb318ca40e721131d4933966c2cc1/.gitattributes.
Access to model mistralai/Mixtral-8x7B-Instruct-v0.1 is restricted and you are not in the authorized list. Visit https://huggingface.co/mistralai/Mixtral-8x7B-Instruct-v0.1 to ask for access.
```

## Solution

ref: [Cannot access gated repo You must be authenticated to access it.](https://huggingface.co/mistralai/Mistral-7B-Instruct-v0.2/discussions/93#6622158124eb2673fe12b354)  

open webpage of that haggingface model, and go to ```model card``` and 
clicked "Agree and access repository".  

Then we will be able to start downloading.
