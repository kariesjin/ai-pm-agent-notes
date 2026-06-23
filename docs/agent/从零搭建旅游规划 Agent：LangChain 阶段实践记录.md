# 从零搭建旅游规划 Agent：LangChain 阶段实践记录
    搭建这个旅游规划 Agent，一方面是为了验证近期对 LangChain、Tool Calling、Memory、用户画像等 Agent 能力的学习成果；另一方面也是为了沉淀一个可持续迭代的 AI 产品项目。
    这是第一版本，第一版本使用langchain架构，LLM Api选用deepseek。
# 项目目标
**根据用户需求规划旅游路线**

    1、天气查询确定天气状况，如天气不适合，则向用户二次确认行程
    2、往返路线规划：根据用户给出的所在地，规划往返的行程路线
    3、旅游路线规划：根据旅游地点的网上信息，合理规划旅游路线。可以在景点推荐后追加酒店推荐
    4、行李清单：根据用户给出的信息，提供详细旅游所需要的行李清单
    5、预算计算：计算全部预算

# Agent开始迭代！
## Langchain Agent V1 基础版
### 环境准备
    1、首先需要安装python，可以官网自行下载安装
    2、pip install -U langchain langchain-openai   安装 LangChain 核心框架，以及用于连接 DeepSeek/OpenAI兼容接口的模型适配包。langchain：LangChain 核心框架包    langchain-openai：LangChain 连接 OpenAI兼容模型的包
### 完整代码示例
```python
from langchain.agents import create_agent
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool


# 1. 连接 DeepSeek
llm = ChatOpenAI(
    model="deepseek-chat",
    api_key="你的DEEPSEEK_API_KEY",
    base_url="https://api.deepseek.com"
)


# 2. 定义工具：天气查询
@tool
def weather_tool(city: str) -> str:
    """
    查询城市天气。
    参数 city 是城市名称，例如：东京、上海、北京。
    """
    weather_db = {
        "东京": "东京今天晴天，22°C，适合户外游玩，不太需要带伞。",
        "上海": "上海今天阴天，28°C，可能有小雨，建议带伞。",
        "北京": "北京今天小雨，18°C，建议带伞并安排室内景点。"
    }

    return weather_db.get(city, f"暂时没有查到{city}的天气信息。")


# 3. 创建 Agent
agent = create_agent(
    model=llm,
    tools=[weather_tool],
    system_prompt="""
你是一个专业的旅游规划Agent。

你的任务：
1. 理解用户的旅游需求
2. 如果用户的问题涉及天气、是否带伞、适合户外活动，请调用 weather_tool
3. 根据工具结果和用户需求，生成清晰、实用的旅游建议
4. 输出要按天规划，尽量具体
"""
)


# 4. 命令行循环
while True:
    user_input = input("\n请输入旅游需求（输入 exit 退出）：")

    if user_input == "exit":
        print("已退出旅游Agent。")
        break

    result = agent.invoke({
        "messages": [
            {"role": "user", "content": user_input}
        ]
    })

    print("\n🤖 旅游Agent：")
    print(result["messages"][-1].content)
    print("\n" + "-" * 50)
```
### 代码解读
#### LLM定义
```python
    # 1. 连接 DeepSeek
    llm = ChatOpenAI(
    model="deepseek-chat",
    api_key="你的DEEPSEEK_API_KEY",
    base_url="https://api.deepseek.com"
)
```
这一步用来定义项目当中用到的大模型。在Agent项目中，可能使用到多个大模型，那么在定义的时候，就通过名字进行区分。比如：

    `planner_llm`：负责规划
    `router_llm`：负责判断任务类型
    `writer_llm`：负责最终表达润色
然后在使用时，根据需要调用大模型。
```python
    agent = create_agent(
    model=planner_llm,
    tools=[weather_tool, attraction_tool, budget_tool],
    system_prompt="你是旅游规划Agent..."
)
```
#### 工具声明
```python
# 2. 定义工具：天气查询
@tool
def weather_tool(city: str) -> str:
    """
    查询城市天气。
    参数 city 是城市名称，例如：东京、上海、北京。
    """
    weather_db = {
        "东京": "东京今天晴天，22°C，适合户外游玩，不太需要带伞。",
        "上海": "上海今天阴天，28°C，可能有小雨，建议带伞。",
        "北京": "北京今天小雨，18°C，建议带伞并安排室内景点。"
    }

    return weather_db.get(city, f"暂时没有查到{city}的天气信息。")
```
这一步在定义大模型能够调用的工具有哪些，同时也在变相的设定大模型的能力边界。

其中的说明函数很重要：
```python
"""
查询城市天气。
参数 city 是城市名称，例如：东京、上海、北京。
"""
```
模型会根据这段描述的内容来决定什么时候调用工具。

多个工具都用@tool声明即可。

#### Create_agent组装器
``` python
agent = create_agent(
    model=llm,
    tools=[weather_tool],
    system_prompt="..."
)
```
这一步就是把
    
    模型+工具+规则

组装成一个Agent。

### 运行说明
    输入 python agent.py运行后，就能够成功调用大模型、工具并根据规则思考、输出。

## 旅游Agent V2 版本
### 完整代码示例
```python
from langchain.agents import create_agent
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool


# 1. 连接 DeepSeek
llm = ChatOpenAI(
    model="deepseek-chat",
    api_key="你的DEEPSEEK_API_KEY",
    base_url="https://api.deepseek.com"
)


# 2. 工具一：天气查询
@tool
def weather_tool(city: str) -> str:
    """
    查询城市天气。
    当用户问到天气、温度、是否带伞、是否适合户外活动时使用。
    参数 city 是城市名称，例如：东京、上海、北京。
    """
    weather_db = {
        "东京": "东京今天晴天，22°C，适合户外游玩，不太需要带伞。",
        "上海": "上海今天阴天，28°C，可能有小雨，建议带伞。",
        "北京": "北京今天小雨，18°C，建议带伞并安排室内景点。"
    }

    return weather_db.get(city, f"暂时没有查到{city}的天气信息。")


# 3. 工具二：景点推荐
@tool
def attraction_tool(city: str, preference: str) -> str:
    """
    根据城市和用户偏好推荐景点。
    当用户需要规划路线、推荐景点、安排每天行程时使用。
    参数 city 是城市名称。
    参数 preference 是用户偏好，例如：美食、购物、亲子、动漫、历史、人文、自然风景。
    """
    attraction_db = {
        "东京": {
            "动漫": "秋叶原、池袋乙女路、三鹰之森吉卜力美术馆、中野百老汇",
            "美食": "筑地场外市场、新宿思出横丁、银座、浅草小吃街",
            "购物": "涩谷、新宿、银座、表参道、原宿",
            "历史": "浅草寺、上野公园、皇居外苑、明治神宫"
        },
        "上海": {
            "美食": "城隍庙、云南南路、静安寺周边、武康路咖啡街区",
            "购物": "南京西路、淮海中路、徐家汇、前滩太古里",
            "历史": "外滩、豫园、武康路、上海博物馆"
        },
        "北京": {
            "历史": "故宫、天坛、颐和园、八达岭长城、雍和宫",
            "美食": "牛街、簋街、南锣鼓巷、前门大街",
            "亲子": "北京动物园、科技馆、自然博物馆、环球影城"
        }
    }

    city_data = attraction_db.get(city)

    if not city_data:
        return f"暂时没有{city}的景点数据。"

    result = city_data.get(preference)

    if result:
        return f"{city}适合{preference}偏好的景点有：{result}。"

    all_attractions = "；".join([f"{k}：{v}" for k, v in city_data.items()])
    return f"没有精确匹配到{preference}偏好，以下是{city}可选景点：{all_attractions}"


# 4. 工具三：预算估算
@tool
def budget_tool(city: str, days: int, level: str) -> str:
    """
    根据城市、天数和消费档位估算旅游预算。
    当用户提到预算、花费、多少钱、费用、穷游、舒适游时使用。
    参数 city 是城市名称。
    参数 days 是旅行天数。
    参数 level 是消费档位，只能是：经济、舒适、豪华。
    """
    daily_budget = {
        "东京": {
            "经济": 600,
            "舒适": 1000,
            "豪华": 1800
        },
        "上海": {
            "经济": 350,
            "舒适": 700,
            "豪华": 1300
        },
        "北京": {
            "经济": 400,
            "舒适": 750,
            "豪华": 1400
        }
    }

    city_budget = daily_budget.get(city)

    if not city_budget:
        return f"暂时没有{city}的预算数据。"

    per_day = city_budget.get(level)

    if not per_day:
        return "消费档位请输入：经济、舒适、豪华。"

    total = per_day * days

    return f"{city}{days}天{level}游的预估预算约为 {total} 元，不含大交通费用。"


# 5. 创建 Agent
agent = create_agent(
    model=llm,
    tools=[weather_tool, attraction_tool, budget_tool],
    system_prompt="""
你是一个专业的旅游规划Agent。

你可以使用以下工具：
1. weather_tool：查询天气
2. attraction_tool：根据城市和偏好推荐景点
3. budget_tool：估算旅游预算

你的工作流程：
1. 先理解用户的城市、天数、预算、偏好
2. 如果问题涉及天气、带伞、户外活动，调用 weather_tool
3. 如果问题涉及景点、路线、行程安排，调用 attraction_tool
4. 如果问题涉及预算、费用、消费水平，调用 budget_tool
5. 最后综合工具结果，输出一个清晰、可执行的旅游方案

输出要求：
- 按天规划
- 每天包含上午、下午、晚上
- 说明为什么这样安排
- 如果工具数据不足，要明确说明，不要编造
"""
)


# 6. 命令行循环
while True:
    user_input = input("\n请输入旅游需求（输入 exit 退出）：")

    if user_input == "exit":
        print("已退出旅游Agent。")
        break

    result = agent.invoke({
        "messages": [
            {"role": "user", "content": user_input}
        ]
    })

    print("\n🤖 旅游Agent：")
    print(result["messages"][-1].content)
    print("\n" + "-" * 50)
```

### 代码新增说明
与V1版本相比，V2版本增加了两个tool，分别用来查景点和查预算。
```python
# 2. 工具一：天气查询
@tool
def weather_tool(city: str) -> str:
    """
    查询城市天气。
    当用户问到天气、温度、是否带伞、是否适合户外活动时使用。
    参数 city 是城市名称，例如：东京、上海、北京。
    """

# 3. 工具二：景点推荐
@tool
def attraction_tool(city: str, preference: str) -> str:
    """
    根据城市和用户偏好推荐景点。
    当用户需要规划路线、推荐景点、安排每天行程时使用。
    参数 city 是城市名称。
    参数 preference 是用户偏好，例如：美食、购物、亲子、动漫、历史、人文、自然风景。
    """
    

# 4. 工具三：预算估算
@tool
def budget_tool(city: str, days: int, level: str) -> str:
    """
    根据城市、天数和消费档位估算旅游预算。
    当用户提到预算、花费、多少钱、费用、穷游、舒适游时使用。
    参数 city 是城市名称。
    参数 days 是旅行天数。
    参数 level 是消费档位，只能是：经济、舒适、豪华。
    """
```
在create_agent中，增加了对工具的使用描述，并修改思考流程。
```python
# 5. 创建 Agent
agent = create_agent(
    model=llm,
    tools=[weather_tool, attraction_tool, budget_tool],
    system_prompt="""
你是一个专业的旅游规划Agent。

你可以使用以下工具：
1. weather_tool：查询天气
2. attraction_tool：根据城市和偏好推荐景点
3. budget_tool：估算旅游预算

你的工作流程：
1. 先理解用户的城市、天数、预算、偏好
2. 如果问题涉及天气、带伞、户外活动，调用 weather_tool
3. 如果问题涉及景点、路线、行程安排，调用 attraction_tool
4. 如果问题涉及预算、费用、消费水平，调用 budget_tool
5. 最后综合工具结果，输出一个清晰、可执行的旅游方案

输出要求：
- 按天规划
- 每天包含上午、下午、晚上
- 说明为什么这样安排
- 如果工具数据不足，要明确说明，不要编造
"""
)
```
## 旅游Agent V3 版本
在V1中，我们定义了Agent的基础形态，同时赋予它假的查询天气的功能，在V2版本中，增加了两个工具，分别是查询景点工具和查询预算工具，当然这些都不是真实数据，只是用来测试的固定返回值。

在V3版本中，我们主要更新的是：当信息不完整时，先追问，不要乱规划。即：

    Agent 先判断信息是否足够。
    如果足够，就规划。
    如果不够，就先追问关键问题。

在agent.py中增加一个tool。
```python
@tool
def requirement_check_tool(user_request: str) -> str:
    """
    检查用户的旅游需求是否完整。
    当用户只表达了模糊旅游意图，但缺少城市、天数、预算、偏好等关键信息时使用。
    参数 user_request 是用户原始输入。
    """
    missing_items = []

    # 检查城市
    city_keywords = ["东京", "上海", "北京", "大阪", "京都", "南京", "杭州", "苏州", "成都", "重庆"]
    has_city = any(city in user_request for city in city_keywords)
    if not has_city:
        missing_items.append("目的地城市")

    # 检查天数
    day_keywords = ["1天", "2天", "3天", "4天", "5天", "一日", "两天", "三天", "四天", "五天", "几天"]
    has_days = any(day in user_request for day in day_keywords)
    if not has_days:
        missing_items.append("旅行天数")

    # 检查预算
    budget_keywords = ["预算", "元", "块", "经济", "舒适", "豪华", "穷游", "多少钱", "费用"]
    has_budget = any(word in user_request for word in budget_keywords)
    if not has_budget:
        missing_items.append("预算或消费档位")

    # 检查偏好
    preference_keywords = ["美食", "购物", "动漫", "历史", "亲子", "自然", "博物馆", "拍照", "休闲", "人文"]
    has_preference = any(word in user_request for word in preference_keywords)
    if not has_preference:
        missing_items.append("旅行偏好")

    if missing_items:
        return "用户需求信息不完整，缺少：" + "、".join(missing_items)

    return "用户需求信息基本完整，可以开始规划。"
```
创建Agent时，修改tool列表。
```python
tools=[
    requirement_check_tool,
    weather_tool,
    attraction_tool,
    budget_tool
    ]
```
同时，修改system_prompt：
```python
system_prompt="""
你是一个专业的旅游规划Agent。

你可以使用以下工具：
1. requirement_check_tool：检查用户旅游需求是否完整
2. weather_tool：查询天气
3. attraction_tool：根据城市和偏好推荐景点
4. budget_tool：估算旅游预算

你的工作流程必须遵守：

第一步：
如果用户的旅游需求比较模糊，例如只说“想去某地玩”“帮我安排一下”“推荐个路线”，你必须先调用 requirement_check_tool 检查信息是否完整。

第二步：
如果 requirement_check_tool 返回“用户需求信息不完整”，你不要直接生成完整行程。
你应该先向用户追问缺失信息。
追问时最多问 3 个最关键问题，不要一次问太多。

第三步：
如果用户需求信息基本完整，你再根据需要调用其他工具：
- 涉及天气、带伞、户外活动，调用 weather_tool
- 涉及景点、路线、行程安排，调用 attraction_tool
- 涉及预算、费用、消费水平，调用 budget_tool

第四步：
综合工具结果，输出清晰、可执行的旅游方案。

输出要求：
- 信息不足时：只追问，不要强行规划
- 信息足够时：按天规划
- 每天包含上午、下午、晚上
- 说明为什么这样安排
- 如果工具数据不足，要明确说明，不要编造
"""
```
如此，Agent会按照system_prompt中设定的工作流程执行，会在需求模糊时，调用requirement_check_tool检索信息完整性，如不完整，则追问。

## 旅游Agent V4 版本
在V4版本中，我们为Agent增加了多轮对话的记忆系统，能够让他记住对话。
#### 第一步：导入需要的包。
```python
from langchain_core.runnables.history import RunnableWithMessageHistory
from langchain_core.chat_history import InMemoryChatMessageHistory
```
#### 第二步，创建memory存储。
```python
store = {}
```
#### 第三步，定义session memory。
```python
def get_session_history(session_id: str):
    if session_id not in store:
        store[session_id] = InMemoryChatMessageHistory()
    return store[session_id]
```
#### 第四步，把agent再包一层。
```python
agent_with_memory = RunnableWithMessageHistory(
    agent,
    get_session_history,
    input_messages_key="messages"
)
```
#### 第五步，修改agent的调用方式。
```python
result = agent_with_memory.invoke(
    {
        "messages": [
            {"role": "user", "content": user_input}
        ]
    },
    config={"configurable": {"session_id": "user-1"}}
)
```
至此，修改完成，每轮对话的信息，Agent都会存入memory中，并在后续对话中作为context的一部分返回。

## 旅游Agent V5 版本
在V5版本中，我们要为Agent增加用户画像功能。用来存储用户画像，方便在后续的提问中进行格式化定制回答。
#### 第一步，建立user_profile.json，用于存储用户画像。
```python
{
  "travel_preferences": [],
  "budget_preference": "",
  "pace_preference": "",
  "food_preference": [],
  "avoid_preferences": []
}
```

#### 第二步，增加update_user_profile_tool，当用户表达长期偏好时更新用户画像
```python 
import json
import os

PROFILE_FILE = "user_profile.json"


def load_user_profile() -> dict:
    if not os.path.exists(PROFILE_FILE):
        return {
            "travel_preferences": [],
            "budget_preference": "",
            "pace_preference": "",
            "food_preference": [],
            "avoid_preferences": []
        }

    with open(PROFILE_FILE, "r", encoding="utf-8") as f:
        return json.load(f)


def save_user_profile(profile: dict) -> None:
    with open(PROFILE_FILE, "w", encoding="utf-8") as f:
        json.dump(profile, f, ensure_ascii=False, indent=2)


@tool
def read_user_profile_tool() -> str:
    """
    读取用户长期旅游偏好画像。
    当你需要根据用户过往偏好进行个性化旅游推荐时使用。
    """
    profile = load_user_profile()
    return json.dumps(profile, ensure_ascii=False)
```
#### 第三步，在tool列表中，增加update_user_profile_tool
```python
tools=[
    requirement_check_tool,
    read_user_profile_tool,
    update_user_profile_tool,
    weather_tool,
    attraction_tool,
    budget_tool
    ]
```
#### 第四步，修改sysem_prompt
```python
你是一个专业的旅游规划Agent。

你可以使用以下工具：
1. requirement_check_tool：检查用户旅游需求是否完整
2. read_user_profile_tool：读取用户长期旅游偏好画像
3. update_user_profile_tool：更新用户长期旅游偏好画像
4. weather_tool：查询天气
5. attraction_tool：根据城市和偏好推荐景点
6. budget_tool：估算旅游预算

你的工作流程必须遵守：

第一步：识别用户是否表达了长期偏好。
如果用户说出类似：
- 我喜欢动漫
- 我喜欢美食
- 我不喜欢赶行程
- 我预算有限
- 我喜欢慢节奏旅行
你应该调用 update_user_profile_tool 更新用户画像。

第二步：在做旅游规划前，优先调用 read_user_profile_tool 读取用户画像。
如果用户本轮没有说明偏好，但画像里有偏好，你要结合画像做个性化推荐。

第三步：如果用户的旅游需求比较模糊，例如只说“想去某地玩”“帮我安排一下”“推荐个路线”，你必须先调用 requirement_check_tool 检查信息是否完整。

第四步：
如果 requirement_check_tool 返回“用户需求信息不完整”，你不要直接生成完整行程。
你应该先向用户追问缺失信息。
追问时最多问 3 个最关键问题，不要一次问太多。

第五步：
如果用户需求信息基本完整，你再根据需要调用其他工具：
- 涉及天气、带伞、户外活动，调用 weather_tool
- 涉及景点、路线、行程安排，调用 attraction_tool
- 涉及预算、费用、消费水平，调用 budget_tool

第六步：
综合用户画像、本轮需求和工具结果，输出清晰、可执行的旅游方案。

输出要求：
- 信息不足时：只追问，不要强行规划
- 信息足够时：按天规划
- 每天包含上午、下午、晚上
- 明确说明哪些推荐是基于用户画像
- 如果工具数据不足，要明确说明，不要编造
```
至此，V5版本的Agent就有了用户画像记忆与调用功能。
## 旅游Agent V6 版本
V6 是本项目 LangChain 阶段的收尾版本。后续会将当前能力迁移到 LangGraph，用图结构显式管理流程节点和状态。
更新点主要有两点。

    1、增加 Agent 执行链路可视化，即调试信息显示.这里看到的是 Agent 的执行链路和工具调用日志，不是模型内部真实思考过程.
    2、优化Agent的输出格式

#### 第一步，引入所需的库
```python
from langchain_core.tracers.stdout import ConsoleCallbackHandler
```

#### 第二步，开启调试
```python
 result = agent_with_memory.invoke(
        {
        "messages": [
            {"role": "user", "content": user_input}
            ]
        },
        config={"configurable": {"session_id": "user-1"},
                 "callbacks": [ConsoleCallbackHandler()]
        }
    )
```
#### 运行后，就能看到模型的思考执行过程，大致过程如下：
```markdown
[llm/start] ChatOpenAI 接收用户输入
tool_calls:
- weather_tool(city="东京")
- attraction_tool(city="东京", preference="动漫")
- attraction_tool(city="东京", preference="美食")
- budget_tool(city="东京", days=3, level="经济")

[tool/end] weather_tool -> {"weather": "晴天", "temp": 22, "suggest": "适合户外"}
[tool/end] attraction_tool -> 东京适合动漫偏好的景点有：秋叶原...
[tool/end] budget_tool -> 东京3天经济游的预估预算约为 1800 元
```
通过这段日志，可以确认 Agent 并不是直接生成答案，而是先识别任务，再调用多个工具，最后融合工具结果生成最终行程。
#### 第四步，在system_prompt中增加输出的结构化约束语句。
```python
    system_prompt="""
你是一个专业的旅游规划Agent。

你可以使用以下工具：
1. requirement_check_tool：检查用户旅游需求是否完整
2. read_user_profile_tool：读取用户长期旅游偏好画像
3. update_user_profile_tool：更新用户长期旅游偏好画像
4. weather_tool：查询天气
5. attraction_tool：根据城市和偏好推荐景点
6. budget_tool：估算旅游预算

你的工作流程必须遵守：

第一步：识别用户是否表达了长期偏好。
如果用户说出类似：
- 我喜欢动漫
- 我喜欢美食
- 我不喜欢赶行程
- 我预算有限
- 我喜欢慢节奏旅行
你应该调用 update_user_profile_tool 更新用户画像。

第二步：在做旅游规划前，优先调用 read_user_profile_tool 读取用户画像。
如果用户本轮没有说明偏好，但画像里有偏好，你要结合画像做个性化推荐。

第三步：如果用户的旅游需求比较模糊，例如只说“想去某地玩”“帮我安排一下”“推荐个路线”，你必须先调用 requirement_check_tool 检查信息是否完整。

第四步：
如果 requirement_check_tool 返回“用户需求信息不完整”，你不要直接生成完整行程。
你应该先向用户追问缺失信息。
追问时最多问 3 个最关键问题，不要一次问太多。

第五步：
如果用户需求信息基本完整，你再根据需要调用其他工具：
- 涉及天气、带伞、户外活动，调用 weather_tool
- 涉及景点、路线、行程安排，调用 attraction_tool
- 涉及预算、费用、消费水平，调用 budget_tool

第六步：
综合用户画像、本轮需求和工具结果，输出清晰、可执行的旅游方案。

输出要求：
- 信息不足时：只追问，不要强行规划
- 信息足够时：按天规划
- 每天包含上午、下午、晚上
- 明确说明哪些推荐是基于用户画像
- 如果工具数据不足，要明确说明，不要编造

输出必须严格遵守结构：

{
  "destination": "",
  "days": "",
  "itinerary": [
    {
      "day": 1,
      "morning": "",
      "afternoon": "",
      "evening": ""
    }
  ],
  "budget_estimate": "",
  "notes": ""
}
"""
```
至此，一个简单的旅游Agent项目就算是完成了。只要把tool改成真实的API调用，就能进行真实的旅游路线规划。
## 当前版本限制

1. 天气、景点、预算工具目前仍是 mock 数据，并未接入真实 API。
2. 用户画像目前存储在本地 JSON 文件中，不支持多用户隔离和数据库持久化。
3. 需求检查工具基于关键词规则，泛化能力有限。
4. 输出格式主要依赖 system_prompt 约束，尚未使用 Pydantic 或 JSON Schema 做强校验。
5. 当前流程仍主要由 LLM 自主决策工具调用，流程控制不够显式；下一阶段将使用 LangGraph 进行节点化改造。

# 总结
  到 V6 为止，本项目已经完成了一个旅游规划 Agent 的 LangChain 阶段闭环：

- 使用 DeepSeek 作为大模型；
- 使用 LangChain `create_agent` 构建 Tool Calling Agent；
- 通过 `@tool` 定义天气、景点、预算、需求检查、用户画像等工具；
- 使用 `RunnableWithMessageHistory` 实现会话级短期记忆；
- 使用本地 JSON 文件实现用户画像长期记忆；
- 通过 ConsoleCallbackHandler 观察 Agent 执行链路；
- 通过 system_prompt 约束输出结构。

下一阶段将迁移到 LangGraph，把当前由 prompt 驱动的流程，改造成显式的图结构流程，例如：

Planner Node → Requirement Check Node → Memory Node → Tool Node → Final Response Node