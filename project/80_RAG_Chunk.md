# 自然语言处理:第八十章  RAG分块策略：主流方法（递归、jina-seg）+前沿推荐（Meta-chunking、Late chunking、SLM-SFT）

**本人项目地址大全：[Victor94-king/NLP__ManVictor: CSDN of ManVictor](https://github.com/Victor94-king/NLP__ManVictor)**

<br />

***写在前面: 笔者更新不易，希望走过路过点个关注和赞，笔芯!!!***

***写在前面: 笔者更新不易，希望走过路过点个关注和赞，笔芯!!!***

***写在前面: 笔者更新不易，希望走过路过点个关注和赞，笔芯!!!***

<br />



大多数常用的数据分块方法（chunking）都是基于规则的，采用 fixed chunk size（译者注：将数据或文本按照固定的大小进行数据分块）或 overlap of adjacent chunks（译者注：让相邻的数据块具有重叠内容，确保信息不会丢失。） 等技术。对于具有多个层级结构的文档，可以使用 Langchain 提供的 RecursiveCharacterTextSplitter，这种方法允许将文档按照不同的层级进行分割。

然而，在实际应用中，由于预定义的规则（比如数据分块大小（chunk size）或重叠部分的大小（size of overlapping parts））过于死板，基于规则的数据分块方法很容易导致检索到的上下文（retrieval contexts）不完整或包含 noise（译者注：指不需要的、干扰性的信息或数据，可能会对分析或处理造成干扰或误导的数据。） 的数据块过大等问题

* 长文本向量化的挑战

  在基于 Transformer 架构的向量化模型中，每个词汇都会被映射为一个高维向量。为了表示整段文本的语义，通常采用对词向量取平均，或使用特殊标记（如 `[CLS]`）位置的向量作为整体表示。然而，当直接对过长的文本进行向量化时，会面临以下挑战：

  * **语义信息稀释** ：长文本往往涵盖多个主题或观点，整体向量难以准确捕捉细节语义，导致语义信息被稀释，无法充分体现文本的核心内容。
  * **计算开销增大** ：处理长文本需要更多的计算资源和存储空间，增加了模型的计算复杂度，影响系统的性能和效率。
  * **检索效率降低** ：过长的向量在检索过程中可能会降低匹配精度，导致检索结果的相关性下降，同时也会降低检索的速度和效率。
* 提升检索和生成质量的必要性

  为了克服上述挑战，**合理的文本分块策略**显得尤为重要。通过对文本进行适当的切分，可以有效提升检索和生成的质量：

  * **提高检索准确率** ：将文本分块后，得到的文本片段具有更精细的粒度，能够更准确地匹配用户的查询意图，提升检索结果的相关性和准确度。
  * **优化系统性能** ：缩短单个文本块的长度，减少了模型在向量化和检索过程中的计算和存储开销，提高了系统的处理效率和响应速度。
  * **增强大模型的回答质量** ：为大语言模型提供更相关和精炼的文本块，有助于模型更好地理解语境，从而生成更准确、连贯和贴切的回答。

> 通过对文本进行合理的分块，不仅可以提高向量化过程中的语义表达能力，还能在检索阶段取得更高的匹配精度，最终使得 RAG 系统能够为用户提供更优质的服务。

# 1.文本分块策略对大模型输出的影响

## 1.1 文本分块过长的影响

在构建 RAG（Retrieval-Augmented Generation）系统时，文本分块的长度对大模型的输出质量有着至关重要的影响。**过长的文本块**会带来一系列问题：

1. **语义模糊** ：当文本块过长时，在向量化过程中，细节语义信息容易被平均化或淡化。这是因为向量化模型需要将大量的词汇信息压缩成固定长度的向量表示，导致无法精准捕捉文本的核心主题和关键细节。结果就是，生成的向量难以代表文本的重要内容，降低了模型对文本理解的准确性。
2. **降低召回精度** ：在检索阶段，系统需要根据用户的查询从向量数据库中检索相关文本。过长的文本块可能涵盖多个主题或观点，增加了语义的复杂性，导致检索模型难以准确匹配用户的查询意图。这样一来，召回的文本相关性下降，影响了大模型生成答案的质量。
3. **输入受限** ：大语言模型（LLM）对输入长度有严格的限制。过长的文本块会占据更多的输入空间，减少可供输入的大模型的文本块数量。这限制了模型能够获取的信息广度，可能导致遗漏重要的上下文或相关信息，影响最终的回答效果。

## 1.2 文本分块过短的影响

相反，**过短的文本块**也会对大模型的输出产生不利影响，具体表现为：

1. **上下文缺失** ：短文本块可能缺乏必要的上下文信息。上下文对于理解语言的意义至关重要，缺乏上下文的文本块会让模型难以准确理解文本的含义，导致生成的回答不完整或偏离主题。
2. **主题信息丢失** ：段落或章节级别的主题信息需要一定的文本长度来表达。过短的文本块可能只包含片段信息，无法完整传达主要观点或核心概念，影响模型对整体内容的把握。
3. **碎片化问题** ：大量的短文本块会导致信息碎片化，增加检索和处理的复杂度。系统需要处理更多的文本块，增加了计算和存储的开销。同时，过多的碎片化信息可能会干扰模型的判断，降低系统性能和回答质量。

通过上述分析，可以得出结论： **合理的文本分块策略是提升 RAG 系统性能和大模型回答质量的关键** 。为了在实际应用中取得最佳效果，需要在以下方面进行权衡和优化：

1. **根据文本内容选择切分策略** ：不同类型的文本适合不同的切分方法。

* **逻辑紧密的文本** ：对于论文、技术文档等段落内逻辑紧密的文本，应尽量保持段落的完整性，避免过度切分，以保留完整的语义和逻辑结构。
* **语义独立的文本** ：对于法规条款、产品说明书等句子间逻辑相对独立的文本，可以按照句子进行切分。这种方式有助于精确匹配特定的查询内容，提高检索的准确性。

1. **考虑向量化模型的性能** ：评估所使用的向量化模型对于不同长度文本的处理能力。

* **长文本处理** ：如果向量化模型在处理长文本时容易丢失信息，应适当缩短文本块的长度，以提高向量表示的精确度。
* **短文本优化** ：对于能够有效处理短文本的模型，可以适当切分文本，但要注意保留必要的上下文信息。

1. **关注大模型的输入限制** ：大语言模型对输入长度有一定的限制，需要确保召回的文本块能够全部输入模型。

* **输入长度优化** ：在切分文本块时，控制每个块的长度，使其既包含完整的语义信息，又不超过模型的输入限制。
* **信息覆盖** ：确保切分后的文本块能够覆盖知识库中的关键信息，避免遗漏重要内容。

1. **实验与迭代** ：没有一种放之四海而皆准的最佳实践，需要根据具体的应用场景进行实验和调整。

* **性能评估** ：通过实验评估不同切分策略对检索准确性和生成质量的影响，从而选择最适合的方案。
* **持续优化** ：根据模型的表现和用户反馈，不断优化切分策略，提升系统的整体性能。

# 2. 常见的文本分块策略

在 RAG（Retrieval-Augmented Generation，检索增强生成）系统中，**文本分块策略**的选择对系统性能和大模型的生成质量有着重要影响。合理的文本分块能提高检索准确性，并为大模型生成提供更好的上下文支持。下面，将深入探讨几种常见的文本分块方法及其应用场景。

常用的文本分块方法包括： **固定大小分块、基于 NTLK 分块、特殊格式分块、深度学习模型分块、智能体式分块** 。

## 2.1 固定大小文本切块(递归方法)

**固定大小文本切块**是最简单直观的文本分块方法。它按照预先设定的固定长度，将文本划分为若干块。这种方法实现起来相对容易，但在实际应用中，需要注意以下几点：

* **问题与挑战**

  * **上下文割裂** ：简单地按照固定字符数截断文本，可能会打断句子或段落，导致上下文信息丢失。这会影响后续的文本向量化效果和语义理解。
  * **语义完整性受损** ：文本块可能包含不完整的句子或思想，影响检索阶段的匹配精度，以及大模型生成回答的质量。
* **改进方法**

  * **引入重叠** ：在相邻文本块之间引入一定的重叠部分，确保上下文的连贯性。比如，每个文本块与前一个块有 50 个字符的重叠。这有助于保留句子的完整性和段落的连贯性。
  * **智能截断** ：在切分文本时，尽量选择在标点符号或段落结束处进行截断，而不是严格按照字符数。这可以避免打断句子，保持语义的完整性。
* 实践工具：LangChain 的 RecursiveCharacterTextSplitter**

大模型应用开发框架 **LangChain** 提供了 `RecursiveCharacterTextSplitter`，优化了固定大小文本切块的缺陷，推荐在通用文本处理中使用。

 **使用示例** ：

```
from langchain.text_splitter import RecursiveCharacterTextSplitter
text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=200,
    chunk_overlap=50,
    length_function=len,
    separators=["\n", "。", ""]
)
text = "..."# 待处理的文本
texts = text_splitter.create_documents([text])
for doc in texts:
    print(doc)

```

 **参数说明** ：

* `chunk_size`：文本块的最大长度（如 200 个字符）。
* `chunk_overlap`：相邻文本块之间的重叠长度（如 50 个字符）。
* `length_function`：用于计算文本长度的函数，默认为 `len`。
* `separators`：定义了一组分割符列表，用于在切分文本时优先选择合适的位置。

 **工作原理** ：

`RecursiveCharacterTextSplitter` 按照 `separators` 中的分割符顺序，递归地对文本进行切分：

1. **初步切分** ：使用第一个分割符（如 `"\n"`，表示段落分隔）对文本进行初步切分。
2. **检查块大小** ：如果得到的文本块长度超过了 `chunk_size`，则使用下一个分割符（如 `"。"`，表示句子分隔）进一步切分。
3. **递归处理** ：依次使用剩余的分割符，直到文本块长度符合要求或无法再切分。
4. **合并块** ：如果相邻的文本块合并后长度不超过 `chunk_size`，则进行合并，确保块的长度尽可能接近 `chunk_size`，同时保留上下文完整性。

## 2.2 基于 NLTK 、spaCy 的文本切块

**NLTK（Natural Language Toolkit）** 是广泛使用的 Python 自然语言处理库，提供了丰富的文本处理功能。其中，**`sent_tokenize`** 方法可用于自动将文本切分为句子。

 **原理** ：**`sent_tokenize`** 基于论文 **《Unsupervised Multilingual Sentence Boundary Detection》** 的方法，使用无监督算法为缩写词、搭配词和句子开头的词建立模型，然后利用这些模型识别句子边界。这种方法在多种语言（主要是欧洲语言）上都取得了良好效果。

* **预训练模型缺失** ：NLTK 官方并未提供中文分句模型的预训练权重，需要用户自行训练。
* **训练接口可用** ：NLTK 提供了训练接口，用户可以基于自己的中文语料库训练分句模型。
* **在 LangChain 中的应用**

LangChain 集成了 NLTK 的文本切分功能，方便用户直接调用。

 **使用示例** ：

```
from langchain.text_splitter import NLTKTextSplitter
text_splitter = NLTKTextSplitter()
text = "..."# 待处理的文本
texts = text_splitter.split_text(text)
for doc in texts:
    print(doc)

```

* **扩展：基于 spaCy 的文本切块**

**spaCy** 是另一款强大的自然语言处理库，具备更高级的语言分析能力。LangChain 也集成了 spaCy 的文本切分方法。

 **使用方法** ：

只需将 **`NLTKTextSplitter`** 替换为 **`SpacyTextSplitter`**：

```

from langchain.text_splitter import SpacyTextSplitter

text_splitter = SpacyTextSplitter()

text = "..."# 待处理的文本
texts = text_splitter.split_text(text)

for doc in texts:
    print(doc)
```

***提示：使用 spaCy 时，需要先下载对应语言的模型。例如，处理中文文本时，需要下载中文模型包。***

## 2.3 特殊格式文本切块（HTML、Markdown）

在实际应用中，常常需要处理具有特殊内在结构的文本，如 **HTML、Markdown、LaTeX、代码文件** 等。这些文本的结构信息对于理解其内容至关重要，简单的文本切分方法可能会破坏其原有结构，导致上下文信息丢失。

* **保留结构信息** ：在切分文本时，应尽量保留其内在的结构，如标签、标题、代码块等。
* **减少上下文损失** ：避免在关键位置切分文本，以免丢失重要的上下文信息。
* **LangChain 提供的特殊文本切块方法**

LangChain 为用户提供了针对多种特殊格式文本的切块类，方便用户处理不同类型的文本。

**表 4-1 LangChain 提供的特殊文本切块方法**

| 文本格式 | 类名                   |
| -------- | ---------------------- |
| Python   | PythonTextSplitter     |
| markdown | MarkdownTextSplitter   |
| latex    | LatexTextSplitter      |
| html     | HTMLHeaderTextSplitter |

以处理 Markdown 文本为例：

```

from langchain.text_splitter import MarkdownTextSplitter

text_splitter = MarkdownTextSplitter()

text = "..."# 待处理的 Markdown 文本
texts = text_splitter.split_text(text)

for doc in texts:
    print(doc)
```

这些特殊文本切块类针对不同的文本格式，预设了适合的分割符列表，然后调用 `RecursiveCharacterTextSplitter` 进行进一步的切分。例如：

* **`PythonCodeTextSplitter`**：针对 Python 代码的结构，设置适合的分隔符，如函数定义、类定义、注释等。
* **`MarkdownTextSplitter`**：根据 Markdown 的结构，如标题、列表、段落等，进行切分。
* **`LatexTextSplitter`**：识别 LaTeX 文档的章节、公式、环境等进行切分。
* **`HTMLHeaderTextSplitter`**：针对 HTML 文档的标签结构，按照元素层级进行切分。
* **4.3.5 自定义扩展**

LangChain 还预定义了其他编程语言（如 Go、C++、Java）等的分割符列表，方便用户快速定义新的文本切块类。如果需要处理未提供的文本格式，可以参照已有的类实现。

 **自定义示例** ：创建一个用于切分 Java 代码的文本切块类。

```
from langchain.text_splitter import RecursiveCharacterTextSplitter

class JavaCodeTextSplitter(RecursiveCharacterTextSplitter):
    def __init__(self, **kwargs):
        separators = [
            "\n\n",  # 空行
            "\n",    # 换行
            ";",     # 语句结束
            " ",     # 空格
            ""# 无分隔符
        ]
        super().__init__(separators=separators, **kwargs)

text_splitter = JavaCodeTextSplitter(chunk_size=200, chunk_overlap=50)

text = "..."# 待处理的 Java 代码
texts = text_splitter.split_text(text)

for doc in texts:
    print(doc)
```

通过自定义分割符列表和参数设置，可以灵活地适应不同格式文本的切分需求。

## 2.4  基于语义文本切块

* **Embedding-based** （译者注：基于嵌入的数据分块方法，数据被映射到一个低维空间中，以便更好地捕捉其语义信息。）
* **Model-based** （译者注：基于模型的数据分块方法，使用了预先训练好的模型来进行语义分块。）
* **LLM-based** （译者注：基于大语言模型的数据分块方法，使用 LLM 捕捉文本中的语义信息。）

### 2.4.1 **Embedding-based 方法**

LlamaIndex[2] 和 Langchain[3] 都提供了基于嵌入（embedding）的 semantic chunker（译者注：能够将文本或数据按照语义信息进行分块的工具或算法。） 。他们的算法理念大致是差不多的，本文将以 LlamaIndex 为例进行解读。

 **请注意，需要安装最新版本的 LlamaIndex ，才能使用 LlamaIndex 中的 semantic chunker。** 我之前安装的 LlamaIndex 版本号是 0.9.45，并没有包含这个算法。因此，我创建了一个新的 conda 虚拟环境，并安装了更新版本的 LlamaIndex —— 0.10.12：

```
pip install llama-index-core

pip install llama-index-readers-file

pip install llama-index-embeddings-openai

```

值得一提的是，0.10.12 版本的 LlamaIndex 可以根据需求安装所需的组件或模块，因此本文仅安装一些关键组件。

```
(llamaindex_010) Florian:~ Florian$ pip list | grep llama
llama-index-core              0.10.12
llama-index-embeddings-openai 0.1.6
llama-index-readers-file 0.1.5
llamaindex-py-client          0.1.13

```

测试代码如下所示：

```
from llama_index.core.node_parser import (
    SentenceSplitter,
    SemanticSplitterNodeParser,
)
from llama_index.embeddings.openai import OpenAIEmbedding
from llama_index.core import SimpleDirectoryReader


import os
os.environ["OPENAI_API_KEY"] = "YOUR_OPEN_AI_KEY"

# load documents
dir_path = "YOUR_DIR_PATH"
documents = SimpleDirectoryReader(dir_path).load_data()


embed_model = OpenAIEmbedding()
splitter = SemanticSplitterNodeParser(
    buffer_size=1, breakpoint_percentile_threshold=95, embed_model=embed_model
)

nodes = splitter.get_nodes_from_documents(documents)
for node in nodes:
 print('-' * 100)
 print(node.get_content())

```

 **`splitter.get_nodes_from_documents`** [4] 函数的内部实现逻辑，其主要流程如图 2 所示：

![图片](https://mmbiz.qpic.cn/sz_mmbiz_jpg/WZBibiatAX4xzYI49CsicFo6GcUOml2WibcEdXBLu3K2vvqGk8mANDSPWteMyUrJCg1t0ehQq5R8dxQ4gAGmglkgDw/640?wx_fmt=jpeg&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

图 2 中提到的 “sentences” 是一个 Python 列表，其中每个成员都是包含四个键值对的字典，各键的含义如下：

* **sentence** ：当前句子
* **index** ：当前句子的序号
* **combined_sentence** ：用于构建滑动窗口（sliding window），包括 [index - self.buffer_size, index, index + self.buffer_size] 三个句子 **（默认情况下，self.buffer_size = 1）** ，能够用于计算句子间的语义相关性。将当前句子与其前后的句子合并在一起，可以减少不必要的干扰信息，从而更有效地抓取连续句子之间的关联性。
* **combined_sentence_embedding** ：combined_sentence 的嵌入向量

通过以上分析可以明显看出， **基于嵌入向量的 semantic chunking** （译者注：根据语义信息将文本或数据分成有意义的片段或块的过程）  **方法本质上是基于滑动窗口（combined_sentence）计算文本相似度。** 那些相邻且符合阈值的句子会被归入一个语义块。

项目路径中只有一份 BERT 论文文档 [5]。以下是运行结果：

```
(llamaindex_010) Florian:~ Florian$ python /Users/Florian/Documents/june_pdf_loader/test_semantic_chunk.py 
...
...
----------------------------------------------------------------------------------------------------
We argue that current techniques restrict the
power of the pre-trained representations, espe-
cially for the ﬁne-tuning approaches. The ma-
jor limitation is that standard language models are
unidirectional, and this limits the choice of archi-
tectures that can be used during pre-training. For
example, in OpenAI GPT, the authors use a left-to-
right architecture, where every token can only at-
tend to previous tokens in the self-attention layers
of the Transformer (Vaswani et al., 2017). Such re-
strictions are sub-optimal for sentence-level tasks,
and could be very harmful when applying ﬁne-
tuning based approaches to token-level tasks such
as question answering, where it is crucial to incor-
porate context from both directions.
In this paper, we improve the ﬁne-tuning based
approaches by proposing BERT: Bidirectional
Encoder Representations from Transformers.
BERT alleviates the previously mentioned unidi-
rectionality constraint by using a “masked lan-
guage model” (MLM) pre-training objective, in-
spired by the Cloze task (Taylor, 1953). The
masked language model randomly masks some of
the tokens from the input, and the objective is to

predict the original vocabulary id of the
maskedarXiv:1810.04805v2  [cs.CL]  24 May 2019
----------------------------------------------------------------------------------------------------
word based only on its context. Unlike left-to-
right language model pre-training, the MLM ob-
jective enables the representation to fuse the left
and the right context, which allows us to pre-
train a deep bidirectional Transformer. In addi-
tion to the masked language model, we also use
a “next sentence prediction” task that jointly pre-
trains text-pair representations. The contributions
of our paper are as follows:
• We demonstrate the importance of bidirectional
pre-training for language representations. Un-
like Radford et al. (2018), which uses unidirec-
tional language models for pre-training, BERT
uses masked language models to enable pre-
trained deep bidirectional representations. This
is also in contrast to Peters et al. 
----------------------------------------------------------------------------------------------------
...
...

```

测试结果表明，使用这种数据分块方法，得到的数据块粒度相对较粗。

图 2 还显示，这种方法是 page-based 的（译者注：这种方法将文本按照页面进行数据分块处理，而非其他更小的单位，比如句子或段落。），并没有直接解决跨多个页面的数据分块问题。

**通常情况下，基于嵌入的数据分块方法，其性能严重依赖于嵌入模型。实际效果需要未来进一步评估。**

### 2.4.2 **Model-based 方法**

#### **基于 BERT 的文本切分方法**

为了使 BERT 模型能够学习 **两个句子之间的关系** ，在其预训练过程中设计了一个 **二分类任务** ：同时向 BERT 中输入两个句子，预测第二个句子是否是第一个句子的下一句。基于这一思想，可以设计一种 **最朴素的文本切分方法** ，其中 **最小的切分单位是句子** 。

具体而言，在完整的文本上，采用**滑动窗口**的方式，将相邻的两个句子分别输入 BERT 模型进行二分类预测。如果预测得分较低，说明这两个句子之间的语义关系较弱，此处可作为文本的切分点。然而，这种方法存在一定的局限性：

* **上下文信息有限** ：在判断切分点时，只考虑了前后两个相邻句子，未利用更长距离的文本信息，可能导致切分不准确。
* **计算效率低** ：需要对文本中的每对相邻句子进行预测，计算量较大，难以应对大规模文本处理的需求。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/WZBibiatAX4xzYI49CsicFo6GcUOml2WibcEiab5ian2libmbDEOiahRbZ1KfA0T4LomicwEfyjqQKjfwOkrfshnoFic5wng/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

#### **Cross-Segment 模型：引入更长的上下文依赖**

针对上述方法的不足，**Lukasik 等人**在论文_《Text Segmentation by Cross Segment Attention》_中提出了  **Cross-Segment 模型** 。该模型的核心是 **充分利用更长的上下文信息** ，同时提升预测效率。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/WZBibiatAX4xzYI49CsicFo6GcUOml2WibcEH5phiav9eep1zicesdQvxHMqpnVJ7mVVsaibQI7UqFB2x7t4lkXovdYPA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

在 cross-segment BERT 模型（a）中，向模型输入潜在文本分割点附近的局部上下文：左边 k 个tokens，右边 k 个tokens。在 BERT+Bi-LSTM 模型（b）中，首先使用 BERT 模型对每个句子进行编码，然后将句子表征输入到 Bi-LSTM 中。在 hierarchical BERT 模型（c）中，首先使用 BERT 对每个句子进行编码，然后将输出的句子表征输入另一个基于 transformer 的模型。Source: Text Segmentation by Cross Segment Attention.[6]

> 图 4（a）中的 cross-segment BERT 模型，将文本分割定义为逐句分类任务（译者注：sentence-by-sentence classification task，逐句判断其是否是文本分割点）。潜在文本分割点附近的局部上下文（两侧的 k 个 tokens）被输入到模型中。与 [CLS] 相对应的隐藏状态（hidden state）被传递给 softmax 分类器，由其决定是否在潜在断句处进行分割。

主要步骤：

1. **句子向量化** ：利用 BERT 模型分别获取每个句子的向量表示，保留句子的语义信息。
2. **跨段落预测** ：将连续的多个句子的向量表示同时输入到另一个 BERT 或 LSTM 模型中， **一次性预测每个句子是否为文本分段的边界** 。

这种方法的优势在于：

* **综合考虑上下文** ：通过同时处理多个句子，模型能够捕获更长距离的依赖关系，提高切分的准确性。
* **提高效率** ：相比逐一预测相邻句子的方式，批量处理多个句子能够提升计算效率。

#### **SeqModel 模型：自适应滑动窗口与上下文建模**

虽然 Cross-Segment 模型引入了更长的上下文，但在句子向量化过程中，仍是对每个句子**独立**进行的，未充分建模句子间的复杂依赖关系。为此，**Zhang 等人**在论文_《Sequence Model with Self-Adaptive Sliding Window for Efficient Spoken Document Segmentation》_中提出了  **SeqModel 模型** ，对文本切分方法进行了进一步改进。

SeqModel 的特点：

* **同时编码多个句子** ：利用 BERT 对连续的多个句子进行 **联合编码** ，直接建模句子间的依赖关系，获取包含上下文信息的句子表示。
* **预测切分边界** ：在获得句子的上下文表示后，模型预测每个句子后是否需要进行文本分割。
* **自适应滑动窗口** ：引入了**自适应滑动窗口**的方法，根据文本内容动态调整窗口大小， **在不牺牲准确性的前提下，加快推理速度** 。

这种方法不仅提高了切分的准确性，还使得模型在处理长文本时具有更高的效率。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_jpg/WZBibiatAX4xzYI49CsicFo6GcUOml2WibcE8QLfeR1UdPhFIMFLIxP7J4D7R3bwibddc9lEQ6OxSJxxLmTqL2dWv6A/640?wx_fmt=jpeg&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

#### **SeqModel 模型的应用与实现**

值得关注的是，SeqModel 模型的**预训练权重**已经在 **「魔搭社区（ModelScope）」** 上公开发布，支持中文文本的处理。这为开发者在实际项目中应用 SeqModel 模型，提供了便利的条件。

 **使用方法示例** ：

```
from modelscope.outputs import OutputKeys
from modelscope.pipelines import pipeline
from modelscope.utils.constant import Tasks

#初始化文本分割任务的pipeline
p = pipeline(task=Tasks.document_segmentation, model='damo/nlp_bert_document-segmentation_chinese-base')

#输入需要分割的长文本
documents = '这里输入您需要分割的长文本内容'

#执行文本分割
result = p(documents=documents)

#输出分割后的文本结果
print(result[OutputKeys.TEXT])
```

SeqModel 可通过 ModelScope[10] 框架使用。代码如下：

```
from modelscope.outputs import OutputKeys
from modelscope.pipelines import pipeline
from modelscope.utils.constant import Tasks

p = pipeline(
    task = Tasks.document_segmentation,
    model = 'damo/nlp_bert_document-segmentation_english-base'
)

print('-' * 100)

result = p(documents='We demonstrate the importance of bidirectional pre-training for language representations. Unlike Radford et al. (2018), which uses unidirectional language models for pre-training, BERT uses masked language models to enable pretrained deep bidirectional representations. This is also in contrast to Peters et al. (2018a), which uses a shallow concatenation of independently trained left-to-right and right-to-left LMs. • We show that pre-trained representations reduce the need for many heavily-engineered taskspecific architectures. BERT is the first finetuning based representation model that achieves state-of-the-art performance on a large suite of sentence-level and token-level tasks, outperforming many task-specific architectures. Today is a good day')

print(result[OutputKeys.TEXT])

```

在测试数据的末尾添加了一句 "Today is a good day（今天是个好日子）"，但文本分割处理后的 result 变量中并没有以任何方式将 "Today is a good day（今天是个好日子）" 分割开。

```
We demonstrate the importance of bidirectional pre-training for language representations.Unlike Radford et al.(2018), which uses unidirectional language models for pre-training, BERT uses masked language models to enable pretrained deep bidirectional representations.This is also in contrast to Peters et al.(2018a), which uses a shallow concatenation of independently trained left-to-right and right-to-left LMs.• We show that pre-trained representations reduce the need for many heavily-engineered taskspecific architectures.BERT is the first finetuning based representation model that achieves state-of-the-art performance on a large suite of sentence-level and token-level tasks, outperforming many task-specific architectures.Today is a good day

```

> 上述方法还有很大改进空间，建议的一种改进方法是创建专门针对某个项目或任务定制的训练数据，以便进行领域微，这样可以提高模型的性能。此外，优化模型架构也是一个改进点

基于深度学习模型的文本切分方法，利用了预训练模型对语言的深层次理解，从最初的朴素方法到 Cross-Segment，再到 SeqModel 模型的不断改进：

* **朴素方法** ：以句子为最小单位，利用 BERT 预测相邻句子的联系，但上下文考虑有限，效率较低。
* **Cross-Segment 模型** ：引入更长的上下文，批量预测切分点，提高了准确性和效率。
* **SeqModel 模型** ：同时编码多个句子，建模句子间的依赖关系，采用自适应滑动窗口，进一步提升性能。

这些方法的演进，体现了研究者们在文本切分领域的不懈探索和创新。通过选择合适的模型和方法，可以更精确地对文本进行切分，提高下游任务的效果，满足多样化的实际应用需求。

## 2.5 **LLM-based 方法**

这篇题为《 Dense X Retrieval: What Retrieval Granularity Should We Use? 》的论文介绍了一种新的检索单位，称为 proposition 。 **proposition 被定义为文本中的 atomic expressions** （译者注：不能进一步分解的单个语义元素，可用于构成更大的语义单位） **，用于检索和表达文本中的独特事实或特定概念，能够以简明扼要的方式表达，使用自然语言完整地呈现一个独立的概念或事实，不需要额外的信息来解释。**

那么，如何获取所谓的 proposition 呢？本文通过构建提示词并与 LLM 交互来获取。

LlamaIndex 和 Langchain 都实现了相关算法，下面将使用 LlamaIndex 进行演示。

LlamaIndex 的实现思路是使用论文中提供的提示词来生成 proposition ：

```
PROPOSITIONS_PROMPT = PromptTemplate(
    """Decompose the "Content" into clear and simple propositions, ensuring they are interpretable out of
context.
1. Split compound sentence into simple sentences. Maintain the original phrasing from the input
whenever possible.
2. For any named entity that is accompanied by additional descriptive information, separate this
information into its own distinct proposition.
3. Decontextualize the proposition by adding necessary modifier to nouns or entire sentences
and replacing pronouns (e.g., "it", "he", "she", "they", "this", "that") with the full name of the
entities they refer to.
4. Present the results as a list of strings, formatted in JSON.

Input: Title: ¯Eostre. Section: Theories and interpretations, Connection to Easter Hares. Content:
The earliest evidence for the Easter Hare (Osterhase) was recorded in south-west Germany in
1678 by the professor of medicine Georg Franck von Franckenau, but it remained unknown in
other parts of Germany until the 18th century. Scholar Richard Sermon writes that "hares were
frequently seen in gardens in spring, and thus may have served as a convenient explanation for the
origin of the colored eggs hidden there for children. Alternatively, there is a European tradition
that hares laid eggs, since a hare’s scratch or form and a lapwing’s nest look very similar, and
both occur on grassland and are first seen in the spring. In the nineteenth century the influence
of Easter cards, toys, and books was to make the Easter Hare/Rabbit popular throughout Europe.
German immigrants then exported the custom to Britain and America where it evolved into the
Easter Bunny."
Output: [ "The earliest evidence for the Easter Hare was recorded in south-west Germany in
1678 by Georg Franck von Franckenau.", "Georg Franck von Franckenau was a professor of
medicine.", "The evidence for the Easter Hare remained unknown in other parts of Germany until
the 18th century.", "Richard Sermon was a scholar.", "Richard Sermon writes a hypothesis about
the possible explanation for the connection between hares and the tradition during Easter", "Hares
were frequently seen in gardens in spring.", "Hares may have served as a convenient explanation
for the origin of the colored eggs hidden in gardens for children.", "There is a European tradition
that hares laid eggs.", "A hare’s scratch or form and a lapwing’s nest look very similar.", "Both
hares and lapwing’s nests occur on grassland and are first seen in the spring.", "In the nineteenth
century the influence of Easter cards, toys, and books was to make the Easter Hare/Rabbit popular
throughout Europe.", "German immigrants exported the custom of the Easter Hare/Rabbit to
Britain and America.", "The custom of the Easter Hare/Rabbit evolved into the Easter Bunny in
Britain and America." ]

Input: {node_text}
Output:"""
)

```

在前面的章节 “01 Embedding-based 方法” 中，已经安装了 LlamaIndex 0.10.12 的关键组件。但如果想要使用  DenseXRetrievalPack ，还需要运行 pip install llama-index-llms-openai 安装。安装完成后，当前的 LlamaIndex 相关组件如下所示：

```
(llamaindex_010) Florian:~ Florian$ pip list | grep llama
llama-index-core                    0.10.12
llama-index-embeddings-openai       0.1.6
llama-index-llms-openai             0.1.6
llama-index-readers-file            0.1.5
llamaindex-py-client                0.1.13

```

在 LlamaIndex 中，DenseXRetrievalPack 是一个需要单独下载的软件包。这里直接在测试代码中下载。测试代码如下：

```
from llama_index.core.readers import SimpleDirectoryReader
from llama_index.core.llama_pack import download_llama_pack

import os
os.environ["OPENAI_API_KEY"] = "YOUR_OPENAI_KEY"

# Download and install dependencies
DenseXRetrievalPack = download_llama_pack(
    "DenseXRetrievalPack", "./dense_pack"
)

# If you have already downloaded DenseXRetrievalPack, you can import it directly.
# from llama_index.packs.dense_x_retrieval import DenseXRetrievalPack

# Load documents
dir_path = "YOUR_DIR_PATH"
documents = SimpleDirectoryReader(dir_path).load_data()

# Use LLM to extract propositions from every document/node
dense_pack = DenseXRetrievalPack(documents)

response = dense_pack.run("YOUR_QUERY")

```

这段测试代码主要使用 DenseXRetrievalPack 类的构造函数。因此，有必要分析 DenseXRetrievalPack 类的源代码 [11]。

```
class DenseXRetrievalPack(BaseLlamaPack):
    def __init__(
        self,
        documents: List[Document],
        proposition_llm: Optional[LLM] = None,
        query_llm: Optional[LLM] = None,
        embed_model: Optional[BaseEmbedding] = None,
        text_splitter: TextSplitter = SentenceSplitter(),
        similarity_top_k: int = 4,
    ) -> None:
        """Init params."""
        self._proposition_llm = proposition_llm or OpenAI(
            model="gpt-3.5-turbo",
            temperature=0.1,
            max_tokens=750,
        )

        embed_model = embed_model or OpenAIEmbedding(embed_batch_size=128)

        nodes = text_splitter.get_nodes_from_documents(documents)
        sub_nodes = self._gen_propositions(nodes)

        all_nodes = nodes + sub_nodes
        all_nodes_dict = {n.node_id: n for n in all_nodes}

        service_context = ServiceContext.from_defaults(
            llm=query_llm or OpenAI(),
            embed_model=embed_model,
            num_output=self._proposition_llm.metadata.num_output,
        )

        self.vector_index = VectorStoreIndex(
            all_nodes, service_context=service_context, show_progress=True
        )

        self.retriever = RecursiveRetriever(
            "vector",
            retriever_dict={
                "vector": self.vector_index.as_retriever(
                    similarity_top_k=similarity_top_k
                )
            },
            node_dict=all_nodes_dict,
        )

        self.query_engine = RetrieverQueryEngine.from_args(
            self.retriever, service_context=service_context
        )

```

如代码所示，该构造函数的思路是首先使用 text_splitter 将文档划分为 `nodes`（译者注：将文档按照其最初的格式分割而成的最小单元。） ，然后调用 self._gen_propositions 生成 propositions 来获取相应的 `sub_nodes`（译者注：根据 nodes 生成的 propositions 所对应的文档片段或子集） 。然后，使用 nodes + sub_nodes 构建 VectorStoreIndex，并通过 RecursiveRetriever 进行检索。递归检索器（recursive retriever）可以通过处理文档的小数据块（small chunks）来找到所需信息，就像可以直接去书籍中的某个小节或段落查找一样。但是，如果在这些小数据块（small chunks）中找不到完整的信息，递归检索器（recursive retriever）会将相关的大数据块（larger chunks）传递到生成阶段（generation stage）进一步处理，就像在书中某个小节或段落查找资料时，如果需要更多信息，就会翻到相关的章节或整本书一样。

项目路径中只有一份 BERT 论文文档。通过调试，发现 sub_nodes[].text 的内容并非原始文本，里面的内容已经被改写过了：

```
> /Users/Florian/anaconda3/envs/llamaindex_010/lib/python3.11/site-packages/llama_index/packs/dense_x_retrieval/base.py(91)__init__()
     90 
---> 91         all_nodes = nodes + sub_nodes
     92         all_nodes_dict = {n.node_id: n for n in all_nodes}


ipdb> sub_nodes[20]
IndexNode(id_='ecf310c7-76c8-487a-99f3-f78b273e00d9', embedding=None, metadata={}, excluded_embed_metadata_keys=[], excluded_llm_metadata_keys=[], relationships={}, text='Our paper demonstrates the importance of bidirectional pre-training for language representations.', start_char_idx=None, end_char_idx=None, text_template='{metadata_str}\n\n{content}', metadata_template='{key}: {value}', metadata_seperator='\n', index_id='8deca706-fe97-412c-a13f-950a19a594d1', obj=None)
ipdb> sub_nodes[21]
IndexNode(id_='4911332e-8e30-47d8-a5bc-ed7cbaa8e042', embedding=None, metadata={}, excluded_embed_metadata_keys=[], excluded_llm_metadata_keys=[], relationships={}, text='Radford et al. (2018) uses unidirectional language models for pre-training.', start_char_idx=None, end_char_idx=None, text_template='{metadata_str}\n\n{content}', metadata_template='{key}: {value}', metadata_seperator='\n', index_id='8deca706-fe97-412c-a13f-950a19a594d1', obj=None)
ipdb> sub_nodes[22]
IndexNode(id_='83aa82f8-384a-4b06-92c8-d6277c4162bf', embedding=None, metadata={}, excluded_embed_metadata_keys=[], excluded_llm_metadata_keys=[], relationships={}, text='BERT uses masked language models to enable pre-trained deep bidirectional representations.', start_char_idx=None, end_char_idx=None, text_template='{metadata_str}\n\n{content}', metadata_template='{key}: {value}', metadata_seperator='\n', index_id='8deca706-fe97-412c-a13f-950a19a594d1', obj=None)
ipdb> sub_nodes[23]
IndexNode(id_='2ac635c2-ccb0-4e62-88c7-bcbaef3ef38a', embedding=None, metadata={}, excluded_embed_metadata_keys=[], excluded_llm_metadata_keys=[], relationships={}, text='Peters et al. (2018a) uses a shallow concatenation of independently trained left-to-right and right-to-left LMs.', start_char_idx=None, end_char_idx=None, text_template='{metadata_str}\n\n{content}', metadata_template='{key}: {value}', metadata_seperator='\n', index_id='8deca706-fe97-412c-a13f-950a19a594d1', obj=None)
ipdb> sub_nodes[24]
IndexNode(id_='e37b17cf-30dd-4114-a3c5-9921b8cf0a77', embedding=None, metadata={}, excluded_embed_metadata_keys=[], excluded_llm_metadata_keys=[], relationships={}, text='Pre-trained representations reduce the need for many heavily-engineered task-specific architectures.', start_char_idx=None, end_char_idx=None, text_template='{metadata_str}\n\n{content}', metadata_template='{key}: {value}', metadata_seperator='\n', index_id='8deca706-fe97-412c-a13f-950a19a594d1', obj=None)

```

sub_nodes 和 nodes 之间的关系如图 7 所示，这个索引结构按照 small-to-big 的方式进行排序组织。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_jpg/WZBibiatAX4xzYI49CsicFo6GcUOml2WibcEOf6BKGjuctBm1Dg06LDoicibtbrdImlic7hICicuniaLe71Ga36jg6tueVg/640?wx_fmt=jpeg&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

图 7：按照 small-to-big 的方式进行排序组织的索引结构。

这种索引结构是通过 self._gen_propositions[12] 构建的，代码如下：

```
 async def _aget_proposition(self, node: TextNode) -> List[TextNode]:
 """Get proposition."""
        inital_output = await self._proposition_llm.apredict(
            PROPOSITIONS_PROMPT, node_text=node.text
 )
        outputs = inital_output.split("\n")

        all_propositions = []

 for output in outputs:
 if not output.strip():
 continue
 if not output.strip().endswith("]"):
 if not output.strip().endswith('"') and not output.strip().endswith(
 ","
 ):
                    output = output + '"'
                output = output + " ]"
 if not output.strip().startswith("["):
 if not output.strip().startswith('"'):
                    output = '"' + output
                output = "[ " + output

 try:
                propositions = json.loads(output)
 except Exception:
 # fallback to yaml
 try:
                    propositions = yaml.safe_load(output)
 except Exception:
 # fallback to next output
 continue

 if not isinstance(propositions, list):
 continue

            all_propositions.extend(propositions)

 assert isinstance(all_propositions, list)
        nodes = [TextNode(text=prop) for prop in all_propositions if prop]

 return [IndexNode.from_text_node(n, node.node_id) for n in nodes]

 def _gen_propositions(self, nodes: List[TextNode]) -> List[TextNode]:
 """Get propositions."""
        sub_nodes = asyncio.run(
            run_jobs(
 [self._aget_proposition(node) for node in nodes],
                show_progress=True,
                workers=8,
 )
 )

 # Flatten list
 return [node for sub_node in sub_nodes for node in sub_node]

```

对每个原始节点（original node），都异步调用 self._aget_proposition，通过 PROPOSITIONS_PROMPT 获取 LLM 返回的 inital_output，然后根据 inital_output 获取 propositions 并构建 TextNode。最后，将这些 TextNode 与原始节点（original node）关联起来，即使用 [IndexNode.from_text_node(n, node.node_id) for n in nodes] 。

有一件事需要多说一句：原论文使用 LLM 生成的 propositions 作为训练数据，微调了一个文本生成模型。这个文本生成模型目前是公开可访问的 [13]，总体来看， **这种利用 LLM 构建 propositions 的数据分块方法能够实现更精细的数据分块。** 能够与原始节点（original node）构成一个 small-to-big 的索引结构，从而为 semantic chunking（译者注：将文本分成具有语义相关性的片段。） 方法提供了一种新思路。

# 3. 文本分块的优化策略

## 3.1 保持语义完整性

### **3.1.1 避免句子拆分**

在文本切分过程中，应尽量避免将句子拆分。 **句子是表达完整语义的基本单位** ，拆分句子可能导致语义破碎，影响向量化表示的准确性和模型对文本的理解。例如，句子中包含的主谓宾结构或修饰关系在被截断后，会失去原有的含义，使得模型难以准确捕捉文本的核心内容。

 **实践建议** ：

* **按标点切分** ：使用句号、问号、感叹号等标点符号作为切分点，确保每个文本块包含完整的句子。
* **分割符的优先级** ：在设定分割符时，将句子结束符号置于高优先级，保证切分过程中优先考虑句子边界。

#### **3.1.2 考虑段落关联性**

除了句子层面， **段落也是表达完整思想的重要单位** 。段落内的句子通常围绕一个主题展开，具有紧密的逻辑关联。将逻辑关联紧密的段落拆分到不同的文本块，可能导致上下文割裂，影响模型对整体语义的把握。

 **实践建议** ：

* **段落完整切分** ：尽量将同一段落的内容保留在同一个文本块中，避免因切分导致的语义断裂。
* **结合主题分割** ：对于长段落，可以根据主题或语义转折点进行切分，确保每个文本块内部主题统一。

## 3.2 控制文本块长度

### **3.2.1 设定合理的长度阈值**

文本块的长度对向量化模型和大语言模型（LLM）的处理性能有直接影响。过长的文本块可能导致：

* **向量表示稀释** ：重要的语义信息被淹没，降低检索精度。
* **模型输入限制** ：超过大模型的输入长度，无法处理全部内容。

过短的文本块则可能缺乏上下文，导致语义不完整。

 **实践建议** ：

* **评估模型性能** ：根据向量化模型和大模型对不同长度文本的处理效果，设定适当的文本块长度阈值。
* **常用长度参考** ：通常，文本块长度可以设定为 200～500 个字符，根据具体情况调整。

### **3.2.2 动态调整**

不同类型的文本对文本块长度的要求可能不同。灵活调整长度阈值，可以更好地适应多样化的文本内容。

 **实践建议** ：

* **文本类型分析** ：针对新闻、法律文档、技术手册等不同类型的文本，分析其结构特点，设定合适的长度。
* **自适应切分** ：开发动态调整机制，根据文本内容和结构实时调整文本块长度，实现个性化处理。

## 3.3 重叠切分

### **3.3.1 方法**

**重叠切分**是指在文本块之间引入一定的重叠部分，使相邻文本块共享部分内容。具体方法是：

* **设定重叠长度** ：如设定每个文本块之间有 50 个字符的重叠。
* **滑动窗口切分** ：采用滑动窗口的方式切分文本，每次移动的步长小于窗口大小。

### **3.3.2 优点**

* **保留上下文连接** ：重叠部分使得相邻文本块之间的上下文信息得到保留，避免因切分导致的语义断裂。
* **增强模型理解** ：大模型在生成回答时，能够参考前后文信息，产生更加连贯和准确的结果。

 **实践建议** ：

* **合理设定重叠量** ：根据文本特点和模型需求，设定适当的重叠长度，既保留上下文，又不增加过多的冗余信息。
* **效率考虑** ：注意重叠切分会增加文本块的数量，需权衡处理效率和上下文保留之间的关系。

## 3.4 结合向量化模型性能

### **3.4.1 适配模型特性**

不同的向量化模型（如 BERT、GPT、Sentence Transformers）在处理文本长度和语义信息方面有不同的表现。

 **实践建议** ：

* **模型特性研究** ：深入了解所使用向量化模型的特点，对其在不同文本长度下的性能进行评估。
* **选择合适策略** ：根据模型的优势，选择短文本或长文本的分块策略。例如，如果模型对短文本的向量表示效果更好，则应倾向于较短的文本块。

### **3.4.2 优化向量表示**

针对长文本可能出现的**语义稀释**问题，可以采用更高级的向量表示方法。

 **实践建议** ：

* **加权平均** ：对文本中的重要词汇（如关键词、专有名词）赋予更高权重，增强它们在向量表示中的影响。
* **注意力机制** ：利用注意力机制（Attention）聚焦文本中的关键部分，生成更具代表性的向量。
* **分层编码** ：对文本进行层次化编码，先编码句子，再编码段落，逐层组合，保持语义结构。

## 3.5 考虑大模型的输入限制

### **3.5.1 输入长度控制**

大语言模型对输入文本的长度有严格限制（如 GPT-3 的最大输入长度为 2048 个标记）。在召回阶段，需要确保选取的文本块总长度不超过模型的限制。

 **实践建议** ：

* **统计文本块长度** ：在将文本块输入模型前，计算其总长度，确保不超出模型限制。
* **截断策略** ：如长度超出限制，可考虑截断低相关性的内容，保留核心文本块。

### **3.5.2 优先级排序**

不同的文本块对回答的贡献度不同，应根据其与查询的相关性进行排序，优先输入最重要的内容。

 **实践建议** ：

* **相关性评分** ：为每个召回的文本块计算与查询的相关性得分。
* **择优选择** ：根据相关性得分，从高到低选择文本块，直至达到模型的输入长度限制。

### **3.5.3 内容精炼**

当重要的文本块过长时，可以对其进行压缩或摘要，确保核心信息得以保留。

 **实践建议** ：

* **自动摘要** ：利用摘要模型对长文本块生成精简的摘要。
* **关键句提取** ：提取文本块中最能代表核心内容的句子，供大模型参考。

优化文本分块策略需要**综合考虑语义完整性、文本块长度、向量化模型性能以及大模型的输入限制**等因素。通过避免句子拆分、保持段落关联、引入重叠切分、适配模型特性和合理控制输入长度，可以有效提升 RAG 系统的检索效果和大模型回答的质量。

# 4. 前沿方法推荐

## 4.1 Meta-Chunking （2024.8.16）

Meta-Chunking旨在优化检索增强生成（RAG）中的文本块处理，从而提升知识密集型任务的质量。研究提出在句子和段落之间引入一种新的粒度，即Meta-Chunking，它由段落内具有深层语言逻辑联系的多个句子组成。为实现这一概念，研究者设计了基于困惑度（PPL）的分割方法，该方法在性能和速度之间取得平衡，并能精确确定文本块的边界。同时，考虑到不同文本的复杂性，研究还提出了结合PPL分割和动态合并的策略，以实现细粒度和粗粒度文本分割之间的平衡。在11个数据集上的实验表明，Meta-Chunking能有效提升基于RAG的单跳和多跳问答性能。此外，研究还发现PPL分割在不同规模和类型的模型上表现出显著的灵活性和适应性

* 标题：Meta-Chunking: Learning Efficient Text Segmentation via Logical Perception，
* github:https://github.com/IAAR-Shanghai/Meta-Chunking/tree/386dc29b9cfe87da691fd4b0bd4ba7c352f8e4ed

基于一个核心原则：允许块大小变化，以更有效地捕获和维护内容的逻辑完整性。这种动态的粒度调整可确保每个分段块都包含完整且独立的思想表达，从而避免在分段过程中逻辑链中断。这不仅增强了文档检索的相关性，还提高了内容清晰度。

如下图所示，例句呈现出递进关系，但语义相似度较低，可能导致例句完全分离。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/WZBibiatAX4xzYI49CsicFo6GcUOml2WibcE1gbTJ58lV9jTPdzwLB1tmG9ZAXNcam76KeIsMb9HRicOw0E6Q7ia3FPQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/WZBibiatAX4xzYI49CsicFo6GcUOml2WibcEfRKicYbHxeeoUeDN3lglQysb997QU05Mza2oSWaI2LpBiaLlKnbFYr8w/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

* 第一种分块，称为Margin Sampling Chunking，大概思路是让LLM来做二分类，大模型输出是个词表的概率分布，这里他们做了一个对“是” 、 “否”的概率差，判断是否符合阈值。
* 第二种分块，称为Perplexity Chunking，计算每个句子在上下文下的困惑度（如果困惑度高，说明模型对这段文本比较懵逼，所以不建议切分）。每次找到序列中困惑度最小的句子，并且如果这个句子前后2句都小于当前这个句子，那就可以切分了。算困惑度可以利用固定长度的kv-cache，来保证显存问题

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/WZBibiatAX4xzYI49CsicFo6GcUOml2WibcE2jJiaWIcnfr49KtVPZ53RKsxb4ak7n4dtULjSX8nkIPbibnWxTCrcgDQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/WZBibiatAX4xzYI49CsicFo6GcUOml2WibcEVHxK5uUhE1ib80y81OZ4UsOMUSr2tbQWpb1np0Okym35kS1iaKptPtZQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

Margin Sampling Chunking:

1. 将文本分割成一系列句子。
2. 对于相邻的句子，使用 LLM 进行二元分类，判断是否需要分割。
3. LLM 输出两个选项的概率，计算概率差异 Margin。
4. 将 Margin 与预设阈值进行比较，如果 Margin 大于阈值，则分割句子；否则，合并句子。

Perplexity Chunking:

1. 将文本分割成一系列句子。
2. 使用 LLM 计算每个句子基于其上下文的 PPL 值。
3. 分析 PPL 值的分布特征，识别潜在的文本块边界（即 PPL 值的局部最小值）。
4. 将句子分割成多个文本块，每个文本块包含一个或多个连续的句子。

* 两种策略的优缺点：

  * 优点：分割结果更加客观，效率更高，并能够有效地捕捉文本的逻辑结构。
  * 缺点：需要分析 PPL 值的分布特征，可能需要一定的计算量。
  * 优点：可以有效地降低对模型规模的需求，使得小模型也能胜任文本分块任务。
  * 缺点：分割结果可能受到 LLM 模型的影响，且效率相对较低。
  * Margin Sampling Chunking:
  * Perplexity Chunking:

## 4.2 Late Chunking（jina 2024.8.4）

* 论文Late Chunking: Contextual Chunk Embeddings Using Long-Context Embedding Models：https://arxiv.org/abs/2409.04701
* github：https://github.com/jina-ai/late-chunking
* jina博客：https://jina.ai/news/late-chunking-in-long-context-embedding-models/
* 项目链接：https://colab.research.google.com/drive/15vNZb6AsU7byjYoaEtXuNu567JWNzXOz?usp=sharing

想象一下,你在整理一大堆文件。传统的方法是先把文件分成小堆,然后再逐一处理。而后期分块却反其道而行之:先对整个文件进行全面处理,然后再进行分类整理。这听起来可能有点反直觉,但在AI的世界里,这种方法却显示出了惊人的效果。具体来说,后期分块是这样工作的:

1. 先用一个长上下文嵌入模型(比如 jina-embeddings-v2-small-en，具有 8K 上下文长度)"读懂"整个文档
2. 然后再把这个"理解"切成小块

> Late Chunking is now available in jina-embeddings-v3 API. 8192-token

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/WZBibiatAX4xzYI49CsicFo6GcUOml2WibcELRJbFibPUcOyWHTqnakiaSRakm88XhoEYEVoXuZIEeNCTZ51nNCzGr7w/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

传统方法涉及使用句子、段落或最大长度限制来预先分割文本。之后，会反复应用嵌入模型到这些结果块中。为了为每个块生成一个嵌入，许多嵌入模型会在这些词级别嵌入上使用均值池化，以输出一个单一的嵌入向量。 而“后期分块”方法首先将嵌入模型的转换层应用于整个文本或尽可能多的文本。这为文本中的每个标记生成了一个包含整个文本信息的向量表示序列。随后，对这个标记向量序列的每一部分应用平均池化，产生考虑整个文本上下文的每个部分的嵌入。与生成独立同分布（i.i.d.）分块嵌入的简单编码方法不同，**后期分块创建了一组分块嵌入，其中每个嵌入都“依赖于”之前的嵌入，从而为每个分块编码了更多的上下文信息。**

为了解决这个问题，利用了最近的嵌入模型可以处理的长输入序列jina-embeddings-v2-base-en。这些模型支持更长的输入文本，例如 8192 个标记jina-embeddings-v2-base-en或大约十页标准文本。这种大小的文本段不太可能具有只能通过更大的上下文来解决的上下文依赖关系。然而，仍然需要更小的文本块的向量表示，部分原因是 LLM 的输入大小有限，但主要是因为短嵌入向量的信息容量有限。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/WZBibiatAX4xzYI49CsicFo6GcUOml2WibcEHg4zWu7yp2J9BAj5iaaaAScMMDbxP2EWY6Wl8fetlpB5o1Uia9OTfsAg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/WZBibiatAX4xzYI49CsicFo6GcUOml2WibcEYbuiaTD3I0PIoYhWCNKeGgrFjMLpRU7cCD948fCiaRsSROzhTfBSiaicug/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

简单的编码方法（如上图左侧所示）在处理文本之前先对文本进行分块，使用句子、段落和最大长度限制来先验地分割文本，然后将嵌入模型应用于生成的块。而后期分块则首先将嵌入模型的转换器部分应用于整个文本或尽可能大的部分。这会为每个标记生成一个向量表示序列，其中包含来自整个文本的文本信息。为了为文本生成单个嵌入，许多嵌入模型会将这些标记表示应用于均值池化以输出单个向量。而后期分块则将均值池化应用于这个标记向量序列的较小段，从而为每个块生成考虑整个文本的嵌入。

| Text                                                                                                                                  | Similarity Traditional | Similarity Late Chunking |
| ------------------------------------------------------------------------------------------------------------------------------------- | ---------------------- | ------------------------ |
| Berlin is the capital and largest city of Germany, both by area and by population."                                                   | 0.84862185             | 0.849546                 |
| Its more than 3.85 million inhabitants make it the European Union's most populous city, as measured by population within city limits. | 0.7084338              | 0.82489026               |
| The city is also one of the states of Germany, and is the third smallest state in the country in terms of area.                       | 0.7534553              | 0.84980094               |

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/WZBibiatAX4xzYI49CsicFo6GcUOml2WibcEnLy2St52g904ULEYmyL4VnkkeKQNypGT9tT7NtbgxmeFVn9bBldGyw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

为了通过几个简单的示例验证此方法的有效性，使用BeIR中的一些检索基准对其进行了测试。这些检索任务包括一个查询集、一个文本文档语料库和一个 QRels 文件，该文件存储了与每个查询相关的文档 ID 的信息。为了识别查询的相关文档，可以对文档进行分块，将其编码为嵌入索引，并确定每个查询嵌入的最相似块 (kNN)。由于每个块对应一个文档，因此可以将块的 kNN 排名转换为文档的 kNN 排名（对于在排名中多次出现的文档，仅保留第一次出现的文档）。之后，可以将结果排名与对应于真实 QRels 文件的排名进行比较，并计算 nDCG@10 等检索指标。

| Dataset   | AVG Document Length (characters) | Traditional Chunking (nDCG@10) | Late Chunking (nDCG@10) | No Chunking (nDCG@10) |
| --------- | -------------------------------- | ------------------------------ | ----------------------- | --------------------- |
| SciFact   | 1498.4                           | 64.20%                         | **66.10%**        | 63.89%                |
| TRECCOVID | 1116.7                           | 63.36%                         | 64.70%                  | **65.18%**      |
| FiQA2018  | 767.2                            | 33.25%                         | **33.84%**        | 33.43%                |
| NFCorpus  | 1589.8                           | 23.46%                         | 29.98%                  | **30.40%**      |
| Quora     | 62.2                             | 87.19%                         | 87.19%                  | 87.19%                |

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/WZBibiatAX4xzYI49CsicFo6GcUOml2WibcEwQJ1oNvBakicQWYXs8jE2pgsv99ASHvp8osMJrRcgfHqCjg2cYap7vQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

文档越长，后期分块策略就越有效。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/WZBibiatAX4xzYI49CsicFo6GcUOml2WibcE78TubV87Q1KUibpMPZXVzLiaic4RrCTswOJMYGZKySr3eOR1wv7SM6W2w/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

后期分块不需要对嵌入模型进行额外的训练。它可以应用于任何使用均值池的长上下文嵌入模型

Late Chunking is now available in jina-embeddings-v3 API. 8192-token

## 4.3 Anthropic (2024.9.20)

链接：https://www.anthropic.com/news/contextual-retrieval

Anthropic 引入了一种称为上下文检索的单独策略。 Anthropic 的方法是一种解决上下文丢失问题的强力方法，其工作原理如下：

* 每个块都会与完整的文档一起发送给LLM 。
* LLM为每个块添加相关上下文。
* 这会产生更丰富、信息更丰富的嵌入。

> 这本质上是上下文丰富，其中全局上下文使用LLM显式硬编码到每个块中，这在成本、时间和存储方面都很昂贵。此外，尚不清楚这种方法是否能够适应块边界，因为LLM依赖于准确且可读的块来有效地丰富上下文

## 4.4  Small Language Model, SLM--qwen0.5

* simple-qwen-0.5: 这是最简单的模型，主要根据文档的结构元素识别边界。它简单高效，适合基本的分块需求。https://huggingface.co/jinaai/text-seg-lm-qwen2-0.5b
* topic-qwen-0.5: 这个模型受到了思维链 (Chain-of-Thought) 推理的启发，它会先识别文本中的主题，比如“第二次世界大战的开始”，然后用这些主题来定义分块边界。它能保证每个文本块在主题上保持连贯，特别适合处理复杂的多主题文档。初步测试表明，它分块的效果很不错，很接近人类的直觉。https://huggingface.co/jinaai/text-seg-lm-qwen2-0.5b-cot-topic-chunking
* summary-qwen-0.5: 这个模型不仅能识别文本边界，还能给每个文本块生成摘要。在 RAG 应用里，文本块摘要很有用，尤其是在长文档问答这种任务上。当然，代价就是训练时需要更多的数据。https://huggingface.co/jinaai/text-seg-lm-qwen2-0.5b-summary-chunking

> 这三个模型都只返回文本块的头部，也就是每个文本块的截断版本。它们不生成完整的文本块，而是输出关键点或子主题。简单来说，因为它只提取关键点或子主题，这就相当于抓住了片段的核心意思，通过文本的语义转换，来更准确地识别边界，保证文本块的连贯性。检索的时候，会根据这些 “文本块头部” 把文档文本切片，然后再根据切片结果重建完整的片段。相当于先用 “文本块头部” 做了个索引，需要的时候再根据索引找到对应的完整片段。这样既能保证检索的精准度，又能提高效率，避免处理过多的冗余信息。

接着，对数据添加了噪声，包括打乱数据、插入随机字符 / 单词 / 字母、随机删除标点符号以及去除所有换行符。

这些数据增强方法虽然有一定效果，但仍有局限性，的最终目标是使模型能够生成连贯的文本，并且正确处理代码片段等结构化内容。

所以还使用了 GPT-4o 生成代码、公式和列表来进一步丰富数据集，以提升模型处理这些元素的能力。

* 训练设置

  训练模型过程中，的设置如下：

  * **框架** ：用了 Hugging Face 的 `transformers` 库，还集成了 `Unsloth` 来优化模型。这对于优化内存使用和加速训练非常重要，让可以用大型数据集高效地训练小型模型。
  * **优化器和调度器** ：用了 AdamW 优化器，加上线性学习率调度策略和 warmup  机制，可以帮在训练初期稳定训练过程。
  * **实验跟踪** ：用 Weights & Biases 跟踪所有训练实验，并记录了训练损失、验证损失、学习率变化、整体模型性能等等关键指标。这种实时追踪可以让了解模型的进展情况，在需要时快速调整参数，获得最好的学习效果。
* 训练过程

  是用 `qwen2-0.5b-instruct` 作为基础模型，用 `Unsloth` 训练了三个 SLM 变体，每个变体对应不同的分块策略。训练数据来自 wiki727k，除了文章全文外，还包含了先前 “数据增强” 步骤中提取的章节、主题和摘要等信息。

  1. `simple-qwen-0.5`：用了 10,000 个样本，训练了 5,000 步，很快就收敛了，可以有效地检测文本连贯分块之间的边界。训练损失是 0.16。
  2. `topic-qwen-0.5`：跟 `simple-qwen-0.5` 类似，也用了 10,000 个样本，训练了 5,000 步，训练损失是 0.45。
  3. `summary-qwen-0.5`：用了 30,000 个样本，训练了 15,000 步。这个模型很有潜力，但是训练损失比较高（0.81），说明还需要更多数据，大概两倍的原始样本数量才能完全发挥它的实力。

# 5. 不同分块策略的效果对比

## 5.1 jina vs qwen0.5

来看一下不同分块策略的效果。下面是每种策略生成的三个连续分块示例，还有 Jina 的 Segmenter API 的结果。为了生成这些文本块，先用 Jina Reader 从 Jina AI 博客抓取了一篇文章的纯文本（包括页眉、页脚等所有页面数据），然后分别用不同的分块方法处理。

> https://jina.ai/news/can-embedding-reranker-models-compare-numbers/

* Jina Segmenter API

Jina Segmenter API 的文本分块非常细粒度，它会根据 `\n`、`\t` 等字符进行分块，所以切出来的文本块通常都很小。只看前三个分块的话，它从网站的导航栏提取了 `search\\n`、`notifications\\n` 和 `NEWS\\n`，但完全没有提取到和文章内容相关的东西：

再往后，总算能看到一些博客文章的内容了，但每个分块保留的上下文信息都很少：

* simple-qwen-0.5

`simple-qwen-0.5` 会根据语义结构把博客文章分成更长的文本块，每个文本块的含义都比较连贯：

* `topic-qwen-0.5`

`topic-qwen-0.5` 会先根据文档内容识别主题，然后再根据这些主题进行分块：

* `summary-qwen-0.5`

`summary-qwen-0.5` 不仅能识别文本块边界，还会为每个文本块生成摘要：

## 5.2 later chunk

Weaviate 研究人员做了一系列测试， 结果相当惊人。我们以 weaviate 博客文章的一段内容来测试后期分块与朴素分块。

使用固定大小分块策略（token 数 = 128）导致以下句子被分成了两个不同的分块：

> Weaviate 的原生、多租户架构为需要在保持快速检索和准确性的同时优先考虑数据隐私的客户提供优势。

如下：

| 分块   | 内容                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| ------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| chunk1 | ... 技术堆栈的演变。这种灵活性，结合易于使用，帮助团队更快地将 AI 原型部署到生产环境中。在架构方面，灵活性也至关重要。不同的应用场景有不同的需求。例如，我们与许多软件公司以及在受监管行业中运营的公司合作。他们通常需要多租户功能来隔离数据并保持合规性。在构建检索增强生成（RAG）应用程序时，使用特定于帐户或用户的数据显示结果的上下文，数据必须保留在为其用户组专用的租户内。Weaviate 的原生、多租户架构对于需要优先考虑此类需求的客户来说表现卓越。 |
| chunk2 | .. 在保持数据隐私的同时，实现快速检索和准确性。另一方面，我们支持一些非常大规模的单租户使用案例，侧重于实时数据访问。这些案例中的许多都涉及电子商务和以速度和客户体验为竞争点的行业。                                                                                                                                                                                                                                                                    |

假设有人问:"客户最关心什么?"

| 方法     | AI 的回答                                                                                                                                                                                                                                                                                                                                                                                           |
| -------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 传统方法 | 1. "产品更新, 参加我们的网络研讨会。" (得分: 75.6)2. "在保证速度和准确性的同时注重数据隐私。我们支持一些大规模单租户用例, 主要在电子商务和那些速度和客户体验至关重要的行业。" (得分: 70.1)                                                                                                                                                                                                          |
| 后期分块 | 1. "客户需求多样, 我们引入了不同的存储层。看到客户的产品越来越受欢迎真是令人惊叹。但随着用户增长, 成本可能会急剧上升..." (得分: 74.8)2. "技术选择至关重要。灵活性和易用性有助于团队更快地实施 AI。不同的用例有不同的需求。例如, 软件公司和受监管行业通常需要多租户来隔离数据并确保合规。在构建 AI 应用时, 使用特定用户数据来个性化结果很重要。Weaviate 的多租户架构在这方面表现出色。" (得分: 68.9) |

显然, 后期分块方法给出的答案更加全面和贴切, 更能抓住问题的核心。

* colab链接: https://colab.research.google.com/drive/15vNZb6AsU7byjYoaEtXuNu567JWNzXOz?usp=sharing&ref=jina-ai-gmbh.ghost.io_
* 测试链接: https://weaviate.io/blog/late-chunking_
* 文章: https://weaviate.io/blog/launching-into-production_

# 6. 结合业务场景与文本特点选择合适分块策略

* 不同应用需求

  不同的业务场景对文本处理有着独特的需求和挑战。 **在选择文本分块策略时，首先要充分理解具体应用的目标和要求** 。

  * **专业领域应用** ：如在法律、医学等专业领域，文本往往包含大量的术语和复杂的句法结构。此时，需要确保文本分块能够保留专业术语的完整性和上下文关联，避免因切分不当导致语义误解。
  * **长文档处理** ：对于需要处理长篇文档的应用，如技术文档、研究报告，应采用能够保持段落和章节结构的分块策略，确保大模型能够理解文档的整体逻辑和主题发展。
  * **实时响应场景** ：在聊天机器人、智能客服等需要实时响应的场景中，文本分块策略需要兼顾处理速度和语义完整性，可能需要在分块大小和计算资源之间寻找平衡。
* 分析文本的内在结构

  **不同类型的文本具有不同的结构特征** ，在选择分块策略时，应充分考虑这些特性。

  * **结构化文本** ：如 HTML、Markdown 格式的文本，具有明确的标签和层次结构。采用针对性的分块方法（如 `HTMLHeaderTextSplitter`、`MarkdownTextSplitter`），能够保留文本的结构信息，提升模型的理解能力。
  * **非结构化文本** ：对于散文、对话等非结构化文本，可考虑基于语义的分块方法，利用自然语言处理技术识别句子边界、主题变化点，确保语义的完整性和连贯性。
  * **多语言文本** ：在处理包含多种语言的文本时，需要选择支持相应语言特性的分块工具，并可能需要针对不同语言调整分块参数。

1. 多方案对比

   **实践中，应对不同的文本分块策略进行实验验证** ，以确定最适合特定应用场景的方法。

   * **策略多样性** ：尝试固定大小切分、基于标点的切分、重叠切分、语义切分等多种策略。
   * **工具选择** ：使用不同的文本切块工具（如 `RecursiveCharacterTextSplitter`、`NLTKTextSplitter`、`SpacyTextSplitter`）进行比较，了解其在具体应用中的表现。
2. 制定评估指标

   为客观评估不同分块策略的效果， **需要制定明确的评估指标** 。

   * **检索性能** ：评估向量化和检索阶段的效果，如召回率、准确率、平均检索时间等。
   * **生成质量** ：评估大模型生成回答的准确性、完整性、连贯性，以及与用户查询的相关性。
   * **用户反馈** ：收集用户对系统回答的满意度评价，作为质量评估的重要参考。
3. 数据驱动的优化

   **基于实验数据和评估结果，对分块策略进行优化** 。

   * **参数调优** ：根据实验结果，调整文本块长度、重叠长度、分割符等参数，寻求最佳配置。
   * **问题定位** ：通过分析模型错误案例，找出分块策略中导致性能下降的原因，针对性地改进。
   * **渐进式优化** ：采用迭代的方法，逐步改进分块策略和参数设置，每次优化后进行验证，确保优化方向正确。
