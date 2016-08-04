# bypass
Bypass model implemented with Tensorflow's rnn method is in bypass_rnn.py . It relies on RNNCells defined in ConvRNN.py

## Training a new model
Specify the desired model architecture, training parameters, and file paths in `bypassrnn_params.py`. 
  
Parameters to double-check before each new run:
  - **Model inputs**
    - Change the `DATA_PATH` based on whether you are running on openmind or on the agents
    - `IMAGE_SIZE`: is the cropped image size. So you would use `IMAGE_SIZE = 224` for VGG models but `IMAGE_SIZE = 256` if you do not want any cropping. 
    - `BATCH_SIZE`: should be adjusted based on available GPU memory and size of the model
  - **Running specifications**
    - `NUM_GPUS = 1`: Our code is not yet ready for multi-GPU nor have we seen any substantial gain in using multi-GPUs so far.
  
  - **Saving and restoring variables, writing loss to file**
    - `CHECKPOINT_DIR`: the directory where your variables, checkpoint file, and loss files will be saved. Thus it is also the directory read by `bypassrnn_eval.py` to periodically restore variables from the last checkpoint and run the validation set on the model. Set `SAVE_LOSS` and `SAVE_VARS` to True to output the training loss and save variable (aka model parameters) every `SAVE_VARS_FREQ` iterations. `MAX_TO_KEEP` specifies how many variable files to keep. 
    - `SAVE_FILE`: specify the name of the file which the variables and loss outputs are written to. For example, 
    ```
    SAVE_FILE = './trial'
    ``` 
    will result in the training loss file saved as `./trial_loss.csv` and variables saved in the format `./trial-12000`, where 12000 is the iteration number. 
    - Restoring variables: If you are training a new model, you should set `RESTORE_VARS = False`. Set it to `True` when you wish to restore your model parameters from a variable file, for example, when you wish to continue a run. To restore, specify the starting iteration in `START_STEP` as well as the path to your variable file in `RESTORE_VAR_FILE`. 
    
  - **Training and optimization parameters**
    - `GRAD_CLIP`: gives the option to apply gradient clipping, to a norm of ±1, as specified in Pascanu, et. al. (2013) http://www.jmlr.org/proceedings/papers/v28/pascanu13.pdf. 
    - `KEEP_PROB`: DEPRECATED
    - Under `# Optimization parameters` you can specify the base learning rate, exponential decay factor for the learning rate, the step size for the exponential decay (in `NUM_EPOCHS_PER_DECAY`), and the momentum parameter, if a momentum optimizer is used.
    - `TIME_PENALTY`: refers to the term in the loss function equation and can be set to any value.
    
  - **Model structure**
  
    You will need to know the sizes of each of the layer activations in your model, rather than letting it be determined by the stride of your conv and pool operations. This is because pooling is implemented by looking at the ratio of spatial dimensions from the expected output and from the input to the pooling, and determining the stride from that value. 

    - `T`: The total number of time steps to train or run the model. This can be any `T` > shortest path through graph. For example, if there is a graph with 6 layers plus a fully connected (FC)-softmax layer, then it would take `T = 8` time steps to have an input image reach the softmax output. If there is a bypass from layer 1 -> 4, the shortest path from the input to the output layer would take `T = 5` steps (input -> 1 -> 4 -> 5 -> 6 -> FC/softmax). Then for this model, `T` can be any value > 5. 
    - `LAYER_SIZES`: Specify the state and output sizes for each layer. Layer 0 is for the input image. `state` refers to the output of the convolution or matrix multiplication in FC layers. The `output` size is equal to the `state` size if there is no pooling in that layer. Otherwise, the `output` size should be the size of the activation after pooling. 
    
      For example, if Layer 1 consists of convolution of stride 1, then pooling of stride 2, and Layer 2 is only convolutional with stride 1, then we would have: 
      ```
      1: {'state': [BATCH_SIZE, IMAGE_SIZE / 1, IMAGE_SIZE / 1, 128], 'output': [BATCH_SIZE, IMAGE_SIZE / 2, IMAGE_SIZE / 2, 128]}, # convpool
      2: {'state': [BATCH_SIZE, IMAGE_SIZE / 2, IMAGE_SIZE / 2, 256], 'output': [BATCH_SIZE, IMAGE_SIZE / 2, IMAGE_SIZE / 2, 256]},# conv
      ```
      The sizes, or equivalently, shapes, are in the form: [batch size, spatial, spatial, depth aka num_channels]
    - `LAYERS`: Specify the type of cell for each layer based on which operations should be carried out. These cells are defined in `ConvRNN.py`. `LAYERS` is a dictionary where `LAYERS[i]` is a list, with first entry the class of the cell for layer i, and the second entry a dictionary of parameters for creating that cell. Many of these parameters, if not specified, have default values. These default values can be specified separately in `ConvRNN.py` [TODO: Maybe let those default values in `ConvRNN.py` be carried over from this parameters file.]
      For example, for a ConvPoolRNNCell, that consists of convolution then pooling as the name implies, we can specify the convolutional kernel size (`conv_size`), the convolutional stride (`conv_stride`), the weight initialization (`weight_init`: either `'xavier'`, which is also the default, or `trunc_norm` for a truncated normal distribution), and the bias initialization (`bias_init`).  We can also specify the kernel size for pooling (`pool_size`) where the default sets `pool_size` equal to the stride of pooling, so a pool of stride 2 would have a default kernel size of stride 2. If we specify `'memory': True`, then we can initialize the initial decay parameter with `decay_param_init`. [Note that the actual decay *factor* (proportion of previous state carried over) is the *sigmoid* of the decay *parameter*.] There is also the option of using weight decay ('weight_decay'), as specified in the AlexNet implementation (Krizhevsky et. al., 2012). For more detail, see `ConvRNN.py`
      
      For example:
      ```
      LAYERS = {1: [ConvPoolRNNCell, {'state_size': LAYER_SIZES[1]['state'], 'output_size': LAYER_SIZES[1]['output']
                         'conv_size': 3,  # kernel size for conv
                        'conv_stride': 1,  # stride for conv
                        'weight_decay': WEIGHT_DECAY, # None for none
                        # 'weight_init': use default (currently 'xavier')
                        # 'weight_stddev': only relevant if you use 'trunc_norm' initialization
                        # 'bias_init': use default (currently 0.1),
                        'pool_size': 2,  # kernel size for pool (defaults to = stride determined by layer sizes.)
                        # 'decay_param_init': relevant if you have memory. 
                            # Example: decay_param_init = 0 => decay factor initialized to sigmoid(0) = 0.5
                        'memory': False}] # defaults to True (then uses decay_param_init)
                        
                ... }
      ```
      For FcRNNCells, which are fully connected layers, you can specify the dropout `'keep_prob'`. Currently, you would have to change the `keep_prob` value manually when switching between models for evaluation and for training, but this can be streamlined in the future. [TODO].
      
      For example:
      ``` 
      LAYERS = {... 
                6: [FcRNNCell, {'state_size': LAYER_SIZES[6]['state'],
                                'keep_prob': FC_KEEP_PROB, # TODO change when evaluating
                                'memory': False}]
                }
      ```
  - `BYPASSES`: Specify bypass connections to include in the model structure. Any valid targets of the bypass connections should be Layer 2 through the first FC layer. That is, if you wish to bypass to Layer n, that layer should expect to have 4-D inputs rather than 2-D (such as the activation outputs of an FC layer). Just a comment: Adding bypasses to an FC layer can potentially increase the number of parameters of the model by several million. It may be better to add bypasses that target Convolutional layers only. 
    
    For example, if there are bypasses from layer *i1* to layer *j1* and layer *i2* to layer *j2*, one would write:
  ``` 
  BYPASSES = [(i1, j1), (i2, j2)] # bypasses: list of tuples (from, to)
  ```
  [TODO- easy fix] Currently, the model is not robust to having a redudant list of bypasses connections, so be careful in your specifications. Also, DO NOT specify any connections between adjacent layers, for example (1,2), as these are by default included. Do not attempt to create any feedback connections either, unless you want to write your own code.
- Other model parameters: Several parameters used in the creation of layers may include `WEIGHT_DECAY` and `FC_KEEP_PROB` (set either to `None` if not desired). 
  
  
