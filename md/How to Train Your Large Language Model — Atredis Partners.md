> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.atredis.com](https://www.atredis.com/blog/2024/6/3/how-to-train-your-large-language-model)

> Large Language Models (LLM) such as those provided by OpenAI (GPT3/4), Google (Gemini), Anthropic (Cl......

Large Language Models (LLM) such as those provided by OpenAI (GPT3/4), Google (Gemini), Anthropic (Claude) can be a useful tool to include when conducting security audits or reverse engineering; however, one of the main downsides of using these tools is the data you are reviewing is processed server side, meaning any data analyzed by the tool must be uploaded/sent to the server.

While these services provide privacy policies that may double pinky swear your data is safe, and they will not use it for training if you opt-out, as a consultant we are often working with a client's data that is under NDA, preventing the usage of these services. Outside of cases where an NDA is in place, a policy won't protect you from platform bugs or provider monitoring that may leak your data or research. We have already seen an example of this with OpenAI publicly confirming they monitor the usage of its service to identify potentially 'evil' usage by bad-actors - [https://openai.com/index/disrupting-malicious-uses-of-ai-by-state-affiliated-threat-actors/](https://openai.com/index/disrupting-malicious-uses-of-ai-by-state-affiliated-threat-actors/)

Besides privacy concerns, a few other disadvantages of using a hosted service are:

*   service may go away (outage/sale)
*   modified to prevent malicious use (RE/Exploitation often flagged)
    *   potentially resulting monitoring/account ban
*   costs (usually per-token)

Given these hurdles, smaller models that run locally on your own hardware are a promising path to leveraging a LLM without compromising your privacy or an NDA.

### Comparisons

To be fair, it is worth pointing out the differences between the hosted LLM offerings and the local versions. The big difference is going to be the size of the training dataset and model parameter size - this can be thought of as the amount of 'knowledge' or data stored within the model, more parameters is going to indicate more 'knowledge' it can reference based on your input. OpenAI does not provide the details of GPT4, GPT3 was +100-billion parameters while GPT3.5's size has not been disclosed, speculation/research/guessing indicates it is much smaller (~22b parameters) - due to fine-tuning and/or other'secret sauce'. It is speculated that the original GPT4 is in the +100-trillion parameter range. On the other hand, a local model that will run on consumer hardware is going to be in the 2b-70b range, this obviously is a clear disadvantage and is going to result in lower quality responses when compared to a hosted service.

### Run Whatcha Brung

The actual size of the model you can run is going to be dependent on how much memory you have available - a decent rule is that the model will occupy 2x the memory of the parameter size: 2b/4gb, 7b/14gb, etc. The main exception to this rule is models that have been modified to use smaller values for stored parameters (quantization). Normally a model will use 16-bit floating point values for parameters; however, by clipping these values to smaller units (8/4-bit) the size can be reduced with minimal to no quality drop, resulting in lower memory usage and faster results.

When it comes to actual speed of results, it comes down to where you are running your inference. The best results are going to come from a recent GPU, ideally 24GB VRAM, meaning NVIDIA 3090 or 4090 - a used 3090 is best for the money for a turnkey solution. The next best setup is going to be an Apple Silicon (arm) Macbook/Studio/etc. - while this may be contentious, it is difficult to match the performance due to the shared memory architecture as you are able to use system ram for compute without a performance hit. While it is possible to run these models from system ram using the CPU on x86/64 machines, there is a performance hit compared to the previous options and results are most likely going to be slow - of course there are caveats here, as with anything you will find cases where highly tuned setups can perform well, in this case we are just considering ease of use and time to get started.

### Execution

There are quite a few ways to run models locally, in this case I am using [Ollama](https://ollama.com/) as it `just-works` and is fairly batteries-included for most use cases. Ollama provides installers for OSX, Linux, and Windows. Downloading and running a local model is as easy as executing the command `ollama run` with a model from the [registry](https://ollama.com/library), the required files will automatically download and enter an interactive 'chat' shell:

```
% ollama run phi3
pulling manifest
pulling b26e6713dc74... 100% ▕██████████████████████████████████████████████████████████████████████████████████████████████████████████▏ 2.4 GB
pulling fa8235e5b48f... 100% ▕██████████████████████████████████████████████████████████████████████████████████████████████████████████▏ 1.1 KB
pulling 542b217f179c... 100% ▕██████████████████████████████████████████████████████████████████████████████████████████████████████████▏  148 B
pulling 8dde1baf1db0... 100% ▕██████████████████████████████████████████████████████████████████████████████████████████████████████████▏   78 B
pulling f91db7a2deb9... 100% ▕██████████████████████████████████████████████████████████████████████████████████████████████████████████▏  485 B
verifying sha256 digest
writing manifest
removing any unused layers
success
>>> write me a sentence for my blog post about using a local running LLM to assist reversing binary applications.
 "Utilizing a local language model (LLM) tailored for our environment empowers developers to efficiently reverse binary applications, enhancing the debugging and
understanding of complex software systems."

>>> /bye

```

With the boring up-front details out of the way, we can try and do something useful.

### Hammer Looking for Nails

As with any tool, it works best if you have a specific use case you are looking to apply it to, in this case we will be looking at using a model to help analyze Hex-Rays pseudocode. As pseudocode is close to C, most models that have been trained to include code should be able to aid with this task, in this case I have chosen to work with Mistral's 7b model. While there are larger models available that I could run on my hardware, this model has been found to perform quite well for its size and I really wanted to target something that would be relatively performant on lower end hardware.

### Prompting Your Way Out

The first step in this process is to understand how well the model will execute a given task and identify the instructions needed to get a useful response. The following prompt was executed:

```
Analyze the following code and provide a description of what it does. Based on analysis provide updated variable names to replace the current ones.
unsigned int __fastcall sub_C385C(int a1)
{
  int v1;
  unsigned int v2;

  v1 = byte_11A2DC[a1];
  if ( sub_CE2BC(0) )
    v2 = v1 | 4;
  return v1 & 0xFFFFFFFD;
}

```

The prompt produced the following output:

```
The given code defines a function named `sub_C385C` that takes an integer argument `a1`. Inside the function, an integer variable `v1` is declared and initialized with the value of the memory location pointed to by `byte_11A2DC` with index `a1`.

Next, a check is made whether the result of calling `sub_CE2BC(0)` is true. If it is, then the value of `v1` is OR-ed with the binary number 4. The resulting value is assigned to a new variable `v2`, which remains undeclared in the given code snippet.

Finally, the original variable `v1` has its bits andered using the bitwise AND operator with the binary number -5 (0xFFFFFFFD) and this result is returned from the function.

To improve the readability of the code, you could consider renaming variables as follows:

```c
unsigned int __fastcall get_value_from_array(int arrayIndex)
{
  unsigned int value;

  value = memoryLocation[arrayIndex]; 

  if ( checkCondition() )
    value |= 4; 

  return value & 0xFFFFFFFD; 
}

Note: Replace `memoryLocation` and `checkCondition` with actual variable names.

```

This response is promising looking, but trying to programmatically use this response would be difficult. We can do better by adjusting our prompt to include instructions to return the values in JSON:

```
Analyze the following code and provide a description of what it does. Based on analysis provide a new function name, new variable names, and a comment describing what the code does.Only respond with valid JSON using the keys 'function_name','comment', and an array 'variables'. Values should use plain ascii with no special characters.

unsigned int __fastcall sub_C385C(int a1)
{
  int v1;
  unsigned int v2;

  v1 = byte_11A2DC[a1];
  if ( sub_CE2BC(0) )
    v2 = v1 | 4;
  return v1 & 0xFFFFFFFD;
}

```

The following output was generated:

```
{
  "function_name": "process_byte_value",
  "comment": "This function takes an integer input, retrieves the corresponding byte value from a global array, performs a condition check and returns the processed byte value.",
  "variables": [
    {
      "name": "input",
      "type": "int"
    },
    {
      "name": "byte_value",
      "type": "int"
    },
    {
      "name": "additional_flag",
      "type": "unsigned int"
    },
    {
      "name": "global_array",
      "type": "unsigned int[]",
      "description": "Global array of byte values."
    }
  ]
}



unsigned int process_byte_value(int input) {
  int byte_value;
  unsigned int additional_flag = 0;

  if (sub_CE2BC() == true) { 
    additional_flag = 4;
  }

  byte_value = global_array[input]; 

  return (byte_value & 0xFFFFFFF8) | additional_flag; 

```

Running the same input another time, we get a slight variation on the output

```
{
  "function_name": "processInputByte",
  "comment": "This function processes an input byte by checking a specific condition and modifying the byte accordingly.",
  "variables": [
    {"name": "inputByte", "type": "int"},
    {"name": "byteMask", "type": "unsigned int"},
    {"name": "conditionMet", "type": "bool"},
    {"name": "processedByte", "type": "unsigned int"}
  ]
}


unsigned int processInputByte(int inputByte) {
  unsigned int processedByte = BYTE_11A2DC[inputByte]; 
  bool conditionMet = sub_CE2BC(0); 

  if (conditionMet) {
    processedByte |= 4; 
  }

  return processedByte & 0xFFFFFFFD; 

```

Neither of these are ideal as we only want the JSON structure to parse programmatically. By adjusting the prompt we can try to guide the output towards what we are expecting:

```
Analyze the following code and provide a description of what it does. Based on analysis provide a new function name, new variable names, and a comment describing what the code does. Only respond with valid JSON using the keys 'function_name','comment', and an array 'variables'. Values should use plain ascii with no special characters. Do not respond outside the specified JSON structure.

unsigned int __fastcall sub_C385C(int a1)
{
  int v1;
  unsigned int v2;

  v1 = byte_11A2DC[a1];
  if ( sub_CE2BC(0) )
    v2 = v1 | 4;
  return v1 & 0xFFFFFFFD;
}

```

The resulting output of this prompt:

```
{
  "function_name": "processInputByte",
  "comment": "This function takes an input integer and processes a byte from a lookup table based on the input. If a certain condition is met, a bit is set in the output.",
  "variables": [
    {
      "name": "input",
      "description": "The input integer"
    },
    {
      "name": "byteValue",
      "description": "The value of the byte from the lookup table corresponding to the input"
    },
    {
      "name": "outputBitSet",
      "description": "An output variable that has a bit set if the condition is met"
    },
    {
      "name": "lookupTable",
      "description": "A global lookup table used to retrieve the byte value based on the input"
    }
  ]
}

```

This output is much better, it follows our instructions and produced a valid json structure only; however, if we run the same input again the output varies and returns a slightly different structure, along with details outside the JSON:

```
{
  "function_name": "process_byte_value",
  "comment": "Processes a byte value from the memory location specified by the input parameter and returns the result.",
  "variables": [
    {
      "name": "input_index",
      "type": "int",
      "description": "The index of the byte value to be processed."
    },
    {
      "name": "processed_value",
      "type": "unsigned int",
      "description": "The processed value of the input byte."
    },
    {
      "name": "is_special_condition_met",
      "type": "int",
      "description": "Flag indicating whether a special condition is met or not."
    }
  ]
}

unsigned int process_byte_value(int input_index)
{
  unsigned int processed_value;
  int is_special_condition_met = sub_CE2BC(0);

  processed_value = byte_11A2DC[input_index];

  if (is_special_condition_met)
    processed_value |= 4;

  return processed_value & 0xFFFFFFFD;
}

```

We could continue down this path changing the instructions trying to describe what your expected output should be for the given input, but eventually the response will end up deviating in some way breaking the ability to programmatically interface with it. Instead of trying to describe what we want, we can provide an example of what we want it to generate (single/multi-shot prompting). An important thing to remember when dealing with a LLM is that it is predicting the next token/word to output based on current input, by providing an example exchange you are creating a pattern for it to follow. The following revised prompt shows the original prompt modified to provide a single-shot example:

```
Analyze the following code and provide a description of what it does. Based on analysis provide a new function name, new variable names, and a comment describing what the code does.
Only respond with valid JSON using the keys 'function_name','comment', and an array 'variables'. Values should use plain ascii with no special characters. Do not respond outside the specified JSON structure.
For example, the user will submit code for analysis:

unsigned int __fastcall sub_C1337(int a1,int a2)
{
  int v1;
  unsigned int v2;

  v1 = byte_fff[a1 + a2];
  if ( v1 )
    v2 = v1 + 0xFF
  return v1 & 0xFFFFFFFD * v2;
}

Your response would be:
{
"function_name": "new_function_name",
"comment": "This function uses the arguments a1 and a2 as index into byte_fff, returning a bitwise value based on the result of the called function subCE2BC.",
"variables":[{"original_name":"v1","new_name":"AnalyzedVarName"},"original_name":"v2","new_name":"AnalyzedVarName"}]
}

Analyze the following code:
unsigned int __fastcall sub_C385C(int a1)
{
  int v1;
  unsigned int v2;

  v1 = byte_11A2DC[a1];
  if ( sub_CE2BC(0) )
    v2 = v1 | 4;
  return v1 & 0xFFFFFFFD;
}

```

In this prompt we have provided the instructions as we had previously as well as an example showing what a user would submit and the expected response, followed by the actual code we want analyzed. In this case, the responses match our desired output and are consistent across multiple executions.

Initial generated response:

```
{
  "function_name": "checkAndModifyByteValue",
  "comment": "This function uses the argument a1 as an index into byte_11A2DC. If the sub_CE2BC function returns true, the value at that index is modified by setting the 4th bit to 1. The function then returns the original value with the 3 highest bits cleared.",
  "variables": [
    {
      "original_name": "v1",
      "new_name": "ModifiedByteValue"
    },
    {
      "original_name": "v2",
      "new_name": "ModifiedByteWithBitSet"
    }
  ]
}

```

Followup executions:

```
{
  "function_name": "analyze_byte_from_array_and_apply_condition",
  "comment": "This function uses the argument a1 as index into byte_11A2DC, applies a condition using external function sub_CE2BC, and returns a bitwise value based on the result.",
  "variables": [
    {
      "original_name": "v1",
      "new_name": "AnalyzedByte"
    },
    {
      "original_name": "v2",
      "new_name": "ConditionalValue"
    }
  ]
}

{
  "function_name": "newFunctionName",
  "comment": "This function takes an integer argument a1, retrieves a value from the memory location byte\\_11A2DC with the index a1, and if the sub\\_CE2BC function returns true, sets a new variable value to the original value orpped with 4. The final return is a bitwise AND of the original value with 0xFFFFFFFD.",
  "variables": [
    {
      "original_name": "v1",
      "new_name": "ResultVar"
    },
    {
      "original_name": "v2",
      "new_name": "ModifiedVar"
    }
  ]
}

{
  "function_name": "checkBitAndModifyByte",
  "comment": "This function uses the argument a1 as an index into byte_11A2DC, checks if sub_CE2BC returns true and modifies the value if it does by setting the 4th bit. It then returns the original value with some bits cleared.",
  "variables": [
    {
      "original_name": "v1",
      "new_name": "ByteValue"
    },
    {
      "original_name": "v2",
      "new_name": "ModifiedByte"
    }
  ]
}

```

The results are decent and do not stray from the format/content we provided in our prompt; we can even include more examples with varying content (multi-shot) if we wanted to. At this point we have a basic prompt that does a specific task that we want relatively well, and the response is parsable (JSON) for automated use.

### Light Customization

In the case you have a specific use case (agent/assistant/task) you can configure a version of your underlying pre-trained weights for use through Ollama's [Modelfile](https://github.com/ollama/ollama/blob/main/docs/modelfile.md) interface. Ollama's `Modelfile` provides a lightweight layer to control/configure precomputed weights that can be easily edited and shared with other users. The following shows an example `Modelfile` configured for our potential Hex-Rays assistant using the prompt we created:

```
# defines the base pre-computed weights we want to use
FROM mistral:7b-instruct

# template is the format of the interactions with the model
# this is using templating provided by ollama where .System
# and .Prompt  are replaced with the defined variables 
TEMPLATE "{{ .System }}
[INST]
{{ .Prompt }}
[/INST]
"

# SYSTEM is the prompt/text that the model is started with, there are some special values included within this prompt
# that are described below, for now this is where the prompt we developed earlier goes
SYSTEM """<s>[INST]Analyze the following code and provide a description of what it does. Based on analysis provide a new function name, new variable names, and a comment describing what the code does.
Only respond with valid JSON using the keys 'function_name','comment', and an array 'variables'. Values should use plain ascii with no special characters. Do not respond outside the specified JSON structure.
For example, the user will submit code for analysis:

unsigned int __fastcall sub_C1337(int a1,int a2)
{
  int v1;
  unsigned int v2;

  v1 = byte_fff[a1 + a2];
  if ( v1 )
    v2 = v1 + 0xFF
  return v1 & 0xFFFFFFFD * v2;
}

Your response would be:
{
"function_name": "new_function_name",
"comment": "This function uses the arguments a1 and a2 as index into byte_fff, returning a bitwise value based on the result of the called function subCE2BC.",
"variables":[{"original_name":"v1","new_name":"AnalyzedVarName"},"original_name":"v2","new_name":"AnalyzedVarName"}]
}

Analyze the following code:[/INST]
</s>
"""
PARAMETER stop [INST]
PARAMETER stop [/INST]
# these control internal settings within the model to adjust how it behaves
PARAMETER temperature 1.2
PARAMETER top_k 100
PARAMETER top_p 0.09
PARAMETER num_ctx 4096
PARAMETER repeat_last_n 512
PARAMETER repeat_penalty 1.1

```

To side track for a second, each model has its own prompt format that is required to be used, as well as specific tokens used to indicate what is an instruction as well as start/stop tokens - these values can be found within the Tokenizer configuration file (`tokenizer_config.json`). For instance, the Mistral 7b-Instruct ([https://huggingface.co/mistralai/Mistral-7B-Instruct-v0.1/blob/main/tokenizer_config.json](https://huggingface.co/mistralai/Mistral-7B-Instruct-v0.1/blob/main/tokenizer_config.json)) defines the special values and format we used in our `Modelfile`:

```
{
  ...
  ...
  "bos_token": "<s>",
  "chat_template": "{{ bos_token }}{% for message in messages %}{% if (message['role'] == 'user') != (loop.index0 % 2 == 0) %}{{ raise_exception('Conversation roles must alternate user/assistant/user/assistant/...') }}{% endif %}{% if message['role'] == 'user' %}{{ '[INST] ' + message['content'] + ' [/INST]' }}{% elif message['role'] == 'assistant' %}{{ message['content'] + eos_token + ' ' }}{% else %}{{ raise_exception('Only user and assistant roles are supported!') }}{% endif %}{% endfor %}",
  "clean_up_tokenization_spaces": false,
  "eos_token": "</s>",
  ...
  ...
}

```

Not all models use the same `chat_template` structure or beginning-of-string (`bos_token`) or end-of-string (`eos_token`) values, so it is worth understanding where those formats and tokens come from.

After saving this file to disk (`idapal_simple.modelfile`), it can be loaded using Ollama:

```
% ollama create iDAPalSimple -f idapal_simple.modelfile
transferring model data
using existing layer sha256:e8a35b5937a5e6d5c35d1f2a15f161e07eefe5e5bb0a3cdd42998ee79b057730
using existing layer sha256:43070e2d4e532684de521b885f385d0841030efa2b1a20bafb76133a5e1379c1
using existing layer sha256:2af02daf0820d322d7e928bec1d7398090f4679c49c24c67575b5cffa7f08668
creating new layer sha256:cd2ecea3b680dc83a3d3981ec5efcca96c74f7b5794e7008d8f94522bf024f0d
using existing layer sha256:b4f4d07d73c548c4cd652806db6d567ceff891910db2bf5d5410dcc7e76eeb5a
creating new layer sha256:96a6cc4cffa5e48e458c33efa5301f519279a08eb51ae4a9b33e9ac7abce76d9
writing manifest
success

```

After creation, the new configuration can be tested by entering the interactive mode via the `run` command and pasting in the previous code:

```
% ollama run iDAPalSimple
>>> unsigned int __fastcall sub_C385C(int a1)
... {
...   int v1;
...   unsigned int v2;
...
...   v1 = byte_11A2DC[a1];
...   if ( sub_CE2BC(0) )
...     v2 = v1 | 4;
...   return v1 & 0xFFFFFFFD;
... }
{
"function_name": "new_function_name_C385C",
"comment": "This function uses the argument a1 as an index into byte\_11A2DC. It calls sub\_CE2BC with an argument of 0 and sets v2 to the result of
bitwise OR operation between v1 and 4 if sub\_CE2BC returns true. The final return value is obtained by performing a bitwise AND operation between
v1 and 0xFFFFFFFD.",
"variables":[{"original_name":"v1","new_name":"AnalyzedVarName_C385C"},{"original_name":"v2","new_name":"AnalyzedVarName_C385C_v2"}]
}

```

The observed response shows that the model works and is using the configured system prompt, returning the expected result after being provided only a code block as input. Ollama also provides an API that can be accessed locally ([https://github.com/ollama/ollama/blob/main/docs/api.md](https://github.com/ollama/ollama/blob/main/docs/api.md)), this can be used as seen in the following simple Python client:

```
import requests,json

def do_analysis(code):
    url = "http://localhost:11434/api/generate"
    headers = {"Content-Type": "application/json"}
    
    payload = {"model": "iDAPalSimple", "prompt": code, "stream": False,"format": "json"}
    res = requests.post(url, headers=headers, json=payload)
    try:
        t = res.json()['response']
        t = json.loads(t)
        return t
    except:
        print(f'error unpacking response')
        print(res.json()['response'])


input_code = '''unsigned int __fastcall sub_C385C(int a1)
{
  int v1;
  unsigned int v2;

  v1 = byte_11A2DC[a1];
  if ( sub_CE2BC(0) )
    v2 = v1 | 4;
  return v1 & 0xFFFFFFFD;
}'''

result = do_analysis(input_code)
print(result)

% python simple_analysis.py
{'function_name': 'new_function_name', 'comment': 'This function uses the argument a1 as an index into byte_11A2DC. It calls sub_CE2BC with an argument of 0 and sets v2 to the result of bitwise OR operation between v1 and 4 if sub_CE2BC returns true. The final return value is obtained by performing a bitwise AND operation between v1 and 0xFFFFFFFD.', 'variables': [{'original_name': 'v1', 'new_name': 'AnalyzedVarName1'}, {'original_name': 'v2', 'new_name': 'AnalyzedVarName2'}]}

```

At this point, the current configuration and simple Python client could be integrated into an IDA Plugin that would work ok, but we can do better.

### Fine-Tuning - step one: draw two circles

The initial training and creation of model weights that are released is a computationally expensive process, while follow on `fine-tuning` training is much less expensive to conduct. Fine-tuning provides a path to give a pre-trained model a "personality" by introducing new data and/or example interactions that would be considered "ideal" behavior when interacting with a user. The process is iterative and can be conducted multiple times until the model matches the expected behavior when interacting with a user.

While our small local model is never going to compete with a large, hosted service, fine-tuning can be used to boost its performance and compete on specific tasks or knowledge domains. To carry out a fine tune of a model you need complete the following steps:

*   Identify a target knowledge domain
*   Construct a dataset for your target domain
*   Train against your dataset
*   Evaluate trained model

For this task, the knowledge domain is already known - we want to fine tune a model that can be used to aid with analysis of Hex-Rays pseudocode. The next step is constructing a dataset, this is the difficult part. At a high level the dataset that needs to be built will be made of "instruction-following" examples, for instance the following shows what this would look like:

```
{
  "instruction":"Assist the user with a helpful process for drawing an animal.",
  "input":"How do I draw an Owl?",
  "output":"Drawing an Owl is simple, first draw some circles, then you draw the rest of the Owl."
},
{
  "instruction":"Assist the user with an idea for an animal to draw and provide them instructions.",
  "input":"",
  "output":"There are many animals you could choose to draw, my favorite is the Owl. An Owl is a fairly basic animal to draw, first draw some circles, then you draw the rest of the Owl"
}

```

These examples provide two types of instruction-following dataset entries. The first example starts by specifying the instruction to be followed with a matching input (user input) and finally the ideal output (generated result). The second example only provides an instruction along with an example output - the second example is useful for generative only responses that do not require user input, this type is not that useful for our current task but was included as an example. A dataset entry that would be useful for our task would look like the following:

```
{
  "instruction":"Analyze the following IDA Hex Rays pseudocode and generate a valid JSON object containing the keys 'function_name','comment', and an array 'variables' explaining what the code does, suggest a function name based on the analysis of the code, and new variable names based on the analysis of the code.",
  "input":"unsigned int __fastcall sub_C385C(int a1)\n {\n int v1;\n unsigned int v2;\n\n v1 = byte_11A2DC[a1];\n if ( sub_CE2BC(0) )\n v2 = v1 | 4;\n return v1 & 0xFFFFFFFD;\n }",
  "output":"{'function_name': 'new_function_name', 'comment': 'This function uses the argument a1 as an index into byte_11A2DC. It calls sub_CE2BC with an argument of 0 and sets v2 to the result of bitwise OR operation between v1 and 4 if sub_CE2BC returns true. The final return value is obtained by performing a bitwise AND operation between v1 and 0xFFFFFFFD.', 'variables': [{'original_name': 'v1', 'new_name': 'AnalyzedVarName1'}, {'original_name': 'v2', 'new_name': 'AnalyzedVarName2'}]}"
}

```

As a side note, following this exact formatting in JSON will allow for using the [datasets](https://huggingface.co/docs/datasets/en/index) library from Hugging Face and is going to be a common format.

With the exact format needed for training identified, the next problem is that we really need thousands of these examples, ideally with high quality responses. I had considered trying to manually create the required dataset using `tree-sitter` to rewrite valid code with generic variable names while sourcing the function description from documentation, this sounded painful, and I wanted the machine to do the hard work for me. Looking at earlier work done by Stanford for the Alpaca project ([https://crfm.stanford.edu/2023/03/13/alpaca.html](https://crfm.stanford.edu/2023/03/13/alpaca.html)) I decided to try the same style of approach. The basic idea of this workflow is to use a LLM to build your dataset based on a smaller dataset, or in this case an incomplete dataset and train against that:

After some noodling around I came up with the following high-level process:

*   compile libc with full debug/symbol information
*   load the compiled libraries into IDA and export all functions Hex-Rays output into individual files by address
*   strip the compiled libraries and repeat the previous step, exporting all functions Hex-Rays output into a new set of files

This process creates two directories with matching files:

```
/symbol/0x2d7f4.c
/stripp/0x2d7f4.c

```

In this case the file `/symbol/0x2d7f4.c` contains:

```
void __fastcall setname(int category, const char *name)
{
  char *v3; 

  v3 = (char *)nl_global_locale.__names[category];
  if ( v3 != name )
  {
    if ( v3 != "C" )
      j___GI___libc_free(v3);
    nl_global_locale.__names[category] = name;
  }
}

```

And the file `/stripp/0x2d7f4.c` contains:

```
char *__fastcall sub_2D7F4(int a1, char **a2)
{
  char *result; 

  result = (char *)off_170C10[a1 + 16];
  if ( result != (char *)a2 )
  {
    if ( result != "C" )
      result = (char *)j_free();
    off_170C10[a1 + 16] = a2;
  }
  return result;
}

```

With the two sets of data, the next stage of processing is to generate the dataset records. At a high-level this process looks like the following:

*   using the previously created mistral-7b configuration, query using the symbol/debug Hex-Rays output to get a reasonable quality output
*   create a dataset entry by combining the matching STRIPPED Hex-Rays output with the generated output from the symbol/debug Hex-Rays
*   iterate over all the files until complete

After completing this step we have a large completed instruction-following dataset we can use to fine tune against.

### Heavy Customization

There are quite a few options when it comes to carrying out a fine tune of a LLM, at the time of this research project I chose to use [unsloth](https://github.com/unslothai/unsloth). The following projects are also popular and most likely more batteries-included:

*   [https://github.com/OpenAccess-AI-Collective/axolotl](https://github.com/OpenAccess-AI-Collective/axolotl)
*   [https://github.com/oobabooga/text-generation-webui](https://github.com/oobabooga/text-generation-webui)

I went with unsloth for a few reasons, the main reason being underlying code has been tuned to provide a large performance increase (speed/memory usage), also it seemed less likely to abstract or hide parts of the training process that may be useful to see or understand. The unsloth project also provides a Jupyter notebook that can be executed on the Google Colab free tier if you do not have hardware (works perfectly!) - I ended up conducting training on a local Linux host with an NVIDIA 3090. To give an idea of performance, the free Colab tier took **21 minutes** while my 3090 executed the same training in **7 minutes**. Refer to the unsloth repository for install instructions, at the time of this project the installation using `conda` looked like the following:

```
conda create --name unsloth_env python=3.10
conda activate unsloth_env
conda install cudatoolkit xformers bitsandbytes pytorch pytorch-cuda=12.1 -c pytorch -c nvidia -c xformers -c conda-forge -y
pip install "unsloth[conda] @ git+https://github.com/unslothai/unsloth.git"

```

The script used for training was adopted from the examples provided by unsloth, the script uses Hugging Face's Supervised Fine-tuning Trainer ([SFT](https://huggingface.co/docs/trl/sft_trainer)) from the Transformer Reinforcement Learning ([TRL](https://huggingface.co/docs/trl/index)) library:

```
from unsloth import FastLanguageModel
import torch,sys

model = sys.argv[1]
steps = int(sys.argv[2])
training_data = sys.argv[3]

max_seq_length = 4096 
dtype = None 
load_in_4bit = True 

model, tokenizer = FastLanguageModel.from_pretrained(
    
    model_name = model,
    max_seq_length = max_seq_length,
    dtype = dtype,
    load_in_4bit = load_in_4bit,
)

model = FastLanguageModel.get_peft_model(
    model,
    r = 32, 
    target_modules = ["q_proj", "k_proj", "v_proj", "o_proj",
                      "gate_proj", "up_proj", "down_proj",],
    lora_alpha = 16, 
    lora_dropout = 0, 
    bias = "none",    
    use_gradient_checkpointing = True,
    random_state = 3407,
    use_rslora = False,  
    loftq_config = None, 
)

alpaca_prompt = """Below is an instruction that describes a task, paired with an input that provides further context. Write a response that appropriately completes the request.

### Instruction:
{}

### Input:
{}

### Response:
{}"""

EOS_TOKEN = tokenizer.eos_token 
def formatting_prompts_func(examples):
    instructions = examples["instruction"]
    inputs       = examples["input"]
    outputs      = examples["output"]
    texts = []
    for instruction, input, output in zip(instructions, inputs, outputs):
        
        text = alpaca_prompt.format(instruction, input, output) + EOS_TOKEN
        texts.append(text)
    return { "text" : texts, }
pass


from datasets import load_dataset
dataset = load_dataset("json", data_files=training_data, split = "train")
dataset = dataset.map(formatting_prompts_func, batched = True,)


from trl import SFTTrainer
from transformers import TrainingArguments

trainer = SFTTrainer(
    model = model,
    tokenizer = tokenizer,
    train_dataset = dataset,
    dataset_text_field = "text",
    max_seq_length = max_seq_length,
    dataset_num_proc = 2,
    packing = False, 
    args = TrainingArguments(
        per_device_train_batch_size = 4,
        gradient_accumulation_steps = 4,
        warmup_steps = 5,
        max_steps = steps,
        learning_rate = 2e-4,
        fp16 = not torch.cuda.is_bf16_supported(),
        bf16 = torch.cuda.is_bf16_supported(),
        logging_steps = 1,
        optim = "adamw_8bit",
        weight_decay = 0.01,
        lr_scheduler_type = "linear",
        seed = 3407,
        output_dir = "outputs",
        save_strategy= "steps",
        save_steps=50
    ),
)

gpu_stats = torch.cuda.get_device_properties(0)
start_gpu_memory = round(torch.cuda.max_memory_reserved() / 1024 / 1024 / 1024, 3)
max_memory = round(gpu_stats.total_memory / 1024 / 1024 / 1024, 3)
print(f"GPU = {gpu_stats.name}. Max memory = {max_memory} GB.")
print(f"{start_gpu_memory} GB of memory reserved.")


trainer_stats = trainer.train()

used_memory = round(torch.cuda.max_memory_reserved() / 1024 / 1024 / 1024, 3)
used_memory_for_lora = round(used_memory - start_gpu_memory, 3)
used_percentage = round(used_memory         /max_memory*100, 3)
lora_percentage = round(used_memory_for_lora/max_memory*100, 3)
print(f"{trainer_stats.metrics['train_runtime']} seconds used for training.")
print(f"{round(trainer_stats.metrics['train_runtime']/60, 2)} minutes used for training.")
print(f"Peak reserved memory = {used_memory} GB.")
print(f"Peak reserved memory for training = {used_memory_for_lora} GB.")
print(f"Peak reserved memory % of max memory = {used_percentage} %.")
print(f"Peak reserved memory for training % of max memory = {lora_percentage} %.")

model.save_pretrained(f"lora_model_{steps}") 


if True: model.save_pretrained_merged(f"model_{steps}", tokenizer, save_method = "lora",)


if True: model.save_pretrained_gguf(f"model_{steps}", tokenizer, quantization_method = "q4_k_m")

```

The script also defines the following items:

```
output_dir = "outputs",
        save_strategy= "steps",
        save_steps=50

```

This configuration will save a copy of the fine-tuned weights every 50 steps to a directory `outputs` - this is helpful for a few reasons. The first being if an error occurs at some point (crash/power/etc.) you have checkpoints you can restart your training from, the second being it allows you to effectively evaluate how well your training is working by comparing each saved checkpoint. While it may seem at first, more steps are better, this is going to be dependent on how large your dataset is and which settings you have configured - more is not always better.

Running this script to fine tune `mistral-7b-instruct` for 100 steps using the dataset we created would look like the following example output:

```
$ python training/train.py unsloth/mistral-7b-instruct-v0.2-bnb-4bit 100 ./dataset.json
==((====))==  Unsloth: Fast Mistral patching release 2024.2
   \\   /|    GPU: NVIDIA GeForce RTX 3090. Max memory: 23.691 GB. Platform = Linux.
O^O/ \_/ \    Pytorch: 2.2.0. CUDA = 8.6. CUDA Toolkit = 12.1.
\        /    Bfloat16 = TRUE. Xformers = 0.0.24. FA = False.
 "-____-"     Free Apache license: http://github.com/unslothai/unsloth
/mnt/new/unsloth/lib/python3.10/site-packages/transformers/quantizers/auto.py:155: UserWarning: You passed `quantization_config` or equivalent parameters to `from_pretrained` but the model you're loading already has a `quantization_config` attribute. The `quantization_config` from the model will be used.
  warnings.warn(warning_msg)
Unsloth 2024.2 patched 32 layers with 32 QKV layers, 32 O layers and 32 MLP layers.
Detected kernel version 5.4.0, which is below the recommended minimum of 5.5.0; this can cause the process to hang. It is recommended to upgrade the kernel to the minimum version or higher.
GPU = NVIDIA GeForce RTX 3090. Max memory = 23.691 GB.
4.676 GB of memory reserved.
==((====))==  Unsloth - 2x faster free finetuning | Num GPUs = 1
   \\   /|    Num examples = 2,897 | Num Epochs = 3
O^O/ \_/ \    Batch size per device = 4 | Gradient Accumulation steps = 4
\        /    Total batch size = 16 | Total steps = 500
 "-____-"     Number of trainable parameters = 83,886,080
{'loss': 1.4802, 'grad_norm': 1.6030948162078857, 'learning_rate': 4e-05, 'epoch': 0.01}
{'loss': 1.4201, 'grad_norm': 1.4948327541351318, 'learning_rate': 8e-05, 'epoch': 0.01}
{'loss': 1.5114, 'grad_norm': 1.6689960956573486, 'learning_rate': 0.00012, 'epoch': 0.02}
{'loss': 1.1665, 'grad_norm': 0.9258238673210144, 'learning_rate': 0.00016, 'epoch': 0.02}
{'loss': 0.9282, 'grad_norm': 0.6133134961128235, 'learning_rate': 0.0002, 'epoch': 0.03}
{'loss': 0.9292, 'grad_norm': 0.6610234975814819, 'learning_rate': 0.0001995959595959596, 'epoch': 0.03}
{'loss': 0.7517, 'grad_norm': 0.4809339940547943, 'learning_rate': 0.0001991919191919192, 'epoch': 0.04}
{'loss': 0.7554, 'grad_norm': 0.6171303987503052, 'learning_rate': 0.00019878787878787878, 'epoch': 0.04}
{'loss': 0.606, 'grad_norm': 0.564286470413208, 'learning_rate': 0.00019838383838383837, 'epoch': 0.05}
{'loss': 0.6274, 'grad_norm': 0.414183109998703, 'learning_rate': 0.000197979797979798, 'epoch': 0.06}
{'loss': 0.6402, 'grad_norm': 0.3489008843898773, 'learning_rate': 0.0001975757575757576, 'epoch': 0.06}
{'loss': 0.596, 'grad_norm': 0.28150686621665955, 'learning_rate': 0.0001971717171717172, 'epoch': 0.07}
{'loss': 0.5056, 'grad_norm': 0.3132913410663605, 'learning_rate': 0.00019676767676767677, 'epoch': 0.07}
{'loss': 0.5384, 'grad_norm': 0.27469128370285034, 'learning_rate': 0.00019636363636363636, 'epoch': 0.08}
{'loss': 0.5744, 'grad_norm': 0.360963374376297, 'learning_rate': 0.00019595959595959596, 'epoch': 0.08}
{'loss': 0.5907, 'grad_norm': 0.3328467011451721, 'learning_rate': 0.00019555555555555556, 'epoch': 0.09}
{'loss': 0.5067, 'grad_norm': 0.2794954478740692, 'learning_rate': 0.00019515151515151516, 'epoch': 0.09}
{'loss': 0.5563, 'grad_norm': 0.2907596528530121, 'learning_rate': 0.00019474747474747476, 'epoch': 0.1}
{'loss': 0.5533, 'grad_norm': 0.34755516052246094, 'learning_rate': 0.00019434343434343435, 'epoch': 0.1}

```

After training is complete, I used a small script to evaluate how each checkpoint performs. To do this I take the first 10 entries from the training dataset and use the `instruction` and `input` values to generate a new output, as well as generating a new output using an input that was not in the original dataset:

```
from unsloth import FastLanguageModel
import torch,sys

model_name_input = sys.argv[1]

max_seq_length = 4096 
dtype = None 
load_in_4bit = True 

model, tokenizer = FastLanguageModel.from_pretrained(
    
    model_name = model_name_input,
    max_seq_length = max_seq_length,
    dtype = dtype,
    load_in_4bit = load_in_4bit,
    
)

alpaca_prompt = """Below is an instruction that describes a task, paired with an input that provides further context. Write a response that appropriately completes the request.

### Instruction:
{}

### Input:
{}

### Response:
{}"""

EOS_TOKEN = tokenizer.eos_token 
def formatting_prompts_func(examples):
    instructions = examples["instruction"]
    inputs       = examples["input"]
    outputs      = examples["output"]
    texts = []
    for instruction, input, output in zip(instructions, inputs, outputs):
        
        text = alpaca_prompt.format(instruction, input, output) + EOS_TOKEN
        texts.append(text)
    return { "text" : texts, }
pass


from datasets import load_dataset
dataset = load_dataset("json", data_files="data.json", split = "train")
dataset = dataset.map(formatting_prompts_func, batched = True,)

FastLanguageModel.for_inference(model)

samples = []
sample_size = 10
for x in range(0,sample_size):
    instruction = dataset[x]["instruction"]
    input       = dataset[x]["input"]
    output      = ''
    text = alpaca_prompt.format(instruction, input, output) 
    sample = tokenizer([text],return_tensors = "pt").to("cuda")
    out = model.generate(**sample,max_new_tokens=4096,use_cache=True)
    out = tokenizer.batch_decode(out)
    samples.append(out[0])


code = '''int __fastcall sub_75C80(int a1, int a2)
{
  int result; // r0
  _DWORD *i; // r3

  result = a2 - *(_DWORD *)(a1 + 12);
  for ( i = *(_DWORD **)(a1 + 48); i; i = (_DWORD *)*i )
  {
    if ( i[2] < result )
      result = i[2];
  }
  return result;
}'''

text = alpaca_prompt.format(instruction, code, output)
sample = tokenizer([text],return_tensors = "pt").to("cuda")
out = model.generate(**sample,max_new_tokens=4096,use_cache=True)
out = tokenizer.batch_decode(out)
samples.append(out[0])

print('Capturing pre training generation samples')
with open(f'results/eval_log_{model_name_input.replace("/","_")}','w') as log:
    for r in samples:
        log.write(r)

```

For running the script, it seemed easiest to just iterate over the checkpoints in `outputs` using bash:

```
for m in $(ls outputs); do python eval.py outputs/$m; done

```

### Results?

So, with training out of the way, the question is, does it work? Initial testing was performed against the following input:

```
### Instruction:
Analyze the following IDA Hex Rays pseudocode and generate a valid JSON object containing the keys 'function_name','comment', and an array 'variables' explaining what the code does, suggest a function name based on the analysis of the code, and new variable names based on the analysis of the code.

### Input:
int __fastcall sub_B0D04(int a1, int a2)
{
  unsigned int v2; 
  int result; 

  v2 = a1 + a2;
  if ( __CFADD__(a1, a2) )
    return 0;
  result = _libc_alloca_cutoff();
  if ( v2 <= 0x1000 )
    return result | 1;
  return result;
}

```

As expected, the base model did not follow the requested format very well and the function comment is low quality. At 50 training steps, the model 'understands' the expected output and matches perfectly - the somewhat surprising result is that function comment is better at 50 steps compared to 100 steps.

Zooming out a bit and comparing further steps, the format is perfect while the most common error seen is confusion on what gets returned (value vs allocated memory) or inconsistent numeric format (1000 vs 0x1000):

The real check is, how does this compare to the big models...

It is interesting to see that GPT3.5 is no better than our results and in fact performs worse than our 50-step results, failing into the same error as the 100-step result.

Comparing against GPT3.5 feels slightly unfair as it is quite old, what about GPT4?

Well… that result definitely makes this whole exercise feel painful and pointless. The quality of the comment is much higher, and it also captured more variable renames. So, the end result is: just use GPT4, using a small local model is pointless.

### Admitting Defeat and Using GPT4

So now that we tried our best with our small model, we can move on and just use GPT4, just not in the way you would expect. Going back and considering the [Alpaca](https://crfm.stanford.edu/2023/03/13/alpaca.html) project, they call out `using an existing strong language model to automatically generate instruction data`, while so far we have used our small 7b parameter model to generate instruction data. This is where we step back slightly and redo some of our previous work, replace our 'low quality' generated data with 'high quality' values from the current leading model.

Using the OpenAI playground is fairly simple to set up an 'assistant' with our instructions:

With the configuration working as expected, its straight forward to use the API and execute the same original instruction generation we previously had done:

I originally had no expectations related to the cost of this process, to be safe I added 50$ to my account before executing the previous step, I was surprised when it only cost ~16$ at the time:

Seeing that it only cost 16$ for the initial run and the quality of the responses were good, I figured why not use both sets of data and get 2x the high-quality instruction datasets?

With the brand-new high-quality dataset complete we can back up and start a new fine tune of our mistral-7b model, in this case it has been trained for 200 steps taking snapshots every 50 steps. After training is complete, an evaluation was done against a new input that is not in either dataset against our old 'low-quality' fine tune and our new one.

At 50 steps the new GPT4 trained version has already performed much better at capturing variables to rename, interestingly the LLM trained dataset description contains more direct references to the code while the GPT4 description is slightly higher level:

At 100 steps the variable names for the GPT4 trained model are slightly better and the description is slightly more technical, referring to specific items within the code. The LLM trained model has picked up the extra variable renames, but they look to be in line with what the GPT4 trained model had at 50 steps. I also thought it was interesting that the LLM trained model refers to `[2]` as the `third` field (mathematically correct):

At 150 steps the GPT4 trained model has slightly improved the function description while maintaining the variable renames. The LLM trained model has improved the function name to match the GPT4 trained model at 50 steps, while losing variable renames - interestingly it now refers to `[2]` as the `second element` now:

Finally, at 200 steps the GPT4 trained model has slightly tweaked its description. The LLM trained model has rediscovered its variable renames from the 100 steps version and also refined how it references the `[2]` within the code:

Clearly the mistral-7b model fine-tuned against the high-quality dataset from GPT4 performs much better than the previous version. The real test is to now compare it with GPT4 directly......

That response looks like something we have seen already, at this point I would say we have proven it is feasible to fine tune a small local model to perform a specific task at the level of a much larger model.

### Making Friends

So now that we have our fine-tuned local model, we need to hook it into IDA and feed it some Hex-Rays. There are a few other plugins that offer similar functionality:

*   [https://github.com/JusticeRage/Gepetto](https://github.com/JusticeRage/Gepetto)
*   [https://github.com/mrphrazer/reverser_ai](https://github.com/mrphrazer/reverser_ai)
*   [https://github.com/mahaloz/DAILA](https://github.com/mahaloz/DAILA)
*   sorry if I forgot yours, nothing personal :)

I decided to write my own simple version, apologies in advance for any errors or poor design decisions, the underlying fine-tuned model is available to use with whatever you like best. Building off the previous simple python script shown earlier, I again choose to use Ollama's rest service instead of loading the model directly - I like this design for few reasons:

*   minimal Python requirements
*   the service can be running on a remote machine with more compute
*   reload/maintenance/update will not interrupt your weeks long IDA session
*   avoids tying IDA up with a large memory footprint, that one you have had running for weeks now :)

To set up Ollama to use the new model, [download](https://github.com/atredispartners/aidapal) the weights and `Modelfile` in the same directory and configure Ollama:

```
% ollama create aidapal -f aidapal.modelfile
transferring model data
using existing layer sha256:d8ff55be57629cfb21d60d4977ffb6c09071104d08bce8b499e78b10481b0a3a
using existing layer sha256:2af02daf0820d322d7e928bec1d7398090f4679c49c24c67575b5cffa7f08668
using existing layer sha256:0c3d95e257e4029eb818625dbf1627a4ca182eefcdbc360d75c108afda3cf458
using existing layer sha256:3da0ba8b21dda1aba779a536319f87fbed8ee78e80b403ce2c393cec6d58e1a9
creating new layer sha256:5fe21ec0a43781478cefd5a2b4b047651c889e08f1d7e4bf7e8bc5a7413e425a
writing manifest
success

```

Loading the [plugin](https://github.com/atredispartners/aidapal) can be done through the IDA menu (File->Script File). After loading, the script provides a new context menu option when right-clicking within a Hex-Rays window:

In this example the plugin has been configured with a single model, if you have other models loaded within your Ollama service they can be added and will appear within the context menu as well. After activating the menu item, the plugin will query the selected model with the Hex-Rays code and return a dialog when it is complete:

Within this dialog all returned values can be accepted individually by selecting the checkbox (enabled by default) and clicking `Accept`, clicking `Cancel` will reject all and close the dialog.

In this example, the results are accepted and applied fully:

This example shows rejecting the function name and description, only applying the variable renames:

There is also nothing stopping you from accepting all changes multiple times:

Another consideration I had when creating aiDAPal was implementing some form of data lookup like Retrieval Augmented Generation (RAG), but in the spirit of keeping things simple I came up with the idea of treating the IDA database (IDB) as a lookup/knowledge base. The basic idea is whenever the plugin is activated, it will identify any references within the code that is being analyzed and retrieve any comments that exist at the target locations and include them as a multi-line comment before the function that is sent for analysis. An example of this workflow can be seen in the following image:

For this example, the `WDT_ICR` register location is queried for any comments, if one exists it gets extracted and included in our request. Something to consider is that in this case, the `WDT_ICR` register is common and is part of the 'base knowledge' stored within the original trained weights and would have be identified fine without the extra comment. This can be confirmed by querying the underlying model for this information:

```
% ollama run mistral:7b
>>> give me a single sentence description of the WDT_ICR register
 The WDT_ICR (Watchdog Timer Independent Counter Register) is a control register in the watchdog timer unit that triggers a reset upon being written, allowing configuring the watchdog timer

```

By using the IDB as an extra source of knowledge as shown previously, we can use our own information/knowledge to better guide the response. In the following image the comment associated with the `WDT_ICR` register has been changed, resulting in the model returning a different result that considers the additional knowledge that was provided by the IDB:

Currently, this functionality does not extract this information from comments that may be defined at the start of a function; while that would be useful and give context to the current analysis as to what a called function does, this would often result the inclusion of a large number of extra tokens potentially exhausting the underlying models context window and return low quality results.

### The End?

While I am sure I made mistakes along the way, I hope this information is helpful to anyone wanting to fine-tune a LLM for local usage; whether that is making a better version of the one we are sharing or something completely different. It is also worth noting most of this project was executed earlier this year (feb/march), since then a handful of new models have been released that would be interesting to explore/adapt this research to (phi3-med/llama3/Codestral). If you made it this far, thanks for reading.

All files related to this project can be found on our GitHub ([https://github.com/atredispartners/aidapal](https://github.com/atredispartners/aidapal)).