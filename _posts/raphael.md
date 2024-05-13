---
title: 'RAPHAEL: Text-to-Image Generation via Large Mixture of Diffusion Paths'
date: 2024-05-16
permalink: /posts/2024/05/raphael
tags:
  - raphael
  - diffusion model
  - Mixture of Experts
---

In recent years Generative Artificial Intelligence has gained a lot of popularity and usage in our daily lives. Image Generation became popular with models like Stable Diffusion or DALL-E which showed the public what Image Generation is able to do. In this blog post, I aim to introduce and explain a new model, RAPHAEL, which outperforms models like Stable Diffusion and focuses on accurately displaying text in the generated images.

Disclaimer: This is a draft of the final blog post, so further details will be added in most sections

Why image generation?
======
Image Generation can be used to innovate in many areas. For example it can be used to vastly improve the speed and reduce the cost of making marketing ads, for example to create a simple poster we can just use the model to generate it for us instead of hiring actors, buying equipement and so on. Another area would be film making, where instead of manually animating scenes, which is time consuming and expensive, we can just generate the scenes. There are also many applications in Science and Engineering like for example discovering new drugs where these models can be used to find new molecules, which else would be a very time and money consuming process.

Recent models like Stable Diffusion or DALL-E have shown great success but often lack the ability to accurately display text in the generated images like shown below:

<img width="668" alt="image" src="https://github.com/Florian-de/floriandreyer.github.io/assets/64322175/6431ef1f-57b2-4473-a938-a13dc402b5de">

RAPHAEL aims at accurately displaying text in generated images and overall improving image quality.

How to map image generation to a Deep Learning problem
======

Input 
------
The Input for Text-Conditional Image Diffusion Models is just plain text in the application phase. 

Between the user and the Diffusion Model sits an encoder neural network which is responsible for extracting text tokens from the text and embedding them.
The embedding of a text token is a very large vector with many values. Like shown below the embedding vectors transport semantics, for example the difference between the embedding of man and woman is similiar to the difference of uncle and aunt. 

<img width="442" alt="image" src="https://github.com/Florian-de/floriandreyer.github.io/assets/64322175/91b74c71-9a42-4c27-8be5-f26389a05fda">

While training there is also a second input source for learning the denoising of images by feeding the model images, but I won't go into detail about that here.

Output
------
The output of the model are image tokens which are decoded to images by an image decoder.


Loss function 
------

The model uses two loss function combined, by addding them: $L = L_{denoise} + L_{edge}$

The first loss function $L_{denoise} = E_{t,x_0,ϵ∼N(0,I)} ∥ϵ − D_θ (x_t, t)∥^2_2$ is used for the denoising neural network, while the loss function $L_{edge} = Focal(P_θ(M),I_{edge})$ is used to train the transformer blocks with Edge-supervised Learning, Focal(·, ·) denotes the focal loss [19] employed to measure the discrepancy between the predicted and the “ground-truth” edge maps.

Further settings for training 
------
The model uses an AdamW optimizer with a learning rate of 1e-04, weight decay as regularization and a batch size of 2,000 for training. 

AdamW optimization is a stochastic gradient descent method that is based on adaptive estimation of first-order and second-order moments with an added method to decay weights per the techniques discussed in the paper, 'Decoupled Weight Decay Regularization' by Loshchilov, Hutter et al., 2019. (https://keras.io/api/optimizers/adamw/)

RAPHAEL Architecture
======

In general a Diffusion Model consists of two parts: the forward process and the denoising network.

In the forward process the image will get more and more random during the steps of adding gaussian noise with the following formula: $x_t = \sqrt{1-\bar{\alpha}_t}x_0 + \sqrt{\bar{\alpha}_t}ϵ_t$, where $\bar{\alpha}_t = \Pi _{i=1} ^{t} \alpha_i$, $x_0$ as the original image and $\epsilon_t$ being normally distributed as the noise. 

RAPHAEL uses an U-Net Architecture for the denoising network. It consists of 16 transformer blocks, which employ Mixture of Experts. 

![image](https://github.com/Florian-de/floriandreyer.github.io/assets/64322175/1809bc49-fbbd-4402-8717-484f0403f9c1)

What are Mixture of Experts?
------
In general Mixture of Experts (MoE) is the method of using multiple expert models instead of using just a single big model for dividing a problem.

In Transformer blocks the MoE method is normally implemented by replacing the MLP after the Attention Layer with a Gate Network followed by the multiple expert MLPs.

In the RAPHAEL model a transformer block consists of four key components, the Self Attention Layer, the Cross Attention Layer, the Time-MoE Layer and the Space-MoE Layer as the following image shows:

<img width="414" alt="image" src="https://github.com/Florian-de/floriandreyer.github.io/assets/64322175/e5d19f96-5af1-4ba7-97d7-80fc75212dfc">

What are Time-MoE?
------
Time-MoE is a MoE Layer which assigns the image in different denoising time steps to different expert models.

A Time-MoE layer constists of a Text Gate Network and the expert models as shown in the image below. It takes the time step and the features as input.

<img width="225" alt="image" src="https://github.com/Florian-de/floriandreyer.github.io/assets/64322175/d3d0eb5b-1656-4932-83ca-f3c4638861f0">

The Time Gate Network does the assignment using the following formula: $t\_router(t_i)=argmax(softmax(G′(E′_θ(t_i))+ϵ))$

The result of the assignments can be seen in the image below:

<img width="597" alt="image" src="https://github.com/Florian-de/floriandreyer.github.io/assets/64322175/37ea0970-9829-4005-9946-cb757f6d62ba">

What are Space-MoE?
------
Space-MoE is a MoE Layer which assigns specific text tokens to image regions.

A Space-MoE layer constists of a Text Gate Network and the expert models as shown in the image below. It takes text as input.

<img width="335" alt="image" src="https://github.com/Florian-de/floriandreyer.github.io/assets/64322175/3cefc66c-c482-49a0-a75b-f0bbafaba431">

The Text Gate Network does the assignment using the following formula: $route(y_i)=argmax(softmax(G(E_θ(y_i))+ϵ))$

The output of the Space-MoE Layer is built by taking the mean of all expert models, calculated by the following formula: 
$\frac{1}{n_y} \Sigma_{i=1}^{n_y} e_{route(y_i)}(h′(x_t) \cdot M_i)$

As a result of the Space-MoE Layer, as shown in the picture below, different categories activate different diffusion paths 

<img width="316" alt="image" src="https://github.com/Florian-de/floriandreyer.github.io/assets/64322175/681216f0-19cd-4359-abef-51d55217c280">


What is Edge-supervised Learning
------
Edge-supervised Learing uses a edge detection module to extract boundary information, which is then used to supervise the model in preserving detailed image features.

The image below shows the attention map from the transformer block and the edges extracted by the edge detector next to the image.

<img width="698" alt="image" src="https://github.com/Florian-de/floriandreyer.github.io/assets/64322175/74df4905-9526-4a0f-aaa6-a7faa9e8b7fa">

It shows that nearly twice as much people prefer the results of the model using Edge-supervised Learning than people prefering the model without it.

Comparison to other Diffusion Models like Stable Diffusion
------

Useful? -> MoE, Edge-supervised Learning key differences

<img width="564" alt="image" src="https://github.com/Florian-de/floriandreyer.github.io/assets/64322175/e1fc3f6e-3b86-4389-8a69-8cc20c5d2ab5">

Experiments and Benchmarks
======
Experiments on Hyperparams
------

<img width="634" alt="image" src="https://github.com/Florian-de/floriandreyer.github.io/assets/64322175/86940ad0-1702-42b7-9c18-52113266eed3">

The hyperparameter $\alpha$ results in an optimal FID-5k score at about 0.2, smaller and larger alpha values decrease the performance.
The hyperparameter $T_c$ is optimal at about 500, for smaller values is slowly decreases, for bigger is decreases fast.

Experiments on the CLIP score show that model with Space-MoE, Time-MoE and Edge-supervised Learning is the best one and the best CLIP score is between 0.33 and 0.34.

The number of Experts can influence the FID-5k score and the computational complexity. The computational complexity steadily decreases with the increase of the number of 
experts. The FID-5k score gets better very fast at the beginning but flattens out quite fast, for the Time-MoE faster than for the Space-MoE.

Benchmarks
------

<img width="640" alt="image" src="https://github.com/Florian-de/floriandreyer.github.io/assets/64322175/38885b83-95db-40fd-aa1b-bfce8e309df2">

The table shows that RAPHAEL outperforms all competitors on the Zero-shot FID-30k. Especially in comparison with the two popular models Stable Diffusion and DALL-E it has a 21% and 37% better score.

Discussion
======

The most obvious advantage of the model is the accurate text in the generated images. Another advantage is the overall higher image quality, partly due to the Edge-supervised Learning. Another advantage is that MoE Architectures make the models more efficient due to sparse assignements. On the other hand is the high GPU usage for the training and the fact that the model is not open source.