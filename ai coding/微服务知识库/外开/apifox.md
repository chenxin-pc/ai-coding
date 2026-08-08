{
  "mcpServers": {
    "apifox-new-mcp": {
      "url": "https://apifox.com/api/v1/mcp",
      "headers": {
        "Authorization": "Bearer afxp_320d4djHnJvCwGhloGzrZin2HNIaYLOwhcTe",
        "X-Apifox-Api-Version": "2025-09-01"
      }
    }
  }
}

1、通过apifox的MCP服务，获取所有apifox的项目ID，确定项目ID
2、通过项目ID获取所有目录ID，确认接口该放到哪个目录
3、根据apifox接口文档内容，或者修改接口，根据method是否存在，确定是修改还是创建接口

文档说明，作为请求头参数，请求头参数名为：method  参数值为文档apicode的值
请求方式不拿文档中的请求方式，默认为POST
请求参数格式不拿apifox请求参数说明，默认为body-json
请求参数示例直接拿接口文档内容的入参示例作为apifox的参数示例
返回参数示例直接拿接口文档内容的返回示例作为apifox的参数示例


1、适配器实现（技能：openapi-implement-adapter） 
2、适配器接口文档生成（技能：openapi-interface-doc-extractor） 
3、接口上传apifox（技能：openapi-sync-interface-apifox）  
4、使用参数示例进MCP接口调用


