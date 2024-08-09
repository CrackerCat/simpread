> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [paper.seebug.org](https://paper.seebug.org/3210/?continueFlag=be186ae28af6df4b80c1b9a238df8338)

> 作者：Hcamael@知道创宇 404 实验室 英文版: https://paper.seebug.org/3211/

**作者：Hcamael@知道创宇 404 实验室  
英文版: [https://paper.seebug.org/3211/](https://paper.seebug.org/3210/ "https://paper.seebug.org/3211/")**

本文将系统分享从零开始搭建本地大模型问答知识库过程中所遇到的问题及其解决方案。

目前，搭建大语言问答知识库能采用的方法主要包括微调模型、再次训练模型以及增强检索生成（RAG，Retrieval Augmented Generation）三种方案。

而我们的目标是希望能搭建一个低成本、快速响应的问答知识库。由于微调 / 训练大语言模型需要耗费大量资源和时间，因此我们选择使用开源的本地大型语言模型结合 RAG 方案。经过测试，我们发现`llama3:8b`和`qwen2:7b`这种体量的大语言模型能快速响应用户的提问，比较符合我们搭建问答知识库的需求。

我们花了一段时间，对 RAG 各个步骤的原理细节和改进方案进行了一些研究，下面将按照 RAG 的步骤进行讲解。

1.1 简述 RAG 原理
-------------

首先，我们来讲解一下 RAG 的原理。假设我们有三个文本和三个问题，如下所示：

```
context = [
    "北京，上海，杭州",
    "苹果，橘子，桃子",
    "太阳，月亮，星星"
]
questions = ["城市", "水果", "天体"]

```

接下来使用 ollama 的`mxbai-embed-large`模型，把三个文本和三个问题都转化为向量的表示形式，代码如下所示：

```
vector = []
model = "mxbai-embed-large"
for c in context:
    r = self.engine.embeddings(c, model=model)
    vector += [r]
    print("r =", r)
'''
r = [-0.4238928556442261, -0.037000998854637146, ......
'''
qVector = []
for q in questions:
    r = self.engine.embeddings(q, model=model)
    qVector += [r]
    print("r =", r)
'''
q = [-0.3943982422351837,
'''

```

接下来使用 numpy 模块来编写一个函数，计算向量的相似度，代码如下所示：

```
def cosine_similarity(self, a, b):
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))
for i in range(3):
    for j in range(3):
        similar = self.engine.cosine_similarity(qVector[i], vector[j])
        print(f"{questions[i]}和{context[j]}的相似度为：{similar}")
'''
城市和北京，上海，杭州的相似度为：0.6192201604133171
城市和苹果，橘子，桃子的相似度为：0.6163859899608889
城市和太阳，月亮，星星的相似度为：0.5885895914816769

水果和北京，上海，杭州的相似度为：0.6260800765574224
水果和苹果，橘子，桃子的相似度为：0.6598514846105531
水果和太阳，月亮，星星的相似度为：0.619382129127254

天体和北京，上海，杭州的相似度为：0.5648588692973202
天体和苹果，橘子，桃子的相似度为：0.6756043740633552
天体和太阳，月亮，星星的相似度为：0.75651740246562
'''

```

从上面的示例可以看出，计算出的结果值越大，表示两个文本之间的相似度越高。上述过程是 RAG 原理的简化版。

有一定基础的都知道，大语言模型本质上就是计算概率，因此它只能回答训练数据中包含的内容。如果要搭建一个问答知识库，并且该知识库的内容是公开的，那么这些内容很可能已经包含在大模型的训练数据集中。在这种情况下，不需要进行任何额外操作，可以直接使用大模型进行问答，这也是目前大多数人使用大语言模型的普遍方式。

然而，我们要搭建的本地知识库大部分包含私有数据，这些内容并不存在于当前大语言模型的训练数据集中。在这种情况下，可以考虑对大模型进行微调或重新训练，但这种方案既费时又费钱。

因此，更常用的方案是将知识库的内容放入到 prompt 中，让大语言模型通过我们提供的内容来进行回答。但是该方案存在一个问题，那就是 token 的长度限制（上下文长度限制）。比如，本地大语言模型中的`qwen2:7b`，`token`的最大长度为`128k`，表示输入的上下文通过`AutoTokenizer`计算出的长度不能超过 128k，但是正常情况下，知识库的大小超过 128k 是很正常的。我们不可能把所有的知识库内容都放入到 prompt 中，因此就有了 RAG 方案。

RAG 的方案步骤如下所示：

1.  对知识库根据一定的大小进行分片处理。
    
2.  使用 embedding 大模型对分片后的知识库进行向量化处理。
    
3.  使用 embedding 大模型对用户的问题也进行向量化处理。
    
4.  把用户问题的向量和知识库的向量数据进行相似度匹配，找出相似度最高的 k 个结果。
    
5.  把这 k 个结果作为上下文放入到 prompt 当中，跟着用户的问题进行提问。
    

在开头的例子中，`context`变量相当于知识库的内容，`"城市"`就相当于用户提出的问题，接着通过相似度计算，`"北京，上海"`是最接近的信息，所以接着把该信息作为提问的上下文提供给 GPT 进行问答。

一个简单的 prompt 示例如下所示：

```
prompt = [
  {
      "role": "user",
      "content": f"""当你收到用户的问题时，请编写清晰、简洁、准确的回答。
你会收到一组与问题相关的上下文，请使用这些上下文，请使用中文回答用户的提问。不允许在答案中添加编造成分，如果给定的上下文没有提供足够的信息，就回答"不要提供与问题无关的信息，也不要重复。
> 上下文：
>>>
{context}
>>>
> 问题：{question}
"""
  }
]

```

1.2 使用 redis-search 计算相似度
-------------------------

在上述的例子中，是自行编写了一个`cosine_similarity`函数来计算相似度，但是在实际的应用场景中，知识库的数据会非常大，这会导致计算相似度的速度非常慢。

经过研究发现，能对向量进行存储并且快速计算相似度的工具有：

*   redis-search
*   chroma
*   elasticsearch
*   opensearch
*   lancedb
*   pinecone
*   qdrant
*   weaviate
*   zilliz

下面分享一个使用 redis-search 的 KNN 算法来快速找到最相似的 k 个内容的方案。

首先搭建`redis-search`环境，可以使用 docker 一键搭建，如下所示：

```
$ docker pull redis/redis-stack
$ docker run --name redis -p6379:6379 -itd redis/redis-stack

```

接着编写相关 python 代码，如下所示：

```
import redis
from redis.commands.search.query import Query
from redis.commands.search.field import TextField, VectorField

class RedisCache:
    def __init__(self, host: str = "localhost", port: int = 6379):
        self.cache = redis.Redis(host=host, port=port)

    def set(self, key: str, value: str):
        self.cache.set(key, value)

    def get(self, key: str):
        return self.cache.get(key)

    def TextField(self, name: str):
        return TextField(name=name)

    def VectorField(self, name: str, algorithm: str, attributes: dict):
        return VectorField(name=name, algorithm=algorithm, attributes=attributes)

    def getSchema(self, name: str):
        return self.cache.ft(name)

    def createIndex(self, index, schema):
        try:
            index.info()
        except redis.exceptions.ResponseError:
            index.create_index(schema)

    def dropIndex(self, index):
        index.dropindex(delete_documents=True)

    def hset(self, name: str, map: dict):
        self.cache.hset(name=name, mapping=map)

    def query(self, index, base_query: str, return_fields: list, params_dict: dict, num: int, startnum: int = 0):
        query = (
            Query(base_query)
            .return_fields(*return_fields)
            .sort_by("similarity")
            .paging(startnum, num)
            .dialect(2)
        )
        return index.search(query, params_dict)

redis = RedisCache()
info = TextField("info")

algorithm="HNSW"
r = self.engine.embeddings(q, model=model)
DIM = len(r)
attributes={
    "TYPE": "FLOAT32",
    "DIM": DIM,
    "DISTANCE_METRIC": "COSINE"
}
embed = VectorField(
    name=name, algorithm=algorithm, attributes=attributes
)
scheme = (info, embed)
index = redis.getSchema(self.redisIndexName)
redis.createIndex(index, scheme)

def insertData(self, model):
    j = 0
    for file in self.filesPath:
        for i in file.content:
                        embed = self.engine.embeddings(i, model = model)
            emb = numpy.array(embed, dtype=numpy.float32).tobytes()
            im = {
                "info": i,
                "embedding": emb,
            }
            name = f"{self.redisIndexName}-{j}"
            j += 1
            self.redis.hset(name, im)

k = 10
base_query = f"* => [KNN {k} @embedding $query_embedding AS similarity]"
return_fields = ["info", "similarity"]
qr = self.engine.embeddings(question, model = model)
params_dict = {"query_embedding": np.array(qr, dtype=np.float32).tobytes()}
index = self.redis.getSchema(self.redisIndexName)
result = self.redis.query(index, base_query, return_fields, params_dict, k)
for _, doc in enumerate(result.docs):
        print(doc.info, doc.similarity)

```

在了解了 RAG 的原理后，我们就可以尝试编写相关代码，搭建一个本地问答知识库。但是在我们开始行动后，就会发现事情并不会按照我们所预料的发展。

2.1 难点一：大语言模型能力不足
-----------------

在我们的设想中，问答知识库是这样工作的：

1.  根据用户的提问，在知识库中找到最相似 k 个的内容。
2.  把这 k 个内容作为上下文，提供给大模型。
3.  大模型根据用户提问的上下文，给出准确的答案。

但是在实际应用的过程中，程序却无法按照我们的意愿运行，首先是问答的大语言模型能力不足，毕竟我们使用的是`qwen2:7b`这种小体积的大模型，无法和`GPT4`这类的大模型相比较。但是修改一下提问的方式或者上下文和问题之间的顺序，还是能比较好的达到我们预期的效果。

但是更重要的是 embed 大模型的能力同样存在不足，这里用开头例子进行说明，我们把`context`的内容简单修改一下，如下所示：

```
context = [
    "北京，上海，杭州",
    "苹果，橘子，梨",
    "太阳，月亮，星星"
]

```

然后再计算一次相似度，如下所示：

```
城市和北京，上海，杭州的相似度为：0.6192201604133171
城市和苹果，橘子，梨的相似度为：0.6401285511286077
城市和太阳，月亮，星星的相似度为：0.5885895914816769

水果和北京，上海，杭州的相似度为：0.6260800765574224
水果和苹果，橘子，梨的相似度为：0.6977096659034031
水果和太阳，月亮，星星的相似度为：0.619382129127254

天体和北京，上海，杭州的相似度为：0.5648588692973202
天体和苹果，橘子，梨的相似度为：0.7067548826946035
天体和太阳，月亮，星星的相似度为：0.75651740246562

```

我们发现，`"城市"`竟然和`"苹果，橘子，梨"`是最相似的。虽然这只是一个简单的例子，但是在实际的应用中经常会遇到这类的情况，通过用户提问的内容找到的知识库中最相似的内容有可能跟问题并不相关。

这是否是本地 embed 大模型的问题？OpenAI 的 embed 大模型效果如何？接着我们找了一些本地 embed 大模型和 OpenAI 的`text-embedding-3-small`、`text-embedding-ada-002`、`text-embedding-3-large`进行一个简单的测试，判断通过问题找到的最相似的知识库上下文，是否是预期的内容。

最终的结果是，不管是本地的大模型还是 OpenAI 的大模型，成功率都在 50%-60% 之间。embed 大模型的能力差是普遍存在的问题，并不仅仅是本地 embed 大模型的问题。

该问题经过我们一段时间的研究，找到了 “将就” 能用的解决方案，将会在后文细说。

2.2 难点二：提问的复杂性
--------------

理想中的问答知识库是能处理复杂问题，并且能进行多轮对话，而不是一轮提问就结束。

以`Seebug Paper`作为知识库，提出的问题具体到某一篇文章，并且多轮对话之间的问题是相互独立的，这种情况是最容易实现的。比如：

```
user: 帮我总结一下CVE-2024-22222漏洞。   
assistant: ......  
user: CVE-2023-1111漏洞的危害如何？  
assistant: ......  
......  

```

但是想要让问答知识库成为一个好用的产品，不可能把目标仅限于此，在实际的应用中还会产生多种复杂的问题。

1. 范围搜索性提问

参考以下几种问题：

> 2024 年的文章有哪些？  
> CTF 相关的文章有哪些？  
> 跟 libc 有关的文章有哪些？  
> ......

拿第一个问题举例，问题为：`2024年的文章有哪些？`。

接着问答知识库的流程为：

1.  通过 embed 模型对问题进行向量化。
2.  通过 redis 搜索出问题向量数据最接近的 k 个知识库内容。(假设 k=10)
3.  然后把这 10 个知识库内容作为上下文，让 GPT 进行回答。

如果按照上面的逻辑来进行处理，最优的情况就是，问答大模型根据提供的 10 个知识库上下文成功的回答了 10 篇 2024 年的文章。

这样问题就产生了，如果 2024 年的文章有 20 篇呢？也许我们可以提高 k 值的大小，但是 k 值提高的同时也会增加运算时间。虽然看似临时解决了问题，但是却产生了新问题，k 值提高到多少？提高到 20？那如果 2024 年的文章有 50 篇。提高到 100？那如果 2024 年的文章如果只有 1 篇，该方案就浪费了大量运算时间，同时可能会超过大语言模型的 token 限制，如果使用的是商业 GPT(比如 OpenAI)，那么 token 就是金钱，这就浪费了大量资金。如果使用的是开源大语言模型，token 同样有限制，并且在 token 增加的同时，大语言模型的能力也会相应的下降。

2. 多轮对话

参考以下多轮提问：

> user: 2024 年的文章有哪些？  
> assistant: ......  
> user: 还有吗？

问答知识库在处理第二个提问的流程为：

1.  通过 embed 模型对`"还有吗？"`问题进行向量化。
2.  通过 redis 搜索出问题向量数据最接近的 k 个知识库内容。(假设 k=10)
3.  ......

问题产生了，根据`"还有吗？"`搜索出的相似上下文会是我们需要的上下文吗？基本上不可能是。

该问题经过我们一段时间的研究，同样是找到了 “将就” 能用，但是并不优雅的解决方案，将会在后文细说。

2.3 难点三：文本的处理
-------------

在使用`Seebug Paper`搭建问答知识库的过程中，发现两个问题：

1.  文章中的图片怎么处理？
2.  文章长度一般在几 k 到几十 k 之间，因此是需要分片处理的，那么如何分片？

关于图片处理勉强还有一些解决方案：

1.  使用 OCR 识别将图片转换为文字。（效果不太好，因为有些图片重要的不是文字。）
2.  使用 llava 这类的大模型来对图片进行描述和概括（llava 效果不太好，不过 gpt4 的效果会好很多）。
3.  直接加上如下图所示，并且在 prompt 中告诉 GPT，让其需要用到图片时，直接返回图片的链接。

但是分片的问题却不是很好处理，如果要进行分片，那么如何分片呢？研究了`llama_index`和`langchain`框架，基本都是根据长度来进行分片。

比如在`llama_index`中，对数据进行分片的代码如下所示：

```
from llama_index import SimpleDirectoryReader
from llama_index.node_parser import SimpleNodeParser

documents = SimpleDirectoryReader(input_dir="./Documents").load_data()
node_parser = SimpleNodeParser.from_defaults(chunk_size=514, chunk_overlap=80)
nodes = node_parser.get_nodes_from_documents(documents)

```

在上面的示例代码中，设置了`chunk_size`的值为 514，但是在 Paper 中，经常会有内嵌的代码段长度大于 514，这样就把一段相关联的上下文分在了多个不同的 chunk 当中。

假设一个相关联的代码段被分成了 chunk1 和 chunk2，但是根据用户问题搜索出相似度前 10 的内容中只有 chunk2 并没有 chunk1，最终 GPT 只能获取到 chunk2 的上下文内容，缺失了 chunk1 的上下文内容，这样就无法做出正确的回答。

不仅仅是代码段，如果只是通过长度进行分片，那么有可能相关联的内容被分成了多个 chunk。

下面分享一些针对上面提出的难点研究出的解决方案，但是仅仅只是将就能用的方案，都是通过时间换准确率，暂时没找到完美的解决方案。

3.1 rerank 模型
-------------

通过研究 [`QAnything`](https://github.com/netease-youdao/QAnything "QAnything")项目，发现可以使用 rerank 模型来提升 embed 模型的准确率。

举个例子，比如你想取前 10 相似的内容作为提问的上下文。那么可以通过 redis 获取到前 20 相似的内容，接着通过 rerank 模型对这 20 个内容进行重打分，最后获取前 10 分数最高的内容。相关代码示例如下所示：

```
from BCEmbedding import RerankerModel

rerankModel = RerankerModel(model_name_or_path="maidalun1020/bce-reranker-base_v1", local_files_only=True)

......
k = 20
result = self.redis.query(index, base_query, return_fields, params_dict, k)
passages = []
for _, doc in enumerate(result.docs):
    passages += [doc.info]
rerank_results = rerankModel.rerank(question, passages)
info = rerank_results["rerank_passages"]
last_result = info[:10]

```

`rerank`模型本质上就是训练出了一个专门打分的大语言模型，让该模型对问题和答案进行打分。该方案一定程度上可以提升搜索出内容的准确度，但是仍然无法完美解决难点一的问题。

3.2 上下文压缩
---------

通过研究 [`LLMLingua`](https://github.com/microsoft/LLMLingua "LLMLingua")和`langchain`项目，发现了上下文压缩方案。

在上面的例子中，都是以`k=10`来举例的，那么 k 的值等于多少才合适呢，这需要根据知识库分片的大小和大语言模型的能力来调整。

首先，要求上下文的长度加上 prompt 的内容不能超过大语言模型 token 长度的限制。其次，基本所有的大语言模型都会随着上下文的增加导致能力下降。所以需要根据使用的大语言模型找到一个长度阙值，来设置 k 的大小。

在实际应用中，会发现前 k 个上下文中可能大部分内容都跟用户的提问无关，因此可以使用上下文压缩技术，去除无用的内容，减小上下文体积，增加 k 值大小。

由于`LLMLingua`项目使用的模型有点大，跑起来费时费电（跑不起来），所以根据其原理，实现了一个低级版本的压缩代码，如下所示：

```
def compress(self, question: str, context: list[str], maxToken: int = 1024) -> list[str]:
        template = f"下面将会提供问题和上下文，请判断上下文信息是否和问题相关，如果不相关，请回复        result = []
        for c in context:
            qs = template%c
            answer = self.engine.chat(qs)
                                    if "                result += [answer]
        newContent = "\n".join(result)
        question = f"你是一个去重机器人，下面将会提供一组上下文，请你对上下文进行去重处理。*注意*，请*不要*自行发挥，*不要*进行任何添加修改，请直接在上下文内容中进行去重。\n上下文：>>>\n{newContent}\n>>>"
        answer = self.engine.chat(question)
        return answer

```

由于有了上下文压缩方案，我们可以考虑设置 k 的值为一个非常大的值，然后分析计算出的相似度的值，比如我发现在我的案例中，redis 搜索结果相似度大于`0.4`的内容就完全跟提问无关，rerank 重打分分数小于`0.5`的内容完全跟提问无关，所以可以做出以下修改：

```
k = 1000
......
passages = []
for _, doc in enumerate(result.docs):
        if float(doc.similarity) > 0.4:
        break
    passages += [doc.info]
rerank_results = rerankModel.rerank(questionHistory, passages)
info = rerank_results["rerank_passages"]
score = rerank_results["rerank_scores"]
contexts_list = []
for i in range(len(info)):
        if float(score[i]) > 0.5:
        contexts_list += [info[i]]
    else:
        break
contexts = self.compress(question, contexts_list)

```

通过以上方案，一定程度上解决了`范围性搜索提问`的难题。同样，我们可以尝试寻找一个相似度分数的阙值，来解决`多轮对话`的难题，因为`"还有吗？"`这类的多轮对话问题和知识库的相关性非常低，得到的分数都会非常低。这样当我们最终获取到的上下文内容为空时，表明当前为多轮对话，再按照对轮对话的逻辑进行处理。

3.3 知识库分片处理
-----------

经过研究，目前没找到完美的分片方案，我认为针对不同格式的知识库设计针对性的分片方案会比较好。

针对`Seebug Paper`的情况，我们考虑根据`一级标题`来进行分片，每个 chunk 中还需要包含当前文章的基础信息，比如文章名称。如果有代码段，则需要根据 token 的大小来进一步分片。

在一些框架中，不同 chunk 之前会有一部分重叠内容，但是我们研究后发现这种处理方案不会让最终的效果有较大的提升。

经过研究，我们发现固定格式的文档是最佳的知识库素材，例如漏洞应急简报，每篇简报的内容大小适中，并且采用 Markdown 格式便于匹配和处理。我们能根据`漏洞概述、漏洞复现、漏洞影响范围、防护方案、相关链接`来进行分片，每部分的相关性都不大。

我们期望的问答知识库是大语言模型能根据我们提供的知识库**快速**、**准确**的回答用户的提问。目前来看是还是不存在理想中的问答知识库，一方面是由于大语言模型能力的限制，在当前的大语言模型中，`快速响应`和`精准响应`还是一对反义词。

使用 embed 大语言模型寻找相关文档的准确率太低，大部分的优化方案都是通过时间换取准确率。所以还是寄希望于生成式大语言模型未来的发展，是否能达成**真**人工智能。

目前的问答知识库和`RAG`类的框架效果相差不大，都是属于先把框架建好，把大语言模型分割开来，能随意替换各类大语言模型，因此这类框架的能力取决于使用的是什么大语言模型。相当于建造一个机器人，把身体都给搭建好了，但是还缺少一颗优秀的脑子。

1.  [https://github.com/netease-youdao/QAnything](https://github.com/netease-youdao/QAnything "https://github.com/netease-youdao/QAnything")
2.  [https://github.com/microsoft/LLMLingua](https://github.com/microsoft/LLMLingua "https://github.com/microsoft/LLMLingua")

![](https://images.seebug.org/content/images/2017/08/0e69b04c-e31f-4884-8091-24ec334fbd7e.jpeg) 本文由 Seebug Paper 发布，如需转载请注明来源。本文地址：[https://paper.seebug.org/3210/](https://paper.seebug.org/3210/)