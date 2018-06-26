# Overview

This branch contains a number of work-in-progress features for SageMaker TensorFlow,
including:

* MKL optimized in CPU instances (MKL is available in the C5 instance types)
* Pipe mode support (for TF 1.7)
* Script mode (run a TensorFlow script as an ordinary Python script rather than implementing various functions)
* SSL / HTTPS toggle when downloading data from S3.
* Python 2 only
* CPU only
* Available regions: us-west-2, us-east-1, and us-east-2

## How do I run these containers?

Using the Python SDK, running this code from the `test/integration/benchmarks/criteo` directory will
run a training job with 3 ml.c5.9xlarge instances with 2 parameter servers.

```python
from sagemaker.tensorflow import TensorFlow

# Change this to your criteo small clicks or large clicks datasets:
# https://github.com/GoogleCloudPlatform/cloudml-samples/tree/master/criteo_tft#criteo-dataset
CRITEO_DATASET = 's3://my/bucket'

hyperparameters = {
    # sets the number of parameter servers in the cluster.
    'sagemaker_num_parameter_servers': 2,
    's3_channel':                      CRITEO_DATASET,
    'batch_size':                      30000,
    'dataset':                         'kaggle',
    'model_type':                      'linear',
    'l2_regularization':               100,

    # see https://www.tensorflow.org/performance/performance_guide#optimizing_for_cpu
    # Best value for this model is 10, default value in the container is 0.
    # 0 sets the value to the number of logical cores.
    'inter_op_parallelism_threads':    10,

    # environment variables that will be written to the container before training starts
    'sagemaker_env_vars':              {
        # True uses HTTPS, uses HTTP otherwise. Default false
	# see https://docs.aws.amazon.com/sdk-for-cpp/v1/developer-guide/client-config.html
        'S3_USE_HTTPS':  True,
        # True verifies SSL. Default false
        'S3_VERIFY_SSL': True,
        # Sets the time, in milliseconds, that a thread should wait, after completing the
        # execution of a parallel region, before sleeping. Default 0
        # see https://github.com/tensorflow/tensorflow/blob/faff6f2a60a01dba57cf3a3ab832279dbe174798/tensorflow/docs_src/performance/performance_guide.md#tuning-mkl-for-the-best-performance
        'KMP_BLOCKTIME': 25
    }
}

tf = TensorFlow(entry_point='task.py',
                source_dir='trainer',
                train_instance_count=3,
                train_instance_type='ml.c5.9xlarge',
                # pass in your own SageMaker role
                role='MySageMakerRole',
                hyperparameters=hyperparameters)

# This points to the prototype images.
# Change the region (to us-west-2 or us-east-2) or TF version (to 1.7.0) if needed
tf.train_image = lambda: '520713654638.dkr.ecr.us-east-1.amazonaws.com/sagemaker-tensorflow:1.6.0-cpu-py2-script-mode-preview'

# publicly accessible placeholder data. Change the region if needed
tf.fit({'training': 's3://sagemaker-sample-data-us-east-1/spark/mnist/train'})

```
## Available images
- 520713654638.dkr.ecr.us-east-1.amazonaws.com/sagemaker-tensorflow:1.6.0-cpu-py2-script-mode-preview
- 520713654638.dkr.ecr.us-east-2.amazonaws.com/sagemaker-tensorflow:1.6.0-cpu-py2-script-mode-preview
- 520713654638.dkr.ecr.us-west-2.amazonaws.com/sagemaker-tensorflow:1.6.0-cpu-py2-script-mode-preview
- 520713654638.dkr.ecr.us-east-1.amazonaws.com/sagemaker-tensorflow:1.7.0-cpu-py2-script-mode-preview
- 520713654638.dkr.ecr.us-east-2.amazonaws.com/sagemaker-tensorflow:1.7.0-cpu-py2-script-mode-preview
- 520713654638.dkr.ecr.us-west-2.amazonaws.com/sagemaker-tensorflow:1.7.0-cpu-py2-script-mode-preview

## Developer documentation (WIP)

### Directories

- docker/```tf version number```.vanilla/ - Dockerfile with vanilla TF version
- docker/```tf version number``` - Dockerfile with optimized binaries
- src/sagemaker_tensorflow_container/training.py - Training logic in the container


ensure you have the TF binaries in the docker folder:

docker/1.6.0/final/py2/ - tensorflow-1.6.0-cp27-cp27mu-linux_x86_64.whl
docker/1.7.0/final/py2/ - tensorflow-1.7.0-cp27-cp27mu-linux_x86_64.whl
docker/1.8.0/final/py2/ - tensorflow-1.8.0-cp27-cp27mu-linux_x86_64.whl

### Building the Images:

test_build has a parameter version that you can change to build 1.6.0, 1.6.0.vanilla, 1.7.0, 1.7.0.vanilla

in the end, it will push the image to your ecr account, **you have to create the ECR repository**.
```bash
pip install -U .
pip install .[test]
pytest test/test_builder.py
```

### Using Pipe Mode

The 1.7.0 and 1.8.0 images include pipe mode installed, example of usage:

```python
features = {
    'data': tf.FixedLenFeature([], tf.string),
    'labels': tf.FixedLenFeature([], tf.int64),
}

def parse(record):
    parsed = tf.parse_single_example(record, features)
    return ({
        'data': tf.decode_raw(parsed['data'], tf.float64)
    }, parsed['labels'])

ds = PipeModeDataset(channel='training', record_format='TFRecord')
num_epochs = 20
ds = ds.repeat(num_epochs)
ds = ds.prefetch(10)
ds = ds.map(parse, num_parallel_calls=10)
ds = ds.batch(64)
```

More information about pipe mode https://github.com/aws/sagemaker-tensorflow-extensions

### Running benchmarks 

### Only required paramaters
```bash 
pytest test/integration/benchmarks/criteo/test_criteo.py \
 --sagemaker-role MySageMakerRole `# pass in your own SageMaker role` \
 --dataset-location  s3://my/bucket/with/criteo/dataset
```

#### All available settings
```bash
pytest test/integration/benchmarks/criteo/test_criteo.py \
 --sagemaker-role MySageMakerRole `# pass in your own SageMaker role` \
 --dataset-location  s3://my/bucket/with/criteo/dataset \
 --instance-type local `# available types: local mode or any SageMaker CPU instance ml.c5.9xlarge for example` \
 --dataset 'large' `# choices: kaggle, large` \
 --region 'us-west-2' `# regions: us-west-2, us-east-1, us-east-2` \
 --framework-version 1.7.0 `# available versions: 1.6.0, 1.7.0, 1.6.0.vanilla, 1.7.0.vanilla` \
--num-parameter-servers 2 `# number of parameter servers in the training` \
--num-instances 3 `# number of training instances` \
--kmp-blocktime 0 `# MKL optimization setting` \
--inter-op-parallelism-threads 5 `# MKL optimization setting` \
--s3-use-https `# use https, does not use otherwise` \
--s3-verify-ssl `# verify s3 ssl, does not verify otherwise` \
--wait `# wait container to finish training, does not wait otherwise`
```

### Runing benchmark comparisons using Tensorboard
```bash
CHECKPOINT_PATH=s3://bucket/that/sagemaker/role/has/access
for PARAMETER_SERVER in 10 5 4 2 1
do
    pytest test/integration/benchmarks/criteo/test_criteo.py \
     --sagemaker-role SageMakerRole `# pass in your own SageMaker role` \
     --dataset-location  s3://my/bucket/with/criteo/dataset \
     --framework-version 1.7.0 `# available versions: 1.7.0, 1.8.0, 1.7.0.vanilla, 1.8.0.vanilla` \
    --num-parameter-servers ${PARAMETER_SERVER} `# number of parameter servers in the training` \
    --num-instances 10 `# number of training instances`
done

AWS_REGION='us-west-2' tensorboard --port 6006 --host localhost --logdir ${CHECKPOINT_PATH}
```
## FAQ

### What is script mode?

Script mode is a newer, simpler, and cleaner way to execute TensorFlow scripts in SageMaker. The objective is to allow our customers to execute any TensorFlow model available with minimal changes.
 
We released the Chainer container a couple of days ago, and it includes Script mode. Please read the documentation here:https://github.com/aws/sagemaker-containers#getting-started----executing-user-scripts-on-amazon-sagemaker
 
The way to execute the scripts and the environment variables that you have access when you execute script mode will not change when we released TensorFlow Script mode. It will be like Chainer.
 
The hyperparameters  that we created especially for you (num_parameter_servers, s3_channel, sagemaker_env_vars) will change to fit a better user experience. We would like to include your feedback for these changes as well.


https://github.com/aws/sagemaker-containers#how-a-script-is-executed-inside-the-container explains how the user script is executed and how the hyperparameters are passed as script arguments.

### I see the message *tensorflow/core/platform/s3/aws_logging.cc::54 Connection has been released. Continuing.* multiple times in the training logs. What is this message?

The message

```bash
tensorflow/core/platform/s3/aws_logging.cc::54 Connection has been released. Continuing.
```

happens multiple times in the logs because the [TensorFlow S3 filesytem implementation in TensorFlow](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/docs_src/deploy/s3.md) requests ECS credentials everytime that is writing/reading that to S3.

The message comes from [here](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/core/platform/s3/aws_logging.cc?utf8=%E2%9C%93#L54) and it has cpp log level **INFO**:

```c++
...
LOG(INFO) << message;
...
```

You can set the environment variable ```TF_CPP_MIN_VLOG_LEVEL=1``` to not log INFO level cpp messages.

The available [log levels](https://github.com/tensorflow/tensorflow/blob/352142267a1a151b04c6198de83b40b7e979d1d8/tensorflow/core/platform/default/logging.h#L31) are:

```c++
namespace tensorflow {
const int INFO = 0;            // base_logging::INFO;
const int WARNING = 1;         // base_logging::WARNING;
const int ERROR = 2;           // base_logging::ERROR;
const int FATAL = 3;           // base_logging::FATAL;
const int NUM_SEVERITIES = 4;  // base_logging::NUM_SEVERITIES;
```

Instances of the S3 file system in TensorFlow are created everytime that [TensorFlow interacts with S3, and new credentials are requested everytime](https://github.com/tensorflow/tensorflow/blob/r1.6/tensorflow/core/platform/s3/s3_file_system.cc#L315). It is necessary to cache the credentials to reduce the volume of requests.
