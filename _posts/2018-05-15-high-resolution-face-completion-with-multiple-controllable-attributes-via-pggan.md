---
layout: post
title: "[Summary] High Resolution Face Completion with Multiple Controllable Attributes via Fully End-to-End Progressive Generative Adversarial Networks"
description: ""
date: 2018-05-15
tags: [inpainting]
comments: true
share: true
use_math: true
---

![Fig1]({{ site.url }}/images/high_resolution_face_completion_pggan/Fig1.png "Fig1"){: .aligncenter}

[paper](https://arxiv.org/abs/1801.07632)  

<br />

#### Problems



1. 기존의 얼굴 이미지에 대한 image completion 방식들은  저화질의 사진만을 생성해낼 수 있었다.
2. 기존 대부분의 방법들은 이미지 생성에 있어 유저의 의견이 반영될 수 없었다.
3. 기존 방식들은 대부분 이미지의 후처리 또는 복잡한 추론과정을 요구했다.

<br />



#### Ideas



1. U-Net 기반 구조의 fully end-to-end progressive GAN을 이용하여 고화질의 이미지 생성을 해보자. 이때 새롭게 정의된 loss function이 이용될 것이다.
2. Multiple controllable attributes를 이용하여 생성되는 이미지의 내용을 유저가 조절할 수 있도록 한다.
3. 이미지 후처리를 요구하지 않으며 단 한번의 forward pass만으로 이미지를 생성한다. 

<br />



#### Problem Formulation



$$\Lambda$$ 를 이미지 격자(가령, 1024X1024 pixels)라고 하자. 이때 격자 $$\Lambda$$ 위에 정의된 RGB이미지를 $$I_\Lambda$$ 라고 한다. 이때 $$\Lambda_t$$ 와 $$\Lambda_c$$ 는 각각 완성시켜야할 target region과 남겨둬야할 context region을 의미하고($$\Lambda_t \cap \Lambda_c = \emptyset$$ and $$\Lambda_t \cup \Lambda_c = \Lambda$$ ), $$M_{\Lambda_t}$$ 와$$M_{\Lambda_c}$$ 는 각 target region, context region에 대한 mask를 의미한다. 



Image completion의 목적은 그럴듯한 이미지인 $$I^{\text{syn}}$$ 을 생성하는 것이며, 이는 observed image인 $$I^{\text{obs}}$$ ($$I^{real}$$로 부터 $$I_{\Lambda_t}^{\text{obs}}$$ 의 영역이 제거된 이미지로)부터 생성된다. 또한 generator는 attributes를 조정하여 원하는 방향의 이미지를 생성할 수 있다. 이 attributes는 다음과 같은 N차원의 벡터로 표현하도록 한다. $$A = (a_1, \dotsb , a_N)$$ with $$a_i \in \{0, 1\}$$. Generator는 다음과 같이 정의될 수 있다.



<center>
$$
I^{\text{syn}} = G(I^{\text{obs}}, M, A; \theta_G)
$$
</center>



$$\theta_G$$ 는 generator에 대한 모든 파라미터를 의미한다. 여기서 두 context region $$I_{\Lambda_c}^{\text{obs}}$$ 와 $$I_{\Lambda_c}^{\text{syn}}$$ 는 굉장히 유사하도록 유지되어야 한다. 



Generator는 고차원의 거대한 이미지 공간에서 conditional probability model $$P_G(I^{\text{syn}} \mid I^{\text{obs}},M,A)$$ 를 아주 근접하게 추정할 수 있도록 학습해야 한다.



<br />



#### The proposed Fully End-to-End Progressive GAN



![Fig2]({{ site.url }}/images/high_resolution_face_completion_pggan/Fig2.png "Fig2"){: .aligncenter}



GAN은 다음과 같이 discriminator와 generator 사이의 minmax게임으로 정의된다.



<center>
$$
min_G max_D L_{adv}(G, D) = E_{z \sim p_{noise}(z)} \big[ 1 - log(1 - D(G(z))) \big] + E_{I \sim p_{data}(I)} \big[ log D(I) \big]
$$
</center>



이 논문에서는 binary mask image와 attribute이미지를 encoder에 입력하여 latent vector z를 얻고, latent vector z와 attribute vector를 입력으로 완전한 이미지를 생성한다.



<center>
$$
G(I^{obs}, M, A; \theta_G) = G_{compl}(G_{enc}(I^{obs}, M; \theta_G^{enc}), A; \theta_G^{compl})
$$
</center>



위의 Fig2에서 볼 수 있는 것처럼 여기서의 $$G_{compl}$$ 과 $$G_{enc}$$ 는 대칭의 형태를 보인다.



Discriminator의 경우, ground truth image 또는 generator에 의해 생성된 이미지가 입력되며 attribute vector와 이미지의 판별에 대한 두 개의 output branches가 존재한다.



<center>
$$
D(I;\theta_D) = \{ D_{cls}(F(I;\theta_D^F);\theta_D^{cls}), D_{attr}(F(I;\theta_{D}^F); \theta_D^{attr}) \}
$$
</center>



다음으로는 학습에 대한 더 세부적인 사항들에 대해 살펴본다.



<br />

* **Generating $$I^{obs}$$ and $$A^{obs}$$:** 우선 mask M을 sample한 뒤에, M을 이용하여 target region을 제거한다. Mask M을 더는 방법에 대해 알아보자. 첫 번째로 all-zero one-channel image위로 임의의 위치에 임의의 크기의 직사각형을 설정한다. 두 번째로 low resolution noise(e.g. 4X4, drawn from uniform distribution) image가 생성되고 upsampling되면서 임의로 생성된 직사각형 또한 bi-linear interpolation을 통해 크기를 키워간다. 이 과정을 통해 continuous value를 가진 rectangular region을 얻게 되는데, threshold를 이용하여 이를 binary mask로 변환한다(denote by $$M \sim p_{mask}(M)$$). Atribute vector $$A^{obs}$$ 는 다음과 같은 방법을 통해 생성된다.

<center>
$$
A^{obs} =
\begin{cases}
\begin{align}
A^{real},  &&\text{if $p < 0.5$.} \\[2ex]
(a_1, \dotsb, 1-a_i, a_{i+1}, \dotsb, a_N; \text{for all $j$, $a_j \in A^{real}$}, &&\text{otherwise}.
\end{align}
\end{cases}
$$
</center>

where $$p \sim U(0,1)$$ and $$i \in [1, N]$$ is a randomly chosen index. (denote by $$A^{obs} \sim p_{attr}(A^{real})$$)

<br />

* **Adversarial Loss:**

<center>
$$
l(I^{real} M, I^{obs}, A^{obs} \mid G, D) = (1 - log(1 - D_{cls}(I^{syn}))) + logD_{cls}(I^{real}), \\
L_{adv}(G, D) = E_{I^{real} \sim p_{data}(I), M \sim p_{mask}(M), A^{obs} \sim p_{attr}(A^{real})} \big[ l(I^{real}, M, I^{obs}, A^{obs} \mid G, D) \big].
$$
</center>

<br />

* **Attribute Loss:** predicted attribute vector, $$\hat{A}^{real} = D_{attr}(I^{real})$$ 과 $$\hat{A}^{obs}=D_{attr}(I^{obs})$$ 와 각각의 target인, $$A^{real}$$ 과 $$A^{obs}$$ 간의 cross-entropy로 산출된다.

<center>
$$
l_{attr}(I^{real}, A^{real}, M, I^{obs}, A^{obs} \mid G, D) =\\ \sum_{i=1}^N (a_i^{real} log (\hat{a_i^{real}}) + (1 - a_i^{real})log(1-\hat{a_i}^{real})) +\\ \sum_{i=1}^N(a_i^{obs} log(\hat{a_i}^{obs}) + (1-a_i^{obs})log(1 - \hat{a_i^{obs}})).
$$
</center>

<br />

* **Reconstruction Loss:** 이 모델에서는 이미지 전체가 생성되기 때문에 target region과 context region에 대해서 reconstruction loss를 취해준다.

<center>
$$
l_{rec}(I^{real}, M, I^{obs}, A^{obs} \mid G) = \| \alpha \cdot M \odot (I^{real} - I^{syn}) \|_1 +\\
\|(1-\alpha) \cdot (1 - M) \odot (I^{real} - I^{syn}) \|_1.
$$
</center>

where $$\odot$$ represents element-wise multiplication and $$\alpha$$ is the trade-off parameter.

<br />

* **Feature Loss:** 이름 그대로 feature level에서의 reconstruction loss다. $$\phi_j$$ 는 j번째 layer에 대한 activation을 의미한다.

<center>
$$
l_{feat}(I^{real}, M, I^{obs}, A^{obs} | \phi, G) = \| \phi_j(I^{real}) - \phi_j(I^{syn})\|_2^2
$$
</center>

논문에서는 VGG16 pretrained on the ImageNet dataset의 relu2_2 layer가 이용되었다.

<br />

* **Boundary Loss:** target region과 context region 사이의 boundary가 더욱 매끄럽게 생성되게 하기위해 이용된다. Mask image M의 boundary를 blurring하여 w를 얻고, 이를 $$I^{real}$$ 과 $$I^{syn}$$ 의 pixel단위 차이에 대해 곱해준다. 이 방식으로 인해 boundary에 가까운 pixel일 수록 큰 loss를 얻게 될 것이다.

<center>
$$
l_{bdy}(I^{real}, M, I^{obs}, A^{obs} \mid G) = \| w \odot (I^{real} - I^{syn}) \|_1.
$$
</center>

<br />

* **Total Loss:** 위 다섯가지의 loss를 trade-off parameter $$\lambda$$ 와 곱한뒤 모두 더해주면 최종 Loss가 완성된다.

<center>
$$
min_G max_D L(G, D) = L_{adv} (G, D) + \lambda_{attr} \cdot L_{attr}(G, D) +\\
\lambda_{rec} \cdot L_{rec}(G) + \lambda_{feat} \cdot L_{feat} (G, \phi) + \lambda_{bdy} \cdot L_{bdy}(G).
$$
</center>

<br />



* **Network Architectures:** 

![Fig3]({{ site.url }}/images/high_resolution_face_completion_pggan/Fig3.png "Fig3"){: .aligncenter}

<br />



* **Progressive Training:** [PGGAN]({{ site.url }}/2018-05-09/pggan/)과 동일하다.



<br />

#### Experiments

![Fig4]({{ site.url }}/images/high_resolution_face_completion_pggan/Fig4.png "Fig4"){: .aligncenter}

![Fig5]({{ site.url }}/images/high_resolution_face_completion_pggan/Fig5.png "Fig5"){: .aligncenter}

![Fig6]({{ site.url }}/images/high_resolution_face_completion_pggan/Fig6.png "Fig6"){: .aligncenter}



<br />



### Limitation



1. Inference time은 짧으나 training time이 매우 길다. Titan Xp GPU로 1024x1024 생성모델을 만들기 위해 약 3주 정도의 시간이 걸렸다. 
2. 자세히 들여다보면 고화질의 이미지 생성 모델이 low-level skin texture를 학습하는데에는 실패한 것을 볼 수 있다.
3. context가 여러가지 세부적인 texture(예를 들어 주근깨)를 갖고있는 경우 이미지가 다소 뿌옇게 생성되었다.
4. 얼굴에 대한 symmetrical structure를 잡아내지 못했다. (가령, 양 눈의 색깔이 다르게 생성됨.)



<br />



### Conclusion



목적달성 ㅋ