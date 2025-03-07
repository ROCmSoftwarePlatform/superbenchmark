---
id: data-diagnosis
---

# Data Diagnosis

## Introduction

This tool is to filter the defective machines automatically from thousands of benchmarking results according to rules defined in **rule file**.

## Usage

1. [Install SuperBench](../getting-started/installation) on the local machine.

2. Prepare the raw data, rule file, baseline file under current path or somewhere on the local machine.

3. After installing the Superbnech and the files are ready, you can start to filter the defective machines automatically using  `sb result diagnosis` command. The detailed command can be found from [SuperBench CLI](../cli).

  ```
  sb result diagnosis --data-file ./results-summary.jsonl --rule-file ./rule.yaml --baseline-file ./baseline.json --output-file-format excel --output-dir ${output-dir}
  ```

4. After the command finished, you can find the output result file named 'diagnosis_summary.xlsx' / 'diagnosis_summary.json' under ${output_dir}.

## Input

The input mainly includes 3 files:

 - **raw data**: jsonl file including multiple nodes' results automatically generated by SuperBench runner.

    `Tips`: this file can be found at ${output-dir}/results-summary.jsonl after each successful run.

 - **rule file**: It uses YAML format and includes each metrics' rules to filter defective machines for diagnosis.

 - **baseline file**: json file including the baseline values for the metrics.

    `Tips`: this file for some representative machine types will be published in [SuperBench Results Repo](https://github.com/microsoft/superbench-results/tree/main) with the release of Superbench.

### rule file

This section describes how to write rules in **rule file**.

The convention is the same with [SuperBench Config File](https://microsoft.github.io/superbenchmark/docs/superbench-config), please view it first.

Here is an overview of the rule file structure:

scheme:
```yaml
version: string
superbench:
  var:
    ${var_name}: dict
  rules:
    ${rule_name}:
      function: string
      criteria: string
      categories: string
      metrics:
        - ${benchmark_name}/regex
        - ${benchmark_name}/regex
        ...
```

example:
```yaml
# SuperBench rules
version: v0.4
superbench:
  rules:
    failure-rule:
      function: value
      criteria: lambda x:x>0
      categories: Failed
      metrics:
        - kernel-launch/return_code
        - mem-bw/return_code
        - nccl-bw/return_code
        - ib-loopback/return_code
    rule0:
    # Rule 0: If KernelLaunch suffers > 5% downgrade, label it as defective
      function: variance
      criteria: lambda x:x>0.05
      categories: KernelLaunch
      metrics:
        - kernel-launch/event_overhead:\d+
        - kernel-launch/wall_overhead:\d+
    rule1:
    # Rule 1: If H2D_Mem_BW or D2H_Mem_BW test suffers > 5% downgrade, label it as defective
      function: variance
      criteria: lambda x:x<-0.05
      categories: Mem
      metrics:
        - mem-bw/H2D_Mem_BW:\d+
        - mem-bw/D2H_Mem_BW:\d+
    rule2:
    # Rule 2: If NCCL_BW suffers > 5% downgrade, label it as defective
      function: variance
      criteria: lambda x:x<-0.05
      categories: NCCL
      metircs:
        - nccl-bw/allreduce_8589934592_busbw:0
    rule3:
    # Rule 3: If GPT-2, BERT suffers > 5% downgrade, label it as defective
      function: variance
      criteria: lambda x:x<-0.05
      categories: Model
      metrics:
        - bert_models/pytorch-bert-base/throughput_train_float(32|16)
        - bert_models/pytorch-bert-large/throughput_train_float(32|16)
        - gpt_models/pytorch-gpt-large/throughput_train_float(32|16)
```

This rule file describes the rules used for data diagnosis.

They are firstly organized by the rule name, and each rule mainly includes 4 elements:

#### `metrics`

The list of metrics for this rule. Each metric is in the format of ${benchmark_name}/regex, you can use regex after the first '/', but to be noticed, the benchmark name can not be a regex.

#### `categories`

The categories belong to this rule.

#### `criteria`

The criteria used for this rule, which indicate how to compare the data with the baseline value. The format should be a lambda function supported by Python.

#### `function`

The function used for this rule.

2 types of rules are supported currently:

- `variance`: the rule is to check if the variance between raw data and baseline violates the criteria. variance = (raw data - criteria) / criteria

  For example, if the criteria are `lambda x:x>0.05`, the rule is that if the variance is larger than 5%, it should be defective.

- `value`: the rule is to check if the raw data violate the criteria.

  For example, if the criteria are `lambda x:x>0`, the rule is that if the raw data is larger than the 0, it should be defective.

`Tips`: you must contain a default rule for ${benchmark_name}/return_code as the above in the example, which is used to identify failed tests.

## Output

We support different output formats for filtering the defective machines including jsonl, excel, etc. The output includes all defective machines' information including index, failure category, failure details, and detailed metrics.

- index: the name of defective machines.

- Category: categories defined in the rule.

- Defective Details: all violated metrics including metric data and related rule.

- ${metric}: the data of the metrics defined in the rule file. If the rule is `variance`, the form of the data is variance in percentage; if the rule is `value`, the form of the data is raw data.
