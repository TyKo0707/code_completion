# Code completion

The project's focus is on code completion. The pipeline separates code samples into three sections: prefix (before the cursor), middle (missing code), and suffix (after the cursor). 
The middle section's completions are generated using pre-trained open-source models: `tiny_starcoder_py`, `starcoder2_3b`, `starcoder2_7b`, and `starcoder2_15b`.

After creating completions, the outputs are examined both manually and automatically using the target code.

The goal is to establish which metrics correspond best with human evaluation and to assess the quality of code completion across various models.

## Steps of solution
### 0. Exploring the models:
* Explore different models ([explore_tiny_starcoder.ipynb](https://github.com/TyKo0707/code_completion/blob/main/models_explore/explore_tiny_starcoder.ipynb), [explore_big_starcoder.ipynb](https://github.com/TyKo0707/code_completion/blob/main/models_explore/explore_big_starcoder.ipynb)) specified for code generation to find the most suitable for the task.
* Find the optimal config and method of generation for each model.
* Result of this step: decided to use starcoder and starcoder-based models `tiny_starcoder_py`, `starcoder2_3b`, `starcoder2_7b`, and `starcoder2_15b`.


### 1. Data Collection and Preprocessing:
I need to create a dataset consists of prefix, middle part (target), and suffix.
* Took a few files ([data/python_lang](https://github.com/TyKo0707/code_completion/tree/main/data/python_lang)) from the local repository and annotated them. 
Each target is located inside `$$tag ...$$` symbols. 
There are 9 different types of tags (`code_by_description, conditional_statement, var_declaration, class_initialization, function_name, function_parameter, description_by_code, method_call, imports`) that can be later used for troubleshooting or analysis, but as of now it is just a small bonus since the dataset is relatively small. 
Example:
```python 
tensorboard_callback = keras.callbacks.$$method_call TensorBoard(log_dir=logdir, histogram_freq=1)$$
# target part is `TensorBoard(log_dir=logdir, histogram_freq=1)` with tag method_call

def generate_time_based_id():
    # Get the current time in the format YYYYMMDDHHMMSSFFF (year, month, day, hour, minute, second, millisecond)
    return $$code_by_description datetime.now().strftime("%Y%m%d%H%M%S%f")$$
    # target part is `code_by_description datetime.now().strftime("%Y%m%d%H%M%S%f")` with tag code_by_description
```
* Extracted all targets using regex along with suffixes and prefixes from the same file. The resulting dataset consists of 37 code snippets and can be found here: [data/python_dataset.csv](https://github.com/TyKo0707/code_completion/blob/main/data/python_dataset.csv).


### 2. Generation Code Completions:
* Using dataset and 4 models, generate missing code parts and save them into [data/python_dataset_gen.csv](https://github.com/TyKo0707/code_completion/blob/main/data/python_dataset_gen.csv)


### 3. Evaluation:
The most exciting part is here. I need to evaluate generated data manually and automatically and then compare different metrics by task suitability.
#### Goals of the Evaluation
For each generated sample, with its target form and context (prefix and suffix), we will measure the following:
- **Exact Match**: Does the generated code exactly match the target? If yes, skip further measurements.
- **Functional Correctness**: Does the generated code run successfully?
- **Syntactic Similarity**: How close is the generated code to the target, in terms of syntax?
- **Semantic Similarity**: Even if the code isn't identical, does it accomplish the task correctly, given the context?

#### Manual Evaluation Metrics
I’ve defined three manual metrics to assess code generation:
- **Functional Correctness**: Does the generated code compile and run without errors?
- **Factual Correctness**: Does the generated code solve the intended task?
- **Relevance**: How well does the generated code fit the context and solve the task? (This is subjective but important.)

#### Automatic Evaluation Metrics
I applied similar logic to automatic metrics, with some slight differences in focus:

- **Exact Match**: (Exact match) Checks if the generated code is identical to the reference.
- **CHRF3**: (Syntactic similarity) Evaluates character-level similarity, capturing fine details.
- **Edit Distance**: (Syntactic similarity) Counts the number of changes required to match the reference.
- **Embedding Similarity**: (Semantic similarity) Measures how semantically similar the generated code is to the target using embeddings like [CodeT5p-110m-Embedding](https://huggingface.co/Salesforce/codet5p-110m-embedding).
- **ROUGE-L**: (Syntactic similarity) Focuses on matching long subsequences between the generated and target code.
- **Function Correctness (LLM)**: (Functional correctness) Automatically checks whether the generated code runs without errors, using LLMs for efficiency instead of manual tests.
