# Foundation Model benchmarking tool (FMBench) built using Amazon SageMaker

![Foundation Model Benchmarking Tool](https://github.com/aws-samples/foundation-model-benchmarking-tool/blob/main/img/fmbt-small.png?raw=true)

A key challenge with FMs is the ability to benchmark their performance in terms of inference latency, throughput and cost so as to determine which model running with what combination of the hardware and serving stack provides the best price-performance combination for a given workload.

Stated as **business problem**, the ask is “_*What is the dollar cost per transaction for a given generative AI workload that serves a given number of users while keeping the response time under a target threshold?*_”

But to really answer this question, we need to answer an **engineering question** (an optimization problem, actually) corresponding to this business problem: “*_What is the minimum number of instances N, of most cost optimal instance type T, that are needed to serve a workload W while keeping the average transaction latency under L seconds?_*”

*W: = {R transactions per-minute, average prompt token length P, average generation token length G}*

This foundation model benchmarking tool (a.k.a. `FMBench`) is a tool to answer the above engineering question and thus answer the original business question about how to get the best price performance for a given workload. Here is one of the plots generated by `FMBench` to help answer the above question (_the numbers on the y-axis, transactions per minute and latency have been removed from the image below, you can find them in the actual plot generated on running `FMBench`_).

![business question](https://github.com/aws-samples/foundation-model-benchmarking-tool/blob/main/img/business_summary.png?raw=true)

## Description

The `FMBench` is a Python package for running performance benchmarks for **any model** on **any supported instance type** (`g5`, `p4d`, `Inf2`). `FMBench` deploys models on Amazon SageMaker and use the endpoint to send inference requests to and measure metrics such as inference latency, error rate, transactions per second etc. for different combinations of instance type, inference container and settings such as tensor parallelism etc. Because `FMBench` works for any model therefore it can be used not only testing _third party models_ available on SageMaker, _open-source models_ but also _proprietary models_ trained by enterprises on their own data.

>In a production system you may choose to deploy models outside of SageMaker such as on EC2 or EKS but even in those scenarios the benchmarking results from this tool can be used as a guide for determining the optimal instance type and serving stack (inference containers, configuration).

`FMBench` can be run on any AWS platform where we can run Python, such as Amazon EC2, Amazon SageMaker, or even the AWS CloudShell. It is important to run this tool on an AWS platform so that internet round trip time does not get included in the end-to-end response time latency.

The workflow for `FMBench` is as follows:

```
Create configuration file
        |
        |-----> Deploy model on SageMaker
                    |
                    |-----> Run inference against deployed endpoint(s)
                                     |
                                     |------> Create a benchmarking report
```

1. Create a dataset of different prompt sizes and select one or more such datasets for running the tests.
    1. Currently `FMBench` supports datasets from [LongBench](https://github.com/THUDM/LongBench) and filter out individual items from the dataset based on their size in tokens (for example, prompts less than 500 tokens, between 500 to 1000 tokens and so on and so forth). Alternatively, you can download the folder from [this link](https://huggingface.co/datasets/THUDM/LongBench/resolve/main/data.zip) to load the data.

1. Deploy **any model** that is deployable on SageMaker on **any supported instance type** (`g5`, `p4d`, `Inf2`).
    1. Models could be either available via SageMaker JumpStart (list available [here](https://sagemaker.readthedocs.io/en/stable/doc_utils/pretrainedmodels.html)) as well as models not available via JumpStart but still deployable on SageMaker through the low level boto3 (Python) SDK (Bring Your  Own Script).
    1. Model deployment is completely configurable in terms of the inference container to use, environment variable to set, `setting.properties` file to provide (for inference containers such as DJL that use it) and instance type to use.

1. Benchmark FM performance in terms of inference latency, transactions per minute and dollar cost per transaction for any FM that can be deployed on SageMaker.
    1. Tests are run for each combination of the configured concurrency levels i.e. transactions (inference requests) sent to the endpoint in parallel and dataset. For example, run multiple datasets of say prompt sizes between 3000 to 4000 tokens at concurrency levels of 1, 2, 4, 6, 8 etc. so as to test how many transactions of what token length can the endpoint handle while still maintaining an acceptable level of inference latency.

1. Generate a report that compares and contrasts the performance of the model over different test configurations and stores the reports in an Amazon S3 bucket.
    1. The report is generated in the [Markdown](https://en.wikipedia.org/wiki/Markdown) format and consists of plots, tables and text that highlight the key results and provide an overall recommendation on what is the best combination of instance type and serving stack to use for the model under stack for a dataset of interest.
    1. The report is created as an artifact of reproducible research so that anyone having access to the model, instance type and serving stack can run the code and recreate the same results and report.

1. Multiple [configuration files](https://github.com/aws-samples/foundation-model-benchmarking-tool/tree/main/src/fmbench/configs) that can be used as reference for benchmarking new models and instance types.

## Getting started

`FMBench` is available as a Python package on [PyPi](https://pypi.org/project/fmbench) and is run as a command line tool once it is installed. All data that includes metrics, reports and results are stored in an Amazon S3 bucket.

### Quickstart

1. Launch the AWS CloudFormation template included in this repository using one of the buttons from the table below. The CloudFormation template creates the following resources within your AWS account: Amazon S3 buckets, Amazon IAM role and an Amazon SageMaker Notebook with this repository cloned. A read S3 bucket is created which contains all the files (configuration files, datasets) required to run `FMBench` and a write S3 bucket is created which will hold the metrics and reports generated by `FMBench`. The CloudFormation stack takes about 5-minutes to create.

   |AWS Region                |     Link        |
   |:------------------------:|:-----------:|
   |us-east-1 (N. Virginia)    | [<img src="./img/ML-FMBT-cloudformation-launch-stack.png">](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=fmbench&templateURL=https://aws-blogs-artifacts-public.s3.amazonaws.com/artifacts/ML-FMBT/template.yml) |

1. Once the CloudFormation stack is created, navigate to SageMaker Notebooks and open the `fmbench-notebook`.

1. On the `fmbench-notebook` open a Terminal and run the following commands.

    ```{.bash}
    conda create --name fmbench_python311 -y python=3.11 ipykernel
    source activate fmbench_python311;
    pip install fmbench
    ```

1. Now you are ready to `fmbench` with the following command line. This would take about ~30 minutes to complete and will produce a report comparing the performance of Llama2-13b on different inference stacks.

    ```{.bash}
    account=`aws sts get-caller-identity | jq .Account | tr -d '"'`
    fmbench --config-file s3://sagemaker-fmbench-read-${account}/configs/config-llama2-13b-inf2-g5-p4d-quick.yml
    ```

1. The generated reports and metrics are available in the `sagemaker-fmbench-write-<replace_w_your_aws_account_id>` bucket.

### The DIY version (with gory details)

Follow the prerequisites below to set up your environment before running the code:

1. **Python 3.11**: Setup a Python 3.11 virtual environment and install `FMBench`.

    ```{.bash}
    python -m venv .fmbench
    pip install fmbench
    ```

1. **S3 buckets for test data, scripts, and results**: Create two buckets within your AWS account:

    * _Read bucket_: This bucket contains `tokenizer files`, `prompt template`, `source data` and `deployment scripts` stored in a directory structure as shown below. `FMBench` needs to have read access to this bucket.
    
        ```{.bash}
        s3://<read-bucket-name>
            ├── source_data/
            ├── source_data/<source-data-file-name>.json
            ├── prompt_template/
            ├── prompt_template/prompt_template.txt
            ├── scripts/
            ├── scripts/<deployment-script-name>.py
            ├── tokenizer/
            ├── tokenizer/tokenizer.json
            ├── tokenizer/config.json
        ```

        * The details of the bucket structure is as follows:

            1. **Source Data Directory**: Create a `source_data` directory that stores the dataset you want to benchmark with. `FMBench` uses `Q&A` datasets from the [`LongBench dataset`](https://github.com/THUDM/LongBench) or alternatively from [this link](https://huggingface.co/datasets/THUDM/LongBench/resolve/main/data.zip). _Support for bring your own dataset will be added soon_.

                * Download the different files specified in the [LongBench dataset](https://github.com/THUDM/LongBench) into the `source_data` directory. Following is a good list to get started with:

                    * `2wikimqa`
                    * `hotpotqa`
                    * `narrativeqa`
                    * `triviaqa`
                
                    Store these files in the `source_data` directory.

            1. **Prompt Template Directory**: Create a `prompt_template` directory that contains a `prompt_template.txt` file. This `.txt` file contains the prompt template that your specific model supports. `FMBench` already supports the [prompt template](src/fmbench/prompt_template/prompt_template.txt) compatible with `Llama` models.

            1. **Scripts Directory**: `FMBench` also supports a `bring your own script (BYOS)` mode for deploying models that are not natively available via SageMaker JumpStart i.e. anything not included in [this](https://sagemaker.readthedocs.io/en/stable/doc_utils/pretrainedmodels.html) list. Here are the steps to use BYOS.

                1. Create a Python script to deploy your model on a SageMaker endpoint. This script needs to have a `deploy` function that [`2_deploy_model.ipynb`](./src/fmbench/2_deploy_model.ipynb) can invoke. See [`p4d_hf_tgi.py`](./scripts/p4d_hf_tgi.py) for reference.

                1. Place your deployment script in the `scripts` directory in your ***read bucket***. If your script deploys a model directly from HuggingFace and needs to have access to a HuggingFace auth token, then create a file called `hf_token.txt` and put the auth token in that file. The [`.gitignore`](.gitgnore) file in this repo has rules to not commit the `hf_token.txt` to the repo. Today, `FMBench` provides inference scripts for:

                    * [All SageMaker Jumpstart Models](https://docs.aws.amazon.com/sagemaker/latest/dg/jumpstart-foundation-models.html)
                    * [Text-Generation-Inference (TGI) container supported models](https://huggingface.co/text-generation-inference)
                    * [Deep Java Library DeepSpeed container supported models](https://docs.djl.ai/docs/serving/serving/docs/lmi/configurations_large_model_inference_containers.html)


                    Deployment scripts for the options above are available in the [scripts](https://github.com/aws-samples/foundation-model-benchmarking-tool/tree/s3_metrics/scripts) directory, you can use these as reference for creating your own deployment scripts as well.

            1. **Tokenizer Directory**: Place the `tokenizer.json`, `config.json` and any other files required for your model's tokenizer in the `tokenizer` directory. The tokenizer for your model should be compatible with the [`tokenizers`](https://pypi.org/project/tokenizers/) package. `FMBench` uses `AutoTokenizer.from_pretrained` to load the tokenizer.
                >As an example, to use the `Llama 2 Tokenizer` for counting prompt and generation tokens for the `Llama 2` family of models: Accept the License here: [meta approval form](https://ai.meta.com/resources/models-and-libraries/llama-downloads/) and download the `tokenizer.json` and `config.json` files from [Hugging Face website](https://huggingface.co/meta-llama/Llama-2-7b/tree/main) and place them in the `tokenizer` directory.

    * _Write bucket_: All prompt payloads, model endpoint and metrics generated by `FMBench` are stored in this bucket. `FMBench` requires write permissions to store the results in this bucket. No directory structure needs to be pre-created in this bucket, everything is created by `FMBench` at runtime.

        ```{.bash}
        s3://<write-bucket-name>
            ├── <test-name>
            ├── <test-name>/data
            ├── <test-name>/data/metrics
            ├── <test-name>/data/models
            ├── <test-name>/data/prompts

### Steps to run

1. `pip install` the `FMBench` package from PyPi.

1. Create a config file using one of the config files available [here](https://github.com/aws-samples/foundation-model-benchmarking-tool/tree/main/src/fmbench/configs).
    1. The configuration file is a YAML file containing configuration for all steps of the benchmarking process. It is recommended to create a copy of an existing config file and tweak it as necessary to create a new one for your experiment.

1. Create the read and write buckets as mentioned in the prerequisites section. Mention the respective directories for your read and write buckets within the config files.

1. Run the `FMBench` tool from the command line.

    ```{.bash}
    # the config file path could be an S3 path and https path 
    # or even a path to a file on the local filesystem
    fmbench --config-file \path\to\config\file
    ```

1. Depending upon the experiments in the config file, the `FMBench` run may take a few minutes to several hours. Once the run completes, you can find the report and metrics in the write S3 bucket set in the [config file](https://github.com/aws-samples/foundation-model-benchmarking-tool/blob/main/src/fmbench/configs/config-mistral-7b-tgi-g5.yml#L12). The report is generated as a markdown file called `report.md` and is available in the metrics directory in the write S3 bucket.

## Results

Here is a screenshot of the `report.md` file generated by `FMBench`.
![Report](https://github.com/aws-samples/foundation-model-benchmarking-tool/blob/main/img/results.gif?raw=true)


## Building the `FMBench` Python package

The following steps describe how to build the `FMBench` Python package.

1. Clone the `FMBench` repo from GitHub.

1. Make any code changes as needed.

1. Install [`poetry`](https://pypi.org/project/poetry/).
   
    ```{.bash}
    pip install poetry
    ```

1. Change directory to the `FMBench` repo directory and run poetry build.

    ```{.bash}
    poetry build
    ```

1. The `.whl` file is generated in the `dist` folder. Install the `.whl` in your current Python environment.

    ```{.bash}
    pip install .\dist\fmbench-X.Y.Z.tar.gz
    ```

1. Run `FMBench` as usual through the `FMBench` CLI command.

## Pending enhancements

The following enhancements are identified as next steps for `FMBench`.

1. Containerize `FMBench` and provide instructions for running the container on EC2.

1. Support for different payload formats that might be needed for different inference containers. Currently the HF TGI container, and DJL Deep Speed container on SageMaker both use the same format but in future other containers might need a different payload format.

1. Emit live metrics so that they can be monitored through Grafana via live dashboard.

1. Allow users to publish their experiment configs and results by doing a POST to an AWS Lambda that writes results to a common S3 bucket that can serve as storage for a simple website.

1. Create a leaderboard of model benchmarks.

View the [ISSUES](https://github.com/aws-samples/foundation-model-benchmarking-tool/issues) on github and add any you might think be an beneficial iteration to this benchmarking harness.

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the [LICENSE](./LICENSE) file.
