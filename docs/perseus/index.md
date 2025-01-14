# Perseus: Energy Scheduling in Large Model Training

!!! Warning
    Perseus is under active development, and breaking changes may happen.
    Currently, we have all the low-level APIs in place, but it's not a turnkey solution yet.
    This document always reflects the master `HEAD`.

!!! Tip
    Our preprint is out on [arXiv](https://arxiv.org/abs/2312.06902)! :)

## Overview

<img src="img/wide-resnet.gif" width=800px>

Perseus finds the training time--energy Pareto frontier of large model training.
Users can pick any point on the frontier -- be it minimum time, minimum energy, or something in the middle, depending on the training deadline.

Large model training requires the distribution of work to multiple GPUs.
The core observation of Perseus is that work cannot be perfectly split and balanced across every GPU; some GPUs have more work to do and some less.
GPUs with smaller amounts of work finish before GPUs with more amounts of work, but ultimately training throughput is bound by GPUs with the most amount of work.
In other words, GPUs with lighter load are running unnecessarily fast and wasting energy (i.e., there is **energy bloat**).

We reduce energy bloat by controlling the execution speed of each pipeline instruction (forward and backward) in each stage by controlling the GPU's frequency in a fine-grained manner.
We call the assignment of a GPU frequency to each pipeline instruction *frequency plan*, and Perseus gives you **every Pareto-optimal frequency plan** that you can choose any point on the iteration time--energy Pareto frontier.
These plans include frequency plans that do not make training any slower compared to not using Perseus at all, but yield free energy savings.
If you have a bit more leeway as to when training should finish (e.g., You're good as long as training finishes by tomorrow morning), you can pick the frequency plan that slows down training by a couple percentages and save more energy.
Our core algorithm, implemented as a separate library called [`lowtime`](https://github.com/ml-energy/lowtime), **provably guarantees** that for any time deadline, energy consumption is minimal.

## How it's done

Currently, it's a three-step process:

1. **Profile**: Profile the computation time and energy consumption of the forward and backward instructions in *each stage* and *each GPU frequency* and the P2P blocking power consumption of the GPU.
2. **Optimize**: Use [`lowtime`](https://github.com/ml-energy/lowtime) to generate all Pareto-optimal frequency plans.
3. **Choose and start training**: Among all the frequency plans generated by `lowtime`, choose the one that suits your use case.

We have a reference integration with the large model training framework [Merak](https://github.com/ml-energy/merak-zeus), which supports 3D parallelism and automatically tracing and partitioning Hugging Face models.
We've smoothed out some rough edges, integrated Zeus and Perseus, and maintained example scripts for GPT3, BERT, and Wide-ResNet (pretty much any `torchvision` model).

You don't have to be tied to Merak.
If you have your own training framework, and you can integrate Perseus following [our integration guide](integrating.md).

### Profile

In order to run our optimization algorithm, we need the time & energy profiling information of the forward and backward instruction in each stage for every GPU frequency.
The CSV file should look like this for a 4-stage pipeline:

```csv
stage,instruction,frequency,time,energy
0,forward,1740,0.09373254776000976,28.4944
0,forward,1725,0.09390360514322917,28.434366666666666
0,forward,1710,0.09381131331125896,28.288966666666667
...
0,backward,1740,0.24533510557810465,69.5691
0,backward,1725,0.24538559118906658,69.2552
0,backward,1710,0.24548352559407552,68.89453333333334
...
3,backward,690,0.4184921979904175,68.12243333333333
3,backward,675,0.42459266185760497,68.77603333333334
3,backward,660,0.4306272824605306,69.39623333333334
```

Since different frameworks and model implementations will have different performance, it's best to obtain these profiling results on the framework and model you'll be using.
That being said, you can obtain this profiling information in however way you want as long as they have all the columns in the reference CSV file above.
But as a reference, we have implemented an automatic profiler in Merak.
Please refer to the [examples](https://github.com/ml-energy/merak-zeus/tree/main/examples) directory in Merak for profiling instructions.

Finally, we also need to take into account the power consumption of the GPU while it is blocking on P2P communication, i.e., waiting for either the activation or gradient from its neighbor stage.
You can use [our profiling script](https://github.com/ml-energy/zeus/tree/master/examples/perseus/profile_p2p.py) for that.

!!! Tip
    As you profile the time and energy consumption of an instruction, you will scan down from the highest to the lowest frequency.
    However, as you lower the GPU's frequency, both time and energy will start to inflate after some point.
    In other words, those frequencies take more time **and** energy and are simply inefficient (i.e., Pareto-suboptimal), so we won't be running anything with those frequencies.
    Therefore, you actually don't need to profile time and energy for *every* frequency.
    A good heuristic is to scan from higher frequencies to lower ones, and once energy consumption increases more than five *consecutive* times, just stop there.

### Optimize

With the CSV file that holds profiling results, you can use `lowtime` to generate all Pareto-optimal frequency plans.

See [`examples/perseus`](https://github.com/ml-energy/zeus/tree/master/examples/perseus) to find the script `run_optimization.py`.

### Choose and start training

Running `lowtime` optimization will produce a set of frequency assignment files (`freqs_pipeline_%05d.py`).
Each file is also annotated with estimates for time and cost.
The larger the number, the shorter the expected iteration time.

Then, start the Perseus server and plug in the frequency plan you chose:

```console
$ docker exec -it merak-zeus bash
# PERSEUS_SCHEDULER_ARGS='{"solution_path": "path/to/freqs_pipeline_%05d.py"}' uvicorn zeus.optimizer.perseus.server.router:app --port 7787
```

When you run training (with the same `run.sh` but without `--profile true`), the [`PerseusOptimizer`][zeus.optimizer.perseus.optimizer.PerseusOptimizer] integrated into your training framework will automatically talk with the Perseus server to figure out the right GPU frequency to set for the upcoming pipeline instruction and transparently set the GPU's frequency.
