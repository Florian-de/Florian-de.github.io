---
title: 'RAPHAEL: Text-to-Image Generation via Large Mixture of Diffusion Paths'
date: 2024-07-12
permalink: /posts/2024/07/raphael-post
categories: blog
tags:
  - RAPHAEL
  - Diffusion Model
  - Mixture of Experts
  - Computer Vision
  - Generative Model
---

<style>
  h1, h2 {
    color: #064273;
  }
</style>
  
In recent years, Generative Artificial Intelligence has gained a lot of popularity. More and more people use it in their daily and professional lives. Diffusion Models became popular with models like Stable Diffusion and DALL-E which showed the public what Diffusion Models are able to do. \
In this blog post, I aim to introduce and explain a new model, RAPHAEL, which outperforms models like Stable Diffusion and focuses on accurately displaying text in the generated images [[1]](#1). 

# Outline

I will start with the motivation behind Diffusion Models and RAPHAEL specifically. After that, I will give you some background knowledge about Diffusion Models and Mixture of Experts. Then, I will explain the architecture of RAPHAEL to you, followed by an ablation study and some experiments. Next, I will show you a benchmark which compares RAPHAEL to other models. Finally I will discuss the model.

1. [Why Diffusion Models?](#why-diffusion-models)
2. [Why RAPHAEL](#why-raphael)
3. [Background - What is a Diffusion Model?](#background---what-is-a-diffusion-model)
   - [Forward diffusion process](#forward-diffusion-process)
   - [Reverse diffusion process](#reverse-diffusion-process)
5. [Background - What are Mixture of Experts?](#background---what-are-mixture-of-experts)
6. [RAPHAEL Architecture](#raphael-architecture)
   - [The Transformer Block](#the-transformer-block)
   - [What are Time-MoE?](#what-are-time-moe)
   - [What are Space-MoE?](#what-are-space-moe)
   - [What is Edge-supervised Learning?](#what-is-edge-supervised-learning)
7. [Ablation Study](#ablation-study)
8. [Experiments](#experiments)
9. [Benchmarks](#benchmarks)
10. [Discussion](#discussion)
11. [References](#references)

# Why Diffusion Models? 

Have you ever taken a picture of something and later wanted to have more background or just a larger picture? Diffusion Models can be used to solve this problem. For example, Adobe has introduced a Diffusion Model called "Adobe Firefly 3 Model" which can expand the image and even add new objects or remove objects from the picture. 

<img width="750" alt="image" src="https://github.com/Florian-de/Florian-de.github.io/assets/64322175/de53ddee-3e90-4cae-97ee-e27ac328fca9">

Image from Adobe [[2]](#2)

Such technology can not only be used as a useful gadget for our daily lives but also for professional use, for example, for editing images for a commercial. \
It brings even more advantages in the use for science. For example in Biology, scientists use Diffusion Models to improve the image quality of their microscopes, which helps them to gain new insights and overall accelerates their work. 

A more radical application for Diffusion Models is in the field of chemistry, where they are used to find new molecules for a specific purpose, which vastly accelerates the process of finding the right molecule for a given problem, for example in drug discovery. One example of this is EDM, a Diffusion Model that can generate molecules in 3D.

<img width="750" alt="image" src="https://github.com/Florian-Dreyer/Florian-Dreyer.github.io/assets/64322175/6a157792-318e-4c63-8daa-80ded6fc6fdf">

Image from Hoogeboom et al. [[3]](#3)

# Why RAPHAEL? 

RAPHAEL has three main objectives:
* Higher aesthetic appeal
* Accurate reflection of concepts in generated images
* Accurate representation of text in generated images

<img width="750" alt="image" src="https://github.com/Florian-Dreyer/Florian-Dreyer.github.io/assets/64322175/5681446f-a574-46f3-aa80-6afbcbef8515">

Image from Xue et al. [[1]](#1)

The first objective, higher aesthetic appeal, is just a better looking image. As we can see with the images above, particularly with the ones in the first row, RAPHAEL generates very aesthetically appealing images, especially compared to the other models.

So what do I mean by "accurate reflection of concepts in generated images"? \
A good example for that are the images in the second row. 
The accurate reflection of the concepts in the text would be to generate an image with five cars that are on a road, as the text says.
As we can see, RAPHAEL is the only model of the ones shown above that actually shows five cars and not less or more.

The most important objective is accurately representing text in generated images. \
The third row is a great example of that.
Many Diffusion Models fail at displaying text in the generated images, they often just create fantasy text.
RAPHAEL successfully displays the word "RAPHAEL" in the image as the text says, in comparison other models like DALL-E2 make up words like "Raahel".

# Background - What is a Diffusion Model?

Diffusion Models learn to predict noise and remove it to restore structure in data.

<img width="750" alt="image" src="https://github.com/Florian-de/Florian-de.github.io/assets/64322175/ca2a8f9b-b597-473c-af9d-9263b7f4cb41">

Image from Steins [[4]](#4)

As shown in the picture above, Diffusion Models constist of two parts, the forward diffusion process and the reverse diffusion process. 

## Forward diffusion process

In the forward diffusion process the model takes an image as input and adds random noise to the image step by step, starting with the input image $x_0$ and ending in pure noise $x_t$. \
In each step, the process is defined as $q(x_t|x_{t−1}) := \mathcal{N}(x_t; \sqrt{1 − \beta_t}x_{t−1},\beta_tI)$ where $q$ is the process, $x_t$ the output of the current step, $x_{t-1}$ the output of the previous step and $\mathcal{N}$ the normal distribution with $\sqrt{1 − β_t}x_{t−1}$ as the mean $\mu$ and $\beta_tI$ as the variance $\sigma^2$. \
During this process, $\beta_t$ is controlled by a schedule and has values in the range of 0 to 1. Such a schedule could be as simple as a linear schedule, which would increase $\beta_t$ by a constant size each step, but in practice, more advanced schedules are used. \
For efficient computation, the entire process from $x_0$ to $x_t$ can be calculated using a closed form $q(x_t|x_0) = \mathcal{N}(x_t;\sqrt{\bar\alpha_t}x_0, (1 − \bar\alpha_t)I)$ where $\alpha_t := 1 − β_t$ and $\bar\alpha_t := \Pi^{t}_{s=1} \alpha_s$ [[5]](#5).

## Reverse diffusion process

In the reverse diffusion process the model tries to predict the total noise for each timestep, starting with the pure noise $x_t$ and ending in a denoised image $x_0$. \
For the prediction of the noise models typically use a modified UNet Neural Network Architecture. 

<img width="750" alt="image" src="https://github.com/Florian-Dreyer/Florian-Dreyer.github.io/assets/64322175/e72822b6-8866-4494-af7c-30b0365c55ca">


Image from Erdem [[6]](#6)

An example for such an architecture for a text-conditional model is shown above. \
It takes the total noise, the time step and text as input. \
The text is embedded by an encoder network which takes the text as input and has vectors as output. Each vector represents a single text token. Like shown below, the embedding vectors transport semantics. For example, the difference between the embedding of man and woman is similiar to the difference of uncle and aunt. 

<img width="750" alt="image" src="https://github.com/Florian-de/Florian-de.github.io/assets/64322175/66038dff-15fc-417e-8516-1b5d42f77937">

Image from 3Blue1Brown [[7]](#7)

The network starts and ends with a pink rectangle, which represents a ResNet block that takes the data from the previous layer as input. It is used to extract features from the image. \
The blue rectangles represent Downsample Blocks, which take data from the previous layer and data about the timestamp and the text embeddings as input. It is used to downsample the data from the previous layer to the size of the current layer. \
The grey arrows represent skip connections between the Downsampling and the Upsampling Blocks to prevent loss of information. \
The green rectangles represent Upsample Blocks, which takes data from the previous layer, data about the timestamp and the text embeddings and data from the skip connection as input. They are used to predict the noise. \
The orange rectangles represent Self-Attention Blocks, which take the data from the previous Downsample/Upsample Block as input. They are used to learn the connections between the different parts of the image. 

Let me walk you through the actual process of getting from $x_t$ to $x_{t-1}$

<img width="750" alt="image" src="https://github.com/Florian-Dreyer/Florian-Dreyer.github.io/assets/64322175/fd1e98d9-a785-4c69-bda2-0537fd3b016f">

Image from Steins [[4]](#4)

As the image above shows on the left side, the UNet gets the noisy image, the time step embedding, and, for text conditional models such as RAPHAEL, also text as input. \
The output is the predicted total noise in the image $\epsilon_{\theta}(x_t, t)$, so not the noise to get from $x_t$ to $x_{t-1}$ but the entire noise in $x_t$. \
To get to $x_{t-1}$ we follow the computation shown on the right side of the image above. \
We take the input $x_t$ and subtract a part, but only a part, of the predicted noise $\epsilon_{\theta}(x_t, t)$ from it. The details of this computation are shown in the formula in the image above.

# Background - What are Mixture of Experts?

In general, Mixture of Experts (MoE) is the method of replacing a single FFN layer with an expert layer consisting of a router and multiple FFNs, thereby dividing a problem. \
The experts (FFNs) share the same architecture and are trained by the same algorithm. The routing function assigns input data to the best experts. It is implemented by a Router Network, so it is trainable and not fixed. To speed up the inference time, a sparse gating function is used, which assigns the input only to the top-K experts. [[8]](#8) 
A MoE Layer takes the data from the previous layer as input data and outputs some kind of weighted combination of the outputs of the experts and sometimes also a skip connection.[[9]](#9)

<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Image Slider</title>
    <style>
        #slider {
            max-width: 750px;
            max-height: 450px;
            margin: auto;
        }
        #slider img {
            width: 100%;
            height: auto;
            display: none;
        }
        #slider img.active {
            display: block;
        }
    </style>
</head>
<body>
    <div id="slider">
        <img src="https://github.com/Florian-Dreyer/Florian-Dreyer.github.io/assets/64322175/17aadd61-adeb-4c39-a736-2baebeed6859" class="active">
        <img src="https://github.com/Florian-Dreyer/Florian-Dreyer.github.io/assets/64322175/57a1e46d-1703-414b-9e0c-d012d23882da">
        <img src="https://github.com/Florian-Dreyer/Florian-Dreyer.github.io/assets/64322175/9a23718a-1f32-4c52-91ed-1258790fd9ee">
        <img src="https://github.com/Florian-Dreyer/Florian-Dreyer.github.io/assets/64322175/9a56900f-502a-45b2-9d3e-f5815d1aa468">
        <img src="https://github.com/Florian-Dreyer/Florian-Dreyer.github.io/assets/64322175/9045bbd5-cc75-40fd-b42f-a1717c2ca931">
    </div>

    <script>
        let currentIndex = 0;
        const images = document.querySelectorAll('#slider img');
        const totalImages = images.length;

        setInterval(() => {
            images[currentIndex].classList.remove('active');
            currentIndex = (currentIndex + 1) % totalImages;
            images[currentIndex].classList.add('active');
        }, 1000); 
    </script>
</body>
</html>

Image from Hugging Face [[9]](#9)

Now let me walk you through the path of the input data in these two models. \
First, the data goes through the Self-Attention layer [[12]](#12), which is the same for both. After that it goes through the Add + Normalize layer, also the same for both. \
But in the next layer, there is a difference: The non-MoE model just passes the data through the single FFN layer, while the model with MoE first passes the data through the router, which then assigns a number of experts for it, in this example, only one. As we can see, with different input the router can assign different Experts. After the data went through the assigned Expert the output is calculated. \
The last step, passing the data through the second Add + Normalize layer, is again the same for both.

The use of MoE can provide benefits like overall better performance, efficient pretraining or faster inference compared to the use of a single MLP/FFN.

# RAPHAEL Architecture

As explained earlier, in general, a Diffusion Model consists of two parts: the forward diffusion process and the reverse diffusion process.

In the RAPHAEL model, the forward diffusion process is implemented as described in the section about Diffusion Models. \
For the reverse diffusion process, RAPHAEL uses an UNet Architecture as the denoising network. 

The general structure of the used Denoising Network is shown below.

<img width="750" alt="image" src="https://github.com/Florian-de/Florian-de.github.io/assets/64322175/5a8bce05-40a7-403a-991a-927c5d3e13a9">

Image from Xue et al. [[1]](#1)

It takes noise and text as input and outputs images. 

Now you may ask yourself, "But where is the difference to the previously described general Diffusion Models?" The answer lies in the Transformer Blocks.
The UNet architecture deployed in the RAPHAEL model consists of 16 transformer blocks, and I will now go into detail about them.

## The Transformer Block

Every Transformer Block consists of four key components: The Self-Attention layer, the Cross-Attention layer, the Time-MoE layer and the Space-MoE layer, as shown in the image below.


<img width="750" alt="image" src="https://github.com/Florian-de/Florian-de.github.io/assets/64322175/dba935c6-beb3-4891-8b5e-dfa3efd42295">

Image from Xue et al. [[1]](#1)

For an intuitive understanding, you can think of the different paths which the MoEs produce as different "painters," each responsible for a different part of the image, as the authors of the paper write.

As a loss function, RAPHAEL uses the following combined loss function $L = L_{\mathrm{denoise}} + L_{\mathrm{edge}}$.
The second part, $L_{\mathrm{edge}}$, is computed by the Edge-supervised Learning, which I will explain later in the section about Edge-supervised Learning.
For the first part, $L_{\mathrm{denoise}} = E_{t,x_0,ϵ∼N(0,I)} ∥ϵ − D_θ (x_t, t)∥^2_2$, represents the expected squared difference between the predicted noise and the actual noise in the image, as the actual noise is normally distributed with a mean of 0 and a variance of 1.

## What are Time-MoE?

Time-MoE are MoE layers that assign the image in different denoising time steps to different expert models. 

The Time-MoE layer takes the feature data from the Cross-Attention layer as input. The output of the layer is the output of the selected expert. \
A Time-MoE layer constists of a Text Gate Network and the expert models, as shown in the image below. It takes the time step and the features as input.


<img width="750" alt="image" src="https://github.com/Florian-de/Florian-de.github.io/assets/64322175/f6c0fbff-be5b-4a2e-afc9-11bca2800b1e">

Image from Xue et al. [[1]](#1)

The Time Gate Network is implemented as a feedforward network and chooses experts for the different features. This can be formulated with $h\prime(x) = te_{t_{\mathrm{router}}(t_i)}(h_c(x_t))$ where $h_c(x_t)$ represents the features from the Cross-Attention layer and $t_{\mathrm{router}}(t_i)=\mathrm{argmax}(\mathrm{softmax}(G′(E′_θ(t_i))+ϵ))$ which returns the index of the chosen expert. In the formula, $te_i$, represents the different experts. \
An example of the result of the assignments can be seen in the image below:

<img width="750" alt="image" src="https://github.com/Florian-de/Florian-de.github.io/assets/64322175/8eca0dbc-7edc-4985-bfaa-1440bc4df51e">

Image from Xue et al. [[1]](#1)

## What are Space-MoE?

Space-MoE is a MoE layer which assigns specific text tokens to their corresponding image regions. \
The layer takes the data from the Time-MoE layer as input. \
The output of the Space-MoE Layer is built by taking the mean of all expert models, calculated by the following formula: 
$\frac{1}{n_y} \Sigma_{i=1}^{n_y} e_{\mathrm{route}(y_i)}(h′(x_t) \cdot M_i)$ where $M_i$ is a binary two-dimensional matrix representing the image region the i-th text token should correspond to, $\cdot$ is the Hadamard product, and $h\prime(x_t)$ are the features from the Time-MoE layer.  

A Space-MoE layer consists of a Text Gate Network and the expert models, as shown in the image below. It takes text as input.

<img width="750" alt="image" src="https://github.com/Florian-de/Florian-de.github.io/assets/64322175/eba93db2-8496-464a-8262-0f009b1c3759">

Image from Xue et al. [[1]](#1)

The Text Gate Network does the assignment using the formula \$\mathrm{route}(y_i)=\mathrm{argmax}(\mathrm{softmax}(G(E_θ(y_i))+ϵ))\$, which returns the index of the corresponding expert.
It is implemented as a feedforward network with text tokens as input. \
As a result of the Space-MoE layer, as shown in the picture below, different categories activate different diffusion paths.

<img width="750" alt="image" src="https://github.com/Florian-de/Florian-de.github.io/assets/64322175/287fa16b-94d1-412d-87a9-e51d9fd3a60f">

Image from Xue et al. [[1]](#1)

## What is Edge-supervised Learning?

Edge-supervised Learning uses an edge detection module to extract boundary information, which is then used to supervise the model in preserving detailed image features. \
The module takes the attention map $M$ as input, and the output is the predicted edge map. \
Since with larger timesteps t the attention map loses detail, a hyperparameter $T_c$ is used to stop edge-supervised learning when t becomes too large.

The image below shows the attention map from the transformer block and the edges extracted by the edge detector next to the image.

<img width="750" alt="image" src="https://github.com/Florian-de/floriandreyer.github.io/assets/64322175/74df4905-9526-4a0f-aaa6-a7faa9e8b7fa">

Image from Xue et al. [[1]](#1)

Now to the loss function $L_{\mathrm{edge}} = \mathrm{Focal}(P_θ(M),I_{\mathrm{edge}})$, it is calculated using the predicted edge map $P_θ(M)$ and the edge map from the ground truth $I_{\mathrm{edge}}$, from which we take the focal loss. [[13]](#13)

(d) shows that nearly twice as many people prefer the results of the model using Edge-supervised Learning than people preferring the model without it.

# Ablation Study

The LAION-5B and some more internal datasets are used for the ablation study. Images from LAION-5B were previously filtered using the same aesthetic scorer as Stable Diffusion. Only images with a score of at least 4.7 and without watermarks are used. The text description from the LAION-5B datasets were cleaned by removing useless information, for example HTML tags.  

<img width="750" alt="image" src="https://github.com/Florian-de/Florian-de.github.io/assets/64322175/23f35b64-ee85-4886-b310-c7e28d1a2585">

Image from Xue et al. [[1]](#1)

The hyperparameter \$\alpha\$ results in an optimal FID-5k score at about 0.2, smaller and larger alpha values decrease the performance. \
This is explained by the fact that $\alpha$ is part of the threshold which decides if an entry in the Cross-Attention Map is 0 or not, so a bigger $\alpha$ leads to a sparser map, while a smaller to a less sparse map. As the paper describes, the value of 0.2 implies a balance between preserving adequate features and avoiding the use of unimportant features. \
The hyperparameter \$T_c\$ is optimal at about 500, for smaller values, it slowly decreases, and for larger values, it decreases fast. \
This result is logical because a small $T_c$ stops Edge-supervised Learning earlier, which can result in worse results, because the Edge-supervised Learning was used in fewer steps. On the other hand, a large $T_c$ stops edge-supervised learning very late, which can also worsen the results because the edge detection module is no longer able to retrieve useful information from the image at that timestep.

Experiments between the CLIP score and FID-5k show that the model with all three Space-MoE, Time-MoE and Edge-supervised Learning, is overall the best one. \
The best FID-5k value is achieved at a CLIP score of about 0.33 for all models except the model without Space-MoE which has its peak at a CLIP score of about 0.315. Since we want to achieve a high CLIP score and a low FID-5k value, the model with all the modules is, as previously mentioned the best one. This implies that all modules contribute effectively. Interestingly, the model without Space-MoE has its FID-5k peak with a significantly lower CLIP score than the other models while having a quite similar FID-5k value compared to the model without Time-MoE and the model without Edge-supervised Learning. This implies that the Space-MoE have a big contribution to the better text and image similarity of RAPHAEL, since the CLIP score measures the text and image similarity.

The number of experts can influence the FID-5k score and the computational complexity. \
The FID-5k score improves very quickly at the beginning but flattens out quite fast, for the Time-MoE earlier than for the Space-MoE. \
This shows that a larger number of both Space-MoE and Time-MoE has a positive effect on the quality of the output. 
However, the computational complexity grows with an increasing number of experts. \
This result makes sense since a larger number of experts increases the number of computations and therefore slows down the model. But overall, a model using a sparse MoE approach such as RAPHAEL will usually be more effient than a model using just a single more complex FFN/MLP for inference. 

# Experiments

For this section I chose three images from the paper which the authors generated with RAPHAEL. \
We will come back to the main objectives of RAPHAEL and see why these images fulfill them. 

<img width="750" alt="image" src="https://github.com/Florian-Dreyer/Florian-Dreyer.github.io/assets/64322175/118267ec-fff4-4bc6-ad7f-58c2a306a9eb">

Image from Xue et al. [[1]](#1)

The first objective was a higher aesthetic appeal. \
With the three images below, we can see that they are very aesthetically appealing and have a very high overall quality. 
The images also show that RAPHAEL is capable of generating images with different styles, so not only photorealistic or cartoon style but many different styles. 

The second objective was the accurate reflection of concepts in the generated images. \
I think the second image is a great example of that. If you go through the text that was used to generate the image, you can find all the concepts, like the waterfalls, streams or pools of golden glowing bioluminescent water, from the text in the generated image.

For the third objective, the accurate representation of text in generated images, there are no further examples in the paper, but I think the image below from the beginning shows that the model is able to achieve this objective. 

<img width="750" alt="image" src="https://github.com/Florian-Dreyer/Florian-Dreyer.github.io/assets/64322175/9bca3a4b-94df-4516-be55-2282502a2363">

Image from Xue et al. [[1]](#1)

# Benchmarks

For benchmarking, 30,000 images from the MS-COCO 256 x 256 dataset and the zero-shot Frechet Inception Distance (FID) were used. The FID score is usually calculated using the Inception v3 model, which compares the original dataset to the generated one in quality and diversity. 

<img width="750" alt="image" src="https://github.com/Florian-de/Florian-de.github.io/assets/64322175/a226c0c8-8725-4466-bb76-d4c03b79db10">

Image from Xue et al. [[1]](#1)

The table shows that RAPHAEL outperforms all competitors on the Zero-shot FID-30k. \
Especially in comparison with the two popular models, Stable Diffusion and DALL-E, it beets them by 21% and 37%.

# Discussion

The most obvious advantage of the model is the more accurate text in the generated images and the overall higher image quality. As discussed in the experiments section, the Space-MoE improve the text alignment significantly. However, all three key differences from RAPHAEL: The Space-MoE, the Time-MoE and the Edge-supervised Learning contribute equally to the image quality. This can be derived from the fact that, as discussed in the experiments section, the models without one of these components all peak at about the same FID-5k value. \
On the other hand, there is the high GPU usage for the training. The model was trained on 1,000 NVIDIA A100s for two months, totaling about 1.46 million A100 GPU hours. In comparison, Stable Diffusion was trained for about 150,000 A100 GPU hours [[10]](#10) [[11]](#11). The GPU usage for the training of RAPHAEL was about 10 times the GPU usage for Stable Diffusion. \
Another advantage is that the MoE architecture makes the models more efficient during inference due to sparse assignments. \
However, the fact that the model is not open source is also a drawback, as it reduces the benefit for the research community.

A possible future improvement is to enable the model to generate not only images but also videos, like OpenAI did with Sora. 
The generation of videos would open completely new use cases for the model.

To sum it up, RAPHAEL can generate images with accurate text representation, accurate reflection of concepts, and aesthetic appeal, beating competitors like Stable Diffusion or DALL-E2.

# References

<a id="1">[1]</a> 
Xue, Zeyue et al. (May 2023),
"RAPHAEL: Text-to-Image Generation via Large Mixture of Diffusion Paths", 
[NeurIPS 2023](https://arxiv.org/abs/2305.18295)

<a id="2">[2]</a> 
Adobe,
[Generative Fill & Expand](https://helpx.adobe.com/photoshop/using/generative-fill.html)

<a id="3">[3]</a> 
Hoogeboom, Emile et al. (Oct 2022), 
"Equivariant Diffusion for Molecule Generation in 3D“,
[ICML 2021](https://arxiv.org/abs/2203.17003)

<a id="4">[4]</a> 
Steins (Dec 2022),
[Diffusion Model Clearly Explained!](https://medium.com/p/cd331bd41166)

<a id="5">[5]</a> 
Ho, Jonathan et al. (Dec 2020), 
“Denoising Diffusion Probabilistic Models”, 
[NeurIPS 2020]("https://arxiv.org/abs/2006.11239")

<a id="6">[6]</a> 
Erdem, Kemal (Nov 2023),
[Step by Step visual introduction to Diffusion Models](https://erdem.pl/2023/11/step-by-step-visual-introduction-to-diffusion-models)

<a id="7">[7]</a> 
3Blue1Brown (Apr 2024),
[Attention in transformers, visually explained](https://www.youtube.com/watch?v=eMlx5fFNoYc&t=71s)

<a id="8">[8]</a> 
Zixiang Chen et al. (Aug 2022),
"Towards Understanding Mixture of Experts in Deep Learning",
[arXiv](https://arxiv.org/abs/2208.02813)

<a id="9">[9]</a> 
Hugging Face (Dec 2023),
[Mixture of Experts Explained](https://huggingface.co/blog/moe)

<a id="10">[10]</a> 
Emad Mostaque (Aug 2020),
[Post on X](https://x.com/emostaque/status/1563870674111832066)

<a id="11">[11]</a> 
Rombach, Robin et al. (Dec 2021), 
“High-Resolution Image Synthesis with Latent Diffusion Model”, 
[IEEE/CVF conference on computer vision and pattern recognition 2022]("https://arxiv.org/abs/2112.10752")

<a id="12">[12]</a>
Vaswani, Ashish et al. (Jun 2017), 
“Attention Is All You Need”, 
[NeurIPS 2017]("https://arxiv.org/abs/1706.03762")

<a id="13">[13]</a>
Lin, Tsung-Yi et al. (Aug 2017), 
“Focal Loss for Dense Object Detection”, 
[Proceedings of the IEEE International Conference on Computer Vision 2017]("https://arxiv.org/abs/1708.02002")


