# WebWatcher

##### 介绍

- Motivation：解决现有Agent主要以文本为中心，忽略视觉信息的局限性。
- 方法描述
  - 有效利用多种外部工具：
    - Web Image Search，由Google SerApi提供支持，用于检索相关图像，其标题和网页URL，以便更好地理解输入图像。
    - Web Text Search，用于开放域信息检索，检索查询的标题和URL。
    - Visit：由Jina提供，允许导航到特定URL以获取网页摘要，根据LLM操作中指定的“目标”进行定制。
    - Code Interpreter：支持符号计算和数值推理。
    - OCR：一种
- 实验细节