- how to run trained syntaxnet model using tensorflow serving?
  - see https://github.com/dmansfield/parsey-mcparseface-api/issues/1

- i copied parsey-mcparseface-api to api directory and modify little bit. and follow xtknight instructions.

- server
```bash
$ pwd
/path/to/models/syntaxnet/work

$ git clone --recurse-submodules https://github.com/tensorflow/serving

# you need to install gRPC properly
# https://tensorflow.github.io/serving/setup
# https://github.com/dsindex/tensorflow#tensorflow-serving

# apply patch by dmansfield to serving/tf_models/syntaxnet 
$ cd serving/tf_models
$ patch -p1 < ../../api/pr250-patch-a4b7bb9a.diff.txt
$ cd -

# configure serving/tensorflow
$ cd serving/tensorflow
$ ./configure
$ cd -

# modify serving/tensorflow_serving/workspace.bzl for referencing syntaxnet
$ cp api/modified_workspace.bzl serving/tensorflow_serving/workspace.bzl
$ cat api/modified_workspace.bzl
  ... 
  native.local_repository(
    name = "syntaxnet",
    path = workspace_dir + "/tf_models/syntaxnet",
  )
  ...

# append build instructions to serving/tensorflow_serving/example/BUILD
$ cat api/append_BUILD >> serving/tensorflow_serving/example/BUILD

# copy parsey_api.cc, parsey_api.proto to example directory to build
$ cp api/parsey_api* serving/tensorflow_serving/example/

# build parsey_api 
$ cd serving
$ bazel --output_user_root=bazel_root build --nocheck_visibility -c opt -s //tensorflow_serving/example:parsey_api --genrule_strategy=standalone --spawn_strategy=standalone --verbose_failures

# run parsey_api with exported model
$ ./bazel-bin/tensorflow_serving/example/parsey_api --port=9000 ../api/parsey_model
```

- node client
```bash
$ cd api/parsey_client
$ cp index_org.js index.js
$ npm install

# if you have a trouble, check your version of node,npm
$ node --version
v4.4.7
$ npm --version
2.15.8
$ npm install grpc

# send sentences to parsey_api server
$ node index.js
{
  "result": [
    {
      "docid": "-:0",
      "text": "This is the first sentence",
      "token": [
        {
          "word": "This",
          "start": 0,
          "end": 3,
          "head": 4,
          "tag": "DT",
          "category": "DET",
          "label": "nsubj",
          "break_level": "SPACE_BREAK"
        },
...
}
```

- python client
```bash
# how to generate 'parsey_api_pb2.py'?
$ which grpc_python_plugin
# if this returns nothing, gRPC was not properly installed. see https://github.com/tensorflow/serving/issues/42
$ cd serving
$ protoc -I ./  --python_out=. --grpc_out=. --plugin=protoc-gen-grpc=`which grpc_python_plugin` ./tensorflow_serving/example/parsey_api.proto
$ cd ..

# download protobuf_json.py for converting protobuf to json
$ git clone https://github.com/dpp-name/protobuf-json.git
$ cp protobuf-json/protobuf_json.py serving/tensorflow_serving/example/

# copy parsey_client.py to serving/tensorflow_serving/example
$ cp api/parsey_client.py serving/tensorflow_serving/example

# build it
$ cd serving
$ bazel --output_user_root=bazel_root build --nocheck_visibility -c opt -s //tensorflow_serving/example:parsey_client --genrule_strategy=standalone --spawn_strategy=standalone --verbose_failures
$ ls bazel-bin/tensorflow_serving/example/parsey_client

# run
$ bazel-bin/tensorflow_serving/example/parsey_client --server=localhost:9000
23:52 $ bazel-bin/tensorflow_serving/example/parsey_client --server=localhost:9000
D0728 23:52:50.764804093   31201 ev_posix.c:101]             Using polling engine: poll
this is a first sentence.
result {
  docid: "-:0"
  text: "this is a first sentence ."
  token {
    word: "this"
    start: 0
    end: 3
    head: 4
    tag: "DT"
    category: "DET"
    label: "nsubj"
  }
  token {
    word: "is"
    start: 4
    end: 5
    head: 4
    tag: "VBZ"
    category: "VERB"
    label: "cop"
  }
...
}

```

- export model
```bash
# copy parsey_mcparseface.py to serving/tensorflow_serving/example
# buid it
$ cd ../
$ cp api/parsey_mcparseface.py serving/tensorflow_serving/example
$ cd serving
$ bazel --output_user_root=bazel_root build --nocheck_visibility -c opt -s //tensorflow_serving/example:parsey_mcparseface --genrule_strategy=standalone --spawn_strategy=standalone --verbose_failures
$ ls bazel-bin/tensorflow_serving/example/parsey_mcparseface

# run
# this will read model from --model_dir and export to --export_path directory
$ bazel-bin/tensorflow_serving/example/parsey_mcparseface --model_dir=syntaxnet/models/parsey_mcparseface --export_path=exported

```

- what about parsing only case? especially when you trained Korean parser.
```bash
# the parsey_api server can handle conll format.
# so, just export model and use it

# export parsing model only
# replace serving/tensorflow_serving/example/parsey_mcparseface.py with parsey_sejong.py
# build it
$ cd ../
$ cp api/parsey_sejong.py serving/tensorflow_serving/example/parsey_mcparseface.py
$ cd serving
$ bazel --output_user_root=bazel_root build --nocheck_visibility -c opt -s //tensorflow_serving/example:parsey_mcparseface --genrule_strategy=standalone --spawn_strategy=standalone --verbose_failures
$ ls bazel-bin/tensorflow_serving/example/parsey_mcparseface

# run
# this will read model from --model_dir and export to --export_path directory
$ bazel-bin/tensorflow_serving/example/parsey_mcparseface --model_dir=../models_sejong --export_path=exported_sejong

# run parsey_api with exported model
$ ./bazel-bin/tensorflow_serving/example/parsey_api --port=9000 exported_sejong/00000001

# node client
# send conll format to parsey_api server
$ cd ../api/parsey_client
$ cp index_sejong.js index.js
$ node index.js | more
{
  "result": [
    {
      "docid": "-:0",
      "text": "내 가 집 에 가 ㄴ다 .",
      "token": [
        {
          "word": "내",
          "start": 0,
          "end": 2,
          "head": 1,
          "tag": "NP",
          "category": "NP",
          "label": "MOD",
          "break_level": "SPACE_BREAK"
        },
        {
          "word": "가",
          "start": 4,
          "end": 6,
          "head": 4,
          "tag": "JKS",
          "category": "JKS",
          "label": "NP_SBJ",
          "break_level": "SPACE_BREAK"
        },
...
}

# python client
# replace serving/tensorflow_serving/example/parsey_client.py with parsey_sejong_client.py
$ cp api/parsey_sejong_client.py serving/tensorflow_serving/example/parsey_client.py

# parsey_sejong_client.py import konlpy, protobuf_json
# so, you need to install konlpy( http://konlpy.org/ko/v0.4.3/install/ )

# download protobuf_json.py for converting protobuf to json
$ git clone https://github.com/dpp-name/protobuf-json.git
$ cp protobuf-json/protobuf_json.py serving/tensorflow_serving/example/

# build it
$ cd serving
$ bazel --output_user_root=bazel_root build --nocheck_visibility -c opt -s //tensorflow_serving/example:parsey_client --genrule_strategy=standalone --spawn_strategy=standalone --verbose_failures
$ ls bazel-bin/tensorflow_serving/example/parsey_client

# run
$ bazel-bin/tensorflow_serving/example/parsey_client --server=localhost:9000
나는 학교에 간다
nput :  나는 학교에 간다
Parsing :
{"result": [{"text": "나 는 학교 에 가 ㄴ다", "token": [{"category": "NP", "head": 1, "end": 2, "label": "MOD", "start": 0, "tag": "NP", "word": "나"}, {"category": "JX", "head": 4, "end": 6, "label": "NP_SBJ", "start": 4, "tag": "JX", "word": "는"}, {"category": "NNG", "head": 3, "end": 13, "label": "MOD", "start": 8, "tag": "NNG", "word": "학교"}, {"category": "JKB", "head": 4, "end": 17, "label": "NP_AJT", "start": 15, "tag": "JKB", "word": "에"}, {"category": "VV", "head": 5, "end": 21, "label": "MOD", "start": 19, "tag": "VV", "word": "가"}, {"category": "EC", "end": 28, "label": "ROOT", "start": 23, "tag": "EC", "word": "ㄴ다"}], "docid": "-:0"}]}

```