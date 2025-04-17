## Lec1 neuro network
Perceptron: Forward Propagation

Neruo-like
- One output
- Multi output

Training for the weights
- learning rates
- design adaptive learning rate that adapts to the landscape

## Lec2 Deep Sequence Modeling
give us the foundation of the Deep Sequence Modeling -> LLM

Sequences in the Word

### Neurons with Recurrence
Depend not only the current state, but also the prior results.

$$y_t = f(x_t, h_{t-1})$$

Recurrent Neural Networks (RNNs), apply a recurrence relation at every time step to process a sequence.

$$h_t = f_W (x_t, h_{t-1})$$

Terminology:
- $h_t$: cell state
- $f_W$: function with weights W
- $x_t$: input
- $h_{t-1}$: old state

To model sequences, we need to:
- variable length input sequences
- long term dependencies
- order is important
- share parameters across the sequence

### A sequence modeling problem: predict the next word
To build a LM to predict the next word, we should first make the 'plain text' into vector form, which can be well understood by the LM.
- Find corpus of words
- Indexing the words
- Map index to fixed-sized vector (Learned embedding)

The position is very crucial

Backpropagation algorithm:
- Take the derivative of the loss with respect to each parameter
- Shift parameters in order to minimize loss

Backpropagation Through the Time: from the end to the start
- problem: the long-term dependencies are decaying.

Ways to mitigate:
- LSTM: Long Short Term Memory (LSTMs)
- rely on a gated cell to track information throughout many time steps

Limitations of RNNs
- Fixed sized 
- time step by step generation => very slow, and no parallelization
- Not long Memory

How to parallelize?
- Idea 1: Feed everything into dense network, but not scalable and unsupport long memory, no order
- Idea 2: Identify and attend to what's important

### Attention is All You Need 
Attending to the most important parts of an input 
- Identify which parts to attend to -> Similar to search
- Extract the features with high attention  

Basic Idea:
- *Search* with a "Query" into a Big DataBase
- *Extract* the *Keys* of the info 
- *Evaluate* the *Similarity* between *Query* and *Key*, and *Identify* the *Key*
- *Extract* the *Value* based on attention => Return the values highest attention 

Steps:
- Encode position information 
- Extract query, key, value for search (Positional Matrix Multiply the Q/K/V Layer to form the Q/K/V matrix)
- Compute attention weighting => Generate a Matrix => $softmax(\frac{Q \cdot K ^{T}}{scaling}) \cdot V = A(Q, K, V)$ 
- Extract Features with high attention 

natural way to attend 
### Summarize
- Very Intuitive one with great story-telling skills.
- But I like it.