---
date: '2025-02-22T12:26:32Z'
title: 'Machine Learning in Mass Spectrometry'
thumbnail: '/img/spectus/alchymista_v_knihovne1.jpeg'
author: "Ivana Krenkova"
description: "SpecTUS: a new tool in da house"
tags: ["Ivana Křenková", "CERIT-SC", "SpecTUS", "Research"]
colormode: true
---

# Spotlight on SpecTUS: A Game-Changer in Identifying Chemical Compounds 

In today’s fast-paced world of science and technology, spotting and understanding chemical compounds is incredibly important. Whether it's for creating new medicines, checking environmental safety, or solving crimes, knowing exactly what a chemical compound is can be vital. 

We introduced **SpecTUS: Spectral Translator for Unknown Structures**, a cutting-edge breakthrough tool that aims to revolutionize how we identify compounds using mass spectrometry.

## What is Mass Spectrometry?
Mass spectrometry (MS) is an analytical technique based on breaking up molecules into fragments whose masses are measured (technically, the ratio of mass and charge, m/z, is measured but usually z = 1, making no difference).
The results are presented as a mass spectrum, which displays the relative abundances of the ions on the y-axis and their m/z ratios on the x-axis. 

![Figure1](/img/spectus/ms-eng2.png)

The spectrum is fairly unique for any compound and it can be used to identify it reliably.
Mass spectrometers are usually coupled with a chromatographic column, which separates compounds in the sample from one another, so that the measured mass spectra
are not mixed.

{{< image src="/img/spectus/Recetox-Vysokorozlysujici-GC-HRMS_l.png" ratio="4x3" class="col-12 col-md-6" wrapper="text-center w-40" >}}

### How MS Works: Example with Sugar
Sucrose (ordinary sugar) is something we consume every day, but given a glass of sweet tasting drink, have you ever wondered what it's really made of? Using mass spectrometry, we can uncover its composition.

The MS process is relatively straightforward: 

![Figure3](/img/spectus/sugar.png)

* **Ionization**:  Molecules in the sample are exposed to some stress, e.g. a hi-energy electron beam, which breaks them up into charged fragments (ions).
* **Mass measurement**: Leveraging some laws of physics, e.g. that lighter ions are more deflected by mangetic field than heavy ones, m/z of the ions is measured together with their relative abundance.
* **Data processing**: Depending on the specific method, data acquired in the previous step may not represent the mass spectrum directly; they have to be processed, denoised, deconvoluted etc. 

![Figure5](/img/spectus/sugar-spec2-solo.png)

By comparing the acquired spectra to databases of known compounds, we can identify which compounds were present in the sample, including their concentrations. These databases contain information about the mass spectra of thousands of compounds, including sugars, amino acids, hormones, environmental polutants, and other biologically relevant compounds. 
     

### Limitation in Compound Identification in Mass Spectrometry
Databases of mass spectra, used for identifying and quantifying substances, have expanded significantly over the years:
* 1970: 12 thousand molecules
* 1990: 54 thousand molecules
* 2023: 900 thousand molecules
     
Despite this growth, the number of known small molecules (molecular weight < 500 u) is around **10⁹** (1 billion), while the number of possible molecules of these sizes is approximately **10⁶⁰**. This immense chemical space suggests that given an arbitrary compound in the sample, the probability to find its spectrum in the database is still ridiculously low.


## What is SpecTUS? 

Think of SpecTUS as a smart translator for scientists. It's a powerful tool that uses advanced technology to transform complicated data from mass spectrometry, a way of analyzing compounds, into clear, useful information about molecular structures. Traditionally, scientists have had to rely on databases of known compounds, but SpecTUS can decipher unknown compounds without needing those references.

![Figure6](/img/spectus/translator.png)

SpecTUS is a sophisticated deep learning model designed to translate mass spectra data into molecular structures. Simply put, it takes the information from gas chromatography-mass spectrometry (GC-MS), which analyzes compounds, and turns that data into a readable format for scientists, effectively identifying unknown structures without relying on traditional reference databases. 


## Why Choose SpecTUS?

One of the most exciting things about SpecTUS is that it can handle compounds that aren’t in existing databases. Old methods usually match new data against what’s already known. If there’s no match, they hit a wall. SpecTUS jumps over this challenge by predicting what the compound might be based purely on its spectral data, filling in gaps that others can't. 

When tested, SpecTUS shone brightly. It correctly guessed 43% of compounds on its first try from a massive test set, and when given more chances, it reached a 65% accuracy rate—far better than older methods. 


## How Does SpecTUS Work? 

It uses a combination of machine learning algorithms and large datasets to learn patterns in mass spectra and generate corresponding molecular structures. This approach has the potential to revolutionize the field of mass spectrometry by enabling faster and more accurate analysis of complex substances. 

![Figure6](/img/spectus/spectus-method1.png)

SpecTUS is based on an encoder-decoder transformer model, a type of neural network architecture that operates similarly to language translation tools. First, the model was trained on synthetic data generated by other advanced models like NEIMS and RASSP. This pretraining step gave SpecTUS a broad understanding of potential chemical structures. The next step involved finetuning it with real experimental data, further honing its accuracy. 

![Figure7](/img/spectus/prediction.png)


## Why is This Important? 

SpecTUS has endless potential uses. In drug development, it speeds up the discovery of new treatments. In environmental science, it helps detect pollutants swiftly. In forensics, it can quickly figure out mystery substances. 

Plus, it runs smoothly on different hardware—from high-end GPUs to standard CPUs—making it accessible and easy to use, allowing rapid spectra analysis without complex infrastructure. 


## Benefits of SpecTUS 

SpecTUS has several benefits over traditional methods of mass spectrometry analysis: 

* **Improved accuracy**: SpecTUS can accurately predict molecular structures, even for complex or unknown compounds.
* **Increased speed**: SpecTUS can analyze mass spectra much faster than human experts, making it a valuable tool for high-throughput analysis.
* **Interpretability**: SpecTUS provides detailed information about the predicted molecular structure, including the arrangement of atoms and bonds.
     


## Future Prospects 

The future for SpecTUS is bright. There are opportunities to enhance its capabilities by incorporating higher-quality data and expanding its preliminary datasets. These improvements could boost accuracy and open up more avenues for its use in different scientific areas. 

In summary, SpecTUS is a groundbreaking tool in the realm of spectral analysis, offering a smart and effective way to identify chemical compounds that go beyond what traditional database methods can do. As research and technology advance, tools like SpecTUS will play a crucial role in deepening our understanding of the chemical world. 


     
## References 

For more information about SpecTUS, please refer to the following resources: 

* For quick, zero-install demonstration, SpecTUS inference (with limited throughput) is exposed via REST API at our cluster, accessible with a Binder-ready Jupyter notebook at https://github.com/ljocha/spectus-demo.
* Complete source code of SpecTUS is available on GitHub: https://github.com/hejjack/SpecTUS.
* Pretrained model and synthetic datasets are published at https://huggingface.co/MS-ML.
* [Original research paper](https://arxiv.org/abs/2502.05114)
     


 
 
   
  


