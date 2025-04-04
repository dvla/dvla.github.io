---
author: "Nigel Brookes-Thomas"
title: "Experimenting with Retrieval Augmented Generation (RAG) with LLMs"
description: "Giving an LLM some external context with Ruby."
draft: false
date: 2025-04-04
tags: ["Ruby, AI"]
categories: ["Ruby, AI"]
ShowToc: true
TocOpen: true
---

I want to be able to ask an generative AI some questions while giving it the context from which I'd like it to use it's smarts to derive an answer. This is Retrieval-Augmented Generation (RAG).

Being a Ruby engineer, I'm going to pick up my shiny red hammer to attack this problem.

## Running a local LLM

I'm going to run my LLM locally. I'm on a Mac, so I'm wanting the `llama.cpp` library installed. I'm going to need this for some dependencies later.

```bash
brew install llama_cpp
```

Because I'm super lazy and want to experiment by hand, I'm using the open source [Jan](https://jan.ai) to host my model and, really conveniently, it can serve an Open AI compatible API. With Jan, I can easily pick and choose from a variety of different models or load my own.

## Chatting to the LLM

At the begining of this experiment, I wasn't sure which model I wanted to use or how to interface with it so I used the [`langchainrb`](https://github.com/patterns-ai-core/langchainrb) library which provides a high-level, pluggable interface.

I do need to install the [Open AI](https://rubygems.org/gems/ruby-openai) as well.

Then I'm going to configure the Open AI library to use my local server rather than the internet. When I create a LLM client, I need to tell it which model I'm going to use. Since Jan can only load one model, I'm going to use the same one for all interations.

```ruby
require 'langchain'
require 'openai'

# logs are a bit chatty by default
Langchain.logger.level = Logger::ERROR

MODEL = 'llama3.2-3b-instruct'

OpenAI.configure do |c|
  c.uri_base = 'http://127.0.0.1:1337/v1'
end

llm = Langchain::LLM::OpenAI.new(api_key: 'locally-model-no-api-key',
                                 default_options:{
                                   chat_model: MODEL,
                                   completion_model: MODEL,
                                   embedding_model: MODEL } )
```

I also want an assistant client. An assistant stores context to make conversational interations more natural. I'm going to pass a block into the constructor which will be called as the response is streamed rather than wait until a complete result is received because I just want to print the response to the console as it is generated.

```Ruby
assistant = Langchain::Assistant.new(
  llm:,
  instructions: <<~EO_PROMPT
    You are a very skilled and helpful assistant on the HR rules at the DVLA in the UK. You are able to find answers to the questions from the contextual passage snippets provided. Provide as much detail as you can.
  EO_PROMPT
  ) do |response_chunk|
    print response_chunk.dig('delta', 'content')
  end
```


## Collecting my own data

I need to turn my unstructured source information into something a machine can deal with. For this experiment, I actually used some of our HR policies: they're wordy, somewhat complex and the documents can be easily converted from Word to Markdown. I'm going to use Markdown section headers to identify coherent sections of text, something which works for these documents.

I need a vector database to store this in. I've picked [Milvus](https://milvus.io), mostly because it has a trivial [quickstart](https://milvus.io/docs/install_standalone-docker.md) through docker and a useful API wrapper in the [Milvus gem](https://github.com/patterns-ai-core/milvus).

I can take my markdown sections, ask the LLM to generate embeddings, and push these into the vector DB. I also need to create a schema in Milvus, and the values here are almost certainly suboptimal.

```Ruby
db = Milvus::Client.new(
    url: 'http://localhost:19530'
)

# in reality, ask Milvus if the collection exists first
# before creating it
db.collections.create(
  collection_name: 'hr',
  auto_id: true,
  fields: [
    {
      fieldName: "id",
      isPrimary: true,
      autoID: false,
      dataType: "Int64"
    },
    {
      fieldName: "text",
      dataType: "VarChar",
      elementTypeParams: {
        max_length: "10000"
      }
    },
    {
      fieldName: "vector",
      dataType: "FloatVector",
      elementTypeParams: {
        dim: 3072
      }
    }
  ]
)

paras = File.read('hr-rules.md').split('# ').reject(&:empty?)

data = paras.map.with_index do |para, i|
  puts "Adding document #{i}"
  embeddings = llm.embed(text: para).embedding
  {vector: embeddings, text: para} # returning a hash
end

db.entities.insert(collection_name: 'hr', data:)
```

## Chatting

Now I want to chat to this thing. When a person asks a question, I'm going to search the vector DB to locate any context I can find. I'm going to collect these results and pass this with the user query. The assistant client will remember the thread of conversation from one interaction to the next.

```ruby
# Ask Milvus to load the collection
db.collections.load(collection_name: 'hr')
embeddings = []

puts 'Ready to answer questions. Type "exit" to quit.'
loop do
  query = gets
  if query == "exit\n"
    break
  end

  embeddings << llm.embed(text: query, model: 'llama3.2-3b-instruct').embedding
  context = db.entities.hybrid_search(
                                      collection_name: 'hr',
                                      search: embeddings.map {
                                        { anns_field: 'vector',
                                          data: [it], # Ruby v3.4 `it` block keyword
                                          output_fields: ['text'],
                                          limit: 5
                                        }  },
                                      rerank: {
                                        strategy: 'rrf',
                                        params: { k: 10 }
                                      },
                                      limit: 5,
                                      output_fields: ['text'])['data'].map {
                                        it['text']
                                      }.join("\n\n")

  prompt = <<~PROMPT
    Use the following pieces of information enclosed in <context> tags and from previous contexts to provide an answer to the question enclosed in <question> tags. Do not mention the <context> tags in your answer.
    <context>
    #{context}
    </context>
    <question>
    #{query}
    </question>
  PROMPT

  assistant.add_message_and_run!(content: prompt)
end
```

Now I can happily chat with the AI about the information I've stored and ask it questions about it.
