---
title: 「大模型应用」Function Calling
date: 2025-05-03 15:20:17
layout: post
---

# 「大模型应用」Function Calling

## 🌟大模型api的调用

#### 定义client和model

```python
from openai import OpenAI
client = OpenAI(
    base_url='https://tbnx.plus7.plus/v1', # 中转的url地址
    api_key='sk-AVzk4v9UHTxFbmMxWnQo5PAfIzYmAPNmSgo4JMHbmKx0cPrC' # 自己生成的key
)
model = "deepseek-chat"
```

#### 一次简单的api调用

```python
response = client.chat.completions.create(
  model=model,
  messages=[
    {"role": "user", "content": "对模型提问的文本内容"},
  ],
  stream=False	# 非流式输出，模型全部输出完毕后返回给客户
)
```

让我们看看response的结构：

```python
>> response.to_dict()
>> {'id': '02174625885892937063ba0950ffa3b2a02464936c9a2817c3789',
 'choices': [{'finish_reason': 'stop',
   'index': 0,
   'message': {'content': '模型返回的文本内容',
    'role': 'assistant'},
   'content_filter_results': {'hate': {'filtered': False},
    'self_harm': {'filtered': False},
    'sexual': {'filtered': False},
    'violence': {'filtered': False},
    'jailbreak': {'filtered': False, 'detected': False},
    'profanity': {'filtered': False, 'detected': False}}}],
 'created': 1746258893,
 'model': 'deepseek/deepseek-v3-turbo',
 'object': 'chat.completion',
 'system_fingerprint': '',
 'usage': {'completion_tokens': 936,
  'prompt_tokens': 10,
  'total_tokens': 946,
  'completion_tokens_details': {'audio_tokens': 0, 'reasoning_tokens': 0},
  'prompt_tokens_details': {'audio_tokens': 0, 'cached_tokens': 0}}}
```

#### 连续对话的实现

```python
# 创建一个会话列表来不断地追加历史对话纪律
messages = []

while True:
    prompt = input('\n用户提问： ')
    if prompt == "退出":
        break  # 如果输入的是“退出”，则结束循环
    messages.append(
        {
            'role':'user',
            'content':prompt
        }
    )
    completion = client.chat.completions.create(
        model=model,
        messages = messages
    )
		# 提取模型的返回，加入message列表
    response = completion.choices[0].message.content
    print(f"模型回复:{response}")
    messages.append(
        {
            'role':'assistant',
            'content':response
        }
    )
```



## 🌟Function calling

一个Function calling  <=>  一个tool <=>  一个json

### 构建大模型能够调用的函数

#### 示例：本地数据库查询函数

```python
# 初始化本地数据库
def create_and_populate_database():
    # 连接到本地数据库文件
    conn = sqlite3.connect('SportsEquipment.db')  # 指定文件名来保存数据库
    cursor = conn.cursor()

    # 检查是否存在名为 'products' 的表，如果不存在则创建
    cursor.execute("SELECT name FROM sqlite_master WHERE type='table' AND name='products';")
    if cursor.fetchone() is None:
        # 创建表
        cursor.execute('''
        CREATE TABLE products (
            product_id TEXT,
            product_name TEXT,
            description TEXT,
            specifications TEXT,
            usage TEXT,
            brand TEXT,
            price REAL,
            stock_quantity INTEGER
        )
        ''')
        # 数据列表，用于插入表中
        products = [
            ('001', '足球', '高品质职业比赛用球，符合国际标准', '圆形，直径22 cm', '职业比赛、学校体育课', '耐克', 120, 50),
            ('002', '羽毛球拍', '轻量级，适合初中级选手，提供优秀的击球感受', '碳纤维材质，重量85 g', '业余比赛、家庭娱乐', '尤尼克斯', 300, 30),
            ('003', '篮球', '室内外可用，耐磨耐用，适合各种天气条件', '皮质，标准7号球', '学校、社区运动场', '斯伯丁', 200, 40),
            ('004', '跑步鞋', '适合长距离跑步，舒适透气，提供良好的足弓支撑', '多种尺码，透气网布', '长跑、日常训练', '阿迪达斯', 500, 20),
            ('005', '瑜伽垫', '防滑材料，厚度适中，易于携带和清洗', '长180cm，宽60cm，厚5mm', '瑜伽、普拉提', '曼达卡', 150, 25),
            ('006', '速干运动衫', '吸汗快干，适合各种户外运动，持久舒适', 'S/M/L/XL，多色可选', '运动、徒步、旅游', '诺斯脸', 180, 60),
            ('007', '电子计步器', '精确计步，带心率监测功能，蓝牙连接手机应用', '可充电，续航7天', '日常健康管理、运动', 'Fitbit', 250, 15),
            ('008', '乒乓球拍套装', '包括两只拍子和三个球，适合家庭娱乐和业余训练', '标准尺寸，拍面防滑处理', '家庭、社区', '双鱼', 160, 35),
            ('009', '健身手套', '抗滑耐磨，保护手部，适合各种健身活动', '多种尺码，通风设计', '健身房、户外运动', 'Under Armour', 120, 50),
            ('010', '膝盖护具', '减少运动伤害，提供良好的支撑和保护，适合篮球和足球运动', '弹性织物，可调节紧度', '篮球、足球及其他运动', '麦克戴维', 220, 40)
        ]

        # 插入数据到表中
        cursor.executemany('''
        INSERT INTO products (product_id, product_name, description, specifications, usage, brand, price, stock_quantity)
        VALUES (?, ?, ?, ?, ?, ?, ?, ?)
        ''', products)

        # 提交更改以确保数据被保存在文件中
        conn.commit()

    # 检索并打印所有记录以验证插入
    cursor.execute('SELECT * FROM products')
    all_rows = cursor.fetchall()

    conn.close()  # 关闭连接以释放资源

    return all_rows
```

```python
# 在数据库中根据产品名称进行模糊查询
def query_by_product_name(product_name):
    # 连接 SQLite 数据库
    conn = sqlite3.connect('SportsEquipment.db')
    cursor = conn.cursor()

    # 使用SQL查询按名称查找产品。'%'符号允许部分匹配。
    cursor.execute("SELECT * FROM products WHERE product_name LIKE ?", ('%' + product_name + '%',))

    # 获取所有查询到的数据
    rows = cursor.fetchall()

    # 关闭连接
    conn.close()

    return rows
```

### 向大模型描述和传递函数

当涉及到函数调用的时候，我们需要在调用聊天完成 API 时，额外的传递一个 `tools`参数，以告知大模型：你在当前的会话过程中，可以调用`query_by_product_name`参数。

```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "query_by_product_name",
            "description": "Query the database to retrieve a list of products that match or contain the specified product name. This function can be used to assist customers in finding products by name via an online platform or customer support interface.",	# 告知大模型何时调用
            "parameters": {
                "type": "object",
                "properties": {
                    "product_name": {
                        "type": "string",
                        "description": "The name of the product to search for. The search is case-insensitive and allows partial matches."
                    }
                },
                "required": ["product_name"]	# 必须传递的参数
            }
        }

    }
]
```

### 用function calling的api调用

```python
response = client.chat.completions.create(
    model=model,
    messages=messages,
    tools=tools,  # 这里是添加
  	parallel_tool_calls=False  # 禁用并行函数调用,确保模型一次只生成一个函数调用,避免输出与工具的严格模式不匹配（默认为True）
)
```

查看response的结构，可以看到message中多了一项tool_calls：

```python
>> response.to_dict()
>> {'id': 'chatcmpl-BDjfeKhH2OUf5vNIb68RRgmQC0uDX',
 'choices': [{'finish_reason': 'tool_calls',
   'index': 0,
   'logprobs': None,
   'message': {'content': None,
    'role': 'assistant',
    'tool_calls': [{'id': 'call_wsfD0pu4QNuWRQBKgmXF2o2V',
      'function': {'arguments': '{"product_name":"球"}',
       'name': 'query_by_product_name',
       'parameters': None},
      'type': 'function'}]}}],	# function calling信息
 'model': 'gpt-4o',
 'usage': {'completion_tokens': 18,
  'prompt_tokens': 109,
  'total_tokens': 127,
  'completion_tokens_details': {},
  'prompt_tokens_details': {'CachedCreationTokens': 0,
   'cached_tokens_details': {}}},
 'error': {}}
```

当触发函数调用，对于chatgpt来说，content字段的对应的值将会是None；而对于qianwen当其他llm则未必。而tool_calls字段中的内容，将会按照我们对`query_by_product_name`的Json Schema描述中 `required`字段的要求来返回值。

我们可以用```function_call = response.choices[0].message.tool_calls[0]```来提取一个function_call

从function_call中，我们可以：

提取该function_call的函数名 ```function_name = function_call.function.name```

提取该function_call的参数 ```function_args = json.loads(function_call.function.arguments)```



### 并行函数调用

并行调用会在单个响应中生成多个函数调用，在 tool_calls 数组中产生包含多个函数调用的消息。

触发并行函数调用的 prompt 类似于：“你好，你家都卖什么球，什么衣服，什么鞋？”

```python
messages = [
    {"role": "user", "content": "你好，你家都卖什么球，什么衣服，什么鞋？"}
]
```

调用api：

```python
response = client.chat.completions.create(
    model=model,
    messages=messages,
    tools=tools,
    parallel_tool_calls=True # 默认为True，可以省略
)
```

查看response：

```python
>> response.to_dict()
>> {'id': 'chatcmpl-BDjiZ77gaDPNLWNhsq4OwIqlmByUe',
 'choices': [{'finish_reason': 'tool_calls',
   'index': 0,
   'logprobs': None,
   'message': {'content': None,
    'refusal': None,
    'role': 'assistant',
    'tool_calls': [{'id': 'call_2qN14k0XZjz6vn4da3HrRA81',
      'function': {'arguments': '{"product_name": "球"}',
       'name': 'query_by_product_name'},
      'type': 'function'},
     {'id': 'call_pUCbFLbVHP8rFKR31V00oUkY',
      'function': {'arguments': '{"product_name": "衣服"}',
       'name': 'query_by_product_name'},
      'type': 'function'},
     {'id': 'call_PWSRQDS1IlIRSzFZZzp6rWn9',
      'function': {'arguments': '{"product_name": "鞋"}',
       'name': 'query_by_product_name'},
      'type': 'function'}]}}],
 'model': 'gpt-4o-2024-08-06',
 'usage': {'completion_tokens': 69,
  'prompt_tokens': 116,
  'total_tokens': 185,
  'completion_tokens_details': {},
  'prompt_tokens_details': {'CachedCreationTokens': 0,
   'cached_tokens_details': {}}},
 'error': {}}
```

### 多函数调用

如果我们想接入智能电商客服的第二个功能：根据用户对商品的提问查询对应的优化政策，那么接下来我们定义一个` read_store_promotions `函数根据提供的产品名称来读取具体的优惠政策。

```python
def read_store_promotions(product_name):
    # 指定优惠政策文档的文件路径
    file_path = 'store_promotions.txt'

    try:
        # 打开文件并按行读取内容
        with open(file_path, 'r', encoding='utf-8') as file:
            promotions_content = file.readlines()

        # 搜索包含产品名称的行
        filtered_content = [line for line in promotions_content if product_name in line]

        # 返回匹配的行，如果没有找到，返回一个默认消息
        if filtered_content:
            return ''.join(filtered_content)
        else:
            return "没有找到关于该产品的优惠政策。"
    except FileNotFoundError:
        # 文件不存在的错误处理
        return "优惠政策文档未找到，请检查文件路径是否正确。"
    except Exception as e:
        # 其他潜在错误的处理
        return f"读取优惠政策文档时发生错误: {str(e)}"
```

```python
# 重新创建一个包含店铺优惠政策的文本文档
promotions_content = """
店铺优惠政策：
1. 足球 - 购买足球即可享受9折优惠。
2. 羽毛球拍 - 任意购买羽毛球拍两支以上，享8折优惠。
3. 篮球 - 单笔订单满300元，篮球半价。
4. 跑步鞋 - 第一次购买跑步鞋的顾客可享受满500元减100元优惠。
5. 瑜伽垫 - 每购买一张瑜伽垫，赠送价值50元的瑜伽教程视频一套。
6. 速干运动衫 - 买三送一，赠送的为最低价商品。
7. 电子计步器 - 购买任意电子计步器，赠送配套手机APP永久会员资格。
8. 乒乓球拍套装 - 乒乓球拍套装每套95折。
9. 健身手套 - 满200元包邮。
10. 膝盖护具 - 每件商品配赠运动护膝一个。

注意：
- 所有优惠活动不可与其他优惠同享。
- 优惠详情以实际到店或下单时为准。
"""

# 将优惠政策写入文件
file_path = './store_promotions.txt'
with open(file_path, 'w', encoding='utf-8') as file:
    file.write(promotions_content)

file_path
```

定义` read_store_promotions `函数的Json Schema描述，并添加到 tools 列表中。

```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "query_by_product_name",
            "description": "Query the database to retrieve a list of products that match or contain the specified product name. This function can be used to assist customers in finding products by name via an online platform or customer support interface.",
            "parameters": {
                "type": "object",
                "properties": {
                    "product_name": {
                        "type": "string",
                        "description": "The name of the product to search for. The search is case-insensitive and allows partial matches."
                    }
                },
                "required": ["product_name"]
            }
        }

    },
    {
        "type": "function",
        "function": {
            "name": "read_store_promotions",
            "description": "Read the store's promotion document to find specific promotions related to the provided product name. This function scans a text document for any promotional entries that include the product name.",
            "parameters": {
                "type": "object",
                "properties": {
                    "product_name": {
                        "type": "string",
                        "description": "The name of the product to search for in the promotion document. The function returns the promotional details if found."
                    }
                },
                "required": ["product_name"]
            }
        }
    }
]
```

定义一个available_functions字典保存相关函数名，方便索引。

```python
available_functions = {"query_by_product_name": query_by_product_name, "read_store_promotions":read_store_promotions}
```

调用api：

```python
messages = []
# 添加用户的提问到消息列表
prompt = "你家卖健身手套吗？现在有什么优惠？"
messages.append({'role': 'user', 'content': prompt})

# 检查是否需要调用外部函数
completion = client.chat.completions.create(
    model=model,
    messages=messages,
    tools=tools,
    parallel_tool_calls=False  # 这里需要格外注意
)
response = completion.choices[0].message
```

查看response，可以看到返回了两个tool_calls信息：

```python
>> response.to_dict()
>> {'content': '',
 'role': 'assistant',
 'tool_calls': [{'id': '26a44b9b8f384b07ac8452d30c91c1ec',
   'function': {'arguments': '{"product_name": "健身手套"}',
    'name': 'query_by_product_name'},
   'type': 'function'},
  {'id': 'ccadf39e16da4a1594151de8e42e937e',
   'function': {'arguments': '{"product_name": "健身手套"}',
    'name': 'read_store_promotions'},
   'type': 'function'}]}
```



在Function Calling架构中，尽管可以通过多函数和并行函数调用逻辑来调用外部函数，实现一些具体的操作流程，但它仍面临一些局限性。例如，当面对用户的单条复杂请求时，如“你家卖健身手套吗？现在有什么优惠？”，虽然我们配置了两个相应的外部函数，理论上能够处理这一请求，但当前的架构无法自动按照一定的执行顺序依次调用这些函数，并在同一轮对话中直接输出结果。理想的处理流程应该是：首先通过`query_by_product_name`函数确认是否销售健身手套；如果有，接着调用`read_store_promotions`函数获取关于健身手套的优惠政策；最后，结合产品价格和优惠信息，直接为用户计算出最终结果。这种需要规划和连续决策的能力，已经超出了智能助理的常规范围，而更接近于智能代理的“Planning”能力。因此，这种复杂的需求处理揭示了向真正的智能代理迈进的必要性。

