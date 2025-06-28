---
date: '2025-05-26T08:00:32Z'
title: 'Training Neural Networks using MetaCentrum'
thumbnail: '/img/wandb-fr/wandb.png'
description: "Integrated Monitoring via Weights & Biases"
tags: ["Jan Pospíšil, Vladimíra Kimlová, Dominik Zappe", "User Experience", "AI", "FR CESNET", "ZCU"]
colormode: true
draft: false
---

# Training Neural Networks using MetaCentrum with Integrated Monitoring via Weights & Biases

> 💡 **Note** 
> This blog post is brought to you directly by our users as part of our ongoing effort to share knowledge and real-world experience across the community. 
> The content you’re about to read is the result of a project supported through the FR CESNET (Development Fund).
> Final report (in Czech) is available at http://hdl.handle.net/11025/61737.


Training deep learning models is not just about raw compute – it is about observability, reproducibility, and efficiency.
When you run neural network experiments at scale on MetaCentrum, keeping track of performance metrics, hyperparameters, and model versions can quickly get out of hand.
That is where Weights & Biases (WandB) comes in.
In this post, we will show you how to combine MetaCentrum’s HPC resources with WandB’s powerful experiment tracking to streamline your deep learning workflows.

## What is Weights & Biases?

Weights & Biases ([WandB](https://wandb.ai/)) is a platform designed to help machine learning practitioners track their experiments, visualize results, and collaborate more effectively.

It provides tools for logging hyperparameters, metrics, and artefacts, making it easier to reproduce experiments and share results with your team.

## Setting Up WandB

To get started with WandB on MetaCentrum, follow these general steps for integrating WandB into workflows and training scripts:

1. **Install WandB**: First, ensure you have the WandB library installed in your Python environment. You can do this using pip:
```bash
pip install wandb
```
2. **Initialize WandB**: In your training script, initialize WandB with your project name and any relevant configurations:
```python
import wandb

# Simple example of initializing WandB
wandb.init(project="your_project_name", entity="your_wandb_entity")

# More complex example of initialization with hyperparameters and tags
wandb.init(
	name="experiment_name",
	project="your_project_name",
	entity="your_wandb_entity",
	tags=["tag1", "tag2"],
	config={
		"learning_rate": 0.001,
		"batch_size": 32,
		"epochs": 10
	}
)
```
3. **Log Variables**: Use WandB to log metrics during training (and possibly whatever else you want to track, like model weights, gradients, etc.). Here is a simple example of how to log loss and accuracy:
```python
for epoch in range(num_epochs):
	
	# Training code here ...
	
	loss = compute_loss()
	accuracy = compute_accuracy()
	
	wandb.log({"loss": loss, "accuracy": accuracy})
```

## Integrating WandB into MetaCentrum jobs

When running your training scripts on MetaCentrum, you have already integrated the WandB initialization and logging into your training code by following the steps above.
Now you need to ensure that your job scripts are set up to log you into WandB correctly and that the WandB run is properly configured to save logs and possible artefacts.

### Example Job Script
Here is an example of how you might set up a job script to run your WandB-enabled training script on MetaCentrum:

```bash
#!/bin/bash

# Prepare your data, set up the environment, load necessary modules, etc.

# Install WandB and log in
API_KEY="your_wandb_api_key_here"
pip install wandb
python -m wandb login --relogin $API_KEY

# Run your training script
python your_training_script.py
```

## Monitoring and Visualizing Results

Once your job is running, you can monitor the progress of your experiments in real-time on the WandB dashboard.
You can visualize metrics, compare runs, and analyze hyperparameter effects directly in your web browser.

Once your job completes, you can also visualize the results and compare different runs to see how changes in hyperparameters affect model performance, as you can see in the example exported image below:

{{< image src="/img/wandb-fr/wandb_example.png" class="rounded" wrapper="text-center w-40" >}}

## Conclusion

Integrating Weights & Biases with MetaCentrum allows you to leverage the power of both platforms for efficient and effective deep learning workflows.
By tracking your experiments, visualizing results, and collaborating with your team, you can significantly enhance your productivity and the reproducibility of your research.
