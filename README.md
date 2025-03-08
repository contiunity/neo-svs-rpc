# Neo SVS RPC
用于实现歌声合成（Singing voice synthesis）编辑器与引擎标准化的RPC接口。基本上是仿（chāo）照（xí）了OpenAI的接口风格。

## API调用

### 1. 列出声库

请求：
```
GET https://<api端点>/v1/models
```

响应：
```
{
    "object": "list",
    "data": [
        {
            "created": <时间戳>,
            "id": "<组ID>",
            "object": "model",
            "singing.submodels": [
                {
                    "id": "<声库ID>",
                    "metadata": <元数据>
                }
            ],
            "singing.languages": [ "<语言ID>" ]
            "singing.allow_mix": <是否允许声线混合>
        } ...
  ]
}
```

注意事项：
1. 这个接口的返回中可以包含其他模型（如LLM模型等）。应当基于是否包含`singing.submodels`来判断是否是声库。
2. `singing.languages`是声库的语言选择项。若没有这个项或为空，视为不声明语言的单语种声库。
3. 语言ID应当是全小写的标准全称形式，例如：汉语是`mandarin`、英语是`english`、日语是`japanese`。
4. 若`singing.allow_mix`为`true`，则整个组中每个声库全部都可以混合。若不为`true`或不存在，则完全不支持声线混合。不同组的声库不能混合。
5. metadata（元数据）的数据项和utau的`character.txt`相同。

### 2. 创建Session

创建Session。

请求：

```
POST https://<api端点>/v1/audio/singing/editorsession

{
  "singer": "<组ID>:<声库ID>",
  "session_id": null
}
```

返回值：

```
{
  "object": "audio.singing.editorsession"
  "session_id": "<session编号>",
  "notes": [],
  "singer": "<组ID>:<声库ID>",
  "curves": [], ...
}
```
注意事项：

1. 请求的Session编号（`session_id`）必须为null。
2. 若启用声线混合，采用星号（`*`）表示配比，竖线（`|`）分隔，形如：`model-1:opencpop*0.5|yoko*0.5`。
3. 返回的`session_id`必须是UUID。

### 3. 替换各种参数

替换各种参数（包括声库、音符、音符参数、曲线参数等）。

请求：

```
POST https://<api端点>/v1/audio/singing/editorsession

{
  "session_id": "<session编号>",
  "notes": [
    {
      "name": "<音符内容>",
      "dur": <音符时长>,
      "lang": "<语言ID>",
      "param.vel": 1.0,
      "graphemes": {
        "start": <起始点位移>,
        "seq": [
          {"name": "<音素名称>", "dur": <音素时长>}
        ]
      },
      ...
    } ...
  ],
  "singer": "<组ID>:<声库ID>",
  "curves.pit": [ <Pitch采样点值> ... ]
  ...
}
```

返回值：

```
{
  "object": "audio.singing.editorsession"
  "session_id": "<session编号>",
  "notes": <音符序列>,
  "singer": "<组ID>:<声库ID>",
  "curves.pit": [ <Pit序列> ]
  ...
}
```
注意事项：

1. 除了session_id是必须的以外，其他传参都是可选的，不填代表不替换。
2. 音符级参数不填代表自动推理。
3. graphemes的start代表本音符第一个音素的起始点。单位是0.01秒，正数代表提前于音符，负数代表延后。
4. 若启用声线混合，采用星号（`*`）表示配比，竖线（`|`）分隔，形如：`model-1:opencpop*0.5|yoko*0.5`。
5. 音符时长的单位是0.01秒。
6. 曲线参数的项名是`curves.<参数名>`，每0.01一个采样。音符级参数应当包含在notes内的音符项中，项名是`param.<参数名>`。
7. 参数名应该是小写简称，尽可能用三个字母，如果是openutau有的，必须和openutau相同。

### 4. 读取参数

按照“替换各种参数”进行，什么参数都不传就是读取。

### 5. 自动推理曲线参数

同“替换各种参数”中的做法，但是传入的曲线参数是字符串型的`"AUTOMATIC"`。如果不支持推理，报错。

### 6. 推理音频

请求（如果是Session）：
```
POST https://<api端点>/v1/audio/singing

{
  "model": "<组ID>",
  "voice": "<声库ID>",
  "input": {
    "type": "session",
    "session_id": <session编号>
  },
  "retain_session": <是否保留Session>,
  "stream": <是否采用CCS流式>,
  "ccs_wavetype": "<CCS波形传递方式>"
}
```

请求（如果是无状态）：
```
POST https://<api端点>/v1/audio/singing

{
  "model": "<组ID>",
  "voice": "<声库ID>",
  "input": {
    "type": "confile",
    "confile": <confile>
  },
  "stream": <是否采用CCS流式>,
  "ccs_wavetype": "<CCS波形传递方式>"
}
```

返回值（如果是CCS流式）：
```
data: {"type": "audio.singing.ccstream", "voice": {"type": "<传递方式>", "wave": "<传递>"}, "offset": <对应起始位置>, "length": <有效长度>}

data: {"type": "audio.singing.ccstream", "voice": {"type": "<传递方式>", "wave": "<传递>"}, "offset": <对应起始位置>, "length": <有效长度>}

...

data: {"type": "audio.singing.ccstream.stop"}
```

返回值（如果不是CCS流式）：和OpenAI TTS API相同。

注意事项：
1. `model`和`voice`参数是替换session中设定的声库。如果不填，代表按session中的配方。如果只填`voice`，代表替换为同组的声库/声线混合配方。如果只填`model`，报错。
2. confile实际上就是“替换各种参数”中向API端点的传参，但是不包含`session_id`和`singer`项。
3. 无状态模式必须填写`model`和`voice`项，confile中的`singer`项是不被承认的。
4. `ccs_wavetype`方式若不填，代表服务器选择传递方式。若填了，则必须遵照填入的传递方式，若服务器不支持，报错。若`ccs_wavetype`是list类型或逗号分隔的文本，则代表服务器可以选择其中一种方式传递。
5. `offset`和`length`的单位是0.01秒。
6. 传递方式为`path`时，`wave`项是文件url；传递方式为`blockchain`时，`wave`项是IPFS CID；传递方式是`b64e`时，`wave`项是base64。

### 7. 删除Session

请求（第一种方式）：
```
POST http://<api端点>/v1/audio/singing/editorsession

{
  "session_id": "<session编号>",
  "delete": true
}
```

请求（第二种方式）：

```
DELETE http://<api端点>/v1/audio/singing/editorsession/<session编号>
```
