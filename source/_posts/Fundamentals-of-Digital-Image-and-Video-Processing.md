---
title: Fundamentals of Digital Image and Video Processing
date: 2025-12-13 13:57:23
excerpt: 课程笔记
categories: [技术, 计算机视觉]
tags: [CV]
---

# Fundamentals of Digital Image and Video Processing

## Introduction

> Pinhole camera model

- **Concept:** A simplified geometric model mapping a 3D scene onto a 2D image plane via a small aperture (pinhole).

- **Mechanism:** Light rays pass through the pinhole, forming an **inverted** image on the sensor3.

- **Geometry:** Based on **similar triangles**.

- Formula:

  

  $$\frac{h_{object}}{d_{object}} = \frac{h_{image}}{d_{image}}$$

  

  *(Where $h$ is height/size and $d$ is distance from the pinhole)*.

- **Key Insight:** Smaller pinhole = sharper image but less light; Larger pinhole = brighter image but blurrier (geometric blur).

> Sampling and quantization of digital images: matrix representation of images, sampling theory and Nyquist rate

- **Digital Image Definition:** An image $f(x,y)$ where spatial coordinates $(x,y)$ and amplitude $f$ are finite, discrete quantities.
- **Sampling (Spatial Discretization):**
  - Converting continuous spatial coordinates into a discrete grid (rows/columns).
  - Determines the **spatial resolution** (image size, $M \times N$).
- **Quantization (Amplitude Discretization):**
  - Mapping continuous intensity values to discrete digital levels.
  - Determines **gray-level resolution** (e.g., 8-bit = 256 levels).
- **Matrix Representation:** A digital image is represented as a matrix (2D array) of numbers.
- **Sampling Theory & Nyquist Rate:**
  - **Nyquist Rate:** The minimum sampling rate required to fully reconstruct a signal without error.
  - **Rule:** Sampling frequency must be **$> 2 \times f_{max}$** (twice the highest frequency in the image) to avoid **aliasing** (artifacts/Moiré patterns).

>  Down-sampling and up-sampling, image interpolation.

- **Down-sampling (Decimation):**
  - Reduces image size (fewer pixels).
  - **Risk:** Aliasing. (Solution: Apply a low-pass filter/blur *before* down-sampling) .
- **Up-sampling (Zooming):**
  - Increases image size (more pixels).
  - Requires creating new pixel values between existing ones (interpolation).
- **Image Interpolation:**
  - Process of estimating unknown values at fractional image coordinates using known neighbors.
  - **Common Methods:**
    - **Nearest Neighbor:** Assigns value of closest pixel (fast, produces blocky "checkerboard" artifacts).
    - **Bilinear:** Weighted average of 4 nearest neighbors (smoother, blurs edges).
    - **Bicubic:** Weighted average of 16 nearest neighbors (sharper, computationally expensive).

## Intensity Transformations

### **I. Point-wise Methods **

> Point-wise methods: clipping and thresholding, negatives, log, Power-law, Piece-wise linear transforms, bit-plane slicing.
>
> What are they?

*Operates on single pixel coordinates $(x,y)$. Formula: $s = T(r)$.*

- **Clipping & Thresholding:**

  - **What:** Limits pixel values to a specific range (clipping) or converts them to binary 0/1 (thresholding).
  - **Use:** Segmentation or noise reduction.

- **Image Negatives:**

  - **Formula:** $s = L - 1 - r$ (where $L$ is gray levels, e.g., 256).
  - **Use:** Enhances white/grey details embedded in dark regions (e.g., medical X-rays).

- **Log Transformations:**

  - **Formula:** $s = c \log(1 + r)$.
  - **Use:** Expands low-intensity (dark) values and compresses high-intensity values. Essential for displaying Fourier spectra.

- **Power-law (Gamma) Transformations:**

  - **Formula:** $s = c r^\gamma$.
  - **$\gamma < 1$:** Expands dark values (brightens image).
  - **$\gamma > 1$:** Compresses dark values (darkens image).
  - **Use:** Gamma correction for display monitors (CRT/LCD) and contrast manipulation.

- **Piece-wise Linear Transforms:**

  - **Contrast Stretching:** Expands a narrow range of intensity levels to the full dynamic range $[0, L-1]$ to increase contrast.

    **Gray-level Slicing:** Highlights a specific range of intensities (e.g., organ tissue) while dimming or preserving others.

- **Bit-plane Slicing:**

  - **What:** Decomposes an image into binary images based on bit significance (e.g., 8 planes for 8-bit).
  - **Insight:** Higher order bits contain visual structure; lower order bits contain fine detail and noise.

### **II. Global Methods (Histogram Processing)**

> Global methods: histogram equalization, histogram specification.
>
> Why we do histogram equalization/specification, how do we implement them in practice?

*Operates based on the intensity distribution (histogram) of the entire image.*

#### **1. Histogram Equalization **

- **Why:** To flatten the histogram into a **uniform distribution**. It maximizes global contrast, making details visible in both dark and light areas automatically.

- **How to Implement (Discrete):**

  1. **Calculate Normalized Histogram (PDF):** $p_r(r_k) = n_k / MN$ (count pixels $n_k$ for each level $k$, divide by total pixels).

  2. Calculate CDF (Transformation Function): Sum the probabilities.

     $$s_k = (L-1) \sum_{j=0}^{k} p_r(r_j)$$

  3. **Round:** Round $s_k$ to the nearest integer.

  4. **Map:** Replace original gray level $r_k$ with calculated $s_k$.

#### **2. Histogram Specification**

- **Why:** HE is fixed/automatic and can be too harsh (wash-out effect). Specification creates an output image with a **specific, user-defined histogram shape** for controlled enhancement.

- **How to Implement:**

  1. **Equalize Input:** Transform input image $r$ to uniform domain $s$ using HE: $s = T(r)$.

  2. **Equalize Target:** Transform the *specified/target* histogram $z$ to uniform domain $v$ using HE: $v = G(z)$.

  3. Inverse Mapping: Since $s \approx v$ (both uniform), equate them and solve for $z$:

     $$z = G^{-1}(s) \rightarrow z = G^{-1}(T(r))$$.

  4. **Lookup:** Find the $z_q$ whose $G(z_q)$ is closest to $s_k$ and map input pixels to this $z_q$.

## Intensity Transformations

> ▸ Correlation and convolution, kernels, padding
>
> ▸ What filters we have covered? Are they linear? Are they separable? Are they translation invariant?
>
> ▸ How do we de-noise? How do we enhance sharp features?

Here is a systematic, concise cheat sheet based on **Lecture 3 (L3.pdf)** for your exam.

### **I. Fundamentals**

- **Kernel (Mask/Filter):** A small matrix (e.g., $3 \times 3$) defining the operation. The center coefficient aligns with the target pixel $(x,y)$.
- **Correlation:** Sliding the kernel over the image and computing the sum of products (dot product)2.
  - Formula: $g(x,y)=\sum \sum w(s,t)f(x+s,y+t)$.
- **Convolution:** Same as correlation but the **kernel is rotated by 180°** first.
  - Formula: $g(x,y)=\sum \sum w(s,t)f(x-s,y-t)$.
  - **Note:** If the kernel is **symmetric** (e.g., Gaussian, Box), Correlation = Convolution.
- **Padding:** Adding border pixels to handle undefined operations at image boundaries.
  - **Zero-padding:** Pad with 0s (black border).
  - **Replicate:** Pad with the edge pixel value.
  - **Mirror:** Pad with a reflection of the image border.

------

### **II. Filters Covered & Properties**

| **Filter**     | **Type**   | **Linear?** | **Separable?** | **Translation Invariant?** | **Function**                                                 |
| -------------- | ---------- | ----------- | -------------- | -------------------------- | ------------------------------------------------------------ |
| **Mean / Box** | Smoothing  | **Yes**     | **Yes**        | **Yes**                    | Averaging. Reduces noise but blurs edges.                    |
| **Gaussian**   | Smoothing  | **Yes**     | **Yes**        | **Yes**                    | Weighted averaging (bell shape). Isotropic (rotation invariant). Better than Box. |
| **Median**     | Smoothing  | **No**      | **No**         | **Yes**                    | Order-statistic. Best for **Salt-and-Pepper noise**. Preserves edges. |
| **Laplacian**  | Sharpening | **Yes**     | N/A            | **Yes**                    | 2nd-order derivative. Isotropic. Detects edges/details.      |
| **Highboost**  | Sharpening | **Yes**     | N/A            | **Yes**                    | Enhances high-freq details by adding them back to the original image. |

- **Linear:** Output is a linear combination (weighted sum) of input pixels.
- **Separable:** 2D kernel $w$ can be decomposed into two 1D vectors ($uv^T$). Allows faster computation.
- **Translation Invariant:** The same filter mask is applied everywhere; the operation does not depend on pixel location $(x,y)$.

> What are the parameters of a Gaussian Filter? How do theyimpact the results?

**Parameters:**

1. **Standard Deviation ($\sigma$):** This is the primary parameter.
2. **Kernel Size:** The spatial dimensions of the filter mask (e.g., $M \times M$).

**Impact on Results:**

- **$\sigma$ (Standard Deviation):** It controls the **amount of smoothing/blurring**. A larger $\sigma$ creates a wider "bell curve," resulting in a stronger blurring effect.
- **Kernel Size:** It is usually chosen based on $\sigma$ (typically size $\approx 6\sigma$) to capture the full shape of the Gaussian function. A larger $\sigma$ requires a larger kernel size to be effective.



Here are the questions repeated from the images and their simple, concise answers based on the course content.

> **Question 1: Important in practical computation. Why?**

- **Speed/Efficiency:** Separability reduces computational cost significantly. A 2D convolution with an $M \times M$ kernel requires **$M^2$** multiplications per pixel. Separating it into two 1D convolutions requires only **$2M$** multiplications.

> **Question 2: How can you decompose the Box and the Gaussian filters?**

- **Decomposition:** They are decomposed into a column vector multiplied by a row vector ($w = uv^T$).
  - **Box Filter:** A matrix of all 1s is formed by a column of 1s multiplied by a row of 1s.
  - **Gaussian Filter:** The 2D exponential function $e^{-(x^2+y^2)}$ separates mathematically into $e^{-x^2} \cdot e^{-y^2}$.

> **Question 3: When is it the most useful?**

- **Large Kernels:** It is most useful when the filter kernel size is **large**. The difference between $M^2$ operations and $2M$ operations becomes massive as the kernel gets bigger.

------

### **III. Applications**

#### **1. How to De-noise? (Smoothing)**

- **Method:** Apply **Low-pass filters** (Integration/Averaging).
- **Gaussian Noise:** Use **Mean** or **Gaussian** filters. They reduce variance by averaging independent noise20.
- **Salt-and-Pepper Noise:** Use **Median** filter. It rejects outliers (extreme black/white dots) without blurring edges like linear filters.

#### **2. How to Enhance Sharp Features? (Sharpening)**

- **Method:** Apply **High-pass filters** (Differentiation) to highlight intensity transitions.
- **Derivative Properties:**
  - **1st Order:** Detects ramps/steps (produces thick edges).
  - **2nd Order (Laplacian):** Detects fine details and produces **zero-crossings** at edges. Sensitive to noise.
- **Implementation (Highboost / Unsharp Masking):**
  1. Get high-frequency detail (mask) = Original - Blurred version.
  2. Add mask back to original: $g(x,y) = f(x,y) + k \times g_{mask}(x,y)$.



## Spectral Filtering

### **I. Fourier Transform (FT) Fundamentals**

- **Motivation:** To analyze signals in the **frequency domain**. Some features (patterns, noise, texture) are hard to see in the spatial domain but obvious in the spectral domain.
- **Formulation:**
  - **Forward FT (2D):** $F(u,v)=\int_{-\infty}^{\infty}\int_{-\infty}^{\infty}f(x,y)e^{-j2\pi(ux+vy)}dx~dy$. Decomposes image into sine/cosine waves.
  - **Inverse FT (2D):** $f(x,y)=\int_{-\infty}^{\infty}\int_{-\infty}^{\infty}F(u,v)e^{j2\pi(ux+vy)}du~dv$. Reconstructs the image.
  - **Convolution Theorem:** Spatial Convolution $\Leftrightarrow$ Frequency Multiplication. $\mathcal{F}\{f * g\} = F \cdot G$. This is the basis of spectral filtering.
- **Discretization (DFT):**
  - **2D DFT:** $F(u,v)=\sum_{x=0}^{M-1}\sum_{y=0}^{N-1}f(x,y)e^{-j2\pi(ux/M+vy/N)}$. Used for digital images.
  - **Centering:** Multiply input image by $(-1)^{x+y}$ to shift the DC component $F(0,0)$ (average intensity) to the center of the spectrum.
- **General Pipeline of Spectral Filtering:** 
  1. **Transform:** $f(x,y) \rightarrow$ DFT $\rightarrow F(u,v)$.
  2. **Filter:** Multiply by filter function $H(u,v)$: $G(u,v) = H(u,v)F(u,v)$.
  3. **Inverse Transform:** $G(u,v) \rightarrow$ IDFT $\rightarrow$ Take real part $\rightarrow g(x,y)$.

------

### **II. Filters in Frequency Domain**

- **Low-pass Filters (Smoothing):**
  - **Goal:** Attenuate high frequencies (edges, noise), pass low frequencies (smooth areas).
  - **Examples:**
    - **Ideal:** Sharp cutoff (Box). Causes **ringing artifacts** in spatial domain due to Sinc function lobes.
    - **Gaussian:** Smooth cutoff. No ringing (Fourier of Gaussian is Gaussian).
    - **Butterworth:** Trade-off between Ideal and Gaussian.
- **High-pass Filters (Sharpening):**
  - **Goal:** Attenuate low frequencies, pass high frequencies (edges, fine details).
  - **Examples:** Ideal High-pass, Gaussian High-pass, Laplacian (in frequency domain).
  - **Effect:** Enhances edges but emphasizes noise.
- **Selective Filters:**
  - **Goal:** Pass/Block specific frequency bands or directions.
  - **Examples:** Band-reject/pass filters (remove periodic noise like scanner lines), Notch filters.

------

### **III. Advantages & Disadvantages**

| **Filter Type** | **Advantages**                                       | **Disadvantages**                                 |
| --------------- | ---------------------------------------------------- | ------------------------------------------------- |
| **Ideal**       | Sharpest separation of frequencies.                  | **Ringing (Gibbs phenomenon)** in spatial domain. |
| **Gaussian**    | No ringing artifacts; smooth transition.             | Less sharp cutoff; might blur desirable details.  |
| **Butterworth** | Flexible control (order $n$) over sharpness/ringing. | Computationally more complex than Ideal.          |

------

### **IV. Answers to Questions**

**L4: Transform & Sampling**

- **Q: Ranges for $f(t)$ and $F(\mu)$?** 
  - **A:** $(-\infty, +\infty)$ for continuous variables.
- **Q: Sample at $t_0 \neq 0$?** 
  - **A:** Use **Sifting Property**: $\int f(t)\delta(t-t_0)dt = f(t_0)$.
- **Q: FT of unit impulse $\delta(t)$?** 
  - **A:** **1** (Constant). It contains all frequencies equally.
- **Q: Why sample $\tilde{F}(\mu)$ (DFT)?** 
  - **A:** Because the FT of a sampled (discrete) signal is periodic. Sampling one period is sufficient to reconstruct it.
- **Q: Specialness of $F(0,0)$?** 
  - **A:** It is the **DC component** (average intensity of the image). $F(0,0) = MN \times \text{Average}(f)$.
- **Q: Shifted rectangle has same spectrum magnitude?** 
  - **A:** Yes. Translation only changes the **Phase Angle**, not the Magnitude Spectrum.

**L5: Filtering**

- **Q: Image elements corresponding to low/high freq?** 
  - **A:** **Low:** Smooth backgrounds, sky, walls. **High:** Edges, noise, texture, bricks.
- **Q: Why Ideal Filter has unsatisfying details (Ringing)?** 
  - **A:** Sharp cutoff in frequency domain $\Leftrightarrow$ Sinc function in spatial domain (ripples).
- **Q: Why Gaussian rescues ringing?** 
  - **A:** FT of a Gaussian is still a Gaussian (smooth decay, no ripples).
- **Q:Role of $D_0$ in Gaussian? Same as spatial?** 
  - **A:** $D_0$ is cutoff frequency. It is **inversely proportional** to spatial standard deviation $\sigma$. Large $D_0$ (broad freq) $\to$ Small $\sigma$ (sharp spatial).
- **Q: Wrong with adding LP and HP for Band-reject?**
  - **A:** For Gaussian, simply subtracting/adding curves doesn't create a perfect "notch/crater" shape; precise formulas are needed to target specific bands $W$ around $C_0$.

## Image Restoration

### **1. Enhancement vs. Restoration**

- **Image Enhancement:**
  - **Nature:** Heuristic procedure.
  - **Criteria:** Subjective (pleasing to the eye).
  - **Goal:** Sharpen/highlight features of interest.
- **Image Restoration:**
  - **Nature:** Inverse procedure.
  - **Criteria:** Objective (mathematical fidelity).
  - **Goal:** Recover the original image ($f$) from degraded data ($g$).

### **2. Noise Models**

- **Model:** $g(x,y) = f(x,y) + \eta(x,y)$.
- **Common PDFs & Origins:**
  - **Gaussian:** Electronic circuit/sensor noise (low light, high temp).
  - **Rayleigh:** Range imaging.
  - **Erlang (Gamma) / Exponential:** Laser imaging.
  - **Impulse (Salt-and-Pepper):** Quick transients/switching errors.
  - **Periodic:** Electrical/electromechanical interference.
- **Filters:**
  - **Gaussian/Uniform:** Mean Filter.
  - **Impulse:** Order-statistics (Median) Filter.
  - **Periodic:** Band-reject/Notch filter (Spectral domain).

### **3. Degradation Models**

- **Model:** Linear, Position-Invariant (LPI).
  - Spatial: $g(x,y) = h(x,y) * f(x,y) + \eta(x,y)$.
  - Spectral: $G(u,v) = H(u,v)F(u,v) + N(u,v)$.
- **Estimation Methods:**
  1. **Observation:** Estimate from image background/edges.
  2. **Experimentation:** Measure Impulse Response (Point Spread Function).
  3. **Direct Modeling:** Mathematical derivation (e.g., Physics).
- **Examples:**
  - **Atmospheric Turbulence:** $H(u,v) = e^{-k(u^2+v^2)^{5/6}}$.
  - **Motion Blur:** Sinc function caused by relative motion during exposure.

### **4. Inverse Filtering**

- **Pipeline:** FFT of Image ($G$) $\rightarrow$ Divide by Degradation ($H$) $\rightarrow$ Inverse FFT.
- **Formula:** $\hat{F}(u,v) = G(u,v) / H(u,v)$.
- **Main Challenge:** **Noise Amplification.
  - At frequencies where $H(u,v) \approx 0$ (usually high freq), the term $N(u,v)/H(u,v)$ becomes huge, drowning the signal.
- **Overcoming:** Truncate (Low-pass) filtering or use Wiener Filtering.

### **5. Noise + Degradation (Wiener Filter)**

- **Goal:** Minimize Mean Square Error: $\sum (f - \hat{f})^2$.

- Formula:

  $$\hat{F}(u,v) = \left[ \frac{1}{H(u,v)} \frac{|H(u,v)|^2}{|H(u,v)|^2 + S_{\eta}(u,v)/S_{f}(u,v)} \right] G(u,v)$$

- Practical Approx: Replace unknown Signal-to-Noise Ratio (SNR) with constant $K$:

  $$\mathcal{W}(u,v) = \frac{1}{H(u,v)}\frac{|H(u,v)|^{2}}{|H(u,v)|^{2}+K}$$

------

### **Supplement: Answers to In-Slide Questions**

- **Q (Slide 12): Why calculate mean/variance of noises?**
  - **A:** To determine the parameters of the PDF (Probability Density Function) required to model the noise accurately for restoration.
- **Q (Slide 18): Filter for Gaussian noise & theoretical support?**
  - **A:** **Arithmetic Mean Filter**. Support: Averaging independent random variables (noise) with zero mean reduces the variance of the noise.
- **Q (Slide 24/25): What filter (for periodic noise)?**
  - **A:** **Band-reject** or **Notch Filter** (Spectral domain) to remove specific interference frequencies.
- **Q (Slide 32): Role of $k$ in turbulence model?**
  - **A:** $k$ represents the severity of turbulence. Higher $k$ = more severe blur (attenuates high frequencies more).
- **Q (Slide 34): Derive $H(u,v)$ for motion blur?**
  - **A:** Integration of the Fourier time-displacement property. Result is a **Sinc function** ($\frac{\sin(x)}{x}$) multiplied by a phase shift.
- **Q (Slide 40): Source of Inverse Filtering issue? Solution?**
  - **A:** Source is division by zero/near-zero values of $H(u,v)$ which amplifies noise. Solution is **Wiener Filtering** (regularization).
- **Q (Slide 45): Connection between Wiener and Inverse filter?**
  - **A:** If there is **no noise** ($S_\eta = 0$), the Wiener filter term becomes 1, and it reduces exactly to the **Inverse Filter.
- **Q (Slide 49): Geometric Mean Filter is a combination of what?**
  - **A:** It generalizes and combines **Inverse Filtering**, **Wiener Filtering**, and parametric smoothing filters via $\alpha$ and $\gamma$ parameters.

## Color Image Processing

### **1. Color Fundamentals**

- **Physics:** Visible spectrum **400nm (Blue) - 700nm (Red)**.
- **Perception Model:** Light sensed = Illumination $e(\lambda)$ $\times$ Reflectance $r(\lambda)$.
- **Human Visual System (HVS):**
  - **Rods:** ~120M, night vision, low-light, monochromatic.
  - **Cones:** ~6M, day vision, color sensitive. 3 types: **S** (Blue), **M** (Green), **L** (Red).
- **Discrimination:** Humans distinguish thousands of colors vs. only ~24-30 shades of gray.

### **2. Basic Color Models**

| **Model**   | **Type**    | **Use Case**                       | **Key Features**                                             |
| ----------- | ----------- | ---------------------------------- | ------------------------------------------------------------ |
| **RGB**     | Additive    | **Monitors, Cameras**              | Cube geometry. Colors created by adding light to black.      |
| **CMY(K)**  | Subtractive | **Printing**                       | Secondary colors of light. Absorb RGB. K added for true black & cost. |
| **HSI**     | Perceptual  | **Human Description, Image Proc.** | Decouples **Intensity** (I) from Color (H, S). **H**ue=Color attribute, **S**aturation=Purity. |
| **CIE XYZ** | Standard    | **Standardization**                | Device-independent. Derived from spectral data.              |

- **Conversions:**
  - **RGB $\to$ CMY:** $C=1-R$, $M=1-G$, $Y=1-B$ .
  - **CMY $\to$ CMYK:** $K = \min(C,M,Y)$, then normalize $C_{new} = (C-K)/(1-K)$ .
  - **RGB $\leftrightarrow$ HSI:** Geometric formulas (see slides if formulas required).

### **3. Gray-level vs. Color Images**

- **Descriptor:** Color is a powerful descriptor for object detection (e.g., skin color).
- **Data Structure:** Gray = Scalar function $f(x,y)$; Color = Vector function (e.g., $[R, G, B]^T$).
- **Noise:** In color images, noise is often component-independent (exists in each channel).

### **4. Color Image Processing**

- **Pseudo-Coloring:** Assigning colors to gray-scale intensity values.
  - *Purpose:* Visualization (human eye sees more colors than grays).
  - *Methods:* **Intensity Slicing** (Assign color to ranges) , **Gray $\to$ Color Transform** (Use separate curves for R, G, B channels).
- **Full-Color Processing:** Processing vector data directly.
  - *Approaches:*
    1. **Per-component:** Process R, G, B separately (e.g., smoothing).
    2. **Vector-based:** Process pixels as vectors.
  - *Equivalence:* These are equivalent only if the operation is linear and component-independent.

------

### **Supplement: Answers to Slide Questions**

1. **Why add Black (K) to CMY?** 
   - Mixing C+M+Y produces muddy brown, not true black.
   - High ink volume (C+M+Y) makes paper wet/hard to dry.
   - Black ink is cheaper than colored inks.
2. **Number of colors in full-color image?** 
   - Standard 24-bit RGB (8 bits per channel) = $2^{24} \approx 16.7$ million colors.
3. **Pseudo-Color Example 1 (Slide 43): What is the equivalent filter?** 
   - **Band-pass filter.** It selects a specific range of mid-tone intensities (yellow/red regions) to highlight, suppressing lows and highs.
4. **Color Smoothing: RGB vs. HSI?** 
   - **RGB:** Smooth each channel. Can introduce new colors (muddy edges) due to averaging vectors.
   - **HSI:** Smooth **Intensity (I)** only. Preserves original Hue (color) and Saturation. Generally preferred to avoid shifting object colors.
5. **Color Sharpening: RGB vs. HSI?** 
   - **RGB:** Sharpen each channel.
   - **HSI:** Sharpen **Intensity (I)** only. Do **not** sharpen Hue/Saturation, as this increases "speckle" noise without improving visual edges.
6. **Color Noise: HSI components (Slide 65)?**
   - **Which is less noisy?** Usually, **Intensity (I)** contains the coherent structural noise. **Hue (H)** is often very noisy in low-saturation (dark/white) regions because small RGB fluctuations cause large angle (hue) swings.
   - *Strategy:* Filter noise in the I component; handle H/S carefully.

### Calculation

(1) Converting a 24-bit RGB color (60,100,200) to HSI space. Please write down your calculation details. (Hint: Please normalize the value to [0,1] first).

**Calculation Details:**

1. Normalization:

   First, scale the RGB values to the range $[0, 1]$:

   

   $$r = \frac{60}{255} \approx 0.2353$$

   $$g = \frac{100}{255} \approx 0.3922$$

   $$b = \frac{200}{255} \approx 0.7843$$

2. Calculate Intensity ($I$):

   

   $$I = \frac{r + g + b}{3} = \frac{0.2353 + 0.3922 + 0.7843}{3} = \frac{1.4118}{3} \approx \mathbf{0.4706}$$

3. Calculate Saturation ($S$):

   

   $$S = 1 - \frac{3}{r + g + b}[\min(r, g, b)]$$

   $$S = 1 - \frac{3}{1.4118}(0.2353) = 1 - 2.125 \times 0.2353 = 1 - 0.5 = \mathbf{0.5}$$

4. Calculate Hue ($H$):

   First, calculate the angle $\theta$:

   

   $$\theta = \arccos\left\{ \frac{\frac{1}{2}[(r-g) + (r-b)]}{\sqrt{(r-g)^2 + (r-b)(g-b)}} \right\}$$

   - Numerator:

     

     $$0.5 \times ((0.2353 - 0.3922) + (0.2353 - 0.7843)) = 0.5 \times (-0.1569 - 0.549) = -0.35295$$

   - Denominator:

     

     $$(r-g)^2 = (-0.1569)^2 \approx 0.0246$$

     $$(r-b)(g-b) = (-0.549)(-0.3921) \approx 0.2153$$

     $$\text{Denom} = \sqrt{0.0246 + 0.2153} = \sqrt{0.2399} \approx 0.4898$$

   Calculate $\theta$:

   

   $$\theta = \arccos\left(\frac{-0.35295}{0.4898}\right) = \arccos(-0.7206) \approx 136.1^\circ$$

   Check Hue Condition:

   Since $b > g$ ($0.7843 > 0.3922$):

   

   $$H = 360^\circ - \theta = 360^\circ - 136.1^\circ = 223.9^\circ$$


​    Final Answer:

​    $$H \approx 223.9^\circ, \quad S = 0.5, \quad I \approx 0.471$$

## Image Compression and Watermarking

### **1. Fundamentals**

- **Compression Types:**
  - **Lossless:** No data loss ($C \approx 10\%$). For medical/legal/archival. Methods: Huffman, RLE, LZW.
  - **Lossy:** Data lost ($\hat{f} \neq f$), higher compression. Exploits visual perception limitations. Methods: JPEG, JPEG-2000.
- **Redundancy Sources:**
  - **Coding:** Codes used are longer than information entropy requires.
  - **Spatial/Temporal:** Correlation between neighboring pixels or video frames.
  - **Irrelevant (Psycho-visual):** Information ignored by human eyes (e.g., high frequencies).
- **Information Measurement:**
  - **Compression Ratio ($C$):** $C = n_1 / n_2$.
  - **Relative Redundancy ($R$):** $R = 1 - 1/C$.
  - **Entropy ($H$):** Avg bits/symbol limit. $H = -\sum P(a_j) \log P(a_j)$.
  - **Fidelity:**
    - *Objective:* RMSE (Root Mean Square Error), MSSNR .
    - *Subjective:* Human rating (1=Excellent, 6=Unusable).

### **2. Coding Methods**

- **Huffman (Lossless):** Removes **coding redundancy**. Assigns shorter binary codes to frequent symbols. Uniquely decodable.
- **Run-Length Encoding / RLE (Lossless):** Removes **spatial redundancy**. Encodes runs of identical pixels as `(count, value)`. Best for binary/graphics.
- **Symbol-based:** Uses dictionary for repeated patterns (e.g., text/texture atlas).
- **Predictive (Lossless/Lossy):** Encodes difference (error) between pixel and prediction: $e(n) = f(n) - \hat{f}(n)$. Error histogram is peaked (low entropy) .
- **Transform Coding:** Block transform $\rightarrow$ Quantize $\rightarrow$ Encode.

### **3. Compression Standards**

- **JPEG (Lossy):** 
  - **Transform:** **DCT** (Discrete Cosine Transform) on $8\times8$ blocks. Real numbers, high energy compaction.
  - **Steps:**
    1. **Level Shift/Block Split:** $8\times8$ blocks.
    2. **DCT:** Convert spatial to frequency domain.
    3. **Quantization:** **(Main Lossy Step)** Divide by quantization table. Removes high-freq details.
    4. **Zig-Zag Scan:** Groups zeros at end for efficient coding.
    5. **DPCM:** Differential coding for **DC** coefficients.
    6. **RLE:** Run-length coding for **AC** coefficients.
    7. **Huffman:** Final entropy coding.
- **JPEG-2000:**
  - **Transform:** **Wavelet Transform** (Spatial & Frequency localization).
  - **Features:** No blocking artifacts, supports progressive transmission (blur $\to$ sharp).

### **4. Watermarking**

- **Purpose:** Copyright, fingerprinting, authenticity, copy protection.
- **Visible:** Overlay image. $f_w = (1-\alpha)f + \alpha w$.
- **Invisible (LSB):** Embed in Least Significant Bits. Fragile.
- **Invisible (DCT/Robust):** Embed random sequence $\omega$ into $K$ largest DCT coefficients: $c'_i = c_i (1 + \alpha \omega_i)$ .

------

### 💡 **Answers to Slide Questions (Supplement)**

- **Slide 18: Why is subjective criteria necessary if we have objective?**
  - *Ans:* Objective metrics (like RMSE) do not always correlate with human perception. Images are ultimately viewed by humans, so perceptual quality matters most.
- **Slide 26: Decode `010100111100`**
  - *Ans:* Using the table: `01010` ($a_3$) $\to$ `011` ($a_1$) $\to$ `1` ($a_2$) $\to$ `1` ($a_2$) $\to$ `00` ($a_6$) .
- **Slide 27: Discrepancy between Huffman Avg Length & Entropy?**
  - *Ans:* Huffman maps symbols to integer bits. It reduces *coding* redundancy but ignores inter-pixel correlations (*spatial* redundancy). Entropy is the theoretical limit assuming all redundancy is removed.
- **Slide 30: Compress stripe image with RLE?**
  - *Ans:* Yes, extremely high compression ratio. Each row is a single "run" of one color.
- **Slide 44: Why $8\times8$ blocks? What if too large?**
  - *Ans:* Trade-off. **Too small:** Low compression (doesn't capture enough correlation). **Too large:** High computation and visible "blocking artifacts".
- **Slide 46: Why DCT and not DST?**
  - *Ans:* Images have a large DC component (average brightness). DCT (Cosine) is 1 at 0Hz, DST (Sine) is 0. DCT also implies even extension at boundaries, preventing discontinuities (artifacts).
- **Slide 49: Why Zig-Zag scan?**
  - *Ans:* After quantization, non-zero coeffs are in top-left (low freq) and zeros in bottom-right (high freq). Zig-zag creates long runs of zeros to optimize RLE efficiency.
- **Slide 50: Coding scheme for DC components?**
  - *Ans:* **DPCM (Differential Pulse Code Modulation)**. DC values of adjacent blocks are similar; encoding the *difference* requires fewer bits.

## Image Segmentation

### 1. Detecting Local Intensity Change

- **Concept:** Edges are detected via derivatives (discontinuities).
- **First Order Derivative:**
  - Gradient vector $\nabla f = [g_x, g_y]^T$.
  - Magnitude $M(x,y) \approx |g_x| + [cite_start]|g_y|$ (Edge strength).
  - Direction $\alpha(x,y) = \tan^{-1}(g_y/g_x)$ (Perpendicular to edge).
  - Produces thicker edges4.
- **Second Order Derivative:**
  - Laplacian $\nabla^2 f$.
  - Produces double-edge response at ramps; sensitive to noise.
  - **Zero-crossing:** Indicates the center of the edge.

### 2. Edge Detectors

- **Gradient Operators (Masks):**
  - **Roberts/Prewitt:** Simple difference filters.
  - **Sobel:** Smooths noise (center weight 2) while differentiating.
  - **Compass Masks:** Rotated masks to detect specific directions (N, NW, W, etc.).
- **Marr-Hildreth (LoG):**
  - **Logic:** Smooth with Gaussian ($G$) $\rightarrow$ Apply Laplacian ($\nabla^2$) $\rightarrow$ Find Zero-crossings.
  - **Formula:** $\nabla^2[G * f(x,y)]$ (Laplacian of Gaussian).
  - **Approximation:** Difference of Gaussians (DoG).
- **Canny Edge Detector (Standard):**
  1. **Smooth:** Gaussian filter to reduce noise.
  2. **Gradient:** Compute Magnitude and Angle.
  3. **Non-maxima Suppression:** Thin edges to 1-pixel width (keep only local max along gradient direction).
  4. **Hysteresis Thresholding:** Use two thresholds ($T_H, T_L$). Keep strong edges ($>T_H$); keep weak edges ($>T_L$) only if connected to strong ones.

### 3. Forming Edges (Local vs. Global)

- **Goal:** Link broken edge points caused by noise/lighting.
- **Local Processing:** Link pixels $(x,y)$ to neighbors if:
  - Gradient Magnitude is similar ($>T_M$).
  - Gradient Direction is similar ($Angle \pm T_A$).
  - Fill gaps $\le$ length $L$.

### 4. Hough Transform (Global Processing)

- **Purpose:** Find global shapes (lines) among disconnected edge points.
- **Representation:** Use Polar coordinates to avoid slope $\infty$ issues.
  - Equation: $x \cos\theta + y \sin\theta = \rho$.
- **Duality (Connection):**
  - **Image Domain $(x,y)$:** A single point. $\rightarrow$ **Parameter Domain $(\rho, \theta)$:** A sinusoidal curve.
  - **Image Domain:** Points on the same line. $\rightarrow$ **Parameter Domain:** Curves intersecting at one point $(\rho', \theta')$.
- **Algorithm (Finding lines):**
  1. Quantize $(\rho, \theta)$ space into accumulator cells.
  2. **Vote:** For each edge point $(x_i, y_i)$, increment all cells satisfying $x_i \cos\theta + y_i \sin\theta = \rho$.
  3. **Peak:** Local maxima in accumulator = parameters of most likely lines.

------

### 💡 Supplement: Answers to Slide Questions

- **Slide 8: Why prefer central difference $f'(x)=0.5(f(x+1)-f(x-1))$?**
  - **Answer:** It is symmetric around $x$ and accounts for both forward and backward data, providing a more accurate estimate of the derivative.
- **Slide 9: Write the 3rd-order derivative.**
  - **Answer:** $f'''(x) = f(x+2) - 3f(x+1) + 3f(x) - f(x-1)$ (Derived from differencing the 2nd order).
- **Slide 11: Is 2nd order strong response to fine detail always good?**
  - **Answer:** No. It is highly sensitive to noise and produces a "double-edge" response at slow transitions (ramps), which complicates detection.
- **Slide 16: Commonality of directional gradient operators?**
  - Answer:** The sum of coefficients is always 0 (zero response in flat areas), and they are rotated versions of a base mask to maximize response in specific directions ($0^\circ, 45^\circ$, etc.).
- **Slide 27: Main issue of Local Processing for edge linking?**
  - **Answer:** It relies only on local neighborhoods, making it sensitive to noise and unable to bridge large gaps or recognize global structures (like a dotted line).
- **Slide 30: Problem with $y=ax+b$ in Hough Transform?**
  - **Answer:** Vertical lines have infinite slope ($a = \infty$), which cannot be represented in a bounded parameter space.
- **Slide 42: Small objects / Dominant background (Histogram is unimodal)?**
  - **Answer:** Global thresholding (Otsu) fails. **Solution:** Use **Edge information**. Compute the histogram only for pixels with high gradient magnitude (edges) to isolate the object's intensity distribution.
- **Slide 51: Issue with K-Means segmentation?**
  - **Answer:** It is strictly based on color/intensity distance and ignores spatial location. This leads to discontinuous, scattered, or noisy regions (no spatial coherence).

## Image Feature Extraction

Here is a systematic cheat sheet summary of Lecture 10, designed for quick exam reference.

### **I. General Concepts**

- **Feature Definition:** A distinctive attribute used to label or differentiate image regions.
- **Ideal Features:** Should be independent of **location, rotation, and scale** (invariant).
- **Intuition for Detection:**
  - **Flat:** No intensity change in any direction.
  - **Edge:** No change along the edge; significant change perpendicular to it.
  - **Corner:** Significant intensity change in **all directions.

------

### **II. Harris Corner Detection**

#### **1. Mathematical Formulation**

- **Objective:** Maximize the weighted sum of squared differences (SSD) via a sliding window.

- **Approximation:** Uses Taylor expansion to create a quadratic form: $C(x,y) \approx [x,y] M [x,y]^T$.

- Structure Tensor ($M$):

  $$M = \sum_{s,t} w(s,t) \begin{bmatrix} f_x^2 & f_x f_y \\ f_x f_y & f_y^2 \end{bmatrix}$$

  where $f_x, f_y$ are gradients and $w(s,t)$ is the window function (Box or Gaussian).

#### **2. Eigenvalue Analysis**

Let $\lambda_1, \lambda_2$ be eigenvalues of $M$:

- **Flat:** $\lambda_1 \approx 0, \lambda_2 \approx 0$ (Both small).
- **Edge:** $\lambda_1 \gg \lambda_2$ or $\lambda_2 \gg \lambda_1$ (One large, one small).
- **Corner:** $\lambda_1 \approx \lambda_2$ and both are **large**.

#### **3. Response Function ($R$)**

To avoid explicit eigenvalue computation:

- **Formula:** $R = \det(M) - k \cdot \text{Tr}^2(M) = \lambda_1\lambda_2 - k(\lambda_1+\lambda_2)^2$.
- **Decision:** If $R > T$ (Threshold), it is a corner.
- **Constants:** $k$ usually $\in (0, 0.25)$.

#### **4. Invariance Properties**

- **Rotation:** Invariant (Eigenvalues do not change with rotation).

- **Translation:** Invariant.
- **Scale:** **Not Invariant** (A corner may look like an edge when zoomed in).

------

### **III. SIFT (Scale-Invariant Feature Transform)**

- **Goal:** Extract features invariant to scale and rotation (Solves Harris's limitation).
- **The 6-Step Pipeline:**
  1. **Scale Space Construction:** Use Gaussian kernel $L(x,y,\sigma)$ and octaves (downsampling) to mimic blur/distance.
  2. **Initial Keypoints:** Calculate Difference of Gaussians (DoG): $D(x,y,\sigma) = L(k\sigma) - L(\sigma)$. Find local Max/Min in $3 \times 3 \times 3$ neighborhood.
  3. **Localization:** Use Taylor expansion to refine location (sub-pixel) and remove low contrast points.
  4. **Eliminate Edges:** Use Hessian Matrix $H$. Remove if curvature ratio is high (similar to Harris): $\frac{\text{Tr}(H)^2}{\det(H)} > \frac{(r+1)^2}{r}$.
  5. **Orientation:** Compute gradient magnitude and angle. Build 36-bin histogram. Assign peak direction to keypoint (Ensures rotation invariance).
  6. **Descriptor:** $16 \times 16$ region $\to$ $4 \times 4$ sub-regions. 8-bin orientation histogram per sub-region. **Result: 128-dimensional vector** ($4 \times 4 \times 8$).

------

### **IV. Topological Features**

- **Euler Number ($E$):** $E = C - H$
  - $C$: Number of Connected Components.
  - $H$: Number of Holes.
- **Alternative:** $E = V - E + F$ (Vertices - Edges + Faces).

------

### **V. Slides Q&A (High Probability Exam Questions)**

**Q1: How would you define a corner point?** 

- **A:** A point where intensity changes significantly in **all directions.

**Q2: What is the essential quantity we need to compute (for corners)?** 

- **A:** The weighted sum of squared differences (SSD) of the patch intensity as it shifts, approximated by the matrix $M$ involving image gradients $f_x, f_y$.

**Q3: What are the roles of $R, k, T$ in Harris?** 

- **$R$ (Response):** A scalar score indicating "cornerness" derived from eigenvalues.
- **$k$:** An empirical constant ($0.04-0.06$) to weight the trace vs. determinant.
- **$T$ (Threshold):** Minimum score required to accept a point as a corner (noise filtering).

**Q4: Is Harris invariant to Rotation, Translation, or Scale?** 

- **A:** Invariant to **Rotation and Translation**, but **NOT Scale**.




## 计算题

这是一份针对《数字图像处理》Final Review 课件中高频考点的**计算题模拟**。每道题都配有标准、简洁的解题步骤，适合作为考试时的答题模板。

------

### 1. 图像存储量计算 (Image Acquisition)

**考点来源：** Sampling and quantization 1

题目：

假设有一幅 $1024 \times 1024$ 像素的图像，具有 256 个灰度级。请计算存储该图像所需的未压缩数据量（以 MB 为单位）。

**解答：**

1. 确定位深度 (Bit Depth, $k$)：

   灰度级 $L = 256 = 2^8$，因此每个像素需要 $k = 8$ bits (即 1 byte)。

2. 计算总比特数 (Total Bits)：

   

   $$\text{Size} = M \times N \times k = 1024 \times 1024 \times 8 \text{ bits}$$

3. 单位换算 (Convert to MB)：

   

   $$\text{Size (Bytes)} = \frac{1024 \times 1024 \times 8}{8} = 1024 \times 1024 \text{ Bytes}$$

   $$\text{Size (MB)} = \frac{1024 \times 1024 \text{ Bytes}}{1024 \times 1024} = 1 \text{ MB}$$

**答案：** 1 MB

------

### 2. 直方图均衡化 (Histogram Equalization)



**考点来源：** Histogram equalization implementation 2222

题目：

假设一幅 3 bit 的图像 ($L=8$)，总像素数为 $n=64$。已知其灰度分布如下表。请计算均衡化后的灰度级 $s_k$，并填写映射表。

| **灰度级 rk** | **0** | **1** | **2** | **3~7** |
| ------------- | ----- | ----- | ----- | ------- |
| 像素数 $n_k$  | 16    | 16    | 32    | 0       |

解答：

步骤 1：计算概率 $p_r(r_k)$ 和 累积分布函数 $S_k$

$$p_r(r_k) = \frac{n_k}{n}, \quad S_k = \sum_{j=0}^{k} p_r(r_j)$$

- $k=0$ ($r_0=0$):

  $p_r(0) = 16/64 = 0.25$

  $S_0 = 0.25$

- $k=1$ ($r_1=1$):

  $p_r(1) = 16/64 = 0.25$

  $S_1 = 0.25 + 0.25 = 0.50$

- $k=2$ ($r_2=2$):

  $p_r(2) = 32/64 = 0.50$

  $S_2 = 0.50 + 0.50 = 1.00$

步骤 2：计算映射后的灰度 $s_k$

公式：$s_k = \text{round}((L-1) \times S_k) = \text{round}(7 \times S_k)$

- $s_0 = \text{round}(7 \times 0.25) = \text{round}(1.75) = 2$
- $s_1 = \text{round}(7 \times 0.50) = \text{round}(3.5) = 4$
- $s_2 = \text{round}(7 \times 1.00) = \text{round}(7) = 7$

最终映射结果：

原灰度 $0 \rightarrow 2$

原灰度 $1 \rightarrow 4$

原灰度 $2 \rightarrow 7$

------

### 3. 空间滤波 (Spatial Filtering)

**考点来源：** Correlation and convolution calculations 

题目：

给定图像子区域 $f$ 和滤波器核 $w$ (如下)。假设不进行 Padding，计算结果图像中心位置的像素值。请分别计算：(1) 相关 (Correlation), (2) 卷积 (Convolution)。

$$f(x,y) = \begin{bmatrix} 1 & 2 & 3 \\ 4 & 5 & 6 \\ 7 & 8 & 9 \end{bmatrix}, \quad w(x,y) = \begin{bmatrix} 0 & 0 & 0 \\ 0 & 1 & 0 \\ 0 & 0 & -1 \end{bmatrix}$$

解答：

中心位置对应 $f(x,y)$ 中的 5。

(1) 相关 (Correlation): $f \circ w$

直接将核 $w$ 覆盖在图像上，对应元素相乘并求和。

$$\text{Result} = (1\cdot0) + (2\cdot0) + (3\cdot0) + (4\cdot0) + (5\cdot1) + (6\cdot0) + (7\cdot0) + (8\cdot0) + (9\cdot(-1))$$

$$\text{Result} = 5 - 9 = -4$$

(2) 卷积 (Convolution): $f * w$

先将核 $w$ 旋转 180 度得到 $w'$，再进行相关操作。

旋转后的核 $w' = \begin{bmatrix} -1 & 0 & 0 \\ 0 & 1 & 0 \\ 0 & 0 & 0 \end{bmatrix}$

$$\text{Result} = (1\cdot(-1)) + (5\cdot1) + \text{(其余项为0)}$$

$$\text{Result} = -1 + 5 = 4$$

------

### 4. 图像压缩 - 霍夫曼编码 (Huffman Coding)

**考点来源：** Coding methods: Huffman 4

题目：

信源包含 5 个符号，其概率如下。请构建霍夫曼树，给出各符号的编码，并计算平均码长 $L_{avg}$。

Symbols: $\{a_1: 0.4, a_2: 0.2, a_3: 0.2, a_4: 0.1, a_5: 0.1\}$

**解答：**

**步骤 1：构建霍夫曼树 (归并概率最小项)**

1. 初始: $0.4, 0.2, 0.2, \mathbf{0.1, 0.1}$
2. 合并 $0.1+0.1=0.2$。新序列: $\mathbf{0.4, 0.2, 0.2, 0.2}$ (排序后)
3. 合并 $0.2+0.2=0.4$。新序列: $\mathbf{0.4, 0.4, 0.2}$
4. 合并 $0.2+0.4=0.6$。新序列: $\mathbf{0.6, 0.4}$
5. 合并 $0.6+0.4=1.0$。

**步骤 2：分配编码 (路径: 大概率给0，小概率给1，反之亦可)**

- $a_1 (0.4) \rightarrow 1$ (举例)

- $a_2 (0.2) \rightarrow 00$

- $a_3 (0.2) \rightarrow 011$

- $a_4 (0.1) \rightarrow 0100$

- $a_5 (0.1) \rightarrow 0101$

  (注：具体编码取决于每次合并时0/1的分配，但码长是固定的)

  修正后的典型对半分：

- $a_1 (0.4)$: `1` (长度 1)

- $a_2 (0.2)$: `01` (长度 2)

- $a_3 (0.2)$: `000` (长度 3) - *需根据画出的树枝确定，假设此处结果*

- (最简单的验证是看码长是否符合概率大小)

更稳妥的计算 $L_{avg}$ 方法：

平均码长 = $\sum (p_i \times \text{length}_i)$

在上述归并过程中，每次“合并”操作贡献了等于合并概率之和的码长成本。

或者直接画树：

1. (0.1, 0.1) -> 0.2 [层数增加]
2. (0.2, 0.2) -> 0.4
3. (0.2, 0.4) -> 0.6
4. (0.4, 0.6) -> 1.0

最终码长通常为：

$a_1$: 1 bit ($0.4$)

$a_2$: 2 bits ($0.2$)

$a_3$: 3 bits ($0.2$)

$a_4$: 4 bits ($0.1$)

$a_5$: 4 bits ($0.1$)

$$L_{avg} = 0.4(1) + 0.2(2) + 0.2(3) + 0.1(4) + 0.1(4)$$

$$L_{avg} = 0.4 + 0.4 + 0.6 + 0.4 + 0.4 = 2.2 \text{ bits/symbol}$$

------

### 5. 图像分割 - 梯度幅度 (Edge Detection)

**考点来源：** Edge detectors 5

题目：

使用 Prewitt 算子计算图像中心点 $(x,y)$ 的梯度幅度。

图像区域 $Z$: $\begin{bmatrix} 10 & 10 & 10 \\ 10 & 50 & 10 \\ 10 & 10 & 10 \end{bmatrix}$

Prewitt 算子:

$H_x = \begin{bmatrix} -1 & 0 & 1 \\ -1 & 0 & 1 \\ -1 & 0 & 1 \end{bmatrix}$, $H_y = \begin{bmatrix} -1 & -1 & -1 \\ 0 & 0 & 0 \\ 1 & 1 & 1 \end{bmatrix}$

解答：

步骤 1：计算 $x$ 方向梯度 $G_x$

对应位置相乘求和 (注意 $H_x$ 的中心列是 0，只计算左右列差值)。

$$G_x = (10 \cdot 1 + 10 \cdot 1 + 10 \cdot 1) - (10 \cdot 1 + 10 \cdot 1 + 10 \cdot 1) = 30 - 30 = 0$$

步骤 2：计算 $y$ 方向梯度 $G_y$

对应位置相乘求和 (注意 $H_y$ 的中心行是 0，只计算上下行差值)。

$$G_y = (10 \cdot 1 + 10 \cdot 1 + 10 \cdot 1) - (10 \cdot 1 + 10 \cdot 1 + 10 \cdot 1) = 30 - 30 = 0$$

*(注：这是一个孤立点，在这个特定的 Prewitt 算子下，因为周围对称，梯度正好抵消。如果换成 Sobel 且周围不对称，会有值。)*

修改题目数据以展示计算过程：

若图像为 $\begin{bmatrix} 10 & 50 & 50 \\ 10 & 50 & 50 \\ 10 & 50 & 50 \end{bmatrix}$ (垂直边缘)

1. $G_x$: (右列 - 左列)

   $$G_x = (50+50+50) - (10+10+10) = 150 - 30 = 120$$

2. $G_y$: (下行 - 上行)

   $$G_y = (10+50+50) - (10+50+50) = 0$$

3. 幅值 $M(x,y)$:

   $$M = \sqrt{G_x^2 + G_y^2} = \sqrt{120^2 + 0} = 120$$

   或者使用绝对值近似：$M \approx |G_x| + |G_y| = 120$

------

### 6. 颜色模型转换 (Color Image Processing)

考点来源： Slide 10 1

Review 中明确提到 "Basic color models... How to convert from one to the other?"

题目：

已知一个像素的 RGB 分量（归一化到 [0, 1]）为 $R=0.2, G=0.6, B=0.8$。

1. 计算其 CMY（青、品红、黄）分量。
2. 计算其 HSI 模型中的亮度（Intensity, I）分量。

解答：

1. RGB 转 CMY：

CMY 是减色模型，转换非常简单：

$$C = 1 - R = 1 - 0.2 = 0.8$$

$$M = 1 - G = 1 - 0.6 = 0.4$$

$$Y = 1 - B = 1 - 0.8 = 0.2$$

(注：如果考 CMYK，则 $K = \min(C, M, Y) = 0.2$，然后修正 C,M,Y)

2. 计算亮度 I：

HSI 模型中的亮度是 RGB 的平均值：

$$I = \frac{R + G + B}{3} = \frac{0.2 + 0.6 + 0.8}{3} = \frac{1.6}{3} \approx 0.533$$

------

### 7. 霍夫变换 (Hough Transform)

考点来源： Slide 12 2

Review 中明确提到 "How to build connection between lines in the image domain and points in the parameter domain ($p, \theta$)?"

题目：

在图像空间 $(x, y)$ 中有一个边缘点 $A(1, 1)$。请使用霍夫变换公式 $\rho = x \cos\theta + y \sin\theta$，计算当 $\theta$ 分别取 $0^\circ, 45^\circ, 90^\circ$ 时对应的 $\rho$ 值。

**解答：**

- 当 $\theta = 0^\circ$:

  $\rho = 1 \cdot \cos(0^\circ) + 1 \cdot \sin(0^\circ) = 1 \cdot 1 + 1 \cdot 0 = 1$

- 当 $\theta = 45^\circ$:

  $\rho = 1 \cdot \cos(45^\circ) + 1 \cdot \sin(45^\circ) = 1 \cdot \frac{\sqrt{2}}{2} + 1 \cdot \frac{\sqrt{2}}{2} = \sqrt{2} \approx 1.414$

- 当 $\theta = 90^\circ$:

  $\rho = 1 \cdot \cos(90^\circ) + 1 \cdot \sin(90^\circ) = 1 \cdot 0 + 1 \cdot 1 = 1$

*考试意义：如果在参数空间中，来自另一个点 $B$ 的曲线也经过 $(\rho, \theta) = (1.414, 45^\circ)$，则说明点 A 和 B 在图像上位于同一条 45 度角的直线上。*

------

### 8. 逆滤波与噪声放大 (Inverse Filtering)

考点来源： Slide 9 3

Review 提到 "Inverse Filtering... main challenge... How to take care of noise"

题目：

假设图像退化模型为 $G(u,v) = H(u,v)F(u,v) + N(u,v)$。

已知某频率点 $(u_0, v_0)$ 处：

- 退化函数 $H(u_0, v_0) = 0.1$

- 原始信号 $F(u_0, v_0) = 100$

- 噪声 $N(u_0, v_0) = 5$

  请计算：

1. 该点处的观测值 $G$。
2. 直接使用逆滤波 $\hat{F} = G / H$ 恢复出的信号值。
3. 计算恢复误差，并说明为什么逆滤波在此处效果不好。

解答：

1. 计算观测值 $G$：

$$G = H \cdot F + N = 0.1 \times 100 + 5 = 10 + 5 = 15$$

2. 逆滤波恢复 $\hat{F}$：

$$\hat{F} = \frac{G}{H} = \frac{15}{0.1} = 150$$

3. 误差分析：

实际值 $F = 100$，恢复值 $\hat{F} = 150$。误差巨大。

**原因：** 因为 $H$ 的值非常小 ($0.1$)，在除法运算中，噪声项 $N$ 被放大了 $1/H = 10$ 倍 ($5 \times 10 = 50$)，导致结果严重失真。这对应了课件中提到的 "main challenge" 4。

------

### 9. 频率域低通滤波 (Spectral Low-pass Filter)

考点来源： Slide 8 5

Review 提到 "Low-pass filters"

题目：

给定一个 $M \times N = 6 \times 6$ 的图像，进行频率域滤波。设截止频率 $D_0 = 2$。

对于频率域中的点 $(u, v) = (1, 2)$（假设中心已移至 $(0,0)$），判断该点是否会被理想低通滤波器 (Ideal Lowpass Filter) 通过？

解答：

1. 计算距离 $D(u,v)$：

欧几里得距离公式：$D(u,v) = \sqrt{u^2 + v^2}$

$$D(1, 2) = \sqrt{1^2 + 2^2} = \sqrt{1 + 4} = \sqrt{5} \approx 2.236$$

2. 比较与判断：

理想低通滤波器规则：

$$H(u,v) = \begin{cases} 1, & \text{if } D(u,v) \le D_0 \\ 0, & \text{if } D(u,v) > D_0 \end{cases}$$

因为 $2.236 > 2$ ($D > D_0$)，所以 $H(1,2) = 0$。

结论： 该频率分量被截止（滤除），不通过。

------

### 10. 游程编码 (Run-Length Coding)

考点来源： Slide 11 6

Review 提到 "Coding methods: ... run-length"

题目：

给定一行二值图像数据：0 0 0 0 0 1 1 1 0 0 1。请写出其游程编码（Run-Length Encoding）表示。假设第一段总是从 0 开始（或者由对 (值, 长度) 进行编码）。

解答：

标准 RLE 表示（记录长度）：

1. 连续的 0 有 5 个 $\rightarrow 5$
2. 连续的 1 有 3 个 $\rightarrow 3$
3. 连续的 0 有 2 个 $\rightarrow 2$
4. 连续的 1 有 1 个 $\rightarrow 1$

编码结果： (5, 3, 2, 1)

（注：考试时如果是二值图像，通常只需要记录长度序列，默认起始颜色或交替变化；如果是灰度图，则需记录 (灰度值, 长度) 对，例如 (0, 5), (1, 3), (0, 2), (1, 1)）。
