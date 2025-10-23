---
title: MasterGo MCP 尝试
date: 2025-5-15 20:08:46
tags:
  - MasterGo
  - MCP
categories: MasterGo
index_img: /img/mg-logo.png
---

## 使用

1. 复制以下代码到 mcp.json 文件中 (权限问题，7 月 10 号之前凭借 token，任意用户都可以，之后只能编辑席位和研发席位的 token 可以访问 mcp，原文地址：https://mastergo.com/static/advance-notice-prototype.html)

   ```
   {
     "mcpServers": {
       "mastergo-magic-mcp": {
         "command": "npx",
         "args": [
           "-y",
           "@mastergo/magic-mcp",
           "--token=MG_MCP_TOKEN",
           "--url=https://mastergo.com"
         ],
         "env": {
           "NPM_CONFIG_REGISTRY": "https://registry.npmjs.org/"
         }
       }
     }
   }
   ```

   ![image](/img2/masterGo_MCP_1.jpg)

2. 获取 token，替换 mcp.json 文件中的<MG_MCP_TOKEN>
   ![image](/img2/masterGo_MCP_2.jpg)
   ![image](/img2/masterGo_MCP_3.jpg)

3. 保存后，回到设置页面，如果显示绿色，并且出现了 Tools 有 mcp\_\_getDSL 就是成功了。
4. 来到 MasterGo 中，选中一个图层，此时图层是 frame 类型的话，链接上会增加 layerId 参数，然后复制链接
   ![image](/img2/masterGo_MCP_4.jpg)
5. 复制 URL 到 cursor 中，输入一下 prompt 进行生成代码
   ![image](/img2/masterGo_MCP_5.jpg)
6. 以下三个生成效果：图 1 是官方 demo、图 2 是收听领奖的整个页面、图 3 是单个按钮。
   ![image](/img2/masterGo_MCP_6.jpg)
   ![image](/img2/masterGo_MCP_7.jpg)
   ![image](/img2/masterGo_MCP_8.jpg)

### 探索 MCP 实现原理

1. 插件 github 地址：https://github.com/mastergo-design/mastergo-magic-mcp
2. 插件提供 mcp 工具：

   - mcp\_\_getDsl，描述如下，概括为：提供 fileId 和 layerId 字段，返回原始的 DSL。包含所有信息。

   ```

       "Use this tool to retrieve the DSL (Domain Specific Language) data from MasterGo design files and the rules you must follow when generating code.
       This tool is useful when you need to analyze the structure of a design, understand component hierarchy, or extract design properties.
       You must provide a fileId and layerId to identify the specific design element.
       This tool returns the raw DSL data in JSON format that you can then parse and analyze.
       This tool also returns the rules you must follow when generating code.
       The DSL data can also be used to transform and generate code for different frameworks."

   ```

- mcp\_\_getComponentLink，描述如下，概括为：当 mcp_getDsl 返回的数据包含 componentDocumentLinks 数组时。获取到每个组件的文档(masterGo 样式组件的文档),根据文档信息生成对应的组件。

  ```

  When the data returned by mcp\*\*getDsl contains a non-empty componentDocumentLinks array, this tool is used to sequentially retrieve URLs from the componentDocumentLinks array and then obtain component documentation data. The returned document data is used for you to generate frontend code based on components.
  ```

- mcp\_\_getMeta，描述如下，概括为：用来给用户构建一个完整的网站，或者高级网站。提供 fileId 和 layerId 字段，返回网站和页面的规则和结果(在 masterGo 中的图层，图层里有规则和信息)。cursor 会根据元信息，生成 task.md，然后去递归的去寻找 masterGo 设计页面，然后逐一生成页面，并且会加上 mete 信息里的跳转，或者其他事件。使用链接：https://mastergo.com/file/155675508499265?fileOpenFrom=home&page_id=8740%3A3698&devMode=true
- mcp\_\_getComponentGenerator，描述如下，概括为，提供 rootPath、fileId、layerId，返回组件开发工作流的文件(component-workflow.md)，让 cursor 根据 component-workflow.md 规则开发组件。

  ```

  Users need to actively call this tool to get the component development workflow. When Generator is mentioned, please actively call this tool.
  This tool provides a structured workflow for component development following best practices.
  You must provide an absolute rootPath of workspace to save workflow files.

  ```

- mcp\_\_getDsl 返回结果和 masterGo 插件中的 DSL 比较，概括为，几乎完全不一样的字段，MCP 返回的 DSL 数据中的样式数据是稍加处理过的。

  ````

      {
        "dsl": {
          "styles": {
            "paint_1:0363": { "value": ["#FFFFFF"], "token": "Whites/White" },
            "effect_1:0365": { "value": ["box-shadow: 0px 0px 0px 20px rgba(0, 0, 0, 0.02);"], "token": "box shadow" },
            "paint_1:0404": {
              "value": [
                {
                  "url": "https://image-resource.mastergo.com/57000091966935/57000091966937/6d01dc22287b99530477ebcb8831f8bb.png",
                  "filters": ""
                }
              ]
            },
            "paint_1:0373": { "value": ["#313237"], "token": "Blacks/High emphasis" },
            "font_1:0372": {
              "value": {
                "family": "Amiko-Bold",
                "size": 25,
                "decoration": "none",
                "case": "none",
                "lineHeight": "33.79999923706055",
                "letterSpacing": "auto"
              },
              "token": "Headings/Heading"
            },
            "paint_1:0379": { "value": ["#718797"], "token": "Blacks/Mid emphasis" },
            "font_1:0378": {
              "value": {
                "family": "Inter-Regular",
                "size": 16,
                "decoration": "none",
                "case": "none",
                "lineHeight": "26.399999618530273",
                "letterSpacing": "auto"
              },
              "token": "Body/body"
            },
            "paint_1:0401": {
              "value": [
                {
                  "url": "https://image-resource.mastergo.com/57000091966935/57000091966937/b253f033eb4e3f396f8ee23280a1b034.png",
                  "filters": ""
                }
              ]
            },
            "font_1:0399": {
              "value": {
                "family": "Inter-SemiBold",
                "size": 16,
                "decoration": "none",
                "case": "none",
                "lineHeight": "26.399999618530273",
                "letterSpacing": "auto"
              },
              "token": "Body/BodyBold"
            },
            "font_1:0395": {
              "value": {
                "family": "Inter-Regular",
                "size": 12,
                "decoration": "none",
                "case": "none",
                "lineHeight": "16.799999237060547",
                "letterSpacing": "auto"
              },
              "token": "Other/caption"
            },
            "paint_1:0383": { "value": [] },
            "paint_1:0387": { "value": ["#5374FF"], "token": "AppColors/ppl" }
          },
          "nodes": [
            {
              "type": "FRAME",
              "id": "158:0222",
              "name": "simple layer 1",
              "layoutStyle": { "width": 540, "height": 680, "relativeX": 0, "relativeY": 0 },
              "fill": "paint_1:0363",
              "children": [
                {
                  "type": "LAYER",
                  "id": "158:0245",
                  "name": "Rectangle 8",
                  "layoutStyle": { "width": 500, "height": 380.2801208496094, "relativeX": 20, "relativeY": 20 },
                  "flexShrink": 1,
                  "borderRadius": "10px",
                  "fill": "paint_1:0404"
                },
                {
                  "type": "FRAME",
                  "id": "158:0223",
                  "name": "Frame 42",
                  "layoutStyle": {
                    "width": 500,
                    "height": 259.7198791503906,
                    "relativeX": 20,
                    "relativeY": 400.2801208496094
                  },
                  "flexGrow": 1,
                  "flexShrink": 1,
                  "children": [
                    {
                      "type": "FRAME",
                      "id": "158:0224",
                      "name": "Frame 30",
                      "layoutStyle": { "width": 460, "height": 121, "relativeX": 20, "relativeY": 40 },
                      "flexShrink": 1,
                      "children": [
                        {
                          "type": "TEXT",
                          "id": "158:0225",
                          "name": "Use the bluetooth SMS andwidth, then you can",
                          "layoutStyle": { "width": 460, "height": 68 },
                          "flexShrink": 1,
                          "text": [{ "text": "Use the bluetooth SMS andwidth, then you can", "font": "font_1:0372" }],
                          "textColor": [{ "start": 0, "end": 44, "color": "paint_1:0373" }],
                          "textAlign": "left",
                          "textMode": "auto-height"
                        },
                        {
                          "type": "TEXT",
                          "id": "158:0228",
                          "name": "Compressing the monitor won't do anything, we need to synthesize the cross-platform HTTP feed.",
                          "layoutStyle": { "width": 460, "height": 53, "relativeY": 68 },
                          "flexShrink": 1,
                          "text": [
                            {
                              "text": "Compressing the monitor won't do anything, we need to synthesize the cross-platform HTTP feed.",
                              "font": "font_1:0378"
                            }
                          ],
                          "textColor": [{ "start": 0, "end": 94, "color": "paint_1:0379" }],
                          "textAlign": "left",
                          "textMode": "auto-height"
                        }
                      ],
                      "flexContainerInfo": { "flexDirection": "column", "mainSizing": "auto", "crossSizing": "fixed" }
                    },
                    {
                      "type": "FRAME",
                      "id": "158:0231",
                      "name": "Frame 31",
                      "layoutStyle": { "width": 460, "height": 44, "relativeX": 20, "relativeY": 195.71987915039062 },
                      "flexShrink": 1,
                      "children": [
                        {
                          "type": "GROUP",
                          "id": "158:0235",
                          "name": "Group 1",
                          "layoutStyle": { "width": 194, "height": 44 },
                          "children": [
                            {
                              "type": "FRAME",
                              "id": "158:0236",
                              "name": "Frame 23",
                              "layoutStyle": { "width": 194, "height": 44 },
                              "children": [
                                {
                                  "type": "LAYER",
                                  "id": "158:0237",
                                  "name": "Rectangle 19",
                                  "layoutStyle": { "width": 40, "height": 40, "relativeY": 2 },
                                  "borderRadius": "155px",
                                  "fill": "paint_1:0401"
                                },
                                {
                                  "type": "FRAME",
                                  "id": "158:0238",
                                  "name": "Frame 24",
                                  "layoutStyle": { "width": 144, "height": 44, "relativeX": 50 },
                                  "children": [
                                    {
                                      "type": "TEXT",
                                      "id": "158:0239",
                                      "name": "Darlene Robertson",
                                      "layoutStyle": { "width": 144, "height": 27 },
                                      "text": [{ "text": "Darlene Robertson", "font": "font_1:0399" }],
                                      "textColor": [{ "start": 0, "end": 17, "color": "paint_1:0373" }],
                                      "textAlign": "left",
                                      "textMode": "single-line"
                                    },
                                    {
                                      "type": "TEXT",
                                      "id": "158:0242",
                                      "name": "March 12, 2021",
                                      "layoutStyle": { "width": 87, "height": 17, "relativeY": 27 },
                                      "text": [{ "text": "March 12, 2021", "font": "font_1:0395" }],
                                      "textColor": [{ "start": 0, "end": 14, "color": "paint_1:0379" }],
                                      "textAlign": "left",
                                      "textMode": "single-line"
                                    }
                                  ],
                                  "flexContainerInfo": {
                                    "flexDirection": "column",
                                    "mainSizing": "auto",
                                    "crossSizing": "auto"
                                  }
                                }
                              ],
                              "flexContainerInfo": {
                                "flexDirection": "row",
                                "alignItems": "center",
                                "mainSizing": "auto",
                                "crossSizing": "auto",
                                "gap": "10px"
                              }
                            }
                          ]
                        },
                        {
                          "type": "FRAME",
                          "id": "158:0232",
                          "name": "bi:arrow-right-short",
                          "layoutStyle": { "width": 40, "height": 40, "relativeX": 420 },
                          "fill": "paint_1:0383",
                          "children": [
                            {
                              "type": "GROUP",
                              "id": "158:0233",
                              "name": "Group",
                              "layoutStyle": {
                                "width": 20.002233505249023,
                                "height": 17.50320053100586,
                                "relativeX": 10,
                                "relativeY": 11.248419761657715
                              },
                              "children": [
                                {
                                  "type": "PATH",
                                  "id": "158:0234",
                                  "name": "Vector",
                                  "layoutStyle": { "width": 20.002233505249023, "height": 17.50320053100586 },
                                  "path": [
                                    {
                                      "fill": "paint_1:0387",
                                      "data": "M6.96159e-30 8.75158C-1.11022e-15 8.42006 0.131696 8.10212 0.366117 7.8677C0.600537 7.63328 0.918479 7.50158 1.25 7.50158C1.25 7.50158 15.7325 7.50158 15.7325 7.50158C15.7325 7.50158 10.365 2.13658 10.365 2.13658C10.1303 1.90186 9.99842 1.58352 9.99842 1.25158C9.99842 0.91964 10.1303 0.601296 10.365 0.366579C10.5997 0.131863 10.9181 0 11.25 0C11.5819 0 11.9003 0.131863 12.135 0.366579C12.135 0.366579 19.635 7.86658 19.635 7.86658C19.7514 7.98269 19.8438 8.12063 19.9068 8.2725C19.9698 8.42436 20.0022 8.58716 20.0022 8.75158C20.0022 8.916 19.9698 9.0788 19.9068 9.23066C19.8438 9.38253 19.7514 9.52047 19.635 9.63658C19.635 9.63658 12.135 17.1366 12.135 17.1366C11.9003 17.3713 11.5819 17.5032 11.25 17.5032C10.9181 17.5032 10.5997 17.3713 10.365 17.1366C10.1303 16.9019 9.99842 16.5835 9.99842 16.2516C9.99842 15.9196 10.1303 15.6013 10.365 15.3666C10.365 15.3666 15.7325 10.0016 15.7325 10.0016C15.7325 10.0016 1.25 10.0016 1.25 10.0016C0.918479 10.0016 0.600537 9.86988 0.366117 9.63546C0.131696 9.40104 -1.11022e-15 9.0831 6.96159e-30 8.75158C6.96159e-30 8.75158 6.96159e-30 8.75158 6.96159e-30 8.75158Z"
                                    }
                                  ]
                                }
                              ]
                            }
                          ]
                        }
                      ],
                      "flexContainerInfo": {
                        "flexDirection": "row",
                        "justifyContent": "space-between",
                        "mainSizing": "fixed",
                        "crossSizing": "auto",
                        "gap": "135px"
                      },
                      "borderRadius": "10px"
                    }
                  ],
                  "flexContainerInfo": {
                    "flexDirection": "column",
                    "justifyContent": "space-between",
                    "mainSizing": "fixed",
                    "crossSizing": "fixed",
                    "padding": "40px 20px 20px"
                  }
                }
              ],
              "effect": "effect_1:0365",
              "flexContainerInfo": {
                "flexDirection": "column",
                "justifyContent": "center",
                "alignItems": "center",
                "mainSizing": "fixed",
                "crossSizing": "fixed",
                "padding": "20px"
              },
              "borderRadius": "10px"
            }
          ],
          "components": []
        },
        "componentDocumentLinks": [],
        "rules": [
          "token filed must be generated as a variable (colors, shadows, fonts, etc.) and the token field must be displayed in the comment",
          "\n            componentDocumentLinks is a list of frontend component documentation links used in the DSL layer, designed to help you understand how to use the components.\n            When it exists and is not empty, you need to use mcp__getComponentLink in a for loop to get the URL content of all components in the list, understand how to use the components, and generate code using the components.\n            For example: \n              ```js  \n                const componentDocumentLinks = [\n                  'https://example.com/ant/button.mdx',\n                  'https://example.com/ant/button.mdx'\n                ]\n                for (const url of componentDocumentLinks) {\n                  const componentLink = await mcp__getComponentLink(url);\n                  console.log(componentLink);\n                }\n              ```\n          "
        ]
      }

  ````

  ![image](/img2/masterGo_MCP_9.jpg)

### 总结

目前 masterGo MCP 生成代码的原材料是设计文件的 DSL 简单处理过一层之后的 DSL Object，AI 并不能根据原始图层来进行精确的布局，也不能生产动态内容的布局(包括文字个数动态变化，以及其他元素动态个数)，一方面 MCP 提供的 DSL Object 是基于图层信息的，非常杂乱，一方面目前的 AI 处理大量数据是有缺陷的。

### 实操参考：

https://mastergo.com/file/155675508499265?fileOpenFrom=home&page_id=1%3A0
