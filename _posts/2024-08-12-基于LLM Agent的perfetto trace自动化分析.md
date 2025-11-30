---
layout:     post
title:      基于LLM Agent的perfetto trace自动化分析
subtitle:   
date:       2024-08-12
author:     WMY
header-img: img/03.jpg
catalog: true
tags:
    - 工作
---


## 基于LLM Agent的perfetto trace自动化分析


### 1. LLM Agent背景介绍

AIGC正逐步应用到各个领域，改变大家的工作和生活方式。但深入到具体专业领域复杂问题，大模型的分析深度还有待提高，针对该现状，构建智能体（Agent）则是AI工程应用当下的“终极形态”

Agent工作流不是让LLM直接生成最终输出，而是多次提示LLM，使其有机会逐步构建更高质量的输出

LLM Agent主要包括感知、分析、决策和执行四大能力。这些能力相互协同，构成了其基本的工作原理。

在基于LLM的智能体中，LLM充当着智能体的大脑的角色，同时还有3个关键部分：规划、记忆、工具使用。

1.Planning：LLM分解复杂任务，制定并执行多步骤计划来实现目标
2.Memory：它存储和检索过去的信息和经验。这使得它能够在处理用户查询时，利用之前学到的知识和经验，提供更准确的答案 
3.Tools：它能够灵活运用搜索引擎、数据库、API等工具，获取和整理相关信息，来支持任务的完成
4.Action：根据拆解的任务需求正确地调用工具以达到任务的目的
如下图所示：
![](https://wwmmyy2023.github.io/img/agent_tool.png)


### 2. AI Agent如何分析perfetto trace

按照如上所述原理，构建智能体（Agent）需要写一些专业领域的辅助工具，使Agent能自动调用这些tool，来支持相关任务的分析处理，当前支持Tool Calling方式的agent开源框架有LangChain等，可自行搜索。
AI Agent根据用户场景分类等灵活调用这些工具，获取专业的指标数据信息，从而实现对于perfetto trace的分析深度达到专家级水准。
AI Agent 调用工具运行时序图如下：

	用户问题 → AI Agent → 工具选择 → 工具执行 → 结果处理 → 最终回答
	    ↓         ↓         ↓         ↓         ↓         ↓
	  自然语言   LangChain  工具路由   数据查询   结果整合   AI总结
	   输入      等开源框架   机制       执行      处理      输出


![](https://wwmmyy2023.github.io/img/llm.jpg)



### 3. 建立分析perfetto trace工具集

对于Android perfetto trace的分析，Google 提供了TraceProcessor开源软件工具，它能够解析和分析追踪trace数据。同时该库提供了一套查询引擎库以及CPU Usage、APP start等场景标准metric指标的默认实现，可作为自动分析辅助工具集的一部分。

此外基于传统分析经验及分析故障树，将其转化为TraceProcessor自动化解析脚本分析能力，进而形成特有的Agent tool分析工具集。

TraceProcessor可以将perfetto trace文件转成数据库表，也可以使用trace_processor_shell工具在命令行进行操作，对于自动化分析来说，适合采用Traceprocessor api动态操作perfetto trace文件。

#### 3.1 建立TraceProcessor 自助查询工具 示例

按照LangChain Tool Calling框架方式，我们可以将TraceProcessor操作查询perfetto trace文件的功能封装成一个工具，供后续Agent动态调用。示例如下：

	@tool
	def perfetto_analysis_tool(trace_path: str, sql_query_statement: str) -> str:
		"""
		执行自定义SQL查询分析Android perfetto文件。
		该工具通过TraceProcessor库解析指定的trace文件，并执行用户提供的SQL查询，
		返回格式化后的查询结果。支持性能数据分析等Systrace数据查询。
		Args:
			trace_path (str): 本地trace文件路径，支持perfetto(.proto)和systrace(.html)格式
			sql_query_statement (str): 符合SQLite语法的查询语句，支持Trace Processor虚拟表结构
		Returns:
			str: 格式化后的查询结果，包含以下形式：
				- 成功时: 直接返回查询的结果，结果是将查询结果DataFrame格式化后的可读字符串
				- 错误时: 错误描述及排查建议
		Raises:
			FileNotFoundError: 当trace文件不存在时
			ValueError: 当SQL语句为空或包含危险操作时
			RuntimeError: TraceProcessor执行过程中发生错误时
		"""
		if not sql_query_statement.strip().upper().startswith("SELECT"):
			raise ValueError("仅支持SELECT查询语句")
		try:
			# 使用上下文管理器管理资源
			with getTraceProcessor(trace_path) as trace:
				# 执行查询
				result = trace.query(sql_query_statement)
				df = result.as_pandas_dataframe()
				return f"查询结果为:" + df.to_string()
		except Exception as e:
			error_msg = f"执行失败: {str(e)}"
			if "no such table" in str(e).lower():
				error_msg += "\n可能原因：表名错误，请参考Perfetto文档确认表结构"
			return error_msg
		
	def getTraceProcessor(abtrace_file):
		abtrace_file = abtrace_file.replace(" ", "")
		if not os.path.exists(abtrace_file):
			return None
		bin_path = r'./tool/trace_processor_shell'
		if sys.platform.lower() == 'win32':
			bin_path = r'./tool/trace_processor_shell.exe'
		config = TraceProcessorConfig(bin_path)
		trace = TraceProcessor(abtrace_file, config=config)
		return trace



#### 3.2 建立日常专家分析问题辅助工具集

将分析perfetto trace问题分析中的一些可以固化的分析套路，拆解成步骤，并尝试使用 Traceprocessor的sql语句等实现，进一步的将这些语句封装成agent工具，从而agent在分析问题时能动态调用。

##### (1) 建立获取前台应用Top卡顿点信息的tool

比如针对前台应用Top卡顿的识别，有如下主要的Traceprocessor的sql语句：

1.追踪 UI主线程中 doframe耗时TOP的slice信息， 可用如下语句查询

	SELECT pro.pid as pid,slice.id,ABS_TIME_STR(ts + 8*3600*1000*1000*1000) as startTime,slice.ts, slice.dur/1e6 as dur_ms,(slice.ts + slice.dur) as endTime,
	       slice.name, thread.name AS thread_name,thread.upid, is_main_thread,slice.parent_id, pro.name as processname, thread.tid as threadtid, thread.utid as utid 
	FROM slice 
		JOIN thread_track 
		ON slice.track_id = thread_track.id 
		JOIN thread USING(utid) 
		JOIN process pro on thread.upid=pro.upid
	WHERE is_main_thread=1 and slice.name LIKE "%Choreographer#doFrame %"ORDER BY slice.dur DESC limit 15

Traceprocessor的sql语句是否可用，可输入到打开的trace文件的perfetto网站做查询验证，比如输入如上语句，可看到类似如下执行结果：

![](https://wwmmyy2023.github.io/img/perfetto/query_top.jpg)

2.UI主线程中最大叶结点Slice追踪 可用如下sql语句查询

	select  pro.pid as pid,sliceTemp.id,ABS_TIME_STR(sliceTemp.ts + 8*3600*1000*1000*1000) as startTime,sliceTemp.ts, sliceTemp.dur/1e6 as dur_ms,(sliceTemp.ts + sliceTemp.dur) as endTime,
	        sliceTemp.name, thread.name AS thread_name,thread.upid, is_main_thread,sliceTemp.parent_id, pro.name as processname, thread.tid as threadtid, thread.utid as utid
	    from (SELECT * FROM slice    WHERE slice_id NOT IN (SELECT parent_id FROM slice WHERE parent_id IS NOT NULL)) sliceTemp
	    JOIN thread_track ON sliceTemp.track_id = thread_track.id
	    JOIN thread USING(utid)
	    JOIN process pro on thread.upid=pro.upid
	    where
	    is_main_thread=1 and (dur > 30*1e6)
	    order by sliceTemp.dur desc limit 10


3.主线程长时间Running等状态的获取sql语句查询

	select tstate.dur/1e6 as dur_ms, state, thread.name as tname, pro.name as pname, ts, thread.utid,ABS_TIME_STR(ts + 8*3600*1000*1000*1000)
	from thread_state as tstate
		join thread on thread.utid=tstate.utid
		join process as pro USING (upid)
	where  is_main_thread=1 and tstate.dur> 40*1e6 and (state='Running' or state='R' or state='R+' or state='D' or state='D*')
		and pro.name ='{pkgname}' 
	order by dur desc
	limit 10


4.TOP卡顿点识别：slice层级的追踪方法的sql语句查询

	SELECT * FROM slice WHERE parent_id = {slice_id}

5.前台焦点应用及时间区间获取方法的sql语句查询

	select ts as starttime, ts+dur as endtime, slice.name as fgapp from slice join track on slice.track_id=track.id where track.name="Focused app"


知道如上关键 Traceprocessor sql查询语句后，再通过一些细化的逻辑处理，我们就可以将其封装成一个获取前台应用TOP 卡顿点的tool，该辅助工具示例如下：

	@tool
	def perfetto_analysis_jank(titlePkgs: List[str], trace_path: str) -> str:
	    """
	        Android perfetto trace卡顿点获取工具, 可以用于获取top卡顿点的信息
	        Args:
	            titlePkgs: 表示跟用户反馈问题可能相关的app packageName名称列表
	            trace_path: 本地trace文件路径，分析问题该问题是需要把本地trace文件路径传递过来
	
	        Returns:
	            str: 返回的结果字段中：
	                - pid (int): 发生卡顿的进程ID
	                - id (int): [关键字段] Systrace切片唯一标识符（slice_id），用于：
	                    1. 追踪事件调用链路（结合parent_id）
	                    2. 关联其他系统事件（如binder_transaction）
	                    3. 通过slice表查询完整上下文
	                - startTime (str): 格式化时间戳（UTC+8，示例：2023-01-01 10:00:00.123）
	                - ts (int): 卡顿起始时间（微秒级原始时间戳）
	                - jank_dur_ms (float): 卡顿时长（毫秒），阈值>25ms
	                - endTime (int): 卡顿结束时间（ts + dur）
	                - name (str): 事件类型标识（如：Choreographer#doFrame）
	                - thread_name (str): 关联线程名称（如：RenderThread）
	                - upid (int): 进程唯一标识符（关联process表）
	                - is_main_thread (bool): 是否发生在主线程
	                - parent_id (int): 父事件slice_id，用于构建调用链
	                - processname (str): 进程名称（如：com.example.app）
	                - threadtid (int): 系统级线程ID
	                - utid (int): 唯一线程标识符（关联thread表）
	
	        Raises:
	            ValueError: 输入参数验证失败时抛出
	    """
	    trace = getTraceProcessor(trace_path)
	    query, frame_loss_infos, similarityLevel = getTopJankSlices(trace, titlePkgs)
	
	    return f"top卡顿点信息为:" + frame_loss_infos.to_string()




##### (2) 建立获取某线程唤醒链信息的tool

获取调用唤醒链的追踪相关的sql语句查询：

step1:获取slice信息详情 

	select thread.utid, slice.ts as startTime, (slice.ts + slice.dur) as endTime, slice.name as sliname, process.name as pname, thread.name as tname, *
	from slice
	 join thread_track on thread_track.id = slice.track_id
	 left join thread USING (utid)
	 left join process USING (upid)
	where slice.slice_id={slice_id}


step2: 查找这个slice期间它的最大的thread state 时段

	select * from thread_state where utid={slice_utid} and (ts between {timeStart} and {timeEnd} or (ts <{timeStart} and (ts + dur) > {timeStart}) ) order by dur desc limit 1

step3:若sleep最大，则需要找到紧接着的sleep后面是谁唤醒的

	select thread.name as tname,process.name as name,utid,thread_state.* 
	from thread_state  
		left join thread USING (utid)  
		left join process USING (upid)
	where utid={utid} and ts between {thread_maxstate_end} and {timeEnd} order by ts asc limit 10

step4:唤醒源不为空,说明这里找到了唤醒源threadid,继续往下跟

	select thread.name as tname,process.name as name,utid,thread_state.*
	from thread_state
		left join thread USING (utid)
		left join process USING (upid)
	where utid={waker_utid}
		and ((ts between {timeStart} and {thread_state_row['ts']}) or (ts<{timeStart} and (ts+dur) > {timeStart}))
	order by ts desc  limit 200

知道如上关键 Traceprocessor sql查询语句后，再通过一些细化的逻辑处理，我们就可以将其封装成一个获取前台应用TOP 卡顿点的tool，该辅助工具示例如下：

	@tool
	def perfetto_get_Jank_KeyStack(slice_id: int, trace_path: str) -> str:
	    """
	      通过输入perfetto trace切片唯一标识符（slice_id），可以追踪该卡顿点唤醒链路以及唤醒链路上的耗时原因
	      典型应用场景:
	        - 追踪主线程卡顿点对应的唤醒链及唤醒链路耗时原因
	    Args:
	        slice_id: Systrace切片唯一标识符（slice_id）
	        trace_path: 本地trace文件路径（需绝对路径）
	
	    Returns:
	        str:卡顿点关键调用链以及唤醒链路上的耗时原因小结
	
	    Raises:
	        ValueError: 输入参数验证失败时抛出
	
	    """
	    trace = getTraceProcessor(trace_path)
	    reasonStack, finalReason, maxDur, samekeyreasons = getJankPointKeyStack(trace, slice_id, "")
	    if finalReason != "":
	        finalReason = "卡顿瓶颈点" + finalReason
	
	    return f"卡顿点关键调用链:{reasonStack},唤醒链路上的耗时原因小结:{finalReason}"

按照类似如上示例，可以将通常的专家分析经验转成相应的辅助工具，可以不限于如上的 Traceprocessor sql语句方式，也可以文档规则库等，总之输出结果能当llm大模型读懂起到分析辅助作用。


### 4. AI Agent工具调用架构图

实现的专业领域辅助开发工具后，最终运行的AI Agent工具调用架构图如下：

	┌──────────────────────────────────────────────────────────────┐
	│                    PerfettoAnalysisAgent 架构图               │
	├──────────────────────────────────────────────────────────────┤
	│  用户输入层                                                   │
	│  ┌──────────────────────────────────────────────────────────┐│
	│  │ 用户问题如: "手机imei号为:xxx，使用xxx应用发热卡顿原因"       ││
	│  └──────────────────────────────────────────────────────────┘│
	├──────────────────────────────────────────────────────────────┤
	│  AI Agent 核心层                                             │
	│  ┌──────────────────────────────────────────────────────────┐│
	│  │  LangChain Agent Executor                                ││
	│  │  ┌──────────────────────────────────────────────────────┐││
	│  │  │  ChatOpenAI (GPT-4/DeepSeek/Qwen)                    │││
	│  │  │  ├─ 模型配置: base_url, api_key, model                │││
	│  │  │  ├─ 超时设置: timeout=120s                            │││
	│  │  │  └─ 重试机制: max_retries=3                           │││
	│  │  └──────────────────────────────────────────────────────┘││
	│  │  ┌──────────────────────────────────────────────────────┐││
	│  │  │  Prompt Template                                     │││
	│  │  │  ├─ System: 工具调用指导                              │││
	│  │  │  ├─ Human: 用户问题输入                               │││
	│  │  │  └─ Agent: 工具执行结果                               │││
	│  │  └──────────────────────────────────────────────────────┘││
	│  └──────────────────────────────────────────────────────────┘│
	├──────────────────────────────────────────────────────────────┤
	│  工具集成层                                                  │
	│  ┌──────────────────────────────────────────────────────────┐│
	│  │  @tool("perfetto_analysis_tool")                         ││
	│  │  ├─ 功能: 自定义SQL查询分析Android perfetto文件             ││
	│  └──────────────────────────────────────────────────────────┘│
	│  ┌──────────────────────────────────────────────────────────┐│
	│  │  @tool("perfetto_analysis_jank")                         ││
	│  │  ├─ 功能: 用于获取top卡顿点的信息                           ││
	│  └──────────────────────────────────────────────────────────┘│
	│  ┌──────────────────────────────────────────────────────────┐│
	│  │  @tool("perfetto_get_Jank_KeyStack")                     ││
	│  │  ├─ 功能: 追踪卡顿点唤醒链路及耗时原因                       ││
	│  └──────────────────────────────────────────────────────────┘│
	│  ┌──────────────────────────────────────────────────────────┐│
	│  │  @tool("xxxxxx")                                         ││
	│  │  ├─ 功能: ................                               ││
	│  └──────────────────────────────────────────────────────────┘│
	├──────────────────────────────────────────────────────────────┤
	│  流式处理层                                                  │
	│  ┌──────────────────────────────────────────────────────────┐│
	│  │  StreamLogCallbackHandler                                ││
	│  │  ├─ on_tool_start(): 工具开始执行                         ││
	│  │  ├─ on_tool_end(): 工具执行完成                           ││
	│  │  ├─ on_tool_error(): 工具执行错误                         ││
	│  │  └─ on_text(): 文本输出                                   ││
	│  └──────────────────────────────────────────────────────────┘│
	└──────────────────────────────────────────────────────────────┘


### 5. AI运行结果示例：

抓取一段音乐播放时有卡顿的perfetto，采用建立好的 Agent Tool Calling模型进行自动化分析测试，示例如下： 

	tools = [perfetto_analysis_tool, perfetto_analysis_jank, perfetto_get_Jank_KeyStack]

	from langchain.agents import AgentExecutor, create_tool_calling_agent
	agent = create_openai_tools_agent( #create_tool_calling_agent
		llm=llm,
		tools=tools,
		prompt=prompt  # ← 使用本地模板
	)

	agent_executor = AgentExecutor(agent=agent, tools=tools, verbose=True, handle_parsing_errors=True)

	user_issue_desc = "播放QQ音乐时卡顿"
	trace_abs_path = './upload/2024-08-08-20-22-01.perfetto-trace'

	res = agent_executor.invoke(
		{
			"input": f'''用户反馈{user_issue_desc}，相关trace文件：'{trace_abs_path}'，请分析卡顿原因。
			PS:你可以采用提供的自定义工具获取用用户反馈场景相关的top卡顿点、卡顿点区间的CPU loading信息、卡顿点区间对应进程的线程运行状态信息,卡顿点唤醒链路及唤醒链路耗时原因等，
			你需要获取top3卡顿点的关键卡顿信息，并进行综合分析总结出卡顿原因，如果这些信息不足以精准定位卡顿原因，你还可以按照自己的分析逻辑采用自定义工具输入你想要查询的sql语句查询与卡顿相关的信息，最后给出分析结论，分析结论要简洁逐条列出，且不超过200字'''
		}
	)
	print("res:", res['output'])

如下是实际执行过程的关键日志打印：

首先Agent调用了自定义的 perfetto_analysis_jank tool，获取到top卡顿点
![](https://wwmmyy2023.github.io/img/perfetto/query_1.jpg)

紧接着Agent调用了自定义的 perfetto_get_Jank_KeyStack tool，追踪某个top卡顿点的唤醒链信息
![](https://wwmmyy2023.github.io/img/perfetto/query_2.jpg)

另外Agent分析发现当前的分析工具获取信息可能还不够，它又调用了自定义的 perfetto_analysis_tool 工具，自己写sql语句查询关键卡顿点的trace信息：
![](https://wwmmyy2023.github.io/img/perfetto/query_3.jpg)

经过多轮调用，最终得到的结果如下，跟人工分析结果基本一致：

![](https://wwmmyy2023.github.io/img/perfetto/query_4.jpg)

当前采用的是阿里 qwen model，不同的模型分析深度有差异。model可以随时替换，做对比。 

如果想深入分析，可以持续升级增加自定义tool。对于一些分析思路相关固定的场景，可以采用LangGraph框架，分析结果更精准稳定。